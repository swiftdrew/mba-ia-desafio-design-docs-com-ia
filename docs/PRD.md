# PRD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.1
**Data:** 2026-07-16
**Product Manager:** Marcos
**Tech Lead:** Larissa

---

## 1. Resumo e Contexto da Feature

O sistema de pedidos do OMS hoje não possui mecanismo de notificação em tempo real para clientes B2B. Três clientes significativos (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling contínuo em `GET /orders` para saber se pedido mudou de status, resultando em custo operacional alto, latência inaceitável e risco de churn.

Esta feature implementa **webhooks outbound** que notificam clientes instantaneamente quando pedidos mudam de status, eliminando necessidade de polling, melhorando experiência de integração e retendo clientes estratégicos.

---

## 2. Problema e Motivação

### Situação Atual

Clientes B2B usam polling síncrono em `GET /orders`:
- **Atlas Comercial:** Faz request a cada 10 segundos pra ~100 pedidos ativos (1.000 req/dia)
- **MaxDistribuição:** Faz request a cada 30 segundos pra ~50 pedidos (2.880 req/dia)
- **Nova Cargo:** Faz request a cada 20 segundos pra ~80 pedidos (4.320 req/dia)

**Total:** ~8.000 requisições/dia extras de polling (sobrecarga)

### Impacto

1. **Cliente (dor):** Latência alta. Atlas faz poll a cada 10s → espera até 10 segundos pra saber que pedido mudou. Frustrante.
2. **Nós (overhead):** CPU/banda desperdiçada em polling vazio.
3. **Negócio (risco):** Atlas sinalizou em reunião que pode migrar para concorrente se não entregar webhooks até fim de novembro.

### Motivação

Webhooks permitem:
- ✅ Notificação instant (< 10 segundos)
- ✅ Zero polling (reduz 8k req/dia)
- ✅ Melhor UX do cliente
- ✅ Retenção de clientes B2B estratégicos
- ✅ Fundação pra futuras features (notificação de outros eventos)

---

## 3. Público-Alvo e Cenários de Uso

### Público-Alvo Principal

**Clientes B2B integrados via API:**
- Atlas Comercial
- MaxDistribuição
- Nova Cargo
- (Futuros clientes)

**Personas:**
1. **Engenheiro de Integração** (cliente): Configura webhook na plataforma OMS, valida recebimento via X-Signature, deduplica por event_id.
2. **Ops/Devops** (cliente): Monitora saúde do webhook (histórico de entregas), reclama se falha persistente.
3. **Larissa** (Tech Lead, nós): Administra feature, monitora performance do worker.

### Cenários de Uso

#### Cenário 1: Notificação de Entrega
> Atlas recebe pedido PED-001. Status: PENDING. Seu sistema quer saber quando mudar pra SHIPPED para gerar boleto de frete.

```
Cliente configura webhook:
  POST /webhooks
  {
    "url": "https://atlas.com.br/webhooks/orders",
    "events": ["SHIPPED", "DELIVERED"]
  }

Quando pedido vira SHIPPED:
  Nós enviamos POST para https://atlas.com.br/webhooks/orders
  {
    "event_id": "evt-123",
    "event_type": "order.status_changed",
    "order_id": "PED-001",
    "from_status": "PROCESSING",
    "to_status": "SHIPPED",
    "timestamp": "2025-11-15T10:30:00Z"
  }

Atlas recebe, valida X-Signature, processa (gera boleto), responde 200 OK.
```

#### Cenário 2: Recovery de Cliente Offline
> MaxDistribuição teve manutenção de 1 hora, webhook falha.

```
Nós mandamos 5 tentativas com backoff exponencial.
Após 1 hora, cliente volta online.
Tentativa 3 ou 4 (30+ minutos depois) consegue.
Cliente deduplica por event_id e processa sem duplicar.
```

#### Cenário 3: Rotação de Secret por Segurança
> Cliente descobriu que secret foi logado em desenvolvimento.

```
Cliente faz POST /webhooks/:id/rotate-secret
Recebe novo secret: sk_live_newABC123...
Sistema aguarda 24h antes de invalidar secret antigo.
Cliente migra seu código pro novo secret.
Após 24h, secret antigo morre automaticamente.
```

---

## 4. Objetivos e Métricas de Sucesso

### Objetivo Principal

**Eliminar polling de clientes B2B e notificar mudanças de pedido em tempo real (< 10 segundos).**

### Métricas de Sucesso

| Métrica | Meta | Rastreamento | Observação |
|---------|------|--------------|-----------|
| **Latência de entrega < 10 segundos** | Abaixo de 10s | ✅ [09:02] Marcos | Requisito cliente real |
| **Latência P95 de entrega** | < 2 segundos | ℹ️ [09:09-10] Diego/Larissa | Implementação técnica via polling 2s |
| **At-least-once delivery** | Retry 5x com backoff | ✅ [09:24-26] Diego | Requisito cliente real |
| **Taxa de sucesso de entrega** | ≥ 98% | ❓ HIPÓTESE | Não mencionado na reunião; SLO interno a calibrar com Marcos |
| **Eliminar polling de clientes** | Webhook adoption | ✅ [09:00] Marcos | Requisito cliente real |
| **Redução de polling requests** | ≥ 80% | ❓ HIPÓTESE | Não mencionado; métrica de adoção a calibrar com Marcos (80% ou 100%?) |
| **Adoção de clientes** | 100% dos 3 principais | ✅ [09:00] Marcos | Atlas, Max, Nova Cargo usando webhooks |
| **Tempo médio pra regressão** | < 30 minutos | ℹ️ Derivado | Se webhook falhar, tempo até conseguir novamente |
| **Confiabilidade do worker** | ≥ 99.5% | ✅ [09:11] Larissa | Uptime do processo separado |
| **Satisfação do cliente** | ≥ 4/5 | ✅ [09:00-45] Marcos | Feedback com Atlas, Max, Nova |

### Notas sobre Rastreabilidade de Métricas

**Legenda:**
- ✅ **Requisito Cliente Real**: Mencionado explicitamente na reunião de design
- ℹ️ **Derivação Técnica**: Decisão da equipe para atingir requisito cliente
- ❓ **Hipótese a Calibrar**: Não foi mencionado na reunião; requer validação com Marcos

**Métricas que Requerem Validação com Marcos:**
1. **Taxa de sucesso ≥ 98%** — É expectativa cliente ou SLO interno? Se interno, qual número (95%? 99%? 99.5%)?
2. **Redução polling ≥ 80%** — Vocês esperam 80% (adoção parcial) ou 100% (adoção completa)?

Vide `docs/TRACKER.md` (seção "Hipóteses Identificadas") para rastreamento completo.

### Critérios de Sucesso Técnicos

- [ ] Worker nunca perde um evento (persistido em outbox)
- [ ] At-least-once delivery garantido (com dedup no cliente)
- [ ] HMAC-SHA256 validation funciona (cliente consegue validar assinatura)
- [ ] Secret rotation sem downtime (24h grace period)
- [ ] DLQ funciona (eventos falhados recuperáveis manualmente)
- [ ] Zero dados sensíveis em logs (sanitização de secrets)

---

## 5. Escopo

### Incluso (MVP - Dentro de Escopo)

✅ **Webhooks de mudança de status de pedido**
- Eventos disparados quando OrderStatus muda
- Seleção de quais statuses cliente quer receber

✅ **Configuração via API**
- CRUD de webhooks (criar, editar, listar, deletar)
- Geração de secret único por endpoint
- Rotação de secret com grace period 24h

✅ **Retenção e auditoria**
- Histórico de entregas (GET /webhooks/:id/deliveries)
- Dead Letter Queue com reprocessamento manual

✅ **Segurança**
- HMAC-SHA256 autenticação
- TLS obrigatório (HTTPS)
- Validação de URL

✅ **Resiliência**
- Retry com backoff exponencial (5 tentativas)
- Worker em processo separado
- Persistência em tabela outbox

✅ **Observabilidade**
- Logs estruturados
- Métricas Prometheus
- Auditoria de ações admin

### Excluído (Fora de Escopo)

❌ **Email/SMS como fallback**
Enviar email pro cliente se webhook falhar 3x. → Fase 2

❌ **Dashboard UI**
Painel visual pra cliente ver webhooks. → Projeto separado de frontend

❌ **Rate limiting de saída**
Throttle se cliente receber 100+ eventos/minuto. → Observar em produção, implementar se virar problema

❌ **Webhooks inbound**
Cliente enviando webhooks pra nós. → Escopo separado (se necessário)

❌ **Notificação de falha**
Cliente receber alert quando webhook dele falha. → Fase 2 (juntamente com email)

❌ **Multi-worker scaling automático**
Múltiplos workers processando em paralelo. → Implementar quando atingir saturação (observar latência)

---

## 6. Requisitos Funcionais

### RF-001: Criar Webhook
**Descrição:** Cliente autenticado consegue cadastrar novo webhook.

**Atores:** Engenheiro de integração (cliente)

**Fluxo:**
1. Cliente autenticado faz POST /webhooks
2. Fornece URL (HTTPS), lista de statuses
3. Sistema gera secret único
4. Sistema retorna secret uma única vez
5. Cliente armazena secret

**Aceitação:**
- Secret gerado aleatoriamente (32+ chars)
- URL validada pra HTTPS
- Secret retornado apenas na criação
- Evento criado registrado em outbox se existir

---

### RF-002: Editar Webhook
**Descrição:** Cliente consegue atualizar configuração de webhook (URL, eventos, ativar/desativar).

**Fluxo:**
1. Cliente faz PATCH /webhooks/:id
2. Fornece novos valores pra url/events/enabled
3. Sistema valida
4. Sistema atualiza e retorna configuração atual

---

### RF-003: Listar Webhooks
**Descrição:** Cliente consegue ver todos seus webhooks cadastrados.

**Fluxo:**
1. Cliente faz GET /webhooks
2. Sistema retorna paginado (page, pageSize, total)
3. Inclui detalhes: id, url, events, enabled, created_at

---

### RF-004: Obter Webhook
**Descrição:** Cliente consegue ver detalhes de um webhook específico.

**Fluxo:**
1. Cliente faz GET /webhooks/:id
2. Sistema retorna webhook completo

---

### RF-005: Deletar Webhook
**Descrição:** Cliente consegue remover um webhook cadastrado.

**Fluxo:**
1. Cliente faz DELETE /webhooks/:id
2. Sistema deleta webhook
3. Novos eventos não são disparados pra este webhook
4. Histórico de entregas anterior é preservado

---

### RF-006: Rotacionar Secret
**Descrição:** Cliente consegue gerar novo secret pra webhook.

**Fluxo:**
1. Cliente faz POST /webhooks/:id/rotate-secret
2. Sistema gera novo secret
3. Secret anterior fica válido por 24 horas (grace period)
4. Após 24h, secret anterior é descartado

**Aceitação:**
- Nova secret devolvida ao cliente
- Ambos secrets funcionam em transações durante 24h
- Após 24h, tentativa com secret antigo falha com 401

---

### RF-007: Disparar Webhook em Mudança de Status
**Descrição:** Sistema dispara webhook quando status de pedido muda.

**Fluxo:**
1. Admin/Cliente faz PATCH /orders/:id/status
2. OrderService valida transição
3. Dentro da mesma transação SQL:
   - Atualiza order status
   - Insere em order_status_history
   - Debita/replenish stock se necessário
   - Insere em webhook_outbox pra cada webhook configurado
4. Transação commitada (atomic)

**Aceitação:**
- Evento entra em outbox SEMPRE que transação sucede
- Se transação falha, evento não entra (rollback)
- Evento carrega snapshot do pedido no momento da mudança

---

### RF-008: Processar Outbox e Entregar Webhooks
**Descrição:** Worker consome outbox e dispara HTTP requests.

**Fluxo:**
1. Worker roda em processo separado
2. A cada 2 segundos:
   - Busca até 10 eventos com status='pending'
   - Pra cada evento:
     - Calcula HMAC-SHA256(secret, payload)
     - POST webhook_url com headers + payload
     - [Sucesso] marca 'delivered'
     - [Erro] aplica backoff, tenta novamente
     - [5 tentativas] move pra DLQ

**Aceitação:**
- Nenhum evento é perdido (persistido ou entregue)
- Latência P95 < 2 segundos (desde mudança até envio)
- Timeout de 10 segundos por request

---

### RF-009: Listar Histórico de Entregas
**Descrição:** Cliente consegue ver histórico de webhooks entregues.

**Fluxo:**
1. Cliente faz GET /webhooks/:id/deliveries?page=1&pageSize=100&status=delivered
2. Sistema retorna paginado:
   - event_id, status (delivered/pending/failed)
   - timestamp de envio e resposta
   - status HTTP da resposta
   - tempo de resposta (ms)

**Aceitação:**
- Filtra por status (delivered, pending, failed)
- Ordena por timestamp desc (mais recentes primeiro)
- Mostra erro/motivo se falhou

---

### RF-010: Reprocessar Dead Letter (Admin)
**Descrição:** Admin consegue reprocessar evento que falhou permanentemente.

**Fluxo:**
1. Admin autentica com role=ADMIN
2. Faz POST /admin/webhooks/dead-letter/:id/replay
3. Sistema pega evento de DLQ
4. Recoloca em outbox com status='pending'
5. Worker tenta novamente na próxima iteração

**Aceitação:**
- Apenas role=ADMIN consegue
- Ação é auditada (quem fez, quando)
- Event_id é mantido (dedup no cliente ainda funciona)

---

## 7. Requisitos Não Funcionais

### RNF-001: Performance
- **Latência de entrega < 10 segundos** ✅ (requisito cliente [09:02] Marcos)
- **Latência P95 entrega:** < 2 segundos ℹ️ (implementação via polling 2s)
- **Latência P99 entrega:** < 10 segundos ✅ (alinhado com requisito cliente)
- **Throughput:** ≥ 100 eventos/minuto (no início)
- **CPU worker:** < 5% (em idle)

### RNF-002: Disponibilidade
- **Uptime do worker:** ≥ 99.5% ✅ [09:11] Larissa
- **At-least-once delivery:** Garantido com retry 5x ✅ [09:24-26] Diego
- **Taxa de sucesso de entrega:** ≥ 98% ❓ HIPÓTESE (não mencionado; a calibrar)
- **RTO (Recovery Time Objective):** < 5 minutos (se worker cai)

### RNF-003: Segurança
- **HTTPS obrigatório** pra webhook URLs
- **HMAC-SHA256** pra assinatura
- **Secret rotation** com grace period 24h
- **TLS 1.2+** em trânsito
- **Nenhum secret em log** (redação)

### RNF-004: Escalabilidade
- Suportar 100 clientes simultaneamente
- Suportar 100+ eventos/minuto
- Tabela outbox sem degradação até 1M de registros

### RNF-005: Observabilidade
- Logs estruturados (Pino)
- Métricas Prometheus
- Tracing distribuído (OpenTelemetry, opcional)

### RNF-006: Auditoria
- Todas ações de admin logadas (quem, quando, o quê)
- Histórico de rotação de secret
- DLQ como evidence de falhas

---

## 8. Decisões e Trade-offs Principais

### D-1: Outbox Pattern vs Síncrono
**Decisão:** Usar padrão Outbox no MySQL.
**Requisito Cliente:** Latência < 10 segundos [09:02] Marcos
**Trade-off:** +Atomicidade, +Resiliência vs -Latência mínima 2s (implementação), -Necessário worker separado
**Rastreabilidade:** [09:04-07] Bruno, Diego

### D-2: Retry com Backoff vs Indefinido
**Decisão:** 5 tentativas com backoff (1m/5m/30m/2h/12h), depois DLQ.
**Requisito Cliente:** At-least-once delivery [09:24-26] Diego
**Trade-off:** +Clareza operacional, +Finito vs -Cliente offline > 15h não recovera
**Rastreabilidade:** [09:14-17] Diego, Bruno

### D-3: HMAC-SHA256 vs OAuth
**Decisão:** HMAC-SHA256 com secret único por endpoint.
**Requisito Cliente:** Segurança de autenticação [09:20-21] Sofia
**Trade-off:** +Simples pra cliente, +Padrão de mercado vs -Responsabilidade no cliente
**Rastreabilidade:** [09:19-21] Sofia

### D-4: Single-worker vs Redis Streams
**Decisão:** Single worker em polling 2s, sem Redis.
**Requisito Cliente:** Latência < 10 segundos com implementação simples
**Trade-off:** +Sem infra extra, +Simples vs -Não escalável automaticamente, -Single point of failure
**Rastreabilidade:** [09:06-13] Diego, Larissa (nota: P95 < 2s é implementação via polling 2s, não requisito cliente)

---

## 9. Dependências

### Dependências Técnicas
- MySQL/Prisma (já existe)
- Node.js + TypeScript (já existe)
- Pino logger (já existe)
- Zod validation (já existe)

### Dependências de Negócio
- Feedback dos clientes B2B (Atlas, Max, Nova Cargo)
- Code review de segurança (Sofia)
- Alinhamento com timeline (3 sprints)

### Dependências Externas
- Nenhuma (não requer infra externa)

---

## 10. Riscos e Mitigação

### Risco 1: Worker Cai, Eventos Acumulam
**Probabilidade:** Média (processo Node pode crashar)
**Impacto:** Alto (clientes não recebem notificações)
**Mitigação:**
- Health check + restart script
- Alertas se worker down > 5 minutos
- Eventos persistem em outbox, podem ser reprocessados

### Risco 2: Secret Vazado
**Probabilidade:** Baixa (secret é long string, random)
**Impacto:** Alto (attacker faz request fake)
**Mitigação:**
- Rotação rápida (24h grace period)
- Auditoria de acessos a secrets
- TLS obrigatório (HTTPS)
- Documentação de boas práticas (guardar em .env, não em git)

### Risco 3: Backlog na Outbox Cresce Infinito
**Probabilidade:** Baixa (com índices e cleanup)
**Impacto:** Médio (tabela fica lenta)
**Mitigação:**
- Índices em status+created_at
- Monitoramento de tamanho (alerta se > 1M)
- Cleanup de registros entregues com 30+ dias

### Risco 4: Cliente Implementa Dedup Errado
**Probabilidade:** Média (é responsabilidade dele)
**Impacto:** Médio (cliente processa evento 2x)
**Mitigação:**
- Documentação clara no portal de dev
- Exemplos de código (Node, Python, Java)
- Suporte ativo (Marcos ajuda cliente)

### Risco 5: Sinal de Polling Still Popular
**Probabilidade:** Baixa (clientes solicitaram webhooks)
**Impacto:** Baixo (polling coexiste)
**Mitigação:**
- Webhooks é opt-in, polling continua funcionando
- Migração gradual é ok

---

## 11. Critérios de Aceitação

1. ✅ Todos 6 RFs implementados e testados
2. ✅ Latência P95 < 2 segundos (validado em testes)
3. ✅ Taxa de sucesso ≥ 98% (sem retry infinito)
4. ✅ Segurança: HMAC, TLS, secret rotation funciona
5. ✅ 3 clientes B2B conseguem integrar sem help externo
6. ✅ Adoção: 100% dos 3 principais clientes
7. ✅ Sem events perdidos (auditoria)
8. ✅ Documentação de cliente atualizada (portal dev)
9. ✅ Runbook de troubleshooting escrito
10. ✅ Alertas configurados (Datadog/Prometheus)

---

## 12. Estratégia de Testes e Validação

### Testes Unitários
- Schemas Zod (validação de URL, events)
- HMAC signing/verification
- Backoff calculation
- Error code mapping

### Testes de Integração
- OrderService.changeStatus() insere em outbox
- Webhook CRUD endpoints (create/update/delete)
- Secret rotation (old secret válido 24h)
- DLQ reprocessing

### Testes E2E
- Fluxo completo: status muda → webhook entregue em < 2s
- Retry: cliente offline → webhook retentado 5x
- Secret rotation: cliente em transição funciona
- Dedup: cliente recebe 2 eventos, processa 1

### Testes de Carga
- 100 eventos/minuto por 1 hora
- 10 workers simultâneos (futura)
- Query na outbox com 1M registros

### Validação com Cliente Real
- Atlas integra e dispara webhooks
- Valida X-Signature com seu código
- Confirma latência é aceitável

---

## 13. Roadmap Futuro (Fase 2+)

- **Fase 2:** Email/SMS como fallback; rate limiting; dashboard UI
- **Fase 3:** Multi-worker scaling automático; webhooks inbound
- **Fase 4:** Eventos de outros domínios (customer criado, payment aprovado, etc)

---

## 14. Links de Referência

- **Transcrição da reunião:** TRANSCRICAO.md
- **RFC técnica:** docs/RFC.md
- **FDD (detalhe implementação):** docs/FDD.md
- **ADRs:** docs/adrs/
  - [ADR-001: Outbox Pattern](./adrs/ADR-001-outbox-no-mysql.md)
  - [ADR-002: Retry Policy](./adrs/ADR-002-retry-backoff-dlq.md)
  - [ADR-003: HMAC Security](./adrs/ADR-003-hmac-sha256-autenticacao.md)
  - [ADR-004: Idempotência](./adrs/ADR-004-idempotencia-event-id.md)
  - [ADR-005: Worker Architecture](./adrs/ADR-005-worker-polling-separado.md)
  - [ADR-006: Padrões Reuso](./adrs/ADR-006-reuso-padroes-projeto.md)

---

## Aprovação

- [x] **Marcos** (Product Manager) — inicial, v1.0
- [x] **Larissa** (Tech Lead) — inicial, v1.0
- [x] **Bruno** (Senior Engineer, Orders) — inicial, v1.0
- [x] **Diego** (Senior Engineer, Platform) — inicial, v1.0
- [x] **Sofia** (Security Engineer) — inicial, v1.0

**Data de Aprovação v1.0:** 2025-11-01

### Status v1.1 (Atualizado 2026-07-16)

Conforme feedback de Marcos, o PRD foi reformulado para:
- ✅ Reancorar metas em falas reais da transcrição
- ✅ Separar requisitos cliente de derivações técnicas
- ✅ Marcar hipóteses não mencionadas como "a calibrar"

**Pendente de Aprovação:**
- ⏳ **Marcos** — Validação das 2 hipóteses:
  1. Taxa de sucesso ≥ 98% (é OK? qual SLO?)
  2. Redução polling ≥ 80% (é 80% ou 100%?)

Vide `PARA_MARCOS_VALIDACAO.md` para questões detalhadas.

