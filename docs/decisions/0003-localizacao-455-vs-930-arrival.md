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

### Hypothesis 1: 455 ahead of 930 (location set before arrival confirmed)

**Active CTRCs (non-delivered), all periods:**

| Metric | Value |
|--------|-------|
| Total active CTRCs | 48,029 |
| 455 location with NO 930 arrival event at that unit | 13,426 (28.0%) |

**Breakdown by unit (top 5):**

| Unit | Total | No 930 arrival | % |
|------|------:|---------------:|--:|
| BAR | 6,727 | 2,476 | 36.8% |
| VIX | 5,848 | 1,920 | 32.8% |
| SAO | 4,543 | 1,646 | 36.2% |
| BHZ | 4,055 | 1,226 | 30.2% |
| RIO | 2,911 | 870 | 29.9% |

**What the last 930 event says for these phantom arrivals:**

| Last 930 event | Count | Meaning |
|----------------|------:|---------|
| CT-E EMITIDO (97) | 4,224 | Just emitted, hasn't moved |
| SAIDA DA UNIDADE (95) | 3,816 | Departed origin, in transit |
| CHEGADA NA UNIDADE (93/94) at different unit | 633 | Arrived at intermediate, 455 already shows next |

### Hypothesis 2: 930 ahead of 455 (reversed)

| Metric | Value |
|--------|-------|
| Recent 930 arrivals (since Mar 20) | 72,179 |
| Location mismatch (930 arrived, 455 shows different) | 2,593 (3.6%) |

**Conclusion**: The problem is strongly unidirectional — 455 is ahead of 930 in 28% of cases, while the reverse is only 3.6%.

### Time gap: manifest → actual arrival (930)

| Metric | Value |
|--------|-------|
| Median gap | 0 days (same day) |
| Same day arrival | 48% |
| Next day arrival | 45% |
| 2+ days | 3% |

### Collector impact

| Metric | Value |
|--------|-------|
| Total pending in collector | 5,353 |
| Pending with cargo NOT physically at unit | 131 (2.4%) |

### Concrete example (2026-04-01)

`RIO600896-8`: 455 says `localizacao_atual_code = SAO`, manifest destination = SAO, but the last 930 event is **SAIDA DA UNIDADE (95) at RIO** — cargo is in transit RIO→SAO but appears in SAO's collector queue.

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
