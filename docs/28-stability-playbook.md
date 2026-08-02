---
title: Stability playbook — orders / parser / catalog / export
type: runbook
status: current
tags: [stability, runbook, parser, catalog, orders, export, regression]
updated: 2026-07-25
related:
  - "[[26-export-contract]]"
  - "[[26-catalog-source-of-truth]]"
  - "[[09-testing-strategy]]"
  - "[[changelog/codex-project-memory]]"
---

# Stability playbook

> [!info] Назначение
> Чек-лист, который **обязан** быть пройден перед закрытием любой задачи, затрагивающей заказы, парсер, каталог или экспорт. Цель — убрать класс «глупых регрессий» (пустая колонка, рейт-лимит душит UI, заказ выпал из выгрузки, sort_by рухнул на 400).

## 0. Перед началом

- [ ] Открой [[changelog/codex-project-memory]] — короткая «живая» память.
- [ ] Открой [[26-catalog-source-of-truth]] — чтобы случайно не написать «умный» матчер.
- [ ] Если задача про экспорт — открой [[26-export-contract]].

## 1. Заказы (`orders`)

| Зона | Правило |
|---|---|
| Сортировка | Любой новый ключ `sort_by` регистрируется и в `_apply_order_sorting`, и в `OrderQueryParams`, и в UI Select. |
| Пагинация исторических диапазонов | Когда `date_from/date_to` далеко в прошлом — пагинация должна остаться real, а не «list().slice». См. коммит `4881c5a`. |
| Архив | `archived_at` фильтруется по умолчанию. Включение `include_archived` — только из `OrdersPage` для owner. |
| `errors_only` | Включает OR по `status=needs_admin`, `error_text`, и наличию проблемных строк (`unknown_item / bad_qty / stopped`). Не дублировать условие. |
| Pickup-scope | `pickup_admin` видит только заказы с `pickup_place ∈ его точки`. Любая mutating-операция должна вызвать `_ensure_order_pickup_scope`. |
| `raw_text` | Никогда не перезаписывать после первичного приёма — это аудит. |

## 2. Парсер (`backend/app/parser/text_parser.py`)

| Зона | Правило |
|---|---|
| `_MAX_SANE_QTY` | Любая правка лимитов количества — с обновлением тестов. Без unit + qty > 100 → стоп, чтобы не парсить телефон как количество. |
| Цена в строке | `strip_price` обязан удалить `«950 за кг», «485р», «(790 руб)»` **до** `parse_qty`. |
| Юниты | Только через `_normalize_unit`. Никаких ad-hoc «if 'г' in text». |
| Распознавание дробей / диапазонов | Тестируется на real-shape сообщениях (см. `tests/test_parser_upgrade.py`). |
| Изменения парсера | Обязательно — регрессионный тест с реальной формулировкой клиента (в комментарии — источник). |
| Typed runtime | Worker/replay/reparse вызывают `run_order_pipeline_with_db`; legacy API — только adapter. |
| Diagnostics | Успешные и unresolved решения сохраняют score/margin/candidates/reason/version. |
| AI | Только shadow/recommendation по умолчанию; auto-match fail-closed без свежего version-matched evaluation artifact (ручной env-rate не gate). |

## 3. Каталог (`catalogs`, `catalog_items`)

См. [[26-catalog-source-of-truth]]. Кратко:

| Правило | Где живёт |
|---|---|
| Никаких runtime INSERT в `catalog_items` из клиентского заказа | enforced by code review |
| Admin-post sync работает только для подтверждённого admin/sender-chat и сильной формы списка; выключается `feature_catalog_admin_post_sync` | `worker/catalog_heal.py`, `DebugPage` |
| Каталог под старые заказы не переписывается | `feature_catalog_reparse` запускает reparse заказов под текущий каталог |
| Расширение через `aliases` — единственно допустимая правка пост-импорта | `admin_service/app/api/routers/catalogs.py::update_catalog_item` |
| `is_active=0` / `stop_at` / `stop_until` блокируют матч | `_resolve_catalog_item_for_line` |
| `position_order` диктует порядок колонок в distribution-экспорте | `_apply_distribution_catalog_headers` |
| `closed_at <= now` означает закрытый intake, даже если `status` ещё не успел обновиться | `catalog_lifecycle.py`, worker periodic check, admin startup и основные catalog read paths (list/detail/bootstrap) |
| Старый `/catalog_import` с диапазоном в title и пустым `closed_at` чинится только по строгому диапазону дат | `backfill_legacy_catalog_deadlines` |
| Будущий каталог хранится как `scheduled`; в `opened_at` он атомарно заменяет текущий `open` | `collection_automation.py`, `catalog_lifecycle.py` |
| Напоминание прекращается после появления будущего каталога; отчёт/напоминание идемпотентны по slot key | `collection_automation_state` |
| Ручное/автоматическое закрытие intake не ждёт parser debt; quality gate остаётся на final export | `routers/catalogs.py`, `parser_health.py` |

### 3.1. Проверка admin-post ingress

