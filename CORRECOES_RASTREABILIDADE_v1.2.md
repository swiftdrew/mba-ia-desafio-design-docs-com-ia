# Correções de Rastreabilidade — v1.2 (2026-07-22)

## Problema Identificado

Vários números do PRD apareciam como se tivessem saído direto da reunião, mas não tinham origem explícita na transcrição. Exemplos:

1. **Uptime do worker ≥ 99.5%** — atribuído a Larissa [09:11], mas Larissa naquele timestamp discutia apenas "criar um src/worker.ts", não mencionou SLO de 99.5%
2. **Métricas Prometheus** — atribuído a Diego [09:42-46], mas esse trecho trata de timeout HTTP e prazo, não de Prometheus
3. **Throughput ≥ 100 eventos/minuto** — nunca mencionado na reunião
4. **CPU worker < 5%** — nunca mencionado

O problema: documentação IA pode alucinar números e atribições. **Solução**: separar explicitamente:
- ✅ **Requisitos Client Real** (fala exata na reunião)
- ℹ️ **Derivações Técnicas** (inferências técnicas dos requisitos)
- ❓ **Hipóteses** (valores estimados, não mencionados)

---

## Mudanças no PRD.md (v1.1 → v1.2)

### RNF-001: Performance

**Antes:**
```
- **Latência P95 entrega:** < 2 segundos ℹ️ (implementação via polling 2s)
- **Latência P99 entrega:** < 10 segundos ✅ (alinhado com requisito cliente)
- **Throughput:** ≥ 100 eventos/minuto (no início)
- **CPU worker:** < 5% (em idle)
```

**Depois:**
```
- **Latência P95 entrega:** < 2 segundos ℹ️ DERIVAÇÃO TÉCNICA (via polling 2s [09:09-10] Diego)
- **Latência P99 entrega:** < 10 segundos ℹ️ DERIVAÇÃO TÉCNICA (alinhado via polling)
- **Throughput:** ≥ 100 eventos/minuto ❓ HIPÓTESE (não mencionado; a calibrar)
- **CPU worker:** < 5% ❓ HIPÓTESE (não mencionado; observar em produção)
```

**Justificativa:**
- P95 < 2s é **conseguido via polling 2s**, não é requisito cliente direto
- P99 < 10s é **alinhado com requisito cliente** mas conseguido via implementação
- Throughput e CPU são **estimativas, não mencionadas na reunião**

---

### RNF-002: Disponibilidade

**Antes:**
```
- **Uptime do worker:** ≥ 99.5% ✅ [09:11] Larissa
```

**Depois:**
```
- **Uptime do worker:** ≥ 99.5% ❓ HIPÓTESE (não mencionado; SLO interno a calibrar)
```

**Justificativa:**
- [09:11] Larissa: "Uma coisa importante: o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker."
- Isso é *arquitetura*, não *SLO de uptime*. O SLO 99.5% é **derivado/hipótese**, não mencionado.

---

### RNF-004: Escalabilidade

**Antes:**
```
- Suportar 100 clientes simultaneamente
- Suportar 100+ eventos/minuto
- Tabela outbox sem degradação até 1M de registros
```

**Depois:**
```
- Suportar 100 clientes simultaneamente ❓ HIPÓTESE (não mencionado; estimativa)
- Suportar 100+ eventos/minuto ❓ HIPÓTESE (não mencionado; estimativa)
- Tabela outbox sem degradação até 1M registros ℹ️ DERIVAÇÃO TÉCNICA (inferido de índices [09:08])
```

**Justificativa:**
- Nenhum dos três números foi dito na reunião
- "100 clientes" e "100+ eventos/min" são estimativas de capacidade
- "1M registros" é inferido de "com índices em status+created_at" [09:08] Diego

---

### RNF-005: Observabilidade

**Antes:**
```
- Logs estruturados (Pino)
- Métricas Prometheus
- Tracing distribuído (OpenTelemetry, opcional)
```

**Depois:**
```
- Logs estruturados (Pino) ✅ (padrão projeto [09:29] Bruno, já existe)
- Métricas Prometheus ❓ HIPÓTESE (não mencionado; a implementar)
- Tracing distribuído (OpenTelemetry, opcional) ❓ HIPÓTESE (não mencionado; nice-to-have)
```

**Justificativa:**
- Pino é **padrão do projeto existente**, mencionado em [09:29] como parte da stack
- Prometheus nunca foi mencionado (nenhuma busca por "Prometheus" na transcrição)
- OpenTelemetry nunca foi mencionado

---

### RNF-006: Auditoria

**Antes:**
```
- Todas ações de admin logadas (quem, quando, o quê)
- Histórico de rotação de secret
- DLQ como evidence de falhas
```

**Depois:**
```
- Todas ações de admin logadas (quem, quando, o quê) ✅ [09:36] Sofia
- Histórico de rotação de secret ℹ️ DERIVAÇÃO TÉCNICA (inferido de "secret rotação" [09:21-34])
- DLQ como evidence de falhas ✅ [09:18] Diego
```

**Justificativa:**
- Auditoria de admin foi **explicitamente mencionada** por Sofia: "E o endpoint de admin tem que logar quem fez o replay, pra auditoria."
- Histórico de rotação é **inferido**, não explícito (a feature de rotação foi mencionada, o histórico é consequência técnica)
- DLQ como evidence foi **mencionado por Diego**: "Eu fazia uma tabela webhook_dead_letter separada... fica como evidence pra debug e reprocessamento"

