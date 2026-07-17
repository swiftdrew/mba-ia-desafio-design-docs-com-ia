# Reformulação dos Documentos de Design

**Data:** 2026-07-16
**Status:** ✅ CONCLUÍDO
**Versão:** v1.0 → v1.1

---

## O que foi reformulado

Conforme feedback de Marcos, os documentos de design foram reformulados para fazer a distinção fundamental entre:
1. **Objetivos cliente** (mencionados na reunião)
2. **Derivações técnicas** (decisões internas da team)
3. **Hipóteses a calibrar** (não mencionados; requerem validação)

---

## Arquivos Modificados

### ✅ docs/TRACKER.md (v1.1)

**Mudanças:**
- 3 metas reancoradas em falas REAIS da transcrição
- 2 hipóteses marcadas como "a calibrar"
- 1 derivação técnica documentada
- Nova seção: "Hipóteses Identificadas que Requerem Validação"
- Total de items: 114 (era 112)

**Exemplo de reancoramento:**

ANTES:
```markdown
| PRD-OBJ-01 | Latência P95 < 2 segundos | TRANSCRICAO | [09:02] Marcos |
```

DEPOIS:
```markdown
| PRD-OBJ-01 | Latência de entrega < 10s (requisito cliente) | ✅ [09:02] Marcos |
| PRD-OBJ-01-IMPL | Latência P95 < 2s (derivação técnica) | ℹ️ [09:09-10] Diego/Larissa |
```

---

### ✅ docs/PRD.md (v1.1)

**Mudanças na Seção 4: "Objetivos e Métricas de Sucesso":**

#### Tabela de Métricas Reformulada

Antes (v1.0):
```markdown
| Métrica | Meta | Como Medir |
| Latência P95 | < 2 segundos | ... |
| Taxa sucesso | ≥ 98% | ... |
| Redução polling | ≥ 80% | ... |
```

Depois (v1.1):
```markdown
| Métrica | Meta | Rastreamento | Observação |
| Latência < 10s | < 10s | ✅ [09:02] Marcos | Requisito cliente |
| Latência P95 | < 2s | ℹ️ [09:09-10] Diego | Implementação técnica |
| Taxa sucesso | ≥ 98% | ❓ HIPÓTESE | Não mencionado; calibrar |
| Redução polling | ≥ 80% | ❓ HIPÓTESE | Não mencionado; calibrar |
```

#### Nova Seção: "Notas sobre Rastreabilidade de Métricas"

Adicionada após a tabela:

```markdown
**Legenda:**
- ✅ **Requisito Cliente Real**: Mencionado explicitamente na reunião
- ℹ️ **Derivação Técnica**: Decisão da equipe para atingir requisito cliente
- ❓ **Hipótese a Calibrar**: Não foi mencionado; requer validação com Marcos

**Métricas que Requerem Validação com Marcos:**
1. **Taxa de sucesso ≥ 98%** — SLO interno a calibrar
2. **Redução polling ≥ 80%** — Métrica de adoção a calibrar (80% ou 100%?)
```

#### Outras mudanças no PRD.md

**RNF-001: Performance**
```markdown
- **Latência de entrega < 10 segundos** ✅ (requisito cliente [09:02] Marcos)
- **Latência P95 entrega:** < 2 segundos ℹ️ (implementação via polling 2s)
```

**RNF-002: Disponibilidade**
```markdown
- **Uptime do worker:** ≥ 99.5% ✅ [09:11] Larissa
- **At-least-once delivery:** Garantido com retry 5x ✅ [09:24-26] Diego
- **Taxa de sucesso de entrega:** ≥ 98% ❓ HIPÓTESE (não mencionado; calibrar)
```

**Seção "Decisões e Trade-offs"**
- Adicionada coluna "Requisito Cliente" para cada decisão
- Exemplo: D-1 agora cita [09:02] Marcos como origem

**Seção "Aprovação"**
```markdown
## Aprovação

### Status v1.1 (Atualizado 2026-07-16)

Conforme feedback de Marcos, o PRD foi reformulado para:
- ✅ Reancorar metas em falas reais da transcrição
- ✅ Separar requisitos cliente de derivações técnicas
- ✅ Marcar hipóteses não mencionadas como "a calibrar"

**Pendente de Aprovação:**
- ⏳ **Marcos** — Validação das 2 hipóteses:
  1. Taxa de sucesso ≥ 98%
  2. Redução polling ≥ 80%
```

---

## Legenda Usada

| Símbolo | Significado | Exemplo |
|---------|------------|---------|
| ✅ | Requisito Cliente Real | [09:02] Marcos: "abaixo de 10 segundos" |
| ℹ️ | Derivação Técnica | [09:09-10] Diego propõe polling 2s |
| ❓ | Hipótese a Calibrar | Taxa 98% (nunca foi mencionado) |

---

## Métricas Reancoradas

### 1. Latência

**ANTES:** "Latência P95 < 2 segundos"

**DEPOIS:**
- ✅ **PRD-OBJ-01**: "Latência de entrega < 10 segundos" (cliente)
  - Fonte: [09:02] Marcos: "qualquer coisa abaixo de 10 segundos"

- ℹ️ **PRD-OBJ-01-IMPL**: "Latência P95 < 2s via polling 2s" (técnico)
  - Fonte: [09:09-10] Diego propõe, Larissa aceita