- [ ] При выключенном `feature_catalog_admin_post_sync` публикация потреблена как admin-origin, но каталог/позиции не изменены.
- [ ] При включённом флаге повтор одной публикации не создаёт второй каталог или дубли товаров.
- [ ] Память добавляет только безопасные aliases/unit к позиции из публикации.
- [ ] Клиент с похожим текстом не может изменить каталог.
- [ ] `СТОП` затрагивает только перечисленные позиции.
- [ ] Backfill после автосоздания ограничен `*_order_without_catalog` текущего окна и сохраняет `raw_text`, qty и unresolved.
- [ ] Для открытия/закрытия из UI не отправляются старые даты предыдущего lifecycle.
- [ ] Повторное сообщение ответственного обновляет тот же `scheduled`-каталог и сохраняет прежний SKU повторяющейся позиции.
- [ ] Клиентское сообщение не может создать `scheduled`-каталог.
- [ ] Weekly/biweekly расчёт всё равно нормализует окно к настроенным weekday/time.

## 4. Экспорт

См. [[26-export-contract]]. Минимум на каждый PR:

- [ ] Если поменял `_TEMPLATE_METADATA_COLUMNS` — wired в `_fill_template_row` **и** в `_fill_strict_template_row` (через `header_map`)?
- [ ] Если поменял окно дат / `_intersect_catalog_window` — нет ли регрессии «catalog_id beats stale dates»?
- [ ] Если поменял рейт-лимит — обновлён `test_export_rate_limit.py`?
- [ ] Если поменял UI-видимость поля — обновлены `bot_settings.export_*` ключи в `SYSADMIN_UI_DEFAULTS`?
- [ ] Unresolved line не попала в товарную колонку через fuzzy similarity с
  Excel header; её полный source text остался в колонке 4?

## 4.1. Preservation gate для targeted reparse

- [ ] Dry-run ограничен конкретными `catalog_id` и `reason_codes`; широкий
  `all_orders` запускается только после отдельного обоснования.
- [ ] `line_delta == 0` и `source_line_removed_count == 0`. Удалённая строка не
  считается исправлением, даже если unresolved counter уменьшился.
- [ ] Каждый `matched_quantity_decrease` разобран по source text и старому/new
  SKU. Apply запрещён, пока не доказано, что снята ложная связь, а не товар.
- [ ] Before/after audit сравнивает как минимум: число строк, hash/multiset
  исходных текстов, SKU+canonical-unit totals и Excel product totals.
- [ ] В Excel отдельно сверены data-row count и multiset колонки 4: сохранность
  raw order text нельзя выводить только из совпадения SKU totals.
- [ ] После apply те же проверки повторены на фактических DB/XLSX данных, а не
  только на predicted dry-run state.
- [ ] Apply запускался только после clean dry-run; при quantity decrease,
  source-line removal, mapped-SKU substitution, segmentation change, execution
  error или raw-text invariant violation он должен остаться blocked.
- [ ] Historical debt другого закрытого catalog не использован как причина
  блокировать operational export выбранного catalog.

## 5. Безопасность как часть стабильности

- [ ] Все `_sanitize_excel_value` сохранены в новых ячейках Excel.
- [ ] Все экспорт/мутирующие эндпоинты проходят через `require_password_confirmation`.
- [ ] Любой новый эндпоинт под `/exports/`, `/orders/export*` явно отнесён к bucket в `RateLimitMiddleware._bucket_key` и `_get_limit`.
- [ ] См. [[27-security-runbook]].

## Тестовый минимум

```sh
# Backend (парсер + воркер)
python -m pytest -q backend/tests/test_rate_limiter.py \
  backend/tests/test_order_processing_mvp.py \
  backend/tests/test_replay_pipeline_smoke.py \
  backend/tests/test_parser_upgrade.py

# Ingress exactly-once / retry / reconciliation / backfill
python -m pytest -q \
  backend/tests/test_integrations.py \
  backend/tests/test_provider_order_identity.py \
  backend/tests/test_worker_retry.py \
  backend/tests/test_reconciliation.py \
  backend/tests/test_ingress_identity_migration.py \
  backend/tests/test_sync_catalog_from_tg_updates.py

# Admin service — full сюита быстрая
python -m pytest -q admin_service/tests

# Гейт по экспорту (быстрая выборка)
python -m pytest -q \
  admin_service/tests/test_export_template_distribution_route.py \
  admin_service/tests/test_export_template_sanitization.py \
  admin_service/tests/test_orders_export_template_matching.py \
  admin_service/tests/test_export_rate_limit.py \
  admin_service/tests/test_orders_export_stability.py \
  admin_service/tests/test_settings_keys.py

# Frontend
cd admin-web && npm run test

# Gold evaluator safety gate
cd backend && python scripts/evaluate_parser_gold.py \
  --corpus tests/fixtures/parser_gold/v2.json --matcher current \
  --compare tests/fixtures/parser_gold/baseline-v2-current.json
```

CI-эквивалент см. в [[../.github/workflows/ci.yml]].

## Известные «дешёвые» способы регрессий

> [!warning] Эти три кейса — лидеры по количеству «глупых» багов
> 1. **Пустая ячейка в экспорте** — забыли wired metadata-колонку в `_fill_strict_template_row`. Лечение: тест на наличие ключа в `_TEMPLATE_METADATA_COLUMNS` + смок-export, проверяющий конкретную ячейку.
> 2. **Рейт-лимит душит UI** — read-only эндпоинт попал в `export` bucket вместо `security`. Лечение: при изменении `_bucket_key` синхронно править `test_export_rate_limit.py`.
> 3. **Стейл-фильтр по датам зануляет выгрузку** — UI оставил в state дату от прошлой раздачи, новый каталог открыли «сегодня». Лечение: `_intersect_catalog_window` должен сбрасывать стейл-диапазон к окну каталога. См. [[26-export-contract#привязка-к-датам-каталога]].
