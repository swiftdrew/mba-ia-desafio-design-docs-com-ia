# FDD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** 2025-11-01
**Proprietário:** Bruno (Senior Engineer, Orders)
**Revisor:** Diego (Senior Engineer, Platform)

---

## 1. Contexto e Motivação Técnica

### Problema Técnico

O sistema de pedidos (Order Management System) não possui mecanismo de notificação externa. Clientes B2B que desejam saber quando um pedido muda de status precisam fazer polling contínuo em `GET /orders`, resultando em:
- Overhead de requisições (cliente faz polling a cada 10-30 segundos)
- Latência alta (até 30 segundos entre mudança real e cliente saber)
- Difícil de escalar (múltiplos clientes = múltiplas requisições)

### Oportunidade

Implementar sistema de webhooks outbound que:
- Notifique clientes em tempo real (< 10 segundos)
- Reduza overhead de polling
- Garanta confiabilidade (retry, persistência)
- Seja seguro (HMAC, TLS, secrets únicos)

### Escopo de Negócio Atendido

Tres clientes B2B aguardando:
- Atlas Comercial
- MaxDistribuição
- Nova Cargo

Prazo: Fim de novembro 2025

---

## 2. Objetivos Técnicos

1. **Notificação em tempo real:** Entrega de eventos < 10 segundos (p95)
2. **Confiabilidade:** At-least-once com retry automático até 5 tentativas
3. **Segurança:** Autenticação HMAC-SHA256, TLS obrigatório, secrets rotacionáveis
4. **Escalabilidade:** Suportar 100+ eventos/minuto inicialmente, extensível pra multi-worker
5. **Operacionalidade:** Zero infra extra (usa MySQL + Node.js existentes)
6. **Auditoria:** Registrar todas as tentativas de entrega (outbox + DLQ)

---

## 3. Escopo e Exclusões

### Incluso

- [x] Webhooks outbound (nós enviamos, cliente recebe)
- [x] Eventos de mudança de status de pedido
- [x] Retry com backoff exponencial
- [x] HMAC-SHA256 + secret por endpoint
- [x] Rotação de secret com grace period 24h
- [x] Endpoints CRUD de configuração de webhook
- [x] Histórico de entregas (GET /webhooks/:id/deliveries)
- [x] Replay manual de DLQ (admin only)
- [x] Observabilidade (logs, métricas, tracing)

### Excluído (Fora de Escopo)

- [ ] Webhooks inbound (cliente enviando pra gente)
- [ ] Email/SMS como fallback (possível fase 2)
- [ ] Dashboard UI (projeto separado frontend)
- [ ] Rate limiting de saída (observar em produção primeiro)
- [ ] Notificação ao cliente de falha (fase 2)
- [ ] Multi-worker scaling (implementar quando atingir saturação)

---

## 4. Fluxos Detalhados

### 4.1. Fluxo Principal: Mudança de Status com Webhook

```
1. Cliente/Admin faz PATCH /orders/:id/status
   ↓
2. OrderController.changeStatus() recebe request
   ↓
3. OrderService.changeStatus() inicia transação
   ├─ Busca order com items
   ├─ Valida transição de status (state machine)
   ├─ [Se PAID] Debita stock
   ├─ [Se CANCELLED from PAID/PROCESSING] Replenish stock
   ├─ UPDATE orders SET status = ?
   ├─ INSERT order_status_history
   ├─ [NEW] publishWebhookEvent(tx, order, from, to)  ← ATÔMICO
   │  └─ Busca webhooks do customer que querem este status
   │  └─ Para cada webhook, INSERT webhook_outbox com evento
   └─ COMMIT (tudo ou nada)
   ↓
4. Retorna order atualizada
   ↓
5. Worker (polling 2s) lê webhook_outbox (status='pending')
   ├─ Busca até 10 eventos ordenados por created_at
   ├─ Para cada evento:
   │  ├─ Busca webhook config (url, secret)
   │  ├─ Calcula HMAC-SHA256(secret, payload)
   │  ├─ POST webhook_url com headers + payload
   │  ├─ [Se 2xx] UPDATE webhook_outbox SET status='delivered'
   │  ├─ [Se erro] Aplica backoff
   │  └─ [Se 5 tentativas] Move para webhook_dead_letter
   └─ Continua polling
   ↓
6. Cliente recebe HTTP POST em seu webhook_url
   ├─ Valida X-Signature (HMAC)
   ├─ Deduplica por X-Event-Id
   ├─ Processa evento (update seu banco)
   └─ Responde com 200 OK
```

### 4.2. Fluxo de Retry com Backoff

