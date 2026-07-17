# Recomendações de Ajuste no Tracker

**Baseado em:** REVIEW_METAS_QUANTITATIVAS.md
**Data:** 2026-07-16

---

## Resumo

Este documento fornece as **entradas exatas** que devem ser adicionadas ou atualizadas no `docs/TRACKER.md` para resolver as inconsistências encontradas no review.

---

## Alterações Recomendadas no Tracker

### ✏️ Entrada 1: Ajuste PRD-OBJ-01 (Latência)

**ANTES (Linha ~13):**
```markdown
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência P95 < 2 segundos (atual: polling 10-30s) | TRANSCRICAO | [09:02] Marcos |
```

**DEPOIS:**
```markdown
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência de entrega abaixo de 10 segundos (requisito cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01-IMPL | docs/PRD.md | Derivação Técnica | Implementação: worker polling 2s → latência P95 ~2s | DERIVADA | [09:09-10] Diego; Aceito por Larissa [09:10] |
```

**Rationale:**
- A transcrição menciona explicitamente "abaixo de 10 segundos"
- Nenhuma menção a "< 2 segundos" como requisito cliente
- A frequência de polling de 2s é **decisão de implementação**, não objetivo
- Criar entrada separada permite rastrear que essa latência é uma consequência técnica

---

### ✏️ Entrada 2: Ajuste PRD-OBJ-02 (Taxa de Sucesso)

**ANTES (Linha ~14):**
```markdown
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Taxa de sucesso entrega ≥ 98% | TRANSCRICAO | [09:14] Marcos (at-least-once) |
```

**DEPOIS:**
```markdown
| PRD-OBJ-02 | docs/PRD.md | Objetivo | At-least-once delivery com retry exponencial 1m/5m/30m/2h/12h (5 tentativas) | TRANSCRICAO | [09:24-26] Diego; [09:15-17] Diego (backoff) |
| PRD-OBJ-02-HIPOTESE | docs/PRD.md | Hipótese a Calibrar | Taxa de sucesso entrega ≥ 98% (meta SLO operacional) | DERIVADA | Não foi mencionada na transcrição. Requer validação com Marcos se alinhado com expectativa cliente. |
```

**Rationale:**
- Diego **explicitamente discutiu at-least-once**, não taxa percentual de sucesso
- O "98%" nunca aparece na transcrição
- Menciona "99% dos casos" mas em contexto de deduplicação, não SLO
- Criar entrada de hipótese marca que requer calibração com cliente
- Permite rastrear que essa meta foi derivada interna ou padrão de mercado

---

### ✏️ Entrada 3: Ajuste PRD-OBJ-03 (Redução Polling)

**ANTES (Linha ~15):**
```markdown
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Redução polling requests ≥ 80% | TRANSCRICAO | [09:00-02] Marcos (polling atual é pain) |
```

**DEPOIS:**
```markdown
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Eliminar polling de clientes B2B via webhook adoption | TRANSCRICAO | [09:00] Marcos; [09:00-28] Contexto: Atlas, Max, Nova Cargo fazem polling contínuo |
| PRD-OBJ-03-HIPOTESE | docs/PRD.md | Hipótese a Calibrar | Redução ≥ 80% como proxy de adoção | DERIVADA | A calibrar com Marcos: expectativa é 80% de redução (adoção parcial) ou 100% (adoção completa)? |
```

**Rationale:**
- Marcos **mencionou "eliminar" polling**, não "reduzir 80%"
- Nenhuma cifra de percentual foi discutida na reunião
- O PRD documenta ~8.000 req/dia atual, mas sem meta específica de redução
- A meta de "80%" é conservadora, mas precisa validação
- Marca como hipótese para garantir alinhamento antes de considerar sucesso

---

### ➕ Entrada 4: Nova entrada para confirmação

**ADICIONAR (após PRD-OBJ-03-HIPOTESE):**
```markdown
| PRD-VALIDATION-01 | docs/PRD.md | Ação Recomendada | Validar com Marcos: expectativa de taxa sucesso (98%? 99.5%?) | PENDENTE | Requer resposta: é SLO cliente ou interno? |
| PRD-VALIDATION-02 | docs/PRD.md | Ação Recomendada | Validar com Marcos: adoção esperada (80% ou 100% dos 3 clientes?) | PENDENTE | Impacta critério de sucesso da feature |
| PRD-VALIDATION-03 | docs/PRD.md | Ação Recomendada | Clarificar com Marcos: latência 10s é P95, P99 ou máxima? | PENDENTE | Impacta métricas técnicas de monitoramento |
```

**Rationale:**
- Documenta explicitamente que essas questões **precisam de validação antes da Go-Live**
- Impede que feature seja marcada como "sucesso" sem confirmação de Marcos
- Rastreia a ação de validação como parte do processo

---

## Seção de Validação de Integridade (Atualizar)

**ANTES (Linha ~139):**
```markdown
✅ Nenhum item dos documentos foi inventado (100% rastreado)
✅ Nenhum item contradiz a transcrição
✅ Nenhum arquivo mencionado no FDD é inexistente
✅ Todos os requisitos funcionais têm origem na reunião
✅ Todas as decisões técnicas têm context identificável
```

