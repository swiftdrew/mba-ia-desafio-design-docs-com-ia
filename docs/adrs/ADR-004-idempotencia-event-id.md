# ADR-004: Garantia At-Least-Once com X-Event-Id para Idempotência

**Status:** Decided
**Data:** 2025-11-01
**Decisor:** Diego (Senior Engineer, Platform), Larissa (Tech Lead)
**Participantes:** Sofia (Security Engineer), Bruno (Senior Engineer, Orders)

---

## Contexto

O padrão de retry com backoff exponencial (ADR-002) garante entrega, mas com custo: **o cliente pode receber o mesmo evento múltiplas vezes**. Exemplos:
- Webhook deliveries: sucesso, mas response perdida em rede → retry e segunda cópia entregue
- Outbox event duplicado após crash do worker se não houve atualização de status
- Transient failures podem resultar em 2+ entregas do mesmo evento

Precisamos balancear:
- **Entregabilidade:** garantir que nenhum evento é perdido (at-least-once)
- **Deduplicação:** permitir cliente reconhecer duplicatas e ignorar a segunda
- **Complexidade operacional:** não forçar exactly-once (custaria coordenação entre sistemas)

## Decisão

Implementar **At-Least-Once com X-Event-Id para deduplicação no lado do cliente**:

**Mecanismo:**
1. Quando evento é criado na outbox, gerar UUID único: `event_id`
2. Incluir `event_id` no payload JSON
3. Enviar `event_id` também em header: `X-Event-Id`
4. Cliente armazena `event_id` de eventos processados
5. Se receber segundo request com mesmo `event_id`, ignora (dedup em seu banco)

**Tabela webhook_outbox:**
```sql
ALTER TABLE webhook_outbox ADD COLUMN event_id CHAR(36) UNIQUE NOT NULL;
```

**Payload JSON:**
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2025-11-01T14:30:00Z",
  "order_id": "abcd1234",
  "order_number": "ORD-000123",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "customer-xyz",
  "total_cents": 99990
}
```

**Headers:**
```
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
X-Timestamp: 2025-11-01T14:30:00Z
X-Signature: ...
X-Webhook-Id: ...
```

**Implementação - Outbox:**

Arquivo: `src/modules/webhooks/webhook.publisher.ts`
```typescript
import { v4 as uuidV4 } from 'uuid';

export async function publishWebhookEvent(
  tx: PrismaTransaction,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  const webhooks = await tx.webhook.findMany({
    where: {
      customer_id: order.customer_id,
      enabled: true,
      events: { has: toStatus },
    },
  });

  for (const webhook of webhooks) {
    const eventId = uuidV4();  // Único por evento

    const payload = {
      event_id: eventId,
      event_type: 'order.status_changed',
      timestamp: new Date().toISOString(),
      order_id: order.id,
      order_number: order.number,
      from_status: fromStatus,
      to_status: toStatus,
      customer_id: order.customer_id,
      total_cents: order.total_cents,
    };

    await tx.webhook_outbox.create({
      data: {
        id: uuidV4(),
        customer_id: order.customer_id,
        webhook_id: webhook.id,
        event_type: 'order.status_changed',
        event_id: eventId,  // Unique constraint
        payload: JSON.stringify(payload),
        status: 'pending',
      },
    });
  }
}
```

**Implementação - Worker:**

Arquivo: `src/modules/webhooks/webhook.worker.ts`
```typescript
async function sendWebhook(
  webhook: Webhook,
  event: WebhookOutboxEvent,
): Promise<void> {
  const payload = JSON.parse(event.payload);

  const response = await fetch(webhook.url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Event-Id': event.event_id,
      'X-Timestamp': new Date().toISOString(),
      'X-Signature': calculateSignature(webhook.current_secret, JSON.stringify(payload)),
      'X-Webhook-Id': webhook.id,
    },
    body: JSON.stringify(payload),
    timeout: 10000,
  });

  if (!response.ok) {
    throw new Error(`Webhook failed: ${response.status}`);
  }
}
```

**Documentação para cliente:**

No portal de desenvolvedor (Markdown):
```markdown
## Idempotência e Deduplicação

