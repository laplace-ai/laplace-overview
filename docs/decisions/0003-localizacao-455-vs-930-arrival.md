# 0003 — localizacao_atual (455) antecipa chegada antes da confirmação da 930

**Status**: Confirmed — investigation complete, fix pending
**Date**: 2026-04-01
**Context**: Reported during field visit — collector showing cargo not yet physically at the unit

---

## Problem

The `localizacao_atual` / `localizacao_atual_code` field in `base_455_cleaned` updates to the destination unit **when the manifest is emitted or the cargo departs**, not when the cargo physically arrives. This causes the collector (coletor) to show CTRCs for verification at units where the cargo hasn't arrived yet.

The `base_930_cleaned` events (specifically codes 93 "CHEGADA NA UNIDADE DE DESTINO" and 94 "CHEGADA NA UNIDADE") record **actual physical arrival** at units with timestamps.

---

## Investigation (2026-04-01)

### Conceitos-chave

- **`localizacao_atual_code` (455)**: campo que indica "onde a carga está" segundo o sistema SSW. Atualiza com base no manifesto/roteamento, não na chegada física.
- **Eventos 93/94 (930)**: registram a **chegada física real** de uma carga numa unidade. São o ground truth.
- **"Sem confirmação da 930"**: a 455 diz que o CTRC está na unidade X, mas a 930 não tem nenhum evento de chegada (93/94) nessa unidade para esse CTRC.

### Análise 1: CTRCs onde a 455 mostra uma localização sem confirmação de chegada pela 930

Pergunta: dos CTRCs ativos (não entregues), quantos têm `localizacao_atual_code = X` na 455 mas **nenhum evento de chegada (93/94) na unidade X** na 930?

| Metric | Value |
|--------|-------|
| Total de CTRCs ativos | 48,029 |
| Sem confirmação de chegada pela 930 na unidade indicada pela 455 | **13,426 (28.0%)** |

Esses 13,426 CTRCs estão em uma localização segundo a 455 que a 930 nunca confirmou. Para entender o porquê, analisamos o **último evento registrado na 930** para cada um:

| Último evento na 930 | Qtd | O que significa |
|-----------------------|----:|-----------------|
| CT-E EMITIDO (97) | 4,224 | CT-e acabou de ser emitido. A 455 já mostra a unidade emissora como localização. A 930 ainda não registrou nenhum evento de movimentação. Na prática a carga pode estar fisicamente na unidade emissora — o problema aqui é menor. |
| SAIDA DA UNIDADE (95) de outra unidade | 3,816 | **Este é o caso mais problemático.** A carga saiu da unidade A (registrado pela 930), mas a 455 já mostra `localizacao_atual = unidade B` (o destino do manifesto). A carga está em trânsito A→B, mas aparece como "no armazém de B". |
| CHEGADA (93/94) em unidade diferente | 633 | A 930 confirmou chegada na unidade A, mas a 455 já avançou para a unidade B (próximo destino na rota). A carga está fisicamente em A, completou aquela etapa, e seguiu ou vai seguir viagem para B — mas a 455 antecipou e já mostra B. Mesmo fenômeno, numa etapa intermediária da viagem. |
| Outros eventos | ~4,753 | Diversos: aguardando agendamento, saída para entrega, etc. São casos onde a 930 tem algum evento mas não de chegada naquela unidade específica. |

**Distribuição por unidade (top 5):**

| Unidade | Total CTRCs | Sem confirmação 930 | % |
|---------|------------:|--------------------:|--:|
| BAR | 6,727 | 2,476 | 36.8% |
| VIX | 5,848 | 1,920 | 32.8% |
| SAO | 4,543 | 1,646 | 36.2% |
| BHZ | 4,055 | 1,226 | 30.2% |
| RIO | 2,911 | 870 | 29.9% |

### Análise 2: Caso inverso — 930 confirma chegada mas 455 mostra outra localização

Pergunta inversa: partindo dos **eventos de chegada da 930**, quantos CTRCs têm uma `localizacao_atual_code` diferente na 455?

| Metric | Value |
|--------|-------|
| Chegadas recentes na 930 (desde 20/mar) | 72,179 |
| Localização na 455 diverge da chegada 930 | 2,593 (3.6%) |

Este caso é raro. Quando a 930 registra chegada, a 455 geralmente já atualizou (ou até já avançou pro próximo destino). Confirma que o problema é **unidirecional**: a 455 está à frente da 930 em 28% dos casos, o inverso acontece em apenas 3.6%.

### Gap temporal: manifesto (455) → chegada real (930)

Para CTRCs onde ambos os dados existem (455 tem data de manifesto E 930 tem chegada na mesma unidade):

| Metric | Value |
|--------|-------|
| Gap mediano | 0 dias (chega no mesmo dia) |
| Chega no mesmo dia | 48% |
| Chega no dia seguinte | 45% |
| 2+ dias | 3% |

A carga tipicamente chega 0-1 dia após a emissão do manifesto. Ou seja, o CTRC aparece no coletor até 1 dia antes da carga chegar fisicamente.

### Impacto real no coletor

| Metric | Value |
|--------|-------|
| Total de itens pendentes no coletor | 5,353 |
| Itens onde a carga **não chegou fisicamente** | 131 (2.4%) |

Os 131 itens são CTRCs que aparecem na fila de verificação de uma unidade, mas a 930 não registrou chegada nessa unidade — a carga provavelmente está em trânsito.

### Exemplo concreto (2026-04-01)

`RIO600896-8`:
- **455**: `localizacao_atual_code = SAO`, manifesto destino = SAO, data manifesto = 01/04
- **930**: último evento = **SAIDA DA UNIDADE (95) no RIO** em 01/04
- **Realidade**: carga saiu do RIO com destino a SAO, está em trânsito, mas aparece no coletor de SAO como se já estivesse no armazém

---

## Proposed fix

### Approach: confirmed location via 930

Add a `localizacao_confirmada_code` field (or adjust `localizacao_atual_code` derivation) that only reflects a unit when the 930 has an arrival event (cod_ocor IN 93, 94) for that CTRC at that unit.

**Where to implement**: `laplace-data-warehouse` ETL pipeline, since it already processes both base 455 and base 930.

**Logic**:
1. Keep `localizacao_atual_code` from 455 as-is (it still has value for tracking intent/manifest destination)
2. Add `localizacao_confirmada_code`: the most recent unit where 930 has an arrival event (93/94)
3. Downstream consumers (loss_verification, collector API) should use `localizacao_confirmada_code` instead of `localizacao_atual_code` when determining which unit a CTRC is physically at

**Alternative** (simpler, less disruptive):
- In the loss prediction service or API, when building the collector queue, cross-check: only include a CTRC at unit X if 930 has an arrival event at X. If not, exclude it from that unit's queue until arrival is confirmed.

---

## Open questions

1. Should we keep showing CTRCs in the collector with a "em trânsito" badge rather than hiding them entirely?
2. Is `localizacao_confirmada_code` worth adding as a persistent column, or should we compute it on-the-fly in the API?
3. For the emission unit (where `localizacao_atual_code` = `unidade_emissora`), the cargo IS there — no 930 arrival needed. Need to handle this edge case.
