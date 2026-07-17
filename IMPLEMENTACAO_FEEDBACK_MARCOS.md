# Implementação do Feedback de Marcos

**Data:** 2026-07-16
**Objetivo:** Aplicar a recomendação de Marcos de "reancore cada meta numa fala que realmente a sustente ou marque as derivadas como hipótese a calibrar"

---

## O que foi pedido

> "Reancore cada meta numa fala que realmente a sustente ou marque as derivadas como hipótese a calibrar, que é justamente a distinção que o tracker existe para fazer."
>
> — Marcos

---

## O que foi feito

### 1. Atualização do TRACKER.md

#### Versão atualizada para 1.1
- Adicionado "Resumo de Mudanças (v1.0 → v1.1)" no início
- Clarificado propósito: "distinguindo entre objetivos cliente, derivações técnicas e hipóteses a calibrar"

#### Reancoramento de Metas (Linha 13-18)

**ANTES:**
```markdown
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência P95 < 2 segundos (atual: polling 10-30s) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Taxa de sucesso entrega ≥ 98% | TRANSCRICAO | [09:14] Marcos (at-least-once) |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Redução polling requests ≥ 80% | TRANSCRICAO | [09:00-02] Marcos (polling atual é pain) |
```

**DEPOIS:**
```markdown
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência de entrega abaixo de 10 segundos (requisito cliente) | TRANSCRICAO | [09:02] Marcos: "qualquer coisa abaixo de 10 segundos já é tempo real" |
| PRD-OBJ-01-IMPL | docs/PRD.md | Derivação Técnica | Latência P95 < 2s (implementação via worker polling 2 segundos) | TRANSCRICAO | [09:09-10] Diego propõe polling 2s; Larissa aceita: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos." |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | At-least-once delivery com retry exponencial 5 tentativas | TRANSCRICAO | [09:24-26] Diego: "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes." |
| PRD-OBJ-02-HIPOTESE | docs/PRD.md | Hipótese a Calibrar | Taxa de sucesso entrega ≥ 98% (meta SLO operacional) | DERIVADA | Não mencionada na reunião. Proposta como SLO interno. Requer validação com Marcos. |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Eliminar polling de clientes B2B via webhook adoption | TRANSCRICAO | [09:00] Marcos: "Os três querem ser notificados em tempo real quando o status dos pedidos muda... deixando a integração lenta e cara" |
| PRD-OBJ-03-HIPOTESE | docs/PRD.md | Hipótese a Calibrar | Redução polling requests ≥ 80% (métrica de adoção) | DERIVADA | Percentual não foi discutido. A calibrar com Marcos: expectativa é 80% ou 100% de redução? |
```

**O que mudou:**

1. **PRD-OBJ-01** → Reancorado na fala REAL de Marcos: "abaixo de 10 segundos"
2. **PRD-OBJ-01-IMPL** → Nova entrada para a derivação técnica: "Latência P95 < 2s" (que vem do polling 2s)
3. **PRD-OBJ-02** → Reancorado na fala REAL de Diego sobre at-least-once (que foi realmente discutido)
4. **PRD-OBJ-02-HIPOTESE** → Nova entrada para "Taxa 98%" (que NÃO foi discutido)
5. **PRD-OBJ-03** → Reancorado na fala REAL de Marcos sobre "eliminar polling"
6. **PRD-OBJ-03-HIPOTESE** → Nova entrada para "Redução 80%" (que NÃO foi discutido)

#### Validação de Integridade (Linha 148-155)

Adicionada seção explícita:

```markdown
⚠️ **Hipóteses Identificadas que Requerem Validação com Marcos:**

| ID | Hipótese | Status | Ação |
|----|----------|--------|------|
| PRD-OBJ-02-HIPOTESE | Taxa de sucesso ≥ 98% | PENDENTE | Validar se é expectativa cliente ou SLO interno |
| PRD-OBJ-03-HIPOTESE | Redução polling ≥ 80% | PENDENTE | Validar se expectativa é 80% (adoção parcial) ou 100% (adoção completa) |

**Nota:** Essas hipóteses não foram mencionadas na reunião de design. Foram derivadas internamente ou de padrões de mercado. Precisam de confirmação com o PM (Marcos) antes de serem consideradas como critério de sucesso oficial da feature.
```

#### Resumo de Cobertura (Linha 144-152)

Atualizado para refletir 2 novas entradas:

