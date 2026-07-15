# ADR-002: Política de Retry com Backoff Exponencial e Dead Letter Queue

**Status:** Decided
**Data:** 2025-11-01
**Decisor:** Diego (Senior Engineer, Platform), Larissa (Tech Lead)
**Participantes:** Bruno (Senior Engineer, Orders), Marcos (PM)

---

## Contexto

Quando o worker tenta entregar um webhook e o cliente está offline, retorna erro HTTP ou timeout, é necessário decidir:
1. Quantas vezes tentar de novo?
2. Qual o intervalo entre tentativas?
3. Quando desistir e considerar falha permanente?

A aplicação precisa balancear:
- **Resiliência:** Tolerar indisponibilidades planejadas ou transitórias do cliente
- **Operabilidade:** Não deixar eventos "pendurados" indefinidamente
- **Auditoria:** Registrar falhas para debug e eventual reprocessamento manual

Requisitos de negócio: clientes podem ter janelas de manutenção planejada (até 2 horas confirmado), indisponibilidades transitórias, ou desabilitação permanente do endpoint.

## Decisão

Implementar **backoff exponencial com 5 tentativas e Dead Letter Queue**:

**Política de Retry:**
```
Tentativa 1: falha imediata → aguardar 1 minuto
Tentativa 2: após 1m → aguardar 5 minutos (total: ~6m do evento)
Tentativa 3: após 5m → aguardar 30 minutos (total: ~36m do evento)
Tentativa 4: após 30m → aguardar 2 horas (total: ~2h36m do evento)
Tentativa 5: após 2h → aguardar 12 horas (total: ~14h36m do evento)

Se Tentativa 5 falha → DEAD LETTER QUEUE (falha permanente)
```

**Dead Letter Queue:**
- Tabela separada: `webhook_dead_letter`
- Estrutura: `id`, `webhook_outbox_id`, `webhook_id`, `customer_id`, `payload`, `reason` (erro da última tentativa), `failure_timestamp`
- Registro auditável: evidência para investigação, reprocessamento manual via endpoint admin

**Implementação:**

Arquivo: `src/modules/webhooks/webhook.worker.ts`
```typescript
export const retryPolicy = {
  maxAttempts: 5,
  backoffIntervals: [
    60 * 1000,           // 1 minuto
    5 * 60 * 1000,       // 5 minutos
    30 * 60 * 1000,      // 30 minutos
    2 * 60 * 60 * 1000,  // 2 horas
    12 * 60 * 60 * 1000, // 12 horas
  ],
};

async function processOutboxEvent(event: WebhookOutboxEvent): Promise<void> {
  const nextRetryTime = calculateNextRetry(event.attempt_count);

  if (event.attempt_count >= retryPolicy.maxAttempts) {
    // Move to DLQ
    await moveToDeadLetter(event, 'Max attempts reached');
    logger.warn(
      { eventId: event.event_id, attempts: event.attempt_count },
      'Webhook event moved to dead letter queue'
    );
    return;
  }

  try {
    await sendWebhook(event.webhook_url, event.payload, event.headers);
    await markAsDelivered(event.id);
    logger.info({ eventId: event.event_id }, 'Webhook delivered successfully');
  } catch (error) {
    const nextRetryAt = new Date(Date.now() + nextRetryTime);
    await recordFailure(event.id, error.message, nextRetryAt, event.attempt_count + 1);
    logger.warn(
      { eventId: event.event_id, attempt: event.attempt_count + 1, nextRetry: nextRetryAt },
      'Webhook delivery failed, will retry'
    );
  }
}
```

**Configuração do Outbox:**
```sql
ALTER TABLE webhook_outbox ADD COLUMN (
  next_retry_at TIMESTAMP NULL,
  attempt_count INT DEFAULT 0,
  last_error TEXT
);
```

---

## Alternativas Consideradas

### 1. Retry indefinido com backoff exponencial (sem máximo)
**Vantagem:** Nunca desiste, garante entrega eventual se cliente voltar
**Desvantagem:**
- Evento fica "pendurado" pra sempre se cliente sumiu (morreu)
- Tabela cresce indefinidamente
- Impossível distinguir "cliente offline temporário" de "cliente desativado"
- Sem fechamento claro da falha

**Trade-off:** Clareza operacional: é melhor ter limite claro e movimentar pra DLQ do que deixar incerteza.

### 2. Retry agressivo (máximo 3 tentativas, backoff curto 1m/5m/10m)
**Vantagem:** Fecha mais rápido, menos overhead
**Desvantagem:**
- Cliente com manutenção planejada de 2 horas na manhã, ganha 3 tentativas em 30 minutos e morre
- Já temos cliente interno que teve indisponibilidade de 2 horas

**Trade-off:** Requisito de negócio: "qualquer coisa abaixo de 10 segundos já é tempo real"; 3 tentativas é pouco pra cenários reais.

### 3. Retry com queue persistida em file system
**Vantagem:** Não ocupa tabela do banco
**Desvantagem:**
- Menos confiável, arquivo pode ser perdido
- Difícil de escalar pra múltiplos workers
- Sem backup automático

**Trade-off:** Database é mais seguro e auditável.

## Consequências

**Positivas:**
- ✅ Resiliência: cliente com indisponibilidade < 14 horas é retentado
- ✅ Operabilidade clara: limite definido, DLQ serve como inbox de falhas
- ✅ Sem infinito: eventos não ficam pendurados pra sempre
- ✅ Auditoria: DLQ é evidence da falha, timestamps e razão
- ✅ Reprocessamento manual: admin pode fazer replay via endpoint
- ✅ Escalável: backoff exponencial evita bombardeio, suporta crescimento

**Negativas:**
- ❌ Latência total alta: cliente que cai às 9h só vai receber última tentativa às 23h (14h depois)
- ❌ Overhead de tentativas: mesmo cliente offline recebe até 5 chamadas antes de desistir
- ❌ Reprocessamento manual: requer ação humana, não automático
- ❌ Responsabilidade no cliente: se endpoint virou inválido, admin terá que intervir

**Trade-off principal:** Balanço entre resiliência (tolera indisponibilidades) e operabilidade (falhas fechadas em tempo razoável).

---

## Ligações

- Relacionado: ADR-001 (Outbox no MySQL)
- Relacionado: ADR-005 (Worker em Polling)
- Implementação: docs/FDD.md - Seção "Estratégias de Resiliência"