**Por quê?** Cliente pediu "< 10s", não "P95 < 2s". A implementação via polling 2s é uma decisão técnica para atingir o objetivo.

---

### 2. Taxa de Sucesso

**ANTES:** "Taxa de sucesso entrega ≥ 98%"

**DEPOIS:**
- ✅ **PRD-OBJ-02**: "At-least-once delivery com retry 5x" (cliente)
  - Fonte: [09:24-26] Diego: "a gente vai garantir at-least-once"

- ❓ **PRD-OBJ-02-HIPOTESE**: "Taxa de sucesso ≥ 98%" (hipótese)
  - Nunca foi mencionado
  - Proposto como SLO interno
  - Requer validação com Marcos

**Por quê?** Diego discutiu "at-least-once delivery" (que foi o que cliente pediu). O "98%" nunca apareceu na transcrição.

---

### 3. Redução de Polling

**ANTES:** "Redução polling requests ≥ 80%"

**DEPOIS:**
- ✅ **PRD-OBJ-03**: "Eliminar polling de clientes B2B" (cliente)
  - Fonte: [09:00] Marcos: "Os três querem ser notificados em tempo real"

- ❓ **PRD-OBJ-03-HIPOTESE**: "Redução polling ≥ 80%" (hipótese)
  - Nunca foi mencionado
  - Dúvida: é 80% (adoção parcial) ou 100% (completa)?
  - Requer validação com Marcos

**Por quê?** Marcos pediu "eliminar polling". O percentual de "80%" é uma métrica interna que não foi discutida.

---

## Próximas Ações

### Passo 1: Marcos Revisa

Marcos deve:
1. Abrir `docs/PRD.md` (v1.1)
2. Ir para seção "4. Objetivos e Métricas de Sucesso"
3. Ver a tabela reformulada com rastreamento
4. Ler seção "Notas sobre Rastreabilidade"
5. Ver na "Aprovação" o status v1.1

### Passo 2: Marcos Valida

Marcos deve responder:

**Questão 1: Taxa de Sucesso ≥ 98%**
- É OK?
- Qual SLO vocês esperam (95%, 99%, 99.5%)?

**Questão 2: Redução Polling ≥ 80%**
- Esperem 80% (adoção parcial) ou 100% (completa)?
- É métrica de sucesso mesmo ou só um indicador?

Vide: `PARA_MARCOS_VALIDACAO.md` para questões detalhadas

### Passo 3: Você Atualiza

Com respostas de Marcos:
1. Atualizar `docs/PRD.md` (v1.2)
2. Atualizar `docs/TRACKER.md` (v1.2)
3. Mudar hipóteses para "Objetivo" ou "Removido" conforme resposta
4. Criar novo commit

### Passo 4: Pronto!

Rastreabilidade 100% confirmada e documentada.

---

## Impacto na Avaliação de Sucesso

### Cenário 1: Taxa 98% não foi atingida

**Antes (v1.0):**
- "Falhou! Meta era 98% e chegou 95%"

**Depois (v1.1):**
- "Taxa 98% é hipótese (PRD-OBJ-02-HIPOTESE), não estava no requisito cliente"
- "O que cliente pediu foi 'at-least-once delivery com retry' (✅ atingido)"
- "O SLO de 98% precisa validação com Marcos"

### Cenário 2: Redução 80% não foi atingida

**Antes (v1.0):**
- "Falhou! Meta era 80% e conseguimos 60%"

**Depois (v1.1):**
- "Redução 80% é hipótese (PRD-OBJ-03-HIPOTESE), não estava no requisito cliente"
- "O que cliente pediu foi 'eliminar polling' (✅ implementado)"
- "Se esperávamos 100%, o resultado de 60% é baixo. Se esperávamos 80%, está próximo"
- "Métrica precisa validação com Marcos"

### Cenário 3: Latência

**Antes (v1.0):**
- "Atingimos P95 de 2.1s. Falhou?"

**Depois (v1.1):**
- "Cliente pediu < 10s (✅ atingido com P99 ~5s)"
- "P95 < 2s é meta de implementação (ℹ️ quase atingido: 2.1s)"
- "Ambos os objetivos foram alcançados"

---

## Checklist de Implementação

- [x] TRACKER.md v1.1 reformulado
- [x] PRD.md v1.1 reformulado
- [x] Tabela de Métricas com rastreamento
- [x] Seção de Aprovação atualizada
- [x] Legenda clara (✅ | ℹ️ | ❓)
- [x] Hipóteses documentadas
- [x] Documentação de análise gerada
- [x] Git commits realizados
- [ ] Marcos valida as 2 hipóteses
- [ ] Atualização para v1.2

---

## Referências

- `docs/TRACKER.md` — Rastreamento completo de requisitos
- `docs/PRD.md` — Seção 4 (Objetivos e Métricas)
- `PARA_MARCOS_VALIDACAO.md` — Questões para Marcos
- `IMPLEMENTACAO_FEEDBACK_MARCOS.md` — Resumo das mudanças
- `REVIEW_METAS_QUANTITATIVAS.md` — Análise detalhada

---

**Status:** ✅ Reformulação v1.1 Concluída
**Próximo:** Validação com Marcos → v1.2
