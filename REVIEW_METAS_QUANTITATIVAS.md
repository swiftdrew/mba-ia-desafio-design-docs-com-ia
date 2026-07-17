# Review: Validação de Metas Quantitativas do PRD contra Transcrição

**Data do Review:** 2026-07-16
**Revisor:** Claude Code
**Objetivo:** Verificar se cada meta quantitativa do PRD está realmente sustentada por fala específica na transcrição ou se é derivação/hipótese que deve ser marcada como tal no Tracker.

---

## Sumário Executivo

Das **4 metas quantitativas principais** listadas no PRD (seção 4, tabela Métricas de Sucesso):

| Meta | Registrada no Tracker | Sustentada por Fala | Status |
|------|----------------------|-------------------|--------|
| **Latência P95 < 2s** | ✅ PRD-OBJ-01 | ⚠️ **DERIVADA** | 🔴 INCONSISTENTE |
| **Taxa de sucesso ≥ 98%** | ✅ PRD-OBJ-02 | ⚠️ **DERIVADA** | 🔴 INCONSISTENTE |
| **Redução polling ≥ 80%** | ✅ PRD-OBJ-03 | ⚠️ **AMBÍGUA** | 🟡 PARCIAL |
| **Latência P99 < 10s** | ✅ PRD.md L353 | ✅ **EXPLÍCITA** | 🟢 OK |

---

## Análise Detalhada

### 1. **Latência P95 < 2 segundos** (PRD L121, Tabela Métricas)

**Registrada no Tracker como:**
- ID: `PRD-OBJ-01`
- Conteúdo: "Latência P95 < 2 segundos"
- Fonte: `TRANSCRICAO | [09:02] Marcos`

**O que a Transcrição realmente diz:**

```
[09:02] Marcos: Eu perguntei especificamente isso. Pra eles, qualquer coisa
abaixo de 10 segundos já é "tempo real". O importante é que não fique
pendurado e eles tenham que ficar atualizando manualmente.
```

**Análise:**
- ✅ Marcos **explicitamente citou "abaixo de 10 segundos"** como requisito do cliente
- ❌ **Nenhuma menção a 2 segundos** na transcrição
- A meta de **2 segundos aparece de forma derivada:**
  - Diego propõe: `[09:09]` "Polling em loop. A cada 2 segundos, busca os eventos..."
  - Larissa aceita: `[09:10]` "2 segundos atende o requisito de 'abaixo de 10 segundos'"
  - Larissa conclui: `[09:10]` "A latência mínima vai ser 2 segundos no pior caso. Aceitamos."

**Conclusão:** ⚠️ **DERIVADA** — A meta de "P95 < 2s" é uma *implementação técnica* (frequência de polling), não uma meta de negócio extraída da reunião. O requisito real é **"abaixo de 10 segundos"**.

**Recomendação:**
- ✏️ No **Tracker**, mudar `PRD-OBJ-01` para:
  - **Conteúdo:** "Latência de entrega abaixo de 10 segundos (requisito cliente)"
  - **Tipo:** Objetivo
  - **Fonte:** `TRANSCRICAO | [09:02] Marcos`

- 📋 Criar nova entrada derivada no Tracker:
  - **ID:** `PRD-IMPL-P95-2s`
  - **Conteúdo:** "Latência P95 < 2s (implementação via worker polling 2s)"
  - **Tipo:** Derivação técnica / Hipótese a Calibrar
  - **Justificativa:** Implementação técnica proposta por Diego e aceita pela team para atingir o objetivo de < 10s

---

### 2. **Taxa de Sucesso Entrega ≥ 98%** (PRD L122, Tabela Métricas)

**Registrada no Tracker como:**
- ID: `PRD-OBJ-02`
- Conteúdo: "Taxa de sucesso entrega ≥ 98%"
- Fonte: `TRANSCRICAO | [09:14] Marcos (at-least-once)`

**O que a Transcrição realmente diz:**

