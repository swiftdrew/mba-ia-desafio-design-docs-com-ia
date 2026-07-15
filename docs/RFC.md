# RFC: Sistema de Webhooks de Notificação de Pedidos

**Autor:** Larissa (Tech Lead)
**Status:** Approved
**Data:** 2025-11-01
**Versão:** 1.0

**Revisores:**
- Diego (Senior Engineer, Platform)
- Bruno (Senior Engineer, Orders)
- Sofia (Security Engineer)
- Marcos (Product Manager)

---

## TL;DR

Implementar notificação em tempo real de mudanças de status de pedidos para clientes B2B via webhooks outbound. Usar padrão Outbox no MySQL para consistência transacional, worker separado em polling com retry automático. Autenticação via HMAC-SHA256, idempotência via X-Event-Id.

**Latência esperada:** < 2 segundos (no caso médio), < 10 segundos (pior caso com retry inicial)
**Entrega:** At-least-once (cliente implementa dedup se necessário)
**Custo de infra:** Zero (usa MySQL + Node.js existentes)

---

## Contexto e Problema

### Situação Atual

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling contínuo em `GET /orders` para saber se status mudou. Isso gera:
- Sobrecarga de requisições (um cliente fazendo polling a cada 10 segundos)
- Latência alta (cliente só sabe mudança após próximo poll, até 20s de delay)
- Custo operacional (banda, CPU, frustrante)

Atlas já sinalizou migração pra concorrente se não entregar em ~fim de trimestre.

### Requisito de Negócio

"Notificar clientes em tempo real (< 10 segundos) quando status muda."

Hoje interpretado como: qualquer latência < 10 segundos é "tempo real" aceitável. Não é case de sub-segundo.

### Restrição de Segurança

Dados de pedidos (order_id, amounts, customer info) saem da nossa infra para URLs de terceiros. Obrigatório:
- Autenticar que request veio de nós (não attacker fake)
- Detectar tampering de payload
- Permitir rotação de credentials sem break de integração

---

## Proposta Técnica

### Visão Geral

```
Cliente B2B requests feature:
"Notifica quando meu pedido muda"
        ↓
OrderService.changeStatus() executa:
  1. Update orders table
  2. Insert order_status_history
  3. Debit/replenish stock
  4. [NEW] Insert event in webhook_outbox (SAME TRANSACTION)
        ↓
Worker (processo separado, polling 2s):
  Reads webhook_outbox (status='pending')
  ↓ for each event:
  Calculate HMAC-SHA256(secret, payload)
  POST webhook_url with event + signature
  ↓ on success: mark 'delivered'
  ↓ on failure: apply backoff (1m, 5m, 30m, 2h, 12h) → DLQ
```

### Componentes Principais

#### 1. **Outbox Pattern no MySQL**
- Tabela `webhook_outbox`: captura eventos atomicamente com mudança de status
- Mesmo batch de processamento evita inconsistência
- Worker consome de forma desacoplada
- Sem infra extra (usa MySQL existente)

#### 2. **Worker Independente**
- Processo Node.js separado (`npm run worker`)
- Polling a cada 2 segundos
- Lê batch pequeno (≤10 eventos), processa
- Marca como delivered/failed, move para DLQ se falha permanente

#### 3. **Resiliência com Backoff Exponencial**
- 5 tentativas: 1m → 5m → 30m → 2h → 12h
- Total ~15h da primeira falha
- Tolera indisponibilidades planejadas (manutenção até 2h)
- DLQ table para auditoria + reprocessamento manual

#### 4. **Autenticação HMAC-SHA256**
- Cada endpoint recebe secret único
- Assinatura no header X-Signature
- Rotação com grace period 24h (old secret válido em paralelo)

#### 5. **Idempotência via X-Event-Id**
- UUID único por evento, mandado no header + payload
- Cliente deduplica pelo ID
- At-least-once: nós garantimos entrega, cliente garante "não process 2x"

#### 6. **Reuso de Padrões Existentes**
- Módulo em `src/modules/webhooks/` (controller, service, repository, schemas)
- Error codes com prefixo `WEBHOOK_`
- Logger Pino centralizado
- Middleware auth/validate existentes

---

## Alternativas Consideradas

### Alt-1: Webhook Síncrono dentro do OrderService

**Proposta:** Chamar HTTP synchronously quando status muda, dentro da mesma transação.

**Trade-off Rejeitado:**
- ❌ Cliente lento bloqueia mudança de status pra outros pedidos
- ❌ Se cliente offline, não há recovery (rollback? não resolve)
- ❌ Timeout impede fechamento rápido de transação
- ❌ Sem retry automático

**Conclusão:** Viável apenas se cliente garantir SLA de resposta < 100ms (irrealista).

---

### Alt-2: Redis Streams como Event Bus

**Proposta:** Usar Redis Streams como fila, múltiplos consumers (workers), melhor escalabilidade futura.

**Trade-off Rejeitado:**
- ❌ Exige infra Redis Cluster (não existe hoje, overhead de devops)
- ❌ Custo AWS adicional
- ❌ Overkill para latência < 10s (polling simples é suficiente)
- ❌ Dados sensíveis saem do MySQL pra Redis (aumenta surface de segurança)

**Conclusão:** Viable em escala maior (100+ pedidos/segundo). Hoje: overengineering.

---

### Alt-3: AWS SQS + Lambda

**Proposta:** Usar SQS como fila, Lambda serverless como consumer.

**Trade-off Rejeitado:**
- ❌ Vendor lock-in AWS (migrabilidade pior)
- ❌ Latência: SQS tem delay até 1s, + Lambda coldstart até 5s
- ❌ Dados sensíveis em serviço externo (compliance)
- ❌ Custo mais alto que solução in-house

