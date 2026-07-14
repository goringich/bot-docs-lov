---
title: Export contract — template / strict / distribution / pivot
type: spec
status: current
tags: [export, contract, catalog, source-of-truth]
updated: 2026-06-03
related:
  - "[[07-excel-export]]"
  - "[[26-catalog-source-of-truth]]"
  - "[[28-stability-playbook]]"
  - "[[changelog/codex-project-memory]]"
---

# Excel-экспорт: строгий контракт

> [!important] Контракт обязателен
> Любая правка в `admin_service/app/api/routers/orders.py::export_orders_template_*`, `export_orders_pivot_xlsx` или в `_TEMPLATE_METADATA_COLUMNS` обязана пройти через этот документ. Менять колонки или семантику без обновления контракта = регрессия для оператора.

## Эндпоинты

| Эндпоинт | Назначение |
|---|---|
| `GET /orders/export-template` | Гибкий экспорт: пишет в шаблон, может расширять колонки. |
| `GET /orders/export-template-strict` | Жёсткий 1:1: фиксированные первые 4 колонки + товары из каталога; overflow допустим. |
| `GET /orders/export-template-distribution` | Раздача по точкам/каталогу: 1:1 строго каталого-ориентированный, **никаких overflow** колонок. |
| `GET /orders/export-pivot` | Сводная книга по точкам и товарам; Book 1 обязан считать qty теми же каталоговыми правилами, что и distribution. |

## Pivot / Book 1

- Строки = `orders.pickup_place`, колонки = товары каталога в порядке `position_order → id`.
- Для строк, связанных с `catalog_item`, pivot считает количество через тот же shared path, что и `template-distribution` (`_resolve_catalog_item_for_line` + `_extract_line_qty_for_template`). Raw parse сам по себе не может переопределить семантику единицы.
- Если в matched-строке нет явного количества, pivot **не** добавляет выдуманное `1 шт/1 уп/1 банка`; такая строка остаётся audit-case (`bad_qty`) и не увеличивает числовую ячейку.
- Следствие: для одного и того же каталога / окна заказов totals в `export-pivot` Book 1 и в `export-template-distribution` должны совпадать по семантике количества.

## Жёсткий формат таблицы (strict / distribution)

| # | Заголовок | Источник | Можно изменить шаблоном? |
|---|---|---|---|
| 1 | **Имя** (клиент) | `orders.customer_name` | Нет, фикс по позиции |
| 2 | **Телефон** | `orders.phone_last4` | Нет, фикс по позиции |
| 3 | **Точка выдачи** | `orders.pickup_place` | Нет, фикс по позиции |
| 4 | **Текст заказа** | `orders.raw_text` (fallback — собранный summary) | Нет, фикс по позиции |
| 5 .. N | Товары каталога | `catalog_items.title` (порядок — `position_order`) | Нет, заголовки и порядок диктуются каталогом |
| (необязательно) | **Дата** | `source_message_ts ?? created_at`, формат `dd.MM.yyyy HH:mm` | Опц. — пишется, если в шаблоне есть колонка `Дата` после 4-й |

> [!warning] Что **не** может появиться в strict/distribution
> - Новые товарные колонки справа из не-каталожных позиций (для `template-distribution` это запрет).
> - Заголовки товара, отличные от `catalog_items.title`.
> - Числа, переписанные из `raw_text` без подтверждения от каталога.

### Инвариант сортировки товарных позиций

При подготовке каталогового экспорта (strict / distribution) товарные позиции сортируются **идентично в обоих режимах**:

1. **Сортировка**: `position_order` (nullable first) → `catalog_items.id`
2. **Результат**: гарантирует консистентный порядок колонок товаров между strict и distribution экспортами для одного каталога, что исключает confusion о разнице в структуре файла.
3. **Причина**: без идентичной сортировки товарные позиции в strict могли появиться в одном порядке, а в distribution — в другом, вызывая путаницу у оператора при сравнении выгрузок.

### Инвариант покрытия каталожных позиций (добавлено 2026-05-22)

`export-template-strict` гарантирует, что **все** позиции каталога получат свою колонку в выгрузке:

- Позиции, заголовок которых нашёлся в шаблоне через fuzzy-matching → идут в своей template-колонке (заголовок шаблона сохраняется).
- Позиции, которых **нет** в шаблоне → получают новую колонку в конце листа с точным названием из `catalog_items.title` (в порядке position_order → id).
- Это реализовано через `append_missing=True` в `_build_catalog_item_template_column_map`, которая вызывается на этапе инициализации — до обработки строк. В `catalog_item_column_map` теперь есть запись для каждой позиции каталога.
- **Следствие**: строки заказа с одной и той же позицией каталога (даже если в разных заказах хранятся разные `item_title`) попадают в одну и ту же колонку (lookup идёт по `catalog_item_id`, а не по тексту). Это устраняет edge-case с дублированием overflow-колонок.