```
[09:24] Diego: Voltando à entrega: a gente vai garantir at-least-once.
Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem
que estar preparado.

[09:25] Sofia: Isso joga responsabilidade pro cliente.

[09:25] Diego: Joga, mas é o padrão de mercado. Stripe faz assim, GitHub
faz assim. Garantir exactly-once exigiria coordenação dos dois lados e
fica muito mais complexo. At-least-once com event_id resolve 99% dos casos.
```

**Contexto [09:14-17]:**
```
[09:14] Larissa: Beleza. Vamos pra retry. Se o cliente tá offline,
o que a gente faz?

[09:15] Diego: Backoff exponencial. Tenta de novo depois de algum tempo,
vai aumentando o intervalo, e depois de um teto de tentativas considera
falha permanente e move pra DLQ.

[09:15] Bruno: Quantas tentativas?

[09:15] Diego: Eu sugiro 5. Algumas pessoas defendem retry indefinido com
backoff, mas isso traz o problema de evento ficar pendurado pra sempre se
o cliente sumiu. Cinco já dá pra cobrir uma janela de até 12 ou 24 horas.
```

**Análise:**
- ✅ Diego menciona **"at-least-once"** (garantia de entrega mínima)
- ✅ Diego menciona **"resolve 99% dos casos"** (no contexto de deduplicação)
- ❌ **Nenhuma menção explícita a "98% de taxa de sucesso"**
- ❌ **Ninguém discute ou propõe a meta específica de 98%**
- A discussão foi sobre **garantias de entrega** (at-least-once), não sobre **taxa de falha aceitável**

**Conclusão:** ⚠️ **DERIVADA / HIPÓTESE** — O "98%" aparenta ser uma meta operacional extraída (possivelmente do padrão de SLO do setor), mas não foi discutido ou justificado na reunião. Diego menciona "99% dos casos" em contexto diferente (dedup), não como SLO de entrega.

**Recomendação:**
- ✏️ No **Tracker**, mudar `PRD-OBJ-02` para:
  - **Conteúdo:** "At-least-once delivery (garantia de entrega mínima com retry 5x backoff)"
  - **Tipo:** Objetivo
  - **Fonte:** `TRANSCRICAO | [09:24-26] Diego`

- 📋 Criar nova entrada de hipótese:
  - **ID:** `PRD-HIPOTESE-TAXA-98`
  - **Conteúdo:** "Taxa de sucesso entrega ≥ 98% (meta operacional a calibrar)"
  - **Tipo:** Hipótese a Calibrar
  - **Justificativa:** Não foi discutida na reunião. Proposta como meta de qualidade do SLO. Requer validação com Marcos (se é expectativa do cliente) ou data-driven após v1.

---

### 3. **Redução Polling Requests ≥ 80%** (PRD L123, Tabela Métricas)

**Registrada no Tracker como:**
- ID: `PRD-OBJ-03`
- Conteúdo: "Redução polling requests ≥ 80%"
- Fonte: `TRANSCRICAO | [09:00-02] Marcos (polling atual é pain)`

**O que a Transcrição realmente diz:**

```
[09:00] Marcos: Bom dia. Então, a gente recebeu na semana passada um
pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição
e Nova Cargo. Os três querem ser notificados em tempo real quando o
status dos pedidos deles muda na nossa plataforma. Hoje eles ficam
batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa,
e isso tá deixando a integração lenta e cara pra eles.
```

**Contexto do problema (PRD L12-28):**
```
Clientes B2B usam polling síncrono em `GET /orders`:
- **Atlas Comercial:** Faz request a cada 10 segundos pra ~100 pedidos ativos
  (1.000 req/dia)
- **MaxDistribuição:** Faz request a cada 30 segundos pra ~50 pedidos
  (2.880 req/dia)
- **Nova Cargo:** Faz request a cada 20 segundos pra ~80 pedidos
  (4.320 req/dia)

**Total:** ~8.000 requisições/dia extras de polling (sobrecarga)
```