```
Evento falha na tentativa 1:
  ├─ Erro capturado (timeout, 5xx, conexão)
  ├─ UPDATE webhook_outbox:
  │  ├─ status = 'pending' (volta pra fila)
  │  ├─ attempt_count = 1
  │  ├─ last_error = "Connection timeout"
  │  └─ next_retry_at = NOW() + 1 minuto
  └─ Log WARN

Próximo polling em 2s encontra evento com next_retry_at < NOW()?
  ├─ Se SIM: tenta novamente (attempt_count = 2)
  │  └─ Aplicar backoff 5 minutos (total ~6m)
  ├─ Se NÃO: skip (espera próximo polling)

Após 5 tentativas (1m + 5m + 30m + 2h + 12h = ~15h):
  ├─ Cria linha em webhook_dead_letter
  ├─ DELETE de webhook_outbox
  └─ Log ERROR
```

### 4.3. Fluxo de Rotação de Secret

```
Cliente faz POST /webhooks/:id/rotate-secret
   ↓
Validação:
  ├─ Webhook exists
  ├─ User autenticado (JWT)
  └─ Owner do webhook = customer do user (implícito de JWT)
   ↓
WebhookSecurityService.generateSecret()
   ↓
UPDATE webhooks:
  ├─ current_secret ← nova string aleatória
  ├─ previous_secret ← (atual) current_secret
  ├─ secret_rotated_at ← NOW()
   ↓
Schedule setTimeout(24h):
  ├─ previous_secret = NULL
  ├─ Log INFO
   ↓
Retorna { newSecret: "abc123..." }
   ↓
Cliente armazena newSecret e reconfig seu lado
   ↓
Worker valida com AMBOS current_secret E previous_secret (grace period)
```

### 4.4. Fluxo de Replay Manual (DLQ)

```
Admin faz POST /admin/webhooks/dead-letter/:id/replay
   ├─ Valida role = ADMIN
   ├─ Busca linha em webhook_dead_letter
   └─ Log auditoria (quem fez, quando)
   ↓
Transação:
  ├─ DELETE de webhook_dead_letter
  ├─ INSERT em webhook_outbox:
  │  ├─ Usa payload original
  │  ├─ status = 'pending'
  │  ├─ attempt_count = 0
  │  └─ event_id = (mantém original pra dedup)
  └─ COMMIT
   ↓
Próximo polling (2s) tenta entregar novamente
```

---

## 5. Contratos Públicos (API)

### 5.1. Criar Webhook

**Endpoint:** `POST /api/v1/webhooks`

**Autenticação:** JWT (qualquer role)

**Request Body:**
```json
{
  "customer_id": "uuid",
  "url": "https://client.example.com/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"]
}
```

**Response (201 Created):**
```json
{
  "id": "webhook-uuid-123",
  "customer_id": "customer-uuid",
  "url": "https://client.example.com/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "enabled": true,
  "secret": "whsec_abcdef123456789_example",
  "created_at": "2025-11-01T14:30:00Z"
}
```

**Error (400):**
```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must use HTTPS",
    "details": { "provided": "http://..." }
  }
}
```

---

### 5.2. Atualizar Webhook

**Endpoint:** `PATCH /api/v1/webhooks/:id`

**Autenticação:** JWT

**Request Body:**
```json
{
  "url": "https://client.example.com/webhooks/v2",
  "events": ["PAID", "SHIPPED"],
  "enabled": false
}
```

**Response (200 OK):**
```json
{
  "id": "webhook-uuid-123",
  "customer_id": "customer-uuid",
  "url": "https://client.example.com/webhooks/v2",
  "events": ["PAID", "SHIPPED"],
  "enabled": false,
  "created_at": "2025-11-01T14:30:00Z",
  "updated_at": "2025-11-01T15:45:00Z"
}
```

---

### 5.3. Listar Webhooks do Customer

**Endpoint:** `GET /api/v1/webhooks`

**Autenticação:** JWT

**Query Params:**
```
GET /api/v1/webhooks?page=1&pageSize=20
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "webhook-uuid-123",
      "customer_id": "customer-uuid",
      "url": "https://client.example.com/webhooks/orders",
      "events": ["PAID", "PROCESSING", "SHIPPED"],
      "enabled": true,
      "created_at": "2025-11-01T14:30:00Z",
      "updated_at": "2025-11-01T14:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 2,
    "totalPages": 1
  }
}
```

---

### 5.4. Obter Detalhes de Webhook

**Endpoint:** `GET /api/v1/webhooks/:id`

