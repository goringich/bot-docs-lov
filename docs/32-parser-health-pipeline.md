---
title: Parser health pipeline — typed decisions, diagnostics, shadow AI and final preflight
type: spec
status: current
tags: [parser, catalog, diagnostics, ai, reparse, monitoring]
updated: 2026-07-15
related:
  - "[[26-catalog-source-of-truth]]"
  - "[[28-stability-playbook]]"
  - "[[26-export-contract]]"
---

# Parser health pipeline

## Production flow

```text
provider ingress
→ normalization
→ intent
→ metadata
→ segmentation
→ explicit quantity
→ current-catalog candidate generation
→ deterministic ranking
→ local Ollama semantic retrieval (`nomic-embed-text`)
→ calibrated reranker
→ constrained local LLM fallback (only when enabled)
→ score/margin/conflict + holdout safety gate
→ status mapping
→ structured diagnostics
→ operator resolution
→ targeted reparse
→ final preflight
```

`backend/app/order_processing.py::run_order_pipeline_with_db` — общий typed
entrypoint для worker, replay, import и reparse. Legacy parser functions остаются
совместимыми adapters.

## Safety contract

- Candidate universe — только текущий `catalog_id`.
- `catalog_items` никогда не создаются из заказов.
- Quantity без числа не выводится из unit/pack hint.
- Exact/operator alias является catalog truth, но fish→roe остаётся hard conflict.
- `catalog_conflict` может прикрепляться только к кандидату с лексическим
  product evidence. Нулевой конфликт с несвязанной первой строкой каталога не
  является кандидатом. Generic alias (`риет`, `паштет`, `филе`) не может стереть
  явно указанное сырьё или product class клиента.
- `missing_catalog_coverage` ставит оператор; `no_candidates` сам по себе не
  доказывает отсутствие товара в source catalog.
- Auto-match разрешён только без conflict flags, при достаточных score/margin
  и `PARSER_AI_HOLDOUT_FALSE_CONFIDENT_RATE <= 0.001`. Без измеренной holdout
  метрики semantic/LLM возвращают abstention.
- Manual decision ставит `manual_lock`; reparse не меняет такую строку.
- Quantity-first (`2 шт форели`) и descriptor continuation сохраняют одну
  source line; явное количество нельзя отделить от товара metadata/newline
  эвристикой. Pickup aliases после standalone parsing всегда канонизируются
  обратно в operator-owned `PickupPlace.title`.

## Structured storage

Миграции:

- `m9n0o1p2q3r4` — `parser_runs`, `parser_line_decisions`, resolutions,
  reparse jobs и model registry;
- `n0o1p2q3r4s5` — operator examples, AI cache и health snapshots;
- `o1p2q3r4s5t6` — `order_line_id ON DELETE SET NULL`, чтобы rebuild сохранял
  before-history diagnostics.

`raw_text` остаётся только в order source. Diagnostics хранит normalized
fragment, offsets, safe hash, score/margin, candidates, conflicts, reason,
parser/model versions и decision source.

## Reasons

Основные machine-readable reasons:

`no_candidates`, `low_score`, `ambiguous_margin`, `catalog_conflict`,
`unit_conflict`, `quantity_missing`, `segmentation_error`,
`missing_catalog_coverage`, `intent_uncertain`, `stale_reparse`,
`ai_schema_error`, `ai_timeout`, `ai_disagreement`,
`legacy_or_unspecified`.

## AI mode

Default flags:

```env
PARSER_EMBEDDINGS_ENABLED=1
PARSER_RERANKER_MODE=production
PARSER_LLM_MODE=disabled
PARSER_AI_AUTO_MATCH=0
```

Local Ollama embeddings (`nomic-embed-text`) and calibrated pair reranker
расширяют только catalog-scoped candidates. Constrained LLM boundary принимает только
обезличенный fragment, quantity/unit и candidates; schema violation, timeout,
budget exhaustion или SKU вне списка дают abstention.

Semantic layer запускается только после deterministic abstention. Он не имеет
права переиграть `ambiguous_margin`, product/form conflict или unit conflict;
missing catalog coverage остаётся operator decision. Offline gold v2 содержит
раздельные tuning/validation/holdout catalogs и проверяет одновременно top-1,
quantity/unit, status/end-to-end и нулевой false-confident rate.

Automatic mode требует отдельного holdout/shadow safety review. Наличие флага
само по себе недостаточно: gate требует false-confident rate `<= 0.001`.

Git snapshot показывает не предполагаемые defaults, а фактически вычисленные в
процессе reporter значения: `embeddings_enabled`, `reranker_mode/enabled`,
`llm_mode/enabled`, `auto_match_requested/effective` и измеренный holdout rate.
Endpoint/model URL, токены и другие секреты в snapshot не попадают.