**Análise:**
- ✅ Marcos **explicitamente menciona que polling é problema** ("pain point")
- ✅ PRD documenta ~8.000 req/dia de polling atual
- ❌ **Marcos NUNCA cita "80%" como meta**
- ❌ **Ninguém propõe ou discute a cifra de 80%**
- O objetivo é **"eliminar polling"**, não **"reduzir 80%"**

**Menção de "80%" no PRD (L123):** Aparece apenas na tabela de métricas, sem rastreamento na reunião.

**Conclusão:** 🟡 **AMBÍGUA** — O objetivo geral ("eliminar polling") é claro, mas a meta específica de "≥ 80%" é **derivação não justificada**. Poderia ser:
- Conservadora (se alguns clientes mantêm polling híbrido)
- Ou otimista (se espera 100% de adoção de webhooks)

**Recomendação:**
- ✏️ No **Tracker**, mudar `PRD-OBJ-03` para:
  - **Conteúdo:** "Eliminar polling de clientes B2B via webhook adoption"
  - **Tipo:** Objetivo
  - **Fonte:** `TRANSCRICAO | [09:00] Marcos`

- 📋 Criar nova entrada de hipótese:
  - **ID:** `PRD-HIPOTESE-POLLING-80`
  - **Conteúdo:** "Redução ≥ 80% de polling requests como proxy de adoção"
  - **Tipo:** Hipótese a Calibrar
  - **Justificativa:** Objetivo é "eliminar" polling, mas meta de 80% é conservadora. Calibrar com Marcos: é adoção parcial ou total esperada? Impacta avaliação de sucesso.

---

### 4. **Latência P99 < 10 segundos** (PRD L353, seção RNF-001)

**Conteúdo:**
```
### RNF-001: Performance
- **Latência P99 entrega:** < 10 segundos
```

**O que a Transcrição diz:**

```
[09:02] Marcos: Eu perguntei especificamente isso. Pra eles, qualquer coisa
abaixo de 10 segundos já é "tempo real".
```

**Análise:**
- ✅ **Explicitamente mencionado por Marcos como requisito do cliente**
- ✅ **Claramente rastreado**: "abaixo de 10 segundos"
- ✓ Não há conflito ou ambiguidade

**Conclusão:** 🟢 **OK** — Esta meta está corretamente fundamentada.

---

## Tabela de Ajustes Recomendados no Tracker

| ID Atual | Item | Conteúdo Atual | Ajuste Recomendado | Tipo | Justificativa |
|----------|------|----------------|-------------------|------|---------------|
| PRD-OBJ-01 | Latência P95 | "< 2 segundos" | "< 10 segundos (Marcos, [09:02])" | Objetivo | Derivada técnica: 2s é do polling, 10s é do cliente |
| — | **NOVO** | — | "P95 < 2s (implementação via worker polling)" | Derivação Técnica | Implementação técnica de Diego |
| PRD-OBJ-02 | Taxa Sucesso | "≥ 98%" | "At-least-once delivery com retry 5x" | Objetivo | 98% não foi discutido, apenas at-least-once |
| — | **NOVO** | — | "Taxa sucesso ≥ 98% (meta a calibrar)" | Hipótese | Não discutida na reunião, requer validação |
| PRD-OBJ-03 | Redução Polling | "≥ 80%" | "Eliminar polling via webhook adoption" | Objetivo | 80% não foi discutido, apenas "eliminar" |
| — | **NOVO** | — | "Redução ≥ 80% (hipótese de adoção parcial)" | Hipótese | Calibrar com Marcos se é 80% ou 100% |

---

## Ações Recomendadas

### 1. **Atualizar o Tracker** (docs/TRACKER.md)

Marcar as 3 metas problemáticas como **"Derivação Técnica"** ou **"Hipótese a Calibrar"** conforme acima.