**Response (200 OK):**
```json
{
  "id": "webhook-uuid-123",
  "customer_id": "customer-uuid",
  "url": "https://client.example.com/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED"],
  "enabled": true,
  "created_at": "2025-11-01T14:30:00Z",
  "updated_at": "2025-11-01T14:30:00Z"
}
```

---

### 5.5. Deletar Webhook

**Endpoint:** `DELETE /api/v1/webhooks/:id`

**Response (204 No Content)**

---

### 5.6. Rotacionar Secret

**Endpoint:** `POST /api/v1/webhooks/:id/rotate-secret`

**Autenticação:** JWT

**Request Body:** (vazio)

**Response (200 OK):**
```json
{
  "newSecret": "whsec_xyz123abc456def789_example",
  "message": "Secret rotated successfully. Old secret remains valid for 24 hours."
}
```

---

### 5.7. Histórico de Entregas

**Endpoint:** `GET /api/v1/webhooks/:id/deliveries`

**Autenticação:** JWT

**Query Params:**
```
GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=100&status=delivered,failed
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "delivery-uuid-1",
      "event_id": "evt-uuid-123",
      "status": "delivered",
      "http_status": 200,
      "payload": { "event_type": "order.status_changed", "order_id": "ord-123", ... },
      "response_body": "{ \"success\": true }",
      "attempt_number": 1,
      "sent_at": "2025-11-01T14:30:05Z",
      "responded_at": "2025-11-01T14:30:06Z",
      "response_time_ms": 1000
    },
    {
      "id": "delivery-uuid-2",
      "event_id": "evt-uuid-124",
      "status": "failed",
      "error": "Connection timeout",
      "payload": { ... },
      "attempt_number": 3,
      "sent_at": "2025-11-01T14:35:10Z",
      "next_retry_at": "2025-11-01T15:05:00Z"
    }
  ],
  "pagination": { ... }
}
```

---

### 5.8. Replay de Dead Letter (Admin Only)

**Endpoint:** `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

**Autenticação:** JWT + role=ADMIN

**Request Body:** (vazio)

**Response (200 OK):**
```json
{
  "id": "dlq-uuid-1",
  "message": "Event requeued for delivery",
  "event_id": "evt-uuid-999"
}
```

**Log de Auditoria:**
```
INFO admin_action user=admin-123 action=webhook_dlq_replay dead_letter_id=dlq-uuid-1 event_id=evt-uuid-999 timestamp=2025-11-01T16:00:00Z
```

---

### 5.9. Webhook Inbound (Payload do Cliente)

**Método:** `POST` (cliente recebe)

**Headers:**
```
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
X-Webhook-Id: 660e8500-e39c-41d4-a816-446655440001
X-Timestamp: 2025-11-01T14:30:00Z
X-Signature: 2d7e4c91af3f9d8c2b1e6a5f4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c
Content-Type: application/json
```

**Payload:**
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2025-11-01T14:30:00Z",
  "order_id": "ORD-000123",
  "order_number": "ORD-000123",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "customer-uuid",
  "total_cents": 99990
}
```

**Cliente deve responder:**
```json
// 200 OK (sucesso)
{
  "received": true
}

// 400+ (erro, será retentado)
{
  "error": "Invalid signature"
}
```

---

## 6. Matriz de Erros

Todos os códigos de erro do webhook usam prefixo `WEBHOOK_*`:

| Código | HTTP | Descrição | Detalhe |
|--------|------|-----------|---------|
| WEBHOOK_NOT_FOUND | 404 | Webhook não existe | `{ "id": "uuid" }` |
| WEBHOOK_INVALID_URL | 400 | URL não é HTTPS ou inválida | `{ "url": "http://..." }` |
| WEBHOOK_SECRET_REQUIRED | 400 | Secret não fornecido | - |
| WEBHOOK_ALREADY_EXISTS | 409 | URL já cadastrada para customer | `{ "customer_id": "uuid", "url": "..." }` |
| WEBHOOK_CUSTOMER_MISMATCH | 403 | Webhook pertence a outro customer | - |
| WEBHOOK_DELIVERY_NOT_FOUND | 404 | Histórico de entrega não existe | - |
| WEBHOOK_INVALID_SIGNATURE | 401 | X-Signature não bate | (resposta do cliente) |
| WEBHOOK_PAYLOAD_TOO_LARGE | 413 | Evento > 64KB | - |
| WEBHOOK_DISABLED | 422 | Webhook está desabilitado | - |
| WEBHOOK_DELIVERY_FAILED | 502 | Falha ao enviar | `{ "reason": "timeout", "attempts": 3 }` |

---