| Режим | Откуда берутся колонки товаров | Overflow при неизвестном item |
|---|---|---|
| `template` | Из шаблона; может создавать новые колонки для неизвестных | Да: `_ensure_template_column` |
| `template-strict` | Из шаблона + **пре-создаются колонки для всех кат. позиций** | Да: для строк без `catalog_item_id` |
| `template-distribution` | Перезаписывает шаблон точными названиями каталога | Нет |

> [!note] Pickup-place (город): значение берётся напрямую из `orders.pickup_place`, никакого вывода или додумывания не производится (col 3, фиксированная позиция).

См. также [[26-catalog-source-of-truth]] и [[changelog/codex-project-memory#Канонические-инварианты]].

## Привязка к датам каталога

Каталог имеет окно работы — `catalogs.opened_at .. catalogs.closed_at`.

```
# псевдокод (см. orders.py::_intersect_catalog_window)
if catalog_id is set:
    window = (catalog.opened_at, catalog.closed_at)
    user_range = (date_from, date_to)
    if user_range overlaps window:
        effective = clamp(user_range, window)
    else:
        effective = window  # стейл-фильтр игнорируем
else:
    effective = user_range
```

| Сценарий | Поведение |
|---|---|
| Передан `catalog_id`, дат нет | Берётся окно каталога целиком. |
| Передан `catalog_id` + даты внутри окна | Берётся пересечение (даты сужают окно). |
| Передан `catalog_id` + даты вне окна | Стейл-фильтр **игнорируется**, берётся окно каталога. |
| `catalog_id` не существует | **404**, экспорт не выполняется. См. `_get_catalog_active_window(require_exists=True)`. |
| Без `catalog_id` | Берётся пользовательский диапазон, без окна каталога. |

## Поля UI

- `date_from`, `date_to` — опциональные. UI помечает «(необязательно)» и может скрывать через `bot_settings.export_filter_show_date_range`.
- `pickup_place` — опциональное. UI помечает «(необязательно)» и может скрывать через `bot_settings.export_filter_show_pickup`.
- Видимость режимов экспорта управляется ключами `bot_settings.export_mode_template / _strict / _distribution / _flexible / _pivot` (см. [[12-admin-panel#sysadmin-toggles]] и [[31-rate-limit-buckets]]).

## Метаданные шаблона

`_TEMPLATE_METADATA_COLUMNS` — единственное место, где регистрируются «not-product» колонки. Добавление нового поля требует:

1. Добавить запись `(key, header_title, aliases, fixed_column)`.
2. Wired в `_fill_template_row`. Если поле должно появиться и в strict/distribution — также wired в `_fill_strict_template_row` (через `header_map` lookup).
3. Регрессионный тест в [[admin_service/tests/test_orders_export_stability]] — иначе цена ошибки = пустая ячейка без сигнала.

## Безопасность

- Все эндпоинты экспорта требуют `_require_owner_export_access` (только owner) и `require_password_confirmation("export")` — пароль вводится повторно перед скачиванием.
- Все ячейки прогоняются через `_sanitize_excel_value` — защита от формула-инъекции (`=`, `+`, `-`, `@` в начале строки).
- Рейт-лимит: heavy-эндпоинты (`/orders/export*`, `/exports/build`) идут в bucket `export` (20 req/min). `/exports/presets` — в `security` (60 req/min). См. [[31-rate-limit-buckets]].
- Pickup-scope: `_apply_pickup_scope_filter` режет выгрузку по точкам, привязанным к текущему `pickup_admin`.
- Final strict/distribution/pivot и `/exports/build` проходят parser-health preflight. Unresolved `unknown_item`, неразрешённый `bad_qty`, pending reparse или stale/missing diagnostics блокируют выгрузку.
- Owner override допустим только после password confirmation, с reason/comment и audit event `export.parser_health_override`.

## Регрессионная сетка

```sh
python -m pytest -q \
  admin_service/tests/test_export_template_distribution_route.py \
  admin_service/tests/test_export_template_sanitization.py \
  admin_service/tests/test_orders_export_template_matching.py \
  admin_service/tests/test_export_rate_limit.py \
  admin_service/tests/test_orders_export_stability.py \
  admin_service/tests/test_settings_keys.py
```

Если поменялся middleware рейт-лимита — обновить `test_export_rate_limit.py` в том же PR.
