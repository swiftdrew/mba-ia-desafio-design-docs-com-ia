# ADR-001: Padrão Outbox no MySQL para Webhooks

**Status:** Decided
**Data:** 2025-11-01
**Decisor:** Larissa (Tech Lead), Diego (Senior Engineer)
**Participantes:** Bruno (Senior Engineer, Orders), Sofia (Security Engineer)

---

## Contexto

O sistema de notificação de webhooks precisa garantir consistência transacional: quando o status de um pedido muda, o evento de notificação deve ser registrado atomicamente com a mudança de status na banco de dados. Se a transação falhar, tanto o status quanto o evento devem ser revertidos juntos.

A aplicação atual não possui mecanismo de notificação externa, filas ou eventos. O pedido é de 3 clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) que necessitam notificação em tempo real (< 10 segundos) quando pedidos mudam de status.

A implementação precisa ser:
1. **Consistente**: Sem inconsistência entre estado do pedido e eventos disparados
2. **Desacoplada**: Não pode bloquear transações de mudança de status
3. **Resiliente**: Deve lidar com clientes offline
4. **Escalável**: Suportar crescimento sem overengineering (time pequeno, infra existente)

## Decisão

Implementar o **padrão Outbox no MySQL existente**:

1. Criar tabela `webhook_outbox` na mesma instância MySQL usada para orders
2. Quando o status muda no `OrderService.changeStatus()`, inserir linha na `webhook_outbox` **dentro da mesma transação** que atualiza `orders` e `order_status_history`
3. Um **worker separado** em polling (a cada 2 segundos) lê eventos pendentes da outbox e dispara chamadas HTTP
4. Após sucesso, marca como `entregue` na tabela; após falhas persistentes, move para `webhook_dead_letter` table

**Estrutura da tabela webhook_outbox:**
```sql
CREATE TABLE webhook_outbox (
  id CHAR(36) PRIMARY KEY,           -- UUID
  customer_id CHAR(36) NOT NULL,
  webhook_id CHAR(36) NOT NULL,
  event_type VARCHAR(64) NOT NULL,
  payload JSON NOT NULL,             -- Snapshot do evento
  event_id CHAR(36) NOT NULL UNIQUE, -- Para idempotência do cliente
  status ENUM('pending', 'processing', 'delivered', 'failed') DEFAULT 'pending',
  attempt_count INT DEFAULT 0,
  last_error TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  INDEX idx_status_created (status, created_at),
  INDEX idx_customer (customer_id),
  FOREIGN KEY (customer_id) REFERENCES customers(id),
  FOREIGN KEY (webhook_id) REFERENCES webhooks(id)
);
```

**Integração no código:**
- No arquivo `src/modules/orders/order.service.ts`, o método `changeStatus()` já usa transação (`this.prisma.$transaction`)
- Adicionar chamada a função pura: `publishWebhookEvent(tx, order, fromStatus, toStatus)`
- Esta função insere na outbox dentro do mesmo tx, garantindo atomicidade

**Arquivo de integração:** `src/modules/webhooks/webhook.publisher.ts`
```typescript
export async function publishWebhookEvent(
  tx: PrismaTransaction,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  // 1. Buscar todos os webhooks do customer que querem este evento
  const webhooks = await tx.webhook.findMany({
    where: {
      customer_id: order.customer_id,
      enabled: true,
      events: { has: toStatus },
    },
  });

  // 2. Para cada webhook, criar uma linha de outbox
  for (const webhook of webhooks) {
    await tx.webhook_outbox.create({
      data: {
        id: uuidV4(),
        customer_id: order.customer_id,
        webhook_id: webhook.id,
        event_type: 'order.status_changed',
        event_id: uuidV4(),
        payload: JSON.stringify({
          event_id: uuidV4(),
          event_type: 'order.status_changed',
          timestamp: new Date().toISOString(),
          order_id: order.id,
          order_number: order.number,
          from_status: fromStatus,
          to_status: toStatus,
          customer_id: order.customer_id,
          total_cents: order.total_cents,
        }),
        status: 'pending',
      },
    });
  }
}
```

## Alternativas Consideradas

### 1. Implementar com Redis Streams
**Vantagem:** Mais reativo (push), melhor escalabilidade futura
**Desvantagem:** Exige nova infra (Redis Cluster), overhead operacional para time pequeno, custo extra AWS, complexidade de devops

**Trade-off:** Redis é overengineering para current scale. MySQL outbox é suficiente e mantém tudo na mesma tecnologia que já conhecemos.

### 2. Disparar webhooks sincronamente no changeStatus
**Vantagem:** Simples, sem camada extra, resposta imediata
**Desvantagem:**
- Cliente lento bloqueia mudança de status pra outros pedidos
- Se cliente offline, não há retry, perde notificação ou gera timeout
- Não resolve problema de "cliente fora do ar, o que fazer?"

**Trade-off:** Coloca responsabilidade crítica em terceiros, risco operacional alto.

### 3. Usar MySQL TRIGGER para notificar worker externo
**Vantagem:** Mais reativo que polling
**Desvantagem:** MySQL não tem listener nativo como Postgres NOTIFY/LISTEN; trigger só executa SQL, não notifica processo externo; precisaria "improvisar" via arquivo ou HTTP call, fica frágil

**Trade-off:** Solução "gambiarra", menos confiável que polling explícito.

## Consequências

**Positivas:**
- ✅ Consistência garantida: evento sempre entra junto com mudança de status, ou ambos falham
- ✅ Desacoplamento: webhook não bloqueia orders.changeStatus
- ✅ Retry automático com backoff: cliente offline é retentado automaticamente
- ✅ Escalável: single-worker suficiente para <10s latência; multi-worker depois se necessário
- ✅ Sem infra extra: usa MySQL + Node.js já existentes
- ✅ Fácil debug: tabela outbox e DLQ servem como auditoria e evidence de tentativas

**Negativas:**
- ❌ Overhead de I/O: polling a cada 2 segundos faz query na outbox mesmo quando vazia (pouco significativo)
- ❌ Latência mínima de 2 segundos: pior caso é esperar 2 segundos entre status mudar e notificação ser enviada (aceitável, < 10s)
- ❌ Single-worker é gargalo: se escalar futuramente pra múltiplos workers, precisará de lock pessimista ou particionamento por order_id
- ❌ Tabela outbox cresce: precisa de política de arquivamento/limpeza (após 30 dias, registros entregues podem ser deletados)

**Trade-off principal:** Ganho de confiabilidade e simplicidade em troca de latência mínima aceitável e futuro upscaling planejado.

---

## Ligações

- Relacionado: ADR-002 (Retry com Backoff e DLQ)
- Relacionado: ADR-005 (Worker em Polling)
