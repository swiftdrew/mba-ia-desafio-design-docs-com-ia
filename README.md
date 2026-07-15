# Sistema de Webhooks de Notificação de Pedidos: Design Docs Completos

## Sobre o Desafio

Este repositório contém o pacote completo de design docs para o **Sistema de Webhooks de Notificação de Pedidos** do Order Management System (OMS). A feature foi discutida em uma reunião técnica envolvendo Tech Lead, PM, engenheiros e segurança, e toda a decisão foi documentada em forma de PRD, RFC, FDD, 6 ADRs, tracker de rastreabilidade e este README.

O desafio consistiu em transformar a transcrição da reunião técnica (`TRANSCRICAO.md`) em um pacote de documentação acionável, usando IA como ferramenta principal de produção. O resultado é um conjunto de documentos em nível de detalhe suficiente para que o time de engenharia possa iniciar a implementação imediatamente.

## Ferramentas de IA Utilizadas

- **Claude (via Claude Code):** Ferramenta principal. Usada para:
  - Exploração do codebase (estrutura, padrões, convenções)
  - Análise da transcrição (extração de requisitos, decisões, restrições)
  - Geração de conteúdo dos documentos (PRD, RFC, FDD, ADRs)
  - Rastreabilidade (mapeamento de items à origem)

- **Task Tool (Explore Agent):** Para varredura rápida do codebase identificando padrões arquiteturais, estrutura de módulos, padrões de error handling, authentication, logging.

- **Bash + Manual Reading:** Verificação de existência real de arquivos mencionados nos documentos (validação de rastreabilidade).

## Workflow Adotado

A ordem de execução seguiu a recomendação do desafio, com ajustes iterativos:

### 1. Exploração Inicial (Foundation)
- Leitura completa de `TRANSCRICAO.md` para entender contexto, atores, decisões discutidas
- Exploração do codebase (estrutura modular, padrões, patterns de error handling, autenticação)
- Mapeamento mental das 6 decisões principais mencionadas na reunião

### 2. ADRs (Skeleton)
- Produção de 6 ADRs cobrindo as decisões principais:
  - ADR-001: Outbox no MySQL
  - ADR-002: Retry com backoff e DLQ
  - ADR-003: HMAC-SHA256 autenticação
  - ADR-004: At-least-once com X-Event-Id
  - ADR-005: Worker em processo separado
  - ADR-006: Reuso de padrões existentes
- **Motivo:** ADRs formam o esqueleto do "como implementar". RFC, FDD e PRD se constroem em cima delas.

### 3. RFC (Architecture Layer)
- Consolidação da proposta técnica com base nos ADRs
- Inclusão de alternativas descartadas (síncrono, Redis, SQS/Lambda)
- Questões em aberto com status e rastreabilidade à transcrição
- Referências cruzadas aos ADRs já escritos
- **Motivo:** RFC propõe e abre pra revisão, sem descer ao detalhe de implementação.

### 4. FDD (Implementation Layer)
- Detalhe técnico acionável: fluxos, endpoints, payloads, códigos de erro
- Seção "Integração com sistema existente" referenciando 4+ arquivos reais
- Observabilidade (métricas, logs, tracing)
- **Motivo:** FDD é profundo, implementação acontece daqui.

### 5. PRD (Business Layer)
- Consolidação dos requisitos em nível de negócio
- Métricas de sucesso com targets quantitativos
- Público-alvo, cenários de uso, trade-offs
- **Motivo:** PRD é mais alto nível, natural produzir por último.

### 6. TRACKER (Validation Layer)
- Mapeamento de cada item dos docs à origem (transcrição + código)
- Validação de integridade (sem invenção, sem contradição)
- **Motivo:** TRACKER é melhor aliado contra alucinações da IA.

### 7. README (Closure)
- Documentação do processo, learnings, iterações
- **Motivo:** Deixar por último permite refletir sobre toda a jornada.

---

## Prompts Customizados

### Prompt 1: Exploração de Padrões Codebase

```
Explore the codebase structure to understand:
1. How modules are organized (src/modules structure)
2. The pattern used for controllers, services, repositories, schemas, and routes
3. Error handling patterns (AppError classes, error codes)
4. Logger implementation (Pino)
5. Authentication and authorization patterns (JWT, roles)
6. Database patterns (Prisma, transactions)
7. The order.service.ts file structure and changeStatus method
8. HTTP response patterns
9. Middleware structure

Be thorough and list all the key patterns and conventions used in this codebase.
```