## 7. Estratégias de Resiliência

### 7.1. Timeouts

- **HTTP request timeout:** 10 segundos
- **Polling interval:** 2 segundos
- **Grace period para secret antigo:** 24 horas
- **Limpeza de eventos entregues:** 30 dias (arquivamento/delete)

### 7.2. Retry Policy

```
Tentativa 1: Imediato
Tentativa 2: +1 minuto (total ~1m)
Tentativa 3: +5 minutos (total ~6m)
Tentativa 4: +30 minutos (total ~36m)
Tentativa 5: +2 horas (total ~2h36m)
Tentativa 6: +12 horas (total ~14h36m)

Após: DLQ (falha permanente)
```

### 7.3. Circuit Breaker (Opcional, Fase 2)

Se webhook falha 10x em 1 hora, desabilitar temporariamente + alertar.

### 7.4. Rate Limiting (Opcional, Fase 2)

Se cliente receber 100+ eventos/minuto, throttle pra evitar DDoS.

---

## 8. Observabilidade

### 8.1. Métricas

**Prometheus:**
```
webhook_delivery_duration_seconds{webhook_id, status} histogram
webhook_delivery_total{webhook_id, status} counter
webhook_retry_count{webhook_id} counter
webhook_dlq_events_total counter
webhook_outbox_queue_size gauge
webhook_secret_rotations_total counter
```

**Exemplos:**
```
# P95 latência de entrega
histogram_quantile(0.95, webhook_delivery_duration_seconds)

# Taxa de sucesso
webhook_delivery_total{status="delivered"} / webhook_delivery_total{status=~"delivered|failed"}

# Tamanho da fila (eventos pendentes)
webhook_outbox_queue_size
```

### 8.2. Logs Estruturados (Pino)

```json
{
  "level": "INFO",
  "service": "order-management-api",
  "component": "webhook-worker",
  "action": "webhook_delivered",
  "webhook_id": "webhook-uuid-123",
  "event_id": "evt-uuid-456",
  "customer_id": "customer-uuid",
  "http_status": 200,
  "duration_ms": 1050,
  "timestamp": "2025-11-01T14:30:06Z"
}

{
  "level": "WARN",
  "component": "webhook-worker",
  "action": "webhook_delivery_failed",
  "webhook_id": "webhook-uuid-123",
  "event_id": "evt-uuid-456",
  "attempt": 2,
  "error": "Connection timeout",
  "next_retry_at": "2025-11-01T14:35:00Z",
  "timestamp": "2025-11-01T14:30:10Z"
}

{
  "level": "ERROR",
  "component": "webhook-worker",
  "action": "webhook_moved_to_dlq",
  "webhook_id": "webhook-uuid-123",
  "event_id": "evt-uuid-456",
  "final_error": "Max retries exceeded",
  "attempts": 5,
  "total_duration_hours": 14.6,
  "timestamp": "2025-11-02T05:00:00Z"
}
```

### 8.3. Tracing Distribuído (OpenTelemetry, Opcional)

```
OrderService.changeStatus()
  └─ trace_id: abc123...
     └─ publishWebhookEvent()
        └─ span: webhook_outbox_insert
           └─ duration: 15ms

WebhookWorker.processOutboxBatch()
  └─ trace_id: xyz789...
     └─ for each event:
        └─ span: webhook_http_request
           └─ duration: 1050ms
```

---

## 9. Integração com Sistema Existente

### 9.1. OrderService.changeStatus()

**Arquivo:** `src/modules/orders/order.service.ts`

Método existente que executa em transação:
```typescript
async changeStatus(
  id: string,
  input: UpdateOrderStatusInput,
  userId: string,
): Promise<OrderWithRelations> {
  return this.prisma.$transaction(async (tx) => {
    // ... validação, stock management ...
    
    // [NEW] Linha de integração
    await publishWebhookEvent(tx, order, fromStatus, toStatus);
    
    // ... return updated order ...
  });
}
```

**Implementação:** Importar função pura `publishWebhookEvent` do módulo de webhooks.

### 9.2. AppError para Webhook

**Arquivo:** `src/shared/errors/app-error.ts`

Subclasses específicas para webhooks (herdam de AppError):
```typescript
export class WebhookNotFoundError extends AppError { }
export class WebhookInvalidUrlError extends AppError { }
// ... etc, prefixo WEBHOOK_ em todos os errorCode
```

**Integração:** Middleware de erro existente já trata AppError.

### 9.3. Logger (Pino)

**Arquivo:** `src/shared/logger/index.ts`

