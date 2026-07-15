# ADR-005: Worker em Processo Separado com Polling a 2 Segundos

**Status:** Decided
**Data:** 2025-11-01
**Decisor:** Diego (Senior Engineer, Platform), Larissa (Tech Lead)
**Participantes:** Bruno (Senior Engineer, Orders)

---

## Contexto

O webhook outbox (ADR-001) precisa de um **consumer** que leia eventos pendentes e dispare chamadas HTTP. Decisão arquitetural: onde rodar esse consumer?

Opções:
1. **Mesmo processo da API**: Simples, mas compartilha resources, riscos operacionais
2. **Processo separado**: Isolamento, melhor observabilidade, escalabilidade futura
3. **Job/cron scheduler** (tipo Bull, RabbitMQ): Overkill pra current scale

Requisitos:
- Latência máxima < 10s (clientes B2B esperando)
- Confiabilidade: não pode "sumir" sem aviso
- Escalabilidade: começar com single-worker, planejar multi-worker futuro

## Decisão

Implementar **worker em processo separado do Node.js com polling a 2 segundos**:

**Arquitetura:**
```
src/server.ts       → API Express (portas requisições HTTP, muda status)
  ↓ (insere na outbox)
[webhook_outbox]    → Tabela MySQL
  ↓ (polling cada 2s)
src/worker.ts       → Worker (lê outbox, dispara webhooks, retry)
```

**Estrutura de código:**

Arquivo: `src/worker.ts` (entry point separado)
```typescript
import { PrismaClient } from '@prisma/client';
import { logger } from './shared/logger';
import { WebhookWorkerService } from './modules/webhooks/webhook.worker';

const prisma = new PrismaClient();
const worker = new WebhookWorkerService(prisma);

async function main() {
  logger.info('Webhook worker started');

  // Polling loop
  setInterval(async () => {
    try {
      await worker.processOutboxBatch();
    } catch (error) {
      logger.error({ error }, 'Error processing webhook batch');
      // Continue polling even on error
    }
  }, 2000); // 2 seconds

  // Graceful shutdown
  process.on('SIGTERM', async () => {
    logger.info('SIGTERM received, shutting down worker');
    clearInterval(pollingInterval);
    await prisma.$disconnect();
    process.exit(0);
  });
}

main().catch((error) => {
  logger.error({ error }, 'Worker failed to start');
  process.exit(1);
});
```

Arquivo: `src/modules/webhooks/webhook.worker.ts`
```typescript
export class WebhookWorkerService {
  constructor(private prisma: PrismaClient) {}

  /**
   * Processa batch de eventos pendentes da outbox
   */
  async processOutboxBatch(): Promise<void> {
    const BATCH_SIZE = 10; // Processa até 10 eventos por batch

    const events = await this.prisma.webhook_outbox.findMany({
      where: { status: 'pending' },
      orderBy: { created_at: 'asc' },
      take: BATCH_SIZE,
      include: { webhook: true },
    });

    if (events.length === 0) {
      // Nada pra fazer, espera próximo polling
      return;
    }

    logger.debug({ count: events.length }, 'Processing webhook batch');

    for (const event of events) {
      try {
        await this.processEvent(event);
      } catch (error) {
        logger.error(
          { eventId: event.event_id, error },
          'Error processing event, will retry on next batch'
        );
      }
    }
  }

  /**
   * Processa um evento individual
   */
  private async processEvent(event: WebhookOutboxEvent): Promise<void> {
    const webhook = event.webhook;

    // 1. Marcar como "processing" pra evitar duplicação
    await this.prisma.webhook_outbox.update({
      where: { id: event.id },
      data: { status: 'processing' },
    });

    try {
      // 2. Tentar enviar
      await this.sendWebhookRequest(webhook, event);

      // 3. Sucesso: marcar como entregue
      await this.prisma.webhook_outbox.update({
        where: { id: event.id },
        data: { status: 'delivered' },
      });

      logger.info({ eventId: event.event_id, webhookId: webhook.id }, 'Webhook delivered');
    } catch (error) {
      // 4. Falha: aplicar backoff
      await this.handleFailure(event, error);
    }
  }

  /**
   * Envia requisição HTTP para o webhook
   */
  private async sendWebhookRequest(
    webhook: Webhook,
    event: WebhookOutboxEvent,
  ): Promise<void> {
    const payload = JSON.parse(event.payload);
    const signature = this.calculateSignature(webhook.current_secret, JSON.stringify(payload));

    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 10000); // 10s timeout

    try {
      const response = await fetch(webhook.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Event-Id': event.event_id,
          'X-Timestamp': new Date().toISOString(),
          'X-Signature': signature,
          'X-Webhook-Id': webhook.id,
        },
        body: JSON.stringify(payload),
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
    } finally {
      clearTimeout(timeoutId);
    }
  }

  /**
   * Trata falha e aplica política de retry
   */
  private async handleFailure(
    event: WebhookOutboxEvent,
    error: Error,
  ): Promise<void> {
    const attempt = event.attempt_count + 1;
    const maxAttempts = 5;

    if (attempt >= maxAttempts) {
      // Move to DLQ
      await this.moveToDeadLetter(event, error.message);
      return;
    }

    // Calculate next retry time
    const backoffIntervals = [60, 5 * 60, 30 * 60, 2 * 60 * 60, 12 * 60 * 60]; // seconds
    const nextRetrySeconds = backoffIntervals[attempt - 1];
    const nextRetryAt = new Date(Date.now() + nextRetrySeconds * 1000);

    await this.prisma.webhook_outbox.update({
      where: { id: event.id },
      data: {
        status: 'pending',
        attempt_count: attempt,
        last_error: error.message,
        next_retry_at: nextRetryAt,
      },
    });

    logger.warn(
      { eventId: event.event_id, attempt, nextRetryAt },
      'Webhook delivery failed, scheduled retry'
    );
  }

  /**
   * Move evento falhado para DLQ
   */
  private async moveToDeadLetter(
    event: WebhookOutboxEvent,
    reason: string,
  ): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      await tx.webhook_dead_letter.create({
        data: {
          id: uuidV4(),
          webhook_outbox_id: event.id,
          webhook_id: event.webhook_id,
          customer_id: event.customer_id,
          payload: event.payload,
          reason,
          failure_timestamp: new Date(),
        },
      });

      await tx.webhook_outbox.delete({
        where: { id: event.id },
      });
    });

    logger.error(
      { eventId: event.event_id, reason },
      'Webhook moved to dead letter queue'
    );
  }
}
```