**DEPOIS:**
```markdown
✅ Nenhum item dos documentos foi inventado (100% rastreado)
✅ Nenhum item contradiz a transcrição
✅ Nenhum arquivo mencionado no FDD é inexistente
✅ Todos os requisitos funcionais têm origem na reunião
✅ Todas as decisões técnicas têm context identificável

⚠️ **3 metas quantitativas requerem validação:**
  - PRD-OBJ-01: "< 2s" é derivação técnica de "< 10s" (cliente)
  - PRD-OBJ-02: "98%" não foi discutido, apenas at-least-once
  - PRD-OBJ-03: "80%" não foi discutido, apenas "eliminar polling"

📋 **Ações pendentes antes de Go-Live:**
  - Validar com Marcos expectativa de taxa de sucesso
  - Validar com Marcos se adoção esperada é 80% ou 100%
  - Clarificar se latência 10s é P95, P99 ou max
```

---

## Como Aplicar as Mudanças

### Opção 1: Merge Manual
1. Abrir `docs/TRACKER.md`
2. Localizar as linhas 13, 14, 15 (PRD-OBJ-01, 02, 03)
3. Substituir conforme indicado acima
4. Adicionar as 3 novas entradas de validação
5. Atualizar a seção de integridade

### Opção 2: Script (Se preferir automatizar)
```bash
# Backup
cp docs/TRACKER.md docs/TRACKER.md.backup

# Aplicar mudanças (manual ou via script)
# Recomendado: fazer manualmente pra revisar cada alteração
```

---

## Impacto nas Outras Documentações

### PRD.md (opcional, mas recomendado)

**Adicionar seção ao final da tabela de Métricas (após L127):**

```markdown
### Notas sobre Rastreabilidade das Métricas

| Métrica | Mencionada na Reunião? | Observação |
|---------|----------------------|-----------|
| Latência < 10s | ✅ [09:02] Marcos | Requisito explícito do cliente |
| Latência P95 < 2s | ⚠️ Derivada | Implementação via polling 2s (Diego) |
| Taxa sucesso ≥ 98% | ⚠️ Não mencionada | SLO interno, a calibrar com cliente |
| Redução polling ≥ 80% | ⚠️ Não mencionada | Objetivo é "eliminar", % a calibrar |
| Requisito "abaixo de 10s" | ✅ [09:02] Marcos | Original do cliente (Atlas, Max, Nova) |
```

---

## Questões para Validação com Marcos

Recomenda-se **enviar a Marcos** antes de finalizar o PRD v1.1:

### Pergunta 1: Taxa de Sucesso
> "No PRD, documentei uma meta de 'Taxa de sucesso ≥ 98%' (i.e., 98% dos eventos chegam com sucesso). Essa meta é:
>
> A) Expectativa do cliente Atlas/Max/Nova?
> B) SLO interno que vocês querem garantir?
> C) Padrão de mercado que vocês acham apropriado?
>
> Se for B ou C, que número faria mais sentido (98%, 99%, 99.5%)?"

### Pergunta 2: Redução de Polling
> "Vocês esperam que os 3 clientes principais (Atlas, Max, Nova Cargo) **migrem 100%** para webhooks ou é aceitável que fiquem híbridos (80-90% de tráfego em webhooks)?
>
> Isso impacta como a gente mede sucesso da feature."

### Pergunta 3: Clarificação de Latência
> "Você mencionou 'abaixo de 10 segundos' — isso é:
>
> A) P95 (95% dos eventos em < 10s)?
> B) P99 (99% em < 10s)?
> C) Máximo absoluto (todos em < 10s)?
>
> Isso ajuda a gente calibrar o monitoramento."

---

## Próximas Etapas

1. ✏️ **Aplicar alterações no Tracker** conforme indicado acima
2. 💬 **Enviar 3 questões a Marcos** (obter feedback)
3. 🔄 **Atualizar PRD.md** com seção de rastreabilidade (opcional)
4. ✅ **Re-validar Tracker** (rodar checklist de integridade)
5. 📌 **Marcar como "Validado"** quando Marcos responder

---

## Checklist de Implementação

- [ ] Entradas PRD-OBJ-01, 02, 03 atualizadas no Tracker
- [ ] Entradas de derivação técnica adicionadas (PRD-OBJ-01-IMPL, etc)
- [ ] Entradas de hipótese adicionadas (PRD-OBJ-02-HIPOTESE, 03-HIPOTESE)
- [ ] Ações de validação documentadas (PRD-VALIDATION-01, 02, 03)
- [ ] Questões enviadas a Marcos
- [ ] Feedback de Marcos incorporado
- [ ] PRD.md atualizado (seção de rastreabilidade)
- [ ] Tracker re-validado (integridade)
- [ ] Commit com mensagem clara sobre mudanças

---

**Status:** 🔴 PENDENTE
**Responsável:** [Seu Nome]
**Data Recomendada:** Antes de finalizar PRD v1.1

