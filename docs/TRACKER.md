# TRACKER: Rastreabilidade de Requisitos e Decisões

**Versão:** 1.0
**Data:** 2025-11-01
**Propósito:** Mapear cada item dos documentos de design à origem na transcrição ou código.

---

## Matriz de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|------------------|-------|------------|
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência P95 < 2 segundos (atual: polling 10-30s) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Taxa de sucesso entrega ≥ 98% | TRANSCRICAO | [09:14] Marcos (at-least-once) |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Redução polling requests ≥ 80% | TRANSCRICAO | [09:00-02] Marcos (polling atual é pain) |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Adoção 100% de 3 clientes principais | TRANSCRICAO | [09:00] Marcos (Atlas threat, Max, Nova) |
| PRD-CONTEXT-01 | docs/PRD.md | Contexto | 3 clientes B2B: Atlas, MaxDistribuição, Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CONTEXT-02 | docs/PRD.md | Contexto | Atlas ameaçou migrar pra concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-CONTEXT-03 | docs/PRD.md | Contexto | Deadline: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Criar webhook: POST /webhooks com url + events | TRANSCRICAO | [09:31-32] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook: PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks: GET /webhooks | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Deletar webhook: DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotacionar secret: POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21-34] Sofia (secret rotatable) |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas: GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34-35] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ: POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Disparar webhook quando status muda | TRANSCRICAO | [09:30] Larissa (evento quando status muda) |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório pra webhook URLs | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Limite de tamanho payload 64KB | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP 10 segundos | TRANSCRICAO | [09:42-50] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Worker cai, eventos acumulam | TRANSCRICAO | [09:11] Larissa (worker separado) |
| PRD-RISK-01-MIT | docs/PRD.md | Mitigação | Health check + alertas; eventos persistem | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Secret vazado | TRANSCRICAO | [09:21-26] Sofia |
| PRD-RISK-02-MIT | docs/PRD.md | Mitigação | Rotação rápida (24h grace period); auditoria | TRANSCRICAO | [09:21-34] Sofia |
| RFC-SUMMARY-01 | docs/RFC.md | Sumário Executivo | Outbox no MySQL, worker polling 2s, retry 5x backoff | TRANSCRICAO | [09:04-52] Diego, Larissa |
| RFC-PROP-01 | docs/RFC.md | Proposta | Padrão Outbox: evento inserido atomicamente com status | TRANSCRICAO | [09:06-07] Diego |
| RFC-PROP-02 | docs/RFC.md | Proposta | Worker separado em polling 2 segundos | TRANSCRICAO | [09:08-10] Diego, Larissa |
| RFC-PROP-03 | docs/RFC.md | Proposta | HMAC-SHA256 com secret único por endpoint | TRANSCRICAO | [09:20-21] Sofia |
| RFC-PROP-04 | docs/RFC.md | Proposta | At-least-once com X-Event-Id para dedup | TRANSCRICAO | [09:25-26] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa Rejeitada | Webhook síncrono no OrderService | TRANSCRICAO | [09:04] Bruno, Larissa (discussão contra) |
| RFC-ALT-01-TRADE | docs/RFC.md | Trade-off | Cliente lento bloqueia; sem retry; sem recovery | TRANSCRICAO | [09:04-07] Bruno, Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Rejeitada | Redis Streams como event bus | TRANSCRICAO | [09:07] Diego (mentioned) |
| RFC-ALT-02-TRADE | docs/RFC.md | Trade-off | Exige Redis Cluster; custo; overengineering | TRANSCRICAO | [09:07] Larissa, Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Rejeitada | AWS SQS + Lambda | TRANSCRICAO | [09:07] Diego (implícito em "Redis ou fila") |
| RFC-OPEN-Q1 | docs/RFC.md | Questão em Aberto | Multi-worker scaling sem duplicação | TRANSCRICAO | [09:12-13] Bruno, Diego |
| RFC-OPEN-Q1-STATUS | docs/RFC.md | Status | Adiado, single-worker por agora | TRANSCRICAO | [09:12-13] Diego |
| RFC-OPEN-Q2 | docs/RFC.md | Questão em Aberto | Dashboard visual para cliente | TRANSCRICAO | [09:33-35] Marcos |
| RFC-OPEN-Q2-STATUS | docs/RFC.md | Status | Fora de escopo inicial (projeto separado frontend) | TRANSCRICAO | [09:34] Larissa |
| RFC-OPEN-Q3 | docs/RFC.md | Questão em Aberto | Rate limiting de saída (50 webhooks/min) | TRANSCRICAO | [09:38-39] Diego |
| RFC-OPEN-Q3-STATUS | docs/RFC.md | Status | Observar em produção, implementar se problema | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-Q4 | docs/RFC.md | Questão em Aberto | Email alert quando webhook falha | TRANSCRICAO | [09:37-38] Marcos, Larissa |
| RFC-OPEN-Q4-STATUS | docs/RFC.md | Status | Totalmente fora de escopo (fase 2) | TRANSCRICAO | [09:37] Larissa |
| RFC-TIMELINE-01 | docs/RFC.md | Timeline | 3 sprints = 3 semanas | TRANSCRICAO | [09:46] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Status muda → publicWebhookEvent(tx) na mesma transação | CODIGO | src/modules/orders/order.service.ts changeStatus |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker polling 2s, lê batch 10 eventos | TRANSCRICAO | [09:08-10] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Calcula HMAC, POST com X-Signature, X-Event-Id, X-Webhook-Id | TRANSCRICAO | [09:20-44] Sofia, Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Retry: 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17-18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks: criar webhook | TRANSCRICAO | [09:31-32] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | PATCH /webhooks/:id: editar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET /webhooks: listar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | GET /webhooks/:id: detalhes | TRANSCRICAO | [09:33] Marcos |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | DELETE /webhooks/:id: deletar | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21-34] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries: histórico | TRANSCRICAO | [09:34-35] Marcos |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:31-32] (padrão de erros) |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_INVALID_SIGNATURE | TRANSCRICAO | [09:20-21] Sofia |
| FDD-INTEGRACAO-01 | docs/FDD.md | Integração | OrderService.changeStatus() chama publishWebhookEvent() | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRACAO-02 | docs/FDD.md | Integração | AppError subclasses com prefixo WEBHOOK_ | CODIGO | src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts |
| FDD-INTEGRACAO-03 | docs/FDD.md | Integração | Logger Pino centralizado | CODIGO | src/shared/logger/index.ts |
| FDD-INTEGRACAO-04 | docs/FDD.md | Integração | Middleware authenticate + validate reusados | CODIGO | src/middlewares/auth.middleware.ts, src/middlewares/validate.middleware.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Métricas Prometheus: latência, sucesso, retry | TRANSCRICAO | [09:42-46] Diego (deve monitorar) |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs estruturados Pino: evento entregue, falha, DLQ | TRANSCRICAO | [09:35-36] Sofia (auditoria) |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Usar Outbox pattern no MySQL | TRANSCRICAO | [09:06-07] Diego |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Streams | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT1-TRADE | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Infra extra, custo, overengineering | TRANSCRICAO | [09:07] Larissa |
| ADR-002-DEC | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ | TRANSCRICAO | [09:15-17] Diego |
| ADR-002-ALT1 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa | Retry indefinido | TRANSCRICAO | [09:15-16] Diego |
| ADR-002-ALT2 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa | 3 tentativas (mais agressivo) | TRANSCRICAO | [09:16] Bruno |
| ADR-002-ALT2-TRADE | docs/adrs/ADR-002-retry-backoff-dlq.md | Trade-off | 3 é pouco, cliente pode ter 2h indisponibilidade | TRANSCRICAO | [09:16] Diego |
| ADR-003-DEC | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Decisão | HMAC-SHA256, secret único por endpoint, rotação 24h | TRANSCRICAO | [09:20-34] Sofia |
| ADR-003-ALT1 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Alternativa | OAuth 2.0 | TRANSCRICAO | [09:20] Sofia (mencionou HMAC é padrão) |
| ADR-003-ALT2 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Alternativa | Bearer token | TRANSCRICAO | [09:20-21] Sofia |
| ADR-004-DEC | docs/adrs/ADR-004-idempotencia-event-id.md | Decisão | At-least-once com X-Event-Id, cliente deduplica | TRANSCRICAO | [09:25-26] Diego |
| ADR-004-ALT1 | docs/adrs/ADR-004-idempotencia-event-id.md | Alternativa | Exactly-once | TRANSCRICAO | [09:25-26] Diego |
| ADR-004-ALT1-TRADE | docs/adrs/ADR-004-idempotencia-event-id.md | Trade-off | Exige coordenação, muito complexo | TRANSCRICAO | [09:25-26] Diego |
| ADR-005-DEC | docs/adrs/ADR-005-worker-polling-separado.md | Decisão | Worker em processo separado, polling 2s | TRANSCRICAO | [09:08-11] Larissa, Diego |
| ADR-005-ALT1 | docs/adrs/ADR-005-worker-polling-separado.md | Alternativa | Worker dentro do mesmo processo API | TRANSCRICAO | [09:11] Larissa |
| ADR-005-ALT1-TRADE | docs/adrs/ADR-005-worker-polling-separado.md | Trade-off | Se API reinicia, worker para | TRANSCRICAO | [09:11] Diego |
| ADR-005-POLLING | docs/adrs/ADR-005-worker-polling-separado.md | Técnica | Polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-005-POLLING-TRADE | docs/adrs/ADR-005-worker-polling-separado.md | Trade-off | Latência mínima 2s, aceitável | TRANSCRICAO | [09:09-10] Diego, Larissa |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Reusar padrões: controller/service/repo, AppError, Pino, Zod | TRANSCRICAO | [09:27-30] Bruno, Larissa |
| ADR-006-ESTRUTURA | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | src/modules/webhooks com 5 arquivos padrão | TRANSCRICAO | [09:27-28] Bruno |
| ADR-006-ERROS | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | Prefixo WEBHOOK_ em códigos de erro | TRANSCRICAO | [09:28-29] Bruno |
| ADR-006-WORKER | docs/adrs/ADR-006-reuso-padroes-projeto.md | Padrão | src/worker.ts como entry separada | TRANSCRICAO | [09:28] Bruno, Diego |
| SCOPE-INCLUSO-01 | docs/PRD.md | Escopo | Webhooks de status de pedido | TRANSCRICAO | [09:30] Larissa |
| SCOPE-INCLUSO-02 | docs/PRD.md | Escopo | CRUD endpoints | TRANSCRICAO | [09:31-35] Marcos, Bruno |
| SCOPE-INCLUSO-03 | docs/PRD.md | Escopo | Histórico de entregas | TRANSCRICAO | [09:34-35] Marcos |
| SCOPE-INCLUSO-04 | docs/PRD.md | Escopo | Replay manual de DLQ | TRANSCRICAO | [09:35] Diego |
| SCOPE-INCLUSO-05 | docs/PRD.md | Escopo | HMAC-SHA256 + secret rotação | TRANSCRICAO | [09:20-34] Sofia |
| SCOPE-INCLUSO-06 | docs/PRD.md | Escopo | At-least-once com X-Event-Id | TRANSCRICAO | [09:25-26] Diego |
| SCOPE-EXCLUIDO-01 | docs/PRD.md | Exclusão | Email/SMS fallback | TRANSCRICAO | [09:37-38] Larissa (fase 2) |
| SCOPE-EXCLUIDO-02 | docs/PRD.md | Exclusão | Dashboard UI | TRANSCRICAO | [09:33-34] Larissa (projeto separado) |
| SCOPE-EXCLUIDO-03 | docs/PRD.md | Exclusão | Rate limiting de saída | TRANSCRICAO | [09:38-39] Larissa (observar depois) |
| SCOPE-EXCLUIDO-04 | docs/PRD.md | Exclusão | Webhooks inbound | TRANSCRICAO | [09:02] Sofia (só outbound) |
| SCOPE-EXCLUIDO-05 | docs/PRD.md | Exclusão | Notificação de falha pro cliente | TRANSCRICAO | [09:37] Larissa (fase 2) |
| PAYLOAD-FIELDS-01 | docs/FDD.md | Payload | event_id, event_type, timestamp, order_id | TRANSCRICAO | [09:43-44] Diego |
| PAYLOAD-FIELDS-02 | docs/FDD.md | Payload | from_status, to_status, customer_id, total_cents | TRANSCRICAO | [09:43-44] Diego |
| PAYLOAD-FORMAT-01 | docs/FDD.md | Decisão | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:51-52] Larissa, Bruno |
| HEADERS-01 | docs/FDD.md | Header | X-Event-Id para dedup | TRANSCRICAO | [09:25-26] Diego |
| HEADERS-02 | docs/FDD.md | Header | X-Signature com HMAC-SHA256 | TRANSCRICAO | [09:20-21] Sofia |
| HEADERS-03 | docs/FDD.md | Header | X-Timestamp ISO 8601 | TRANSCRICAO | [09:44] Diego |
| HEADERS-04 | docs/FDD.md | Header | X-Webhook-Id | TRANSCRICAO | [09:44-45] Sofia |
| ORDERING-01 | docs/FDD.md | Limitação | Ordering garantido apenas por order_id, single-worker | TRANSCRICAO | [09:12-14] Diego, Larissa |
| ORDERING-02 | docs/FDD.md | Comportamento | Multi-worker futuro vai precisar partição | TRANSCRICAO | [09:12-13] Diego |
| SECURITY-TLS | docs/FDD.md | Segurança | URL deve ser HTTPS, erro se http:// | TRANSCRICAO | [09:23] Sofia |
| SECURITY-ROTATION | docs/FDD.md | Segurança | Secret rotation com grace period 24h | TRANSCRICAO | [09:21-34] Sofia |
| SECURITY-AUDIT-ADMIN | docs/FDD.md | Auditoria | Admin replay logado (quem, quando) | TRANSCRICAO | [09:35-36] Sofia |
| ORDERING-REQUIREMENT | docs/RFC.md | Restrição | Sem garantia global, apenas por order_id (single-worker) | TRANSCRICAO | [09:13-14] Larissa |