Reutilizar logger centralizado:
```typescript
import { logger } from '@/shared/logger';

logger.info({ webhookId, eventId }, 'Webhook delivered');
```

**Redação de secrets:** Já configurada globalmente (redação de `*secret*`).

### 9.4. Middleware de Autenticação

**Arquivo:** `src/middlewares/auth.middleware.ts`

Reutilizar middleware existente (JWT validation, `req.user` population).

### 9.5. Middleware de Validação (Zod)

**Arquivo:** `src/middlewares/validate.middleware.ts`

Usar no roteador de webhooks como em outros módulos.

### 9.6. PrismaClient e Transações

**Arquivo:** `src/config/database.ts`

Usar mesma instância de `PrismaClient()`. Worker cria instância separada (outro processo).

### 9.7. Schemas Zod

**Arquivo:** `src/modules/webhooks/webhook.schemas.ts`

Seguir padrão existente: `{action}{Resource}Schema` + export types via `z.infer<>`.

---

## 10. Critérios de Aceitação Técnicos

- [ ] Tabelas `webhook`, `webhook_outbox`, `webhook_dead_letter` criadas via migration Prisma
- [ ] Endpoint `POST /webhooks` cria webhook com secret gerado
- [ ] Endpoint `PATCH /webhooks/:id` edita webhook
- [ ] Endpoint `GET /webhooks` lista webhooks do customer
- [ ] Endpoint `GET /webhooks/:id` detalhes
- [ ] Endpoint `DELETE /webhooks/:id` deleta
- [ ] Endpoint `POST /webhooks/:id/rotate-secret` gera novo secret, grace period 24h
- [ ] Endpoint `GET /webhooks/:id/deliveries` lista histórico com status/erro
- [ ] Endpoint `POST /admin/webhooks/dead-letter/:id/replay` recoloca na fila
- [ ] `OrderService.changeStatus()` chama `publishWebhookEvent()` dentro da transação
- [ ] Worker lê outbox a cada 2 segundos, processa batch até 10
- [ ] Webhook retries com backoff 1m/5m/30m/2h/12h
- [ ] Após 5 tentativas, move para DLQ
- [ ] HMAC-SHA256 calculado e mandado no header X-Signature
- [ ] X-Event-Id único por evento (UUID)
- [ ] X-Timestamp ISO 8601
- [ ] X-Webhook-Id no header
- [ ] TLS obrigatório (URL validation rejeita http://)
- [ ] Payload máximo 64KB (erro se exceder)
- [ ] Logs estruturados em Pino (evento entregue, falha, DLQ)
- [ ] Métricas Prometheus para latência, contagem, retry
- [ ] Testes unitários: schemas, security, publisher
- [ ] Testes e2e: fluxo completo mudança status → webhook entregue
- [ ] Testes e2e: retry com timeout
- [ ] Testes e2e: secret rotation
- [ ] Documentação de cliente (portal dev) com exemplo HMAC validation
- [ ] Auditoria: quem fez replay de DLQ logado

---

## 11. Riscos e Mitigação

| Risco | Prob | Impacto | Mitigação |
|-------|------|---------|-----------|
| Worker cai, outbox acumula | Média | Alto | Health check + restart script; eventos persistem, can replay |
| Secret vazado | Baixa | Alto | Rotação rápida (24h grace); auditoria de acessos; TLS |
| Cliente implementa dedup errado | Média | Médio | Doc clara; exemplos de código; suporte ativo |
| Backlog outbox cresce infinito | Baixa | Médio | Índices em status+created_at; monitoramento tamanho; cleanup 30d |
| Payload > 64KB | Muito Baixa | Baixo | Validação rejeita; nenhum evento real deve chegar perto |
| DLQ acumula | Baixa | Baixo | Alertas; investigação de causa raiz (client fora? secret errado?) |

---

## 12. Checklist Técnico Pré-Deploy

- [ ] Migrations Prisma rodadas e validadas
- [ ] Worker testado localmente e em staging
- [ ] Load test: 100 eventos/min, latência aceitável
- [ ] Retry test: client offline, verifica que tenta 5x
- [ ] Secret rotation test: cliente em transição, ambos secrets funcionam
- [ ] DLQ test: evento falhado 5x aparece em DLQ
- [ ] Security review (Sofia): HMAC, TLS, secret storage
- [ ] Logs sanitizados (nenhum secret em plaintext)
- [ ] Métricas expostas e alertas configurados
- [ ] Documentação cliente atualizada (portal dev)
- [ ] Runbook pra troubleshooting (worker não responde, backlog alto, etc)
- [ ] Disaster recovery plan (backup outbox, restore)