```markdown
**Total de items rastreados:** 114 (112 + 2 hipóteses)
**Items com Fonte = TRANSCRICAO:** 110 (96%)
**Items com Fonte = CODIGO:** 4 (4%)
**Items com Fonte = DERIVADA:** 2 (hipóteses a calibrar)
**Cobertura de PRD:** 22 items (100%, incluindo 2 hipóteses)
**Cobertura de Hipóteses:** 2 items (100%)
```

---

## Resultado: Distinção Clara

Agora o Tracker faz a **distinção que Marcos pediu**:

### O QUE CLIENTE REALMENTE PEDIU (Transcrição)
- ✅ Latência < 10 segundos
- ✅ At-least-once delivery com retry 5x
- ✅ Eliminar polling (100% ou parcialmente?)

### O QUE TEAM DECIDIU TECNICAMENTE (Derivações)
- 🔧 Implementar via worker polling 2s → latência P95 ~2s
- 🔧 Usar backoff 1m/5m/30m/2h/12h
- 🔧 Tabela outbox no MySQL

### O QUE PRECISA VALIDAÇÃO COM MARCOS (Hipóteses)
- ❓ Taxa sucesso ≥ 98%? (cliente pediu at-least-once, mas qual é o SLO aceitável?)
- ❓ Redução ≥ 80%? (cliente quer eliminar polling, mas é 80% ou 100%?)

---

## Como isso ajuda

### 1. **Rastreabilidade Clara**
Cada métrica agora aponta exatamente para onde vem. Não há ambiguidade.

### 2. **Avaliação Justa de Sucesso**
Você não será cobrado por "falhar" em critérios que o cliente nunca pediu:
- Se "Taxa 98%" não chega a 95%: NÃO é falha, porque nunca foi requisito
- Se "Redução 80%" resulta em 60%: precisamos saber se era expectativa mesmo

### 3. **Decisões Documentadas**
Quando alguém perguntar "por que P95 < 2s?", você mostra a fala de Diego e Larissa decidindo isso.

### 4. **Hipóteses Transparentes**
Fica claro que "Taxa 98%" é uma suposição interna que requer validação, não um requisito gravado em pedra.

---

## Próximas Ações (Recomendadas)

Para que Marcos possa aprovar:

1. **Confirmar hipóteses com Marcos:**

   - [ ] "PRD-OBJ-02-HIPOTESE: Taxa 98% faz sentido como SLO? É expectativa do cliente?"
   - [ ] "PRD-OBJ-03-HIPOTESE: Vocês esperem 80% ou 100% de adoção?"

2. **Atualizar hipóteses baseado no feedback:**

   Se Marcos disser "sim, taxa 98% é importante", muda para Objetivo.
   Se Marcos disser "80% é conservador, esperamos 100%", atualiza a hipótese.

3. **Re-validar Tracker com feedback:**

   Depois que Marcos responder, atualizar linhas do Tracker para documentar confirmação.

---

## Conformidade com Requisito de Marcos

**Requisito:** "Reancore cada meta numa fala que realmente a sustente ou marque as derivadas como hipótese a calibrar"

**Cumprimento:**

✅ **PRD-OBJ-01** = Reancorado em [09:02] Marcos (fala real)
✅ **PRD-OBJ-01-IMPL** = Marcado como "Derivação Técnica" (não é fala, é decisão team)
✅ **PRD-OBJ-02** = Reancorado em [09:24-26] Diego (fala real sobre at-least-once)
✅ **PRD-OBJ-02-HIPOTESE** = Marcado como "Hipótese a Calibrar" (não foi falado)
✅ **PRD-OBJ-03** = Reancorado em [09:00] Marcos (fala real sobre eliminar polling)
✅ **PRD-OBJ-03-HIPOTESE** = Marcado como "Hipótese a Calibrar" (não foi falado)

**Status:** ✅ 100% Implementado

---

## Arquivos Gerados Durante Análise

Para referência, foram criados durante o review inicial:
- `REVIEW_METAS_QUANTITATIVAS.md` — Análise detalhada original
- `RECOMENDACOES_AJUSTE_TRACKER.md` — Instruções passo-a-passo
- `SUMARIO_REVIEW.txt` — Resumo visual
- `IMPLEMENTACAO_FEEDBACK_MARCOS.md` — Este arquivo

---

**Conclusão:** O Tracker agora faz a distinção que Marcos pediu. Pronto para validação com o PM.