**Conclusão:** Viable se já em AWS. Não é caso nosso.

---

## Questões em Aberto

### Q1: Multi-worker Scaling
**Problema:** Se escalarmos pra múltiplos workers (hoje não é necessário), como evitar que 2 workers processem o mesmo evento?

**Decisão adiada:** Não é escopo inicial. Single-worker por agora. Quando atingir ponto de saturação, revisitamos.

**Rastreabilidade:** [09:12-13] Diego e Bruno

### Q2: Dashboard para Cliente
**Problema:** Cliente quer ver histórico de webhooks entregues ("últimos 100 webhooks, sucesso/falha")?

**Decisão adiada:** Não é escopo inicial. Endpoints API sim (GET /webhooks/:id/deliveries). UI dashboard é projeto separado do time de frontend.

**Rastreabilidade:** [09:33-35] Marcos

### Q3: Rate Limiting de Saída
**Problema:** Se pedido X tem 50 mudanças de status em 1 minuto (improvável), cliente recebe 50 webhooks?

**Decisão adiada:** Observar em produção. Se virar problema, implementar batching ou throttling depois. Não é requisito inicial.

**Rastreabilidade:** [09:38-39] Diego

### Q4: Notificação de Falha para Cliente
**Problema:** Cliente quer receber alert (email) quando webhook dele falha N vezes?

**Decisão adiada:** Email está totalmente fora de escopo dessa fase. Possível fase 2.

**Rastreabilidade:** [09:37-38] Marcos

---

## Impacto e Riscos

### Impacto Positivo

- **Negócio:** Retém clientes B2B (Atlas), melhora experiência integração
- **Operação:** Reduz polling, libera banda
- **Arquitetura:** Introduz padrão robusto (outbox) reutilizável em futuros features

### Impacto Negativo

- **Performance:** Worker ocupa outro processo Node.js
- **Operação:** Gerenciar 2 processos em produção (não é trivial, precisa de script/supervisor)
- **Observabilidade:** Novo componente requer monitoramento (latência de entrega, DLQ backlog)

### Riscos Técnicos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|--------|-----------|
| Worker cai, eventos acumulam | Média | Alto | Health check + alertas; eventos persistem na outbox, pode reprocessar depois |
| Secret vazamento | Baixa | Alto | Rotação rápida (24h grace period); auditoria de acessos |
| Backlog na outbox cresce infinito | Baixa | Médio | Índices + monitoramento + cleanup de arquivados |
| Cliente implementa dedup errado | Média | Médio | Documentação clara no portal de dev; suporte ativo |

---

## Decisões Relacionadas

Arquitetura detalhada de cada decisão em ADRs próprios:

- **[ADR-001](./adrs/ADR-001-outbox-no-mysql.md):** Padrão Outbox no MySQL
- **[ADR-002](./adrs/ADR-002-retry-backoff-dlq.md):** Retry com Backoff Exponencial e DLQ
- **[ADR-003](./adrs/ADR-003-hmac-sha256-autenticacao.md):** Autenticação HMAC-SHA256 com Secret por Endpoint
- **[ADR-004](./adrs/ADR-004-idempotencia-event-id.md):** Garantia At-Least-Once com X-Event-Id
- **[ADR-005](./adrs/ADR-005-worker-polling-separado.md):** Worker em Processo Separado com Polling
- **[ADR-006](./adrs/ADR-006-reuso-padroes-projeto.md):** Reuso de Padrões e Infraestrutura Existentes

---

## Timeline

**Estimativa:** 3 sprints de 1 semana

1. **Sprint 1:** Modelagem (outbox, webhooks table, DLQ), esquemas Zod, erros
2. **Sprint 2:** Worker + retry logic, HMAC security, integração OrderService
3. **Sprint 3:** Endpoints CRUD, deliveries history, replay admin, testes ponta-a-ponta
   - (+) Code review segurança (Sofia): 2 dias úteis
   - (+) QA testing: 3 dias úteis

**Prazo acordado:** Fim de novembro com clientes

---

## Próximos Passos

1. Aprovação dessa RFC (hoje)
2. Review dos ADRs por team (Bruno, Diego)
3. Design session: detalhe de schemas Prisma + FDD
4. Setup do backlog e stories JIRA
5. Kick-off Sprint 1

---

## Glossário

- **Outbox:** Tabela que captura eventos em mesma transação que mudança de estado
- **Worker:** Processo Node.js que consome outbox e dispara webhooks
- **DLQ (Dead Letter Queue):** Tabela que armazena eventos que falharam permanentemente
- **At-least-once:** Garantia de que cada evento é entregue >= 1 vez
- **HMAC-SHA256:** Função criptográfica que assina payload com secret compartilhado
- **Grace period:** Janela de tempo em que credential antiga ainda é válida (durante rotação)
- **Idempotência:** Propriedade de operação que pode ser repetida sem efeito colateral

---

## Aprovação

- [x] Larissa (Tech Lead)
- [x] Diego (Senior Engineer, Platform)
- [x] Bruno (Senior Engineer, Orders)
- [x] Sofia (Security Engineer)
- [x] Marcos (Product Manager)

**Data de Aprovação:** 2025-11-01

**Notas dos Revisores:**

- **Larissa:** Pronto, alinha com timeline que passei pro Marcos.
- **Diego:** LGTM, design é sólido. Outbox é padrão que queria usar há tempos.
- **Bruno:** Aprovado. Integração no OrderService é clean.
- **Sofia:** Liberado, HMAC + TLS obrigatório me tranquiliza.
- **Marcos:** Clientes vão gostar. Confirmo prazo com Atlas.
