# Para Marcos — Validação de Hipóteses

**De:** Equipe de Design
**Para:** Marcos (Product Manager)
**Data:** 2026-07-16
**Assunto:** Validação de 2 hipóteses no Tracker antes de Go-Live
**Urgência:** ANTES de finalizar PRD v1.1

---

## Contexto

Durante a análise de rastreabilidade do Tracker (v1.1), identificamos 2 metas quantitativas que **não foram explicitamente mencionadas na reunião de design**, mas foram registradas no PRD.

Conforme sua orientação: **"reancore cada meta numa fala que realmente a sustente ou marque as derivadas como hipótese a calibrar"** — essas 2 metas foram marcadas como **HIPÓTESES A CALIBRAR**.

Agora precisamos de seu feedback para confirmar se essas metas refletem mesmo as expectativas de vocês.

---

## Hipótese #1: Taxa de Sucesso ≥ 98%

**Localização no Tracker:** `PRD-OBJ-02-HIPOTESE`

### O que foi discutido na reunião

Diego falou sobre "**at-least-once delivery**":

> [09:24-26] Diego: "A gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."

**NUNCA foi mencionado um "98% de taxa de sucesso".**

### O que está no PRD agora

Meta: "Taxa de sucesso de entrega ≥ 98%"

### Sua pergunta

**Confirmação necessária:**

- [ ] Sim, vocês esperam que 98% dos webhooks cheguem com sucesso
  - Se sim: qual é a justificativa? (SLO padrão de mercado, expectativa do cliente, outra?)

- [ ] Não, esse número não é acurado. O correto seria:
  - 95%
  - 99%
  - 99.5%
  - Outro: ___________

- [ ] Na verdade, a métrica importante é "at-least-once delivery com retry 5x" e não um percentual específico
  - Se sim: podemos remover a meta de 98% e deixar apenas "at-least-once"

---

## Hipótese #2: Redução de Polling ≥ 80%

**Localização no Tracker:** `PRD-OBJ-03-HIPOTESE`

### O que foi discutido na reunião

Marcos falou sobre **"eliminar polling"**:

> [09:00] Marcos: "Os três querem ser notificados em tempo real quando o status dos pedidos muda... deixando a integração lenta e cara"

**NUNCA foi mencionado um "≥ 80%".**

### O que está no PRD agora

Meta: "Redução de polling requests ≥ 80%"

### O que isso significa

Se a baseline é ~8.000 requisições/dia de polling:
- **80% de redução** = 1.600 req/dia (alguns clientes em híbrido)
- **100% de redução** = 0 req/dia (todos migraram 100%)

### Sua pergunta

**Confirmação necessária:**

- [ ] **80% é a meta correta.**
  - Vocês aceitam que alguns clientes fiquem em híbrido (polling + webhooks)?

- [ ] **Vocês esperam 100% de redução.**
  - Todos os 3 clientes devem migrar completamente para webhooks até o fim de novembro?

- [ ] **A métrica não deveria ser percentual.**
  - O importante é que Atlas, Max e Nova Cargo todos estejam usando webhooks (sim/não), não o percentual de redução

---

## Por que isso importa

Essas 2 metas vão ser usadas para **avaliar sucesso da feature**:

### Cenário A: Se não forem validadas
```
Você implementa a feature.
A feature é entregue.
Depois Marcos diz: "Mas vocês não bateram os 98%!"
Você responde: "Mas 98% nunca foi discutido na reunião..."
```

### Cenário B: Se forem validadas agora
```
Você implementa a feature.
A feature é entregue.
Marcos: "Perfeito, exatamente como alinhamos."
```

---

## Respostas Esperadas

**Para Marcos:** Basta responder:

**Hipótese #1 (Taxa de Sucesso):**
> Minha resposta: [ ] 98% [ ] 95% [ ] 99% [ ] 99.5% [ ] Outro: _____ [ ] Usar apenas "at-least-once"

**Hipótese #2 (Redução de Polling):**
> Minha resposta: [ ] 80% [ ] 100% [ ] Usar métrica de "quantos clientes usando"

---

## Próximas Ações

1. **Você responde** essa validação por aqui ou em um Slack/email
2. **A gente atualiza** o Tracker com as respostas (v1.2)
3. **Pronto!** Tracker com 100% de rastreabilidade confirmada

---

## Referência: O que já foi validado

✅ **PRD-OBJ-01:** Latência < 10 segundos
→ Confirmado: [09:02] Marcos: "qualquer coisa abaixo de 10 segundos já é tempo real"

✅ **PRD-OBJ-02 (parcial):** At-least-once delivery com retry 5x
→ Confirmado: [09:24-26] Diego

❓ **PRD-OBJ-02-HIPOTESE:** Taxa de sucesso ≥ 98%
→ **PENDENTE SUA VALIDAÇÃO**

✅ **PRD-OBJ-03 (parcial):** Eliminar polling
→ Confirmado: [09:00] Marcos

❓ **PRD-OBJ-03-HIPOTESE:** Redução ≥ 80%
→ **PENDENTE SUA VALIDAÇÃO**

---

## Documentação

Você pode encontrar detalhes em:
- `docs/TRACKER.md` (v1.1) — Tracker atualizado
- `IMPLEMENTACAO_FEEDBACK_MARCOS.md` — Resumo das mudanças
- `REVIEW_METAS_QUANTITATIVAS.md` — Análise detalhada original

---

**Aguardando suas respostas!**

Commit: `77eab0d` — docs: reancore metas quantitativas conforme feedback de Marcos