```markdown
| PRD-OBJ-01-BASE | docs/PRD.md | Objetivo | Latência de entrega < 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01-IMPL | docs/PRD.md | Derivação Técnica | Implementar via worker polling 2s → latência P95 ~2s | DECISAO_TECNICA | [09:09-10] Diego |
| PRD-OBJ-02-BASE | docs/PRD.md | Objetivo | At-least-once delivery com retry exponencial | TRANSCRICAO | [09:24-26] Diego |
| PRD-OBJ-02-HIPOTESE | docs/PRD.md | Hipótese a Calibrar | Taxa de sucesso ≥ 98% (SLO operacional) | DERIVADA | Não mencionado na reunião, requer validation |
| PRD-OBJ-03-BASE | docs/PRD.md | Objetivo | Eliminar polling de clientes B2B | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03-HIPOTESE | docs/PRD.md | Hipótese a Calibrar | Redução ≥ 80% como proxy de adoção | DERIVADA | Calibrar com Marcos: expectativa é 80% ou 100% |
```

### 2. **Validar com Marcos**

Enviar a Marcos as seguintes questões **antes da implementação**:

**Pergunta 1:** "Você espera que **80% ou 100%** dos 3 clientes principais migrem completamente para webhooks?"
- Se 100%: mudar métrica para "Redução ≥ 95%"
- Se 80%: confirmar que é meta realista

**Pergunta 2:** "O cliente mencionou expectativa de 'taxa de sucesso' (i.e., quantos eventos chegam)? A meta de 98% é interna ou externa?"
- Se externa (cliente espera): documentar
- Se interna (SLO): justificar por quê 98% vs 99.5% vs outro valor

**Pergunta 3:** "Quando você disse 'abaixo de 10 segundos' você quis dizer P95, P99 ou max?"
- Para documentar com precisão no PRD

### 3. **Atualizar o PRD** (opcional, mas recomendado)

Adicionar seção de **"Observações sobre Métricas"**:

```markdown
### Notas sobre Métricas de Sucesso

- **Latência de entrega < 10s:** Requisito cliente (Marcos, reunião)
- **Latência P95 < 2s:** Implementação técnica via worker polling 2s (pode ser calibrada após MVP)
- **Taxa sucesso ≥ 98%:** Meta operacional de SLO (a validar com Marcos se alinhado com expectativa cliente)
- **Redução polling ≥ 80%:** A calibrar com Marcos — expectativa é adoção total (100%) ou parcial?
```

### 4. **Decisão: Como proceder**

**Opção A (Conservadora):**
- Fazer commit ajustando apenas o Tracker (sem mudar PRD.md)
- Enviar questões de validação a Marcos
- Atualizar PRD.md após feedback

**Opção B (Agressiva):**
- Atualizar PRD.md imediatamente com observações
- Comentar o Tracker com "HIPÓTESE" bem visível
- Sincronizar com Marcos antes de validação final do PRD

---

## Conclusão

O projeto tem **documentação de design excelente**, mas as **metas quantitativas têm inconsistências importantes**:

| Métrica | Status |
|---------|--------|
| Latência P95 < 2s | ⚠️ Derivada técnica, não objetivo |
| Taxa sucesso ≥ 98% | ⚠️ Hipótese não mencionada |
| Redução polling ≥ 80% | ⚠️ Ambígua (80% vs 100%?) |
| Latência < 10s | ✅ Correto |

**Recomendação final:** Usar esta análise como **insumo para validação com Marcos antes de finalizar o PRD v1.1**. A distinção entre "objetivo do cliente" (transcrição) e "meta implementação" (derivada) é exatamente o que o Tracker existe para fazer.

---

**Próximas Ações:**
- [ ] Revisar com Marcos as 3 questões acima
- [ ] Atualizar docs/TRACKER.md com entradas de "Derivação" e "Hipótese"
- [ ] Atualizar PRD.md seção 4 com notas de rastreabilidade
- [ ] Re-validar integridade do Tracker após mudanças
