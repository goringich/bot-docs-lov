---
title: Catalog as source of truth
type: spec
status: current
tags: [catalog, parser, source-of-truth]
updated: 2026-05-01
related:
  - "[[05-parsing-rules]]"
  - "[[26-export-contract]]"
  - "[[changelog/codex-project-memory]]"
---

# Каталог — единственный источник истины

> [!important] Правило в одну строку
> Текст, который оператор вставил при создании каталога — единственный источник истины для названий, единиц и порядка позиций. Всё остальное — производные.

## Что отсюда следует

1. **Парсер не достраивает каталог.** Если в заказе встречается товар, которого нет в каталоге — позиция уходит в `unknown_item` / заказ становится `partial`/`needs_admin`. Никаких runtime-INSERT в `catalog_items` из обработчика заказов или admin-feed.
2. **Заказы перепарсиваются по текущему каталогу**, а не каталог переписывается под старые заказы. Если оператор отредактировал каталог, запускаем reparse заказов.
3. **`catalog_items.unit_hint` / `pack_hint`** диктуют единицу количества для матча. Текст сообщения клиента и заголовок Excel-шаблона на эту семантику не влияют.
4. **Заголовки Excel-шаблона** — это только текстовые подписи для поиска нужной колонки. Никакого «125ГР» в заголовке не превращается в команду «писать в граммах».
5. **Расширять каталог по факту** допускается только через `aliases` (синонимы для распознавания), не через создание новых SKU из runtime-эвристик.

## Куда смотреть в коде

| Слой | Файлы |
|---|---|
| Хранилище каталога | `backend/app/repo/catalog_repo.py` |
| Парсер заказа | `backend/app/parser/text_parser.py`, `backend/app/domain/order_domain.py` |
| Матч строки заказа к SKU | `admin_service/app/api/routers/orders.py::_resolve_catalog_item_for_line`, `_normalize_template_match_text` |
| Импорт каталога текстом | `admin_service/app/api/routers/catalogs.py::import_catalog_text` |
| Reparse заказов после правки каталога | `admin_service/app/api/routers/catalogs.py::run_catalog_reprocess` (см. также `feature_catalog_reparse_all`) |

## Антипаттерны (не делать)

> [!danger] Никогда
> - INSERT в `catalog_items` из обработчика входящих заказов / `tg_updates`.
> - Перезапись `catalog_items.title` по статистике из заказов.
> - Превращение «125ГР» в заголовке Excel в семантику количества: для каталожных позиций единица всегда из `unit_hint`.
> - Игнор `is_active = 0` / `stop_at` / `stop_until` при матче — остановленные позиции не должны утекать в активные заказы.

## Доказательства в тестах

- `test_orders_export_template_matching.py::test_strict_template_export_writes_totals_and_matches_aliases`
- `test_export_template_distribution_route.py::test_distribution_export_uses_catalog_units_*`
- `test_orders_export_stability.py` (см. [[28-stability-playbook#тестовый-минимум]])
