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

### Decomposição refinada dos 28% sem confirmação de chegada

Os 13.426 CTRCs sem evento 93/94 na unidade da 455 se dividem em dois grupos:

**Grupo 1 — Último evento da 930 na MESMA unidade que a 455 mostra (9.316 / 69%)**

A 930 tem atividade nessa unidade, mas não um evento formal de chegada (93/94). A maioria está correta:

| Evento | Qtd | Análise |
|--------|----:|---------|
| CT-E EMITIDO (97) | 3.935 | Carga emitida na unidade — está lá, sem "chegada" porque originou ali |
| SAIDA DA UNIDADE (95) | 1.937 | Carga **já saiu** dessa unidade — 455 desatualizada |
| AGENDAMENTO (12) | 449 | Processos operacionais na unidade — carga está lá |
| SAIDA PARA ENTREGA (87) | 433 | Carga saiu pra entrega dessa unidade |
| RAIO X (40) | 316 | Carga manuseada na unidade — está lá |
| Outros operacionais | ~2.246 | Devolução, agendamento, paletização, etc. |

**Grupo 2 — Último evento da 930 em OUTRA unidade (4.144 / 31%)**

A carga definitivamente **não está** na unidade que a 455 indica:

| Evento | Qtd | Análise |
|--------|----:|---------|
| SAIDA DA UNIDADE (95) | 1.648 | Saiu de outra unidade, em trânsito pro destino da 455 |
| CHEGADA (93/94) em outra | 829 | Chegou numa intermediária, 455 já avançou |
| AGENDAMENTO (12) em outra | 359 | Processos em outra unidade |
| CT-E EMITIDO (97) em outra | 262 | Emitido em outra unidade, 455 já mostra destino |
| Outros | ~1.046 | Diversos eventos em unidades diferentes |

### Edge case: eventos operacionais sem chegada formal (93/94) nem emissão (97)

Para CTRCs recentes (últimos 4 dias, 17.825 ativos):

| Cenário | Qtd | % |
|---------|----:|--:|
| Tem evento 93/94/97 na unidade da 455 | 16.359 | 91.8% |
| **Tem eventos na unidade, mas NÃO 93/94/97** | **323** | **1.8%** |
| Nenhum evento na unidade | 1.143 | 6.4% |

Os 323 CTRCs (1.8%) têm apenas eventos operacionais na unidade (raio-x, agendamento, roteirização) sem evento formal de chegada/emissão. Exemplos:

| CTRC | Emissora | Loc 455 | Eventos na unidade (930) |
|------|----------|---------|--------------------------|
| ARA978446-2 | ARA | SAO | RAIO X (00:05) → SAIDA DA UNIDADE (00:21) |
| BAR361809-9 | BAR | CPQ | AGENDAMENTO (16:21) → PALETIZACAO (14:11) |
| BAR362700-4 | BAR | VIX | AGENDAMENTO → ENTREGA AGENDADA → ROTEIRIZADA → SAIDA ENTREGA |
| CPQ890150-3 | CPQ | BHZ | AGENDAMENTO (11:47) |
| EMS889977-1 | EMS | VIX | AGENDAMENTO → ENTREGA AGENDADA |
| GYN045306-4 | GYN | BHZ | NF LIBERADA → NF RECEBIDA AGENDAMENTO |
| ITL486765-3 | ITL | ITR | PALETIZACAO (17:35) |

São falhas de registro operacional: a carga passou pela unidade sem evento formal de chegada. **Marginal (1.8%) — aceita-se o delay.**

---

## Decisão

### Por que manter a 455 como fonte primária

1. A 455 cobre a **grande maioria** dos casos corretamente (91.8% com confirmação da 930)
2. O caso inverso (930 à frente da 455) é raro (3.6%) — não justifica inverter a lógica
3. A 455 fornece `localizacao_atual_code` já extraído e pronto — a 930 exigiria computar a última unidade a partir da sequência de eventos
4. Todos os consumidores downstream (API, coletor, dashboard) já usam `localizacao_atual_code` da 455

### Fix escolhido: double-check com 930 no loss-prediction

**Onde**: `laplace-service-loss-prediction/src/verification.py`

**Lógica**: manter o fluxo atual, mas antes de inserir na `loss_verification`, confirmar na 930 que existe evento de **chegada (93/94)** ou **emissão (97)** naquela unidade para o CTRC.

- A função `get_930_arrival_time()` **já é chamada** para cada CTRC antes do INSERT e retorna `None` quando não há confirmação
- Fix: se `arrived_at is None` → skip INSERT (não cria registro)
- CTRCs ignorados serão capturados no próximo ciclo quando a 930 registrar o evento de chegada

**Descartado**: adicionar coluna `localizacao_confirmada_code` na `base_455_cleaned` — a 455 é espelho do sistema do cliente (DELETE+INSERT por período), colunas derivadas seriam perdidas a cada sync.

### Edge cases aceitos

- **1.8%** (323 CTRCs) com eventos operacionais mas sem 93/94/97: não serão criados na `loss_verification`. Delay aceito — a maioria já saiu da unidade.
- **3.6%** onde 930 está à frente da 455: delay de 1 ciclo (~5min) até a 455 atualizar. Impacto desprezível.

### Backfill

Registros existentes na `loss_verification` com `verification_status = 'pending'` e `arrived_at IS NULL` são cargas inseridas sem confirmação da 930. Avaliar remoção via script.

---

## Referências

- Issue DW: laplace-ai/laplace-data-warehouse#16
- Issue loss-prediction: laplace-ai/laplace-service-loss-prediction#6