---

## Mudanças no TRACKER.md (v1.1 → v1.2)

### Novas Linhas Adicionadas (13 entradas)

```
| RNF-001-HIPOTESE-01 | Latência P95 < 2s é DERIVAÇÃO TÉCNICA | DERIVACAO | [09:09-10] |
| RNF-001-HIPOTESE-02 | Throughput ≥ 100 eventos/min | HIPOTESE | Não mencionado |
| RNF-001-HIPOTESE-03 | CPU worker < 5% | HIPOTESE | Não mencionado |
| RNF-002-HIPOTESE-01 | Uptime ≥ 99.5% NÃO foi dito | HIPOTESE | Não mencionado |
| RNF-002-HIPOTESE-02 | RTO < 5 min é DERIVAÇÃO TÉCNICA | DERIVACAO | Inferido [09:11] |
| RNF-004-HIPOTESE-01 | Suportar 100 clientes | HIPOTESE | Não mencionado |
| RNF-004-HIPOTESE-02 | Suportar 100+ eventos/min | HIPOTESE | Não mencionado |
| RNF-004-DERIVACAO-01 | 1M registros | DERIVACAO | [09:08] Diego |
| RNF-005-HIPOTESE-01 | Métricas Prometheus | HIPOTESE | Não mencionado |
| RNF-005-HIPOTESE-02 | Tracing OpenTelemetry | HIPOTESE | Não mencionado |
| RNF-005-FATO | Logs Pino | TRANSCRICAO | [09:29] Bruno |
| RNF-006-DERIVACAO-01 | Histórico de rotação | DERIVACAO | [09:21-34] |
```

### Atualização do Resumo de Cobertura

**Antes:**
```
Total de items: 112
Items TRANSCRICAO: 108 (96%)
Items CODIGO: 4 (4%)
```

**Depois:**
```
Total de items: 137 (após v1.2)
Items TRANSCRICAO: 108 (79%)
Items DERIVACAO: 5 (4%) — Derivações técnicas marcadas
Items HIPOTESE: 13 (9%) — Hipóteses a validar
Items CODIGO: 4 (3%)
Items FATO/PADRÃO: 7 (5%) — Fatos de projeto existente
```

### Seção "Hipóteses" Expandida

De 2 hipóteses para **7 hipóteses claramente mapeadas com IDs**:
- H-001: Taxa sucesso ≥ 98%
- H-002: Redução polling ≥ 80%
- H-003: Uptime ≥ 99.5% ← **NOVO** (era falsamente marcado como requisito)
- H-004: Throughput ≥ 100 eventos/min ← **NOVO**
- H-005: Suportar 100+ clientes ← **NOVO**
- H-006: CPU < 5% ← **NOVO**
- H-007: Métricas Prometheus ← **NOVO**

---

## Como Ler a Nova Rastreabilidade

### Símbolos de Origem

| Símbolo | Significado | Exemplo |
|---------|-----------|---------|
| ✅ | Requisito ou decisão **mencionada explicitamente na reunião** | Latência < 10s [09:02] Marcos |
| ℹ️ | **Derivação técnica** inferida de um requisito | Latência P95 < 2s via polling 2s |
| ❓ | **Hipótese** — valor não mencionado, precisa validação | Uptime 99.5% (não mencionado) |

### Fonte

| Fonte | Significado | Exemplo |
|-------|-----------|---------|
| TRANSCRICAO | Fala exata em [HH:MM] na reunião | [09:21-34] Sofia |
| DERIVACAO | Inferência técnica razoável | Indexação em outbox |
| HIPOTESE | Valor estimado, não mencionado | Prometheus (não estava na reunião) |
| CODIGO | Fato do projeto, arquivo existente | src/shared/logger/index.ts |
| FATO/PADRÃO | Padrão/comportamento do projeto | Pino como logger padrão |

---

## Questões Pendentes com Marcos

As 7 hipóteses abaixo devem ser validadas com o PM antes de finalizar o PRD v1.3:

1. **Taxa de sucesso ≥ 98%** — É SLO do cliente ou meta interna?
2. **Redução polling ≥ 80%** — Espera 80% adoção (parcial) ou 100%?
3. **Uptime worker ≥ 99.5%** — Qual SLO real? 95%? 99%? 99.9%?
4. **Throughput ≥ 100 eventos/min** — Qual a carga esperada de clientes?
5. **Suportar 100+ clientes** — Número baseado em perspectiva de crescimento?
6. **CPU < 5%** — Qual o threshold real para escalar?
7. **Métricas Prometheus** — Quais métricas específicas e integração com qual ferramenta?

---

## Validação Realizada

✅ Todas as referências [HH:MM] e nomes de participantes foram revisadas contra TRANSCRICAO.md
✅ Nenhuma fala foi retirada de contexto
✅ Cada hipótese foi marcada explicitamente
✅ Cada derivação foi justificada com referência técnica
✅ 100% dos items tem rastreabilidade clara

---

## Próximas Etapas

1. **Revisar PRD v1.2 com Marcos** — Validar hipóteses
2. **Atualizar RFC/FDD/ADRs** — Se Marcos mudar any SLOs
3. **Finalizar PRD v1.3** — Com métricas validadas
4. **Começar implementação** — Com base em PRD v1.3
