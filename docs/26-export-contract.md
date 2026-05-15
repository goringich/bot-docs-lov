---
title: Export contract — strict / distribution / template
type: spec
status: current
tags: [export, contract, catalog, source-of-truth]
updated: 2026-05-01
related:
  - "[[07-excel-export]]"
  - "[[26-catalog-source-of-truth]]"
  - "[[28-stability-playbook]]"
  - "[[changelog/codex-project-memory]]"
---

# Excel-экспорт: строгий контракт

> [!important] Контракт обязателен
> Любая правка в `admin_service/app/api/routers/orders.py::export_orders_template_*` или в `_TEMPLATE_METADATA_COLUMNS` обязана пройти через этот документ. Менять колонки или семантику без обновления контракта = регрессия для оператора.

## Эндпоинты

| Эндпоинт | Назначение |
|---|---|
| `GET /orders/export-template` | Гибкий экспорт: пишет в шаблон, может расширять колонки. |
| `GET /orders/export-template-strict` | Жёсткий 1:1: фиксированные первые 4 колонки + товары из каталога; overflow допустим. |
| `GET /orders/export-template-distribution` | Раздача по точкам/каталогу: 1:1 строго каталого-ориентированный, **никаких overflow** колонок. |

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
- Видимость режимов экспорта управляется ключами `bot_settings.export_mode_template / _strict / _distribution / _flexible` (см. [[12-admin-panel#sysadmin-toggles]] и [[31-rate-limit-buckets]]).

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