Nós garantimos **at-least-once delivery**: cada evento será entregue
pelo menos uma vez, mas pode ser entregue múltiplas vezes em cenários
de falha transitória ou retry.

Para lidar com isso, **você DEVE implementar deduplicação pelo lado dele**:

1. Armazene o `event_id` de cada webhook recebido no seu banco
2. Ao receber um webhook, procure o `event_id` em seu histórico
3. Se encontrar (duplicata), responda com `200 OK` mas não reprocesse
4. Se não encontrar, processe o evento e armazene o `event_id`

Exemplo em Node.js:
\`\`\`typescript
app.post('/webhooks/orders', async (req) => {
  const eventId = req.headers['x-event-id'];

  // Dedup
  const existing = await db.webhookHistory.findOne({ event_id: eventId });
  if (existing) {
    return res.json({ ok: true, deduped: true });
  }

  // Process
  const payload = req.body;
  await processOrderUpdate(payload);

  // Record
  await db.webhookHistory.create({ event_id: eventId, payload, received_at: new Date() });

  return res.json({ ok: true });
});
\`\`\`

O header `X-Timestamp` pode ser usado para detectar **replay attacks** (cliente recusa eventos muito antigos).
```

---

## Alternativas Consideradas

### 1. Garantir Exactly-Once (entrega exatamente uma vez)
**Vantagem:** Cliente não precisa implementar dedup, mais simples
**Desvantagem:**
- Exigiria transação distribuída entre nós e cliente
- Cliente precisaria confirmar recebimento sincronamente
- Nós precisaríamos manter estado de qual cliente confirmou quais eventos
- Complexidade exponencial com múltiplos clientes
- Latência aumentada (wait for ack)

**Trade-off:** Industry standard é at-least-once. Stripe, GitHub, Twilio todos fazem assim.

### 2. Idempotência via Timestamp + Offset
**Vantagem:** Não precisa armazenar event_id
**Desvantagem:**
- Timestamp pode colidir (múltiplos eventos no mesmo ms)
- Offset não funciona bem se há múltiplos workers
- Mais frágil

**Trade-off:** UUID é mais robusto.

### 3. Sem garantia de idempotência (cliente arca com o problema)
**Vantagem:** Nós simplifcamos a implementação
**Desvantagem:**
- Cliente com ecommerce duplicaria pedidos
- Não é aceitável
- Nos responsabiliza por possíveis danos (double charges)

**Trade-off:** Responsabilidade não pode ficar inteiramente no cliente.

## Consequências

**Positivas:**
- ✅ At-least-once: nenhum evento é perdido
- ✅ Dedup é trivial: cliente só precisa checar um ID
- ✅ Padrão de mercado: Stripe, GitHub, AWS todos usam isso
- ✅ Sem overhead nossa: `event_id` é único por evento, gerado de graça
- ✅ Auditoria: cliente consegue auditar qual event_id processou

**Negativas:**
- ❌ Responsabilidade no cliente: requer implementação correta de dedup
- ❌ Pode quebrar cliente ruim: se cliente não implementar dedup, vai processar 2x
- ❌ Sem garantia de exatidão: nós entregamos, cliente decide o que faz
- ❌ Storage cliente: cliente precisa persistir event_ids processados

**Trade-off principal:** Simplicidade nossa em troca de carga no cliente. Mas é carga razoável e padrão de mercado.

---

## Ligações

- Relacionado: ADR-001 (Outbox no MySQL)
- Relacionado: ADR-002 (Retry com Backoff)
- Relacionado: ADR-003 (HMAC para Autenticação)
- Implementação: docs/FDD.md - Seção "Contratos Públicos" (Headers X-Event-Id)