**Resultado:** Exploração detalhada de 18 padrões/convenções, servindo como base para ADR-006 (Reuso de Padrões).

---

### Prompt 2: Extração de Requisitos da Transcrição

```
Analise a transcrição (TRANSCRICAO.md) e extraia:

1. Todas as decisões FECHADAS (não questionadas no fim)
2. Todos os requisitos FUNCIONAIS explicitamente mencionados
3. Requisitos NÃO FUNCIONAIS (segurança, performance, etc)
4. Pontos EXPLICITAMENTE EXCLUÍDOS ("fora de escopo", "fase 2", "adiado")
5. Restrições (TLS, rate limits, payloadsize)
6. Alternativas REJEITADAS com o trade-off discutido

Para cada item, capture:
- Conteúdo exato (não parafrasear demais)
- Timestamp [hh:mm]
- Quem falou (ou grupo de discussão)
- Contexto (por que foi decidido assim)

Organize em seções por tipo (decisões, requisitos, exclusões, etc).
```

**Resultado:** Arquivo estruturado com ~80 items mapeados, base para PRD, RFC, FDD e TRACKER.

---

## Iterações e Ajustes

### Iteração 1: Primeiros ADRs (Superficialidade)
**Problema:** Os primeiros drafts dos ADRs foram genéricos, com "consequências" muito abstratas ("mais seguro", "mais complexo").

**Ajuste:** Refiz os ADRs pedindo:
- Trade-offs explícitos (com números: "15 horas", "24h grace period")
- Consequências concretas (benchmarks, coeficientes, nomes de classes)
- Referências ao código existente (caminhos reais de arquivos)

**Resultado:** ADR-001 a ADR-006 ficaram acionáveis, com 5-7 páginas cada, não 2-3.

### Iteração 2: RFC Muito Longo
**Problema:** RFC inicial ficou com 8 páginas, violando requisito "2-4 páginas, conciso".

**Ajuste:** Removi detalhes de implementação (que pertencem ao FDD) e mantive apenas:
- Visão geral da solução
- 3 alternativas rejeitadas com trade-offs
- 4 questões em aberto

**Resultado:** RFC reduzido para 3.5 páginas, mantendo clareza.

### Iteração 3: FDD sem Integração com Código Real
**Problema:** Seção "Integração com sistema existente" mencionava arquivos fictícios ("webhook.module.ts", "webhook.factory.ts").

**Ajuste:** Reli o codebase e atualizei com caminhos REAIS:
- `src/modules/orders/order.service.ts` (changeStatus)
- `src/shared/errors/app-error.ts` (AppError base)
- `src/shared/logger/index.ts` (Pino)
- `src/middlewares/auth.middleware.ts` (authenticate)

**Resultado:** FDD integração com 4 arquivos reais, validado no TRACKER.

### Iteração 4: TRACKER Incompleto
**Problema:** Primeiras versões do TRACKER tinham ~60 items. Requisito minimo é 80%.

**Ajuste:** Varrida segunda em docs (PRD, RFC, FDD) e ADRs, capturando:
- Cada RF do PRD como linha
- Cada RNF
- Cada risco + mitigação
- Cada decisão de ADR
- Cada endpoint + contrato de FDD

**Resultado:** TRACKER cresceu para 112 items (100% cobertura), com 96% rastreáveis à transcrição.

### Iteração 5: README Genérico
**Problema:** README inicial usava linguagem muito genérica ("IA gerou documentos").

**Ajuste:** Reescrevi com:
- Descrição concreta da feature (webhooks outbound, at-least-once, HMAC)
- Nomes reais dos prompts que usei
- Exemplos específicos de problemas encontrados (ADRs superficiais, FDD fiction)
- Learnings concretos (TRACKER como validação)

**Resultado:** README atual tem 2 prompts, 5 iterações mapeadas, workflow detalhado.

---

## Como Navegar a Entrega

### Estrutura de Arquivos
```
docs/
├── PRD.md                          # Requisitos de negócio
├── RFC.md                          # Proposta técnica
├── FDD.md                          # Detalhe de implementação
├── TRACKER.md                      # Rastreabilidade
└── adrs/
    ├── ADR-001-outbox-no-mysql.md
    ├── ADR-002-retry-backoff-dlq.md
    ├── ADR-003-hmac-sha256-autenticacao.md
    ├── ADR-004-idempotencia-event-id.md
    ├── ADR-005-worker-polling-separado.md
    └── ADR-006-reuso-padroes-projeto.md
```

### Ordem Recomendada de Leitura

**Para Product Manager / Stakeholder:**
1. Este README (context)