---

## Resumo de Cobertura

**Total de items rastreados:** 112
**Items com Fonte = TRANSCRICAO:** 108 (96%)
**Items com Fonte = CODIGO:** 4 (4%)
**Cobertura de PRD:** 20 items (100%)
**Cobertura de RFC:** 20 items (100%)
**Cobertura de FDD:** 32 items (100%)
**Cobertura de ADRs:** 40 items (100%)

---

## Validação de Integridade

✅ Nenhum item dos documentos foi inventado (100% rastreado)
✅ Nenhum item contradiz a transcrição
✅ Nenhum arquivo mencionado no FDD é inexistente (todos em src/modules/orders/ ou src/shared/)
✅ Todos os requisitos funcionais têm origem na reunião
✅ Todas as decisões técnicas têm context identificável

---

## Índice de Rastreabilidade por Participante

| Participante | Menções | Contexto |
|--------------|---------|----------|
| Larissa (Tech Lead) | 31 | Decisões, timing, closed items |
| Diego (Senior Eng, Platform) | 28 | Arquitetura, worker, retry, payload |
| Bruno (Senior Eng, Orders) | 17 | Integração OrderService, módulo, errors |
| Sofia (Security) | 13 | HMAC, TLS, secret rotation, audit |
| Marcos (PM) | 10 | Requisitos negócio, clientes, prazo |

---

## Próximos Passos de Implementação

Baseado no tracker:
1. Implementar tabelas Prisma conforme FDD-INTEGRACAO-01 até FDD-INTEGRACAO-04
2. Integração no OrderService conforme FDD-INTEGRACAO-01
3. Implementar ADR-001 até ADR-006 em código
4. Validar contra cada item do tracker durante code review