**Scripts no package.json:**
```json
{
  "scripts": {
    "dev": "node --loader ts-node/esm src/server.ts",
    "dev:worker": "node --loader ts-node/esm src/worker.ts",
    "start": "node dist/server.js",
    "start:worker": "node dist/worker.js"
  }
}
```

**Deployment:**
```bash
# Processo da API
$ npm start

# Processo do worker (outro terminal/container)
$ npm run start:worker
```

**Entry point no server (src/server.ts) - NENHUMA mudança:**
O servidor continua normal, apenas a função `publishWebhookEvent()` insere na outbox. Worker lê de forma desacoplada.

---

## Alternativas Consideradas

### 1. Worker dentro do mesmo processo da API
**Vantagem:** Simples, single entry point, sem orquestração de processos
**Desvantagem:**
- Se API reinicia, worker também para (eventos acumulam na outbox)
- Competição por CPU/memory entre requisições HTTP e webhooks
- Difícil de observar separadamente (logs mezclados)
- Não escala bem

**Trade-off:** Risco operacional: se API cai, webhooks param.

### 2. Redis Streams com consumer
**Vantagem:** Altamente escalável, reativo, suporta múltiplos consumers
**Desvantagem:**
- Exige redis infrastructure (não existe hoje)
- Overhead operacional (backup, monitoria, scaling de cluster)
- Custo AWS adicional
- Overkill para <10s latência

**Trade-off:** Overengineering. Polling simples é suficiente.

### 3. AWS SQS + Lambda
**Vantagem:** Serverless, escalável automaticamente, gerenciado
**Desvantagem:**
- Vendor lock-in AWS
- Não rola bem com dados sensíveis (SQS é externa)
- Custo pode ser maior
- Latência: SQS pode levar segundos, + Lambda coldstart

**Trade-off:** Perde benefício de outbox + banco: informação sai para fora.

### 4. MySQL triggers + UDF (User-Defined Function)
**Vantagem:** Reativo, sem polling
**Desvantagem:**
- MySQL não tem listeners nativos
- UDFs são frágeis, difíceis de debugar
- Operação de banco pesada

**Trade-off:** Polling é mais simples e confiável.

## Consequências

**Positivas:**
- ✅ Isolamento: worker não compete com API por resources
- ✅ Resiliência: worker para != API para, eventos acumulam na outbox
- ✅ Observabilidade: logs e métricas separadas do worker
- ✅ Escalabilidade: multi-worker futuro = apenas spawn mais processos
- ✅ Simplicidade: polling é trivial de implementar
- ✅ Sem infra extra: usa MySQL + Node.js existentes
- ✅ Latência aceitável: 2s no pior caso, < 10s target

**Negativas:**
- ❌ Orquestração: precisa gerenciar 2 processos em produção (não é trivial)
- ❌ I/O overhead: polling a cada 2s faz query mesmo quando vazio
- ❌ Single-worker é gargalo: não é multi-worker automaticamente
- ❌ Graceful shutdown: precisa lidar com SIGTERM corretamente
- ❌ Visibilidade: cliente não consegue ver status do worker (precisa de endpoint)

**Trade-off principal:** Simplicidade + resiliência em troca de overhead de orquestração.

---

## Notas Operacionais

**Monitoramento:**
- Métrica: latência média de entrega (deve estar perto de 2s)
- Métrica: quantidade de eventos em `webhook_outbox` com status='pending' (alarme se > 1000)
- Métrica: quantidade de eventos em `webhook_dead_letter` (alarme se > 10)
- Métrica: taxa de falhas de webhook (% de eventos que vão para DLQ)

**Alarmes:**
- Worker não está rodando (heartbeat): check se há eventos novos sendo entregues
- Backlog crescente: eventos acumulando em pending (worker pode estar lento)
- DLQ acumulando: muitas falhas permanentes, investigar

**Escalabilidade futura:**
Se atingir múltiplos workers:
- Cada worker pega batch diferente (select com LIMIT e ORDER BY, sem overlapping)
- Usar lock pessimista: `SELECT ... FOR UPDATE` pra evitar duplicação
- Particionar por customer_id ou webhook_id pra distribuir

---

## Ligações

- Relacionado: ADR-001 (Outbox no MySQL)
- Relacionado: ADR-002 (Retry com Backoff)
- Implementação: docs/FDD.md - Seção "Fluxos Detalhados"
- Referência de código: `src/worker.ts`, `src/modules/webhooks/webhook.worker.ts`