## Operator flow

Owner queue живёт в Analytics:

- посмотреть reason, score/margin и top candidates;
- выбрать существующий SKU;
- подтвердить catalog gap;
- отметить correction и запустить dry-run targeted reparse.

Подтверждённый owner existing SKU добавляет trusted normalized alias только
этому существующему catalog item и запускает targeted reparse всего cluster
для текущей версии каталога. Каждое решение пишет audit, resolution event и
`parser_operator_examples` с positive/negative candidate evidence.

## Final preflight

Strict/distribution/pivot, final `/exports/build` и закрытие каталога блокируются,
если есть unresolved unknown/bad_qty, pending/failed reparse или строки без
diagnostics после миграции.

Owner override требует re-auth, reason/comment и audit. Intake не блокируется.

## Reports and scheduling

Stable GitHub surfaces:

- `runtime-reports/parser-health/current.json`;
- `runtime-reports/parser-health/current.md`;
- `runtime-reports/parser-health/history/YYYY-MM-DD.json`.

One-shot generation:

```sh
docker-compose -f infra/docker-compose.yml --profile ops run --rm parser_health_reporter
```

Timer templates: `infra/systemd/otlichniy-parser-health.{service,timer}`.
PII guard выполняется до записи файлов и DB snapshot.

Для каждого top unresolved cluster sanitized JSON/Markdown содержит до трёх
кандидатов: `rank`, `sku`, `score`, `strategy`, `conflict_flags` и
`source_catalog_id`. В report запрещены internal item/order/user identifiers и
исходный customer text; fragment проходит PII sanitizer и ограничение длины.
Отдельный `catalog_conflict_summary` агрегирует текущее число конфликтов по
source catalog и sanitized top candidate SKU, чтобы нулевой/массовый кандидат
был виден без чтения customer fragments.

`infra/publish_parser_health_report.sh` генерирует snapshot, а при явном
`PARSER_HEALTH_GIT_PUBLISH=1` коммитит только stable report paths и отправляет
их существующим private Git transport. Таймер запускается ежечасно через
`flock`; публикация по умолчанию выключена и не требует хранения GitHub token
в репозитории.

## Reparse

API: `POST /parser-health/reparse`.

Dry-run по умолчанию. Результат содержит `scanned`, `changed`, `resolved`,
`still_unresolved`, `sku_changed`, `status_changed`, `segmentation_changes`,
`possible_false_matches`, `line_delta`, `source_line_removed_count`,
`matched_quantity_decrease_count`, `preservation_review_required`,
`manual_locked`, `errors` и bounded before/after/review samples.

Удаление строки или уменьшение ранее matched SKU quantity никогда не считается
`resolved`: это preservation review. До apply оператор обязан сопоставить
source fingerprints, SKU totals и Excel product totals. Для incident repair
используется самый узкий reason-code/catalog scope; `all_orders` не заменяет
целевой dry-run. Manual lock действует только в той же версии каталога; новая
catalog version/alias или явный owner reparse снимают его контролируемо.
Idempotency обеспечивается key в `parser_reparse_jobs`.

`archive_non_orders=true` — отдельный opt-in для уже сохранённых сообщений,
которые новый intent classifier подтвердил как не-заказы. Автоматический
soft-archive разрешён только когда ни before-, ни after-state не содержит
явного количества. `raw_text` и существующие `order_lines` не удаляются;
snapshot исключает архивную запись через active-order join. Сообщение с любым
quantity блокируется для operator review.

Startup/online catalog-heal не является archival workflow и работает
fail-closed. Он сохраняет заказ целиком и считает отдельные counters, если:

- новый intent больше не считает source message заказом;
- source message принадлежит admin/senderless feed;
- predicted state уменьшает matched SKU quantity, число строк или теряет
  fingerprint исходной строки.

Такие записи попадают соответственно в `non_order_skipped`,
`admin_origin_skipped` или `preservation_blocked`; `updated` не увеличивается.
Обычный targeted reparse без `archive_non_orders=true` также не имеет права
пересобрать подтверждённый non-order в пустой набор строк.

## Rollback

1. Выключить new intake path rollback-deploy предыдущего backend image.
2. Оставить diagnostics tables — они additive и не меняют catalog/orders.
3. При полном schema rollback: `alembic downgrade l8m9n0o1p2q3` только после
   выгрузки operator events/examples.
4. Откат targeted reparse выполняется по audited before-state; `raw_text`
   остаётся неизменным источником повторного rebuild.
