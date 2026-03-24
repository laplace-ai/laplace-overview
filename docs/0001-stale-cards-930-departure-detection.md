# 0001 — Cards antigos no coletor: fix is_delivered + detecção de saída via 930

**Data**: 2026-03-24
**Status**: Implementado, aguardando deploy e validação
**Repos afetados**: `laplace-data-warehouse`, `laplace-service-loss-prediction`, `laplace-platform-api`
**Issues**: [laplace-service-loss-prediction#4](https://github.com/laplace-ai/laplace-service-loss-prediction/issues/4), [laplace-data-warehouse#11](https://github.com/laplace-ai/laplace-data-warehouse/issues/11)

---

## Problema

Cards antigos permaneciam no coletor (visão em dispositivo) mesmo quando a carga já havia saído da unidade ou sido entregue. Exemplos concretos: CTRCs de 18/03 ainda apareciam como pendentes em transferência/distribuição em 24/03 — 6 dias parados.

Duas causas raiz foram identificadas:

### Causa 1: `is_delivered` incompleto na base_455

A coluna `is_delivered` verificava apenas `localizacao_atual == 'CTRC ENTREGUE/BAIXADO'`. Porém, muitos CTRCs têm `localizacao_atual` mostrando a última unidade (ex: "SAO JOSE DOS CAMPOS") enquanto `descricao_da_ultima_ocorrencia` já diz "ENTREGA REGISTRADA VIA MOBILE".

**Dados da investigação:**
- 5.088 CTRCs tinham `is_delivered = false` mas descrição de entrega registrada
- Exemplo concreto: CTRC `BAR322280-2` — a 930 mostrava entrega registrada em 19/03, mas `is_delivered = false` na 455

### Causa 2: ausência de detecção de saída da unidade

O `close_stale_verifications()` dependia exclusivamente da 455 para detectar movimentação. A 455 é um snapshot mensal — se a carga saiu da unidade mas a 455 ainda não atualizou, o card permanecia indefinidamente.

A base_930 (log de eventos do TMS, snapshot diário) tem os eventos exatos de saída com timestamps precisos, mas não era utilizada.

---

## Investigação da base_930

### Eventos de saída identificados

| cod_ocor | descricao_ocor | Frequência (30d) | Uso |
|----------|---------------|------------------|-----|
| 87 | SAIDA P/ ENTREGA | ~51.000 | Saída para distribuição |
| 95 | TRANSFERENCIA | ~24.000 | Saída para transferência |

### Eventos de chegada identificados

| cod_ocor | descricao_ocor | Frequência (30d) | Uso |
|----------|---------------|------------------|-----|
| 93 | CHEGADA NA UNIDADE DE DESTINO | ~9.500 | Chegada na unidade |
| 94 | CHEGADA NA UNIDADE | ~5.400 | Chegada na unidade |

### Eventos de entrega

| cod_ocor | descricao_ocor | Frequência (30d) | Uso |
|----------|---------------|------------------|-----|
| 1 | ENTREGA REALIZADA NORMALMENTE | ~136.000 | Entrega confirmada |
| 11 | ENTREGA REGISTRADA VIA MOBILE | ~86.000 | Entrega via mobile |

### Taxa de retorno

Investigação de retornos (insucesso de entrega + retorno):

- **Total de saídas**: ~51.000
- **Total de retornos**: ~1.700
- **Taxa de retorno**: ~3.3%

Decisão: não considerar retornos. Se uma carga saiu para entrega, é removida do coletor. Se por acaso retornar, o operador verifica fisicamente. A taxa de 3.3% é aceitável.

---

## Decisões tomadas

1. **Fix imediato do `is_delivered`**: expandir a lógica para também verificar `descricao_da_ultima_ocorrencia` com padrões de entrega
2. **Usar 930 para detecção de saída**: novo status `departed` no `loss_verification`, com `departed_at` timestamp
3. **Usar 930 para `arrived_at` preciso**: quando disponível, usar o timestamp do evento de chegada da 930 ao invés de `NOW()`
4. **Retornos ignorados**: cargo que saiu é considerada como partida definitivamente
5. **Manter 455 para detecção de chegada**: 455 identifica "onde está", 930 identifica "quando chegou/saiu" — separação limpa

---

## Alterações por repositório

### laplace-data-warehouse (`fix/is-delivered-logic`)

**Arquivo**: `src/pipelines/base_455/pipeline.py`
- Adicionado passo pós-COPY que verifica `descricao_da_ultima_ocorrencia` com padrões ILIKE:
  - `ENTREGA REALIZADA%`
  - `ENTREGA REGISTRADA%`
  - `ENTREGA EFETUADA%`
- Se match, atualiza `is_delivered = true` mesmo que `localizacao_atual` não diga "CTRC ENTREGUE/BAIXADO"
- Executa via UPDATE SQL após o COPY INTO no pipeline ETL

**Migration**: `scripts/sql/fix-is-delivered-from-description.sql`
- Backfill de dados existentes
- Corrigiu 5.088 CTRCs de `is_delivered = false` para `true`
- Verificação BEFORE/AFTER para auditoria

### laplace-service-loss-prediction (`feat/930-departure-detection`)

**Schema**: nova coluna `departed_at` (TIMESTAMPTZ, nullable) na `loss_verification`
**Schema**: novo valor `departed` no CHECK constraint de `verification_status`

**Migration**: `scripts/sql/add-departed-status.sql`
- Adiciona coluna `departed_at`
- Recria constraint para incluir `departed`

**Arquivo**: `src/verification.py`
- `close_departed_verifications()` — nova função que:
  - Cruza `loss_verification` (pending) com `base_930_cleaned`
  - Busca eventos de saída (cod_ocor 87, 95) e entrega (1, 11) na unidade do card
  - Marca como `verification_status = 'departed'` com `departed_at` = timestamp do evento
  - LEFT JOIN exclusion: só pega cards que ainda não foram fechados
- `_get_930_arrival_time()` — nova função que:
  - Busca evento de chegada (cod_ocor 93, 94) na 930 para a unidade
  - Retorna timestamp preciso se encontrado, `None` se não
- `sync_missing_ctrcs()` atualizado para usar `arrived_at` da 930 com fallback para `NOW()`

**Arquivo**: `src/main.py`
- Pipeline agora chama `close_departed_verifications()` após `close_stale_verifications()`

**Arquivo**: `src/predictor.py`
- Exclui CTRCs com status `departed` do scoring (LEFT JOIN exclusion)

### laplace-platform-api (`feat/930-departure-detection`)

**Arquivo**: `src/queries/loss_verification.py`
- Coletor filtra `verification_status != 'departed'` (cards que saíram não aparecem)
- Serialização inclui `departed_at` na resposta

**Arquivo**: `src/queries/loss_verification_stats.py`
- Stats do dashboard excluem `departed` das métricas
- `departed_at` incluído nos campos retornados

**Arquivo**: `src/queries/loss_predictions.py`
- Visão em tabela exclui CTRCs `departed`

**Arquivo**: `src/queries/loss_predictions_stats.py`
- Stats de predição excluem `departed`

**Arquivo**: `src/queries/analytics_stats.py`
- Analytics excluem `departed`

**Arquivo**: `src/queries/ab_testing_stats.py`
- A/B testing stats excluem `departed`

---

## Migrations executadas (2026-03-24)

| Migration | Resultado |
|-----------|-----------|
| `add-departed-status.sql` | Coluna `departed_at` adicionada, constraint atualizada |
| `fix-is-delivered-from-description.sql` | 5.088 CTRCs corrigidos, 0 restantes |

---

## Deploy

Ordem de deploy (respeitando dependências):

1. **laplace-data-warehouse** — fix do `is_delivered` no ETL
2. **laplace-service-loss-prediction** — departure detection + arrived_at da 930
3. **laplace-platform-api** — exposição de `departed_at` + filtro no coletor

Gateway não precisa de redeploy (nenhuma rota nova).
Frontend não precisa de mudanças (filtro é 100% backend).

---

## Validação pós-deploy

```sql
-- Cards fechados pelo departed detection
SELECT ctrc, unit_code, stage, departed_at, verification_status
FROM loss_verification
WHERE tenant_id = 1 AND verification_status = 'departed'
ORDER BY departed_at DESC LIMIT 20;

-- Cards com arrived_at preciso da 930 (não NOW())
SELECT ctrc, unit_code, arrived_at, created_at,
       EXTRACT(EPOCH FROM (created_at - arrived_at)) / 3600 AS diff_hours
FROM loss_verification
WHERE tenant_id = 1 AND arrived_at != created_at
ORDER BY created_at DESC LIMIT 20;

-- Confirmar backfill do is_delivered
SELECT COUNT(*) FILTER (WHERE is_delivered = false
    AND (descricao_da_ultima_ocorrencia ILIKE '%ENTREGA REALIZADA%'
         OR descricao_da_ultima_ocorrencia ILIKE '%ENTREGA REGISTRADA%'
         OR descricao_da_ultima_ocorrencia ILIKE '%ENTREGA EFETUADA%')
) AS still_broken
FROM base_455_cleaned WHERE tenant_id = 1;
```

---

## Evolução futura

- **Tipos de CTRC configuráveis**: transformar o filtro `'NORMAL'` (do issue de filtro de subcontratos) em parâmetro, permitindo aceitar outros tipos no futuro
- **Detecção de chegada via 930**: migrar a detecção de "em qual unidade está" da 455 para a 930, eliminando latência do snapshot mensal
- **Retornos**: se a taxa de retorno crescer, implementar lógica de "re-abertura" de card quando carga volta para a unidade
