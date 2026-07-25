---
title: Catalog as source of truth
type: spec
status: current
tags: [catalog, parser, source-of-truth]
updated: 2026-07-25
related:
  - "[[05-parsing-rules]]"
  - "[[26-export-contract]]"
  - "[[changelog/codex-project-memory]]"
---

# Каталог — единственный источник истины

> [!important] Правило в одну строку
> Подтверждённый оператором текст ассортимента — единственный источник истины для названий, единиц и порядка позиций. Клиентские заказы никогда не являются источником каталога.

## Допустимые операторские источники

1. Ручной `Import text` / `/catalog_import`.
2. Когда включён sysadmin-флаг `feature_catalog_admin_post_sync`, публикация
   списка товаров от настроенного администратора или `sender_chat` того же
   чата. Сообщение должно иметь сильную форму прайса/каталога: строки с ценами
   либо позиции с явными суффиксами единиц (`КГ / ШТ / УП / БАНКА / ПАЧ / НАБ`).

Второй путь — не «обучение из Telegram вообще», а альтернативный операторский
ingress того же source text. Флаг выключается в `DebugPage → Каталог`.

## Что отсюда следует

1. **Парсер заказа не достраивает каталог.** Если в клиентском заказе встречается товар, которого нет в каталоге — позиция уходит в `unknown_item` / заказ становится `partial`/`needs_admin`. Runtime-INSERT допустим только из подтверждённого admin-post ingress при включённом флаге.
2. **Заказы перепарсиваются по текущему каталогу**, а не каталог переписывается под старые заказы. Если оператор отредактировал каталог, запускаем reparse заказов.
3. **Новый каталог не наследует прошлый ассортимент молча.** Пустое ручное создание остаётся пустым; перенос возможен через явный clone/copy. Подтверждённый admin-post может создать новый каталог только когда открытого каталога нет, заполняет его исключительно позициями самой публикации и переносит строгий диапазон дат публикации в `closed_at`.
4. **`catalog_items.unit_hint` / `pack_hint`** диктуют единицу количества для матча. Текст сообщения клиента и заголовок Excel-шаблона на эту семантику не влияют.
   Преобразование `кг ↔ шт/уп/банка` допустимо только при явном числовом
   `pack_hint`; без него несовместимая единица остаётся `unit_conflict`.
5. **Если пользователь не указал количество, parser не выдумывает его из `unit_hint`.** Matched-строка без явного qty остаётся `bad_qty` / `needs_admin`, а не превращается в `1 шт`, `1 уп` или `1 банка`.
6. **Заголовки Excel-шаблона** — это только текстовые подписи для поиска нужной колонки. Никакого «125ГР» в заголовке не превращается в команду «писать в граммах».
7. **Расширять каталог по факту** допускается только через `aliases` (синонимы для распознавания), не через создание новых SKU из runtime-эвристик.
8. **Matcher и AI видят только текущий catalog ID.** Embedding/reranker/LLM не могут вернуть SKU из другой версии или создать SKU.
9. **Catalog gap подтверждает оператор.** Автоматический `no_candidates` означает abstention, но не является доказательством отсутствия товара в source catalog.
10. **Explicit product form сильнее широкого alias.** `с/м`, `вяленый`,
    `филе`, `икра`, `консервы`, `риет/паштет` нельзя пересекать только потому,
    что совпал species token.
11. **Память каталога — только enrichment.** `catalog_item_memory` добавляет к
    операторской позиции уже подтверждённые aliases/unit, но не создаёт товар
    из клиентского текста и не переносит SKU между разными product classes.
12. **Admin-post idempotent.** Повтор одной публикации обновляет существующую
    позицию, а не создаёт дубль. `СТОП` деактивирует только названные товары;
    повторная обычная публикация может вернуть только позицию, остановленную
    причиной `admin_post_stop`.
13. **Пропущенный ingress восстанавливается узко.** Если каталог был
    автоматически создан после первых заказов, backfill берёт только уже
    диагностированные `*_order_without_catalog` сообщения текущего
    межкаталожного окна (не более 14 дней), повторно прогоняет typed parser и
    оставляет неоднозначные строки unresolved.
14. **Lifecycle отделён от финального качества.** `closed_at` прекращает новый
    intake в срок даже при unresolved; parser-health preflight продолжает
    блокировать финальную выгрузку/ручное завершение без owner override.

Подробный lifecycle: [[32-parser-health-pipeline]].

## Куда смотреть в коде

| Слой | Файлы |
|---|---|
| Хранилище каталога | `backend/app/repo/catalog_repo.py` |
| Парсер заказа | `backend/app/parser/text_parser.py`, `backend/app/domain/order_domain.py` |
| Матч строки заказа к SKU | `admin_service/app/api/routers/orders.py::_resolve_catalog_item_for_line`, `_normalize_template_match_text` |
| Импорт каталога текстом | `admin_service/app/api/routers/catalogs.py::import_catalog_text` |
| Опциональный admin-post ingress | `backend/app/worker/catalog_heal.py::maybe_run_online_catalog_heal` |
| Автозакрытие и legacy repair | `backend/app/catalog_lifecycle.py`, `admin_service/app/catalog_lifecycle.py` |
| Reparse заказов после правки каталога | `admin_service/app/api/routers/catalogs.py::run_catalog_reprocess` (см. также `feature_catalog_reparse_all`) |

## Антипаттерны (не делать)

> [!danger] Никогда
> - INSERT в `catalog_items` из клиентского заказа, unresolved queue или произвольного `tg_updates`.
> - Считать обычное сообщение администратора каталогом без проверки роли/source и сильной формы списка.
> - Перезапись `catalog_items.title` по статистике из заказов.
> - Выдумывание количества из одного только `unit_hint` / `pack_hint`, если пользователь его не написал.
> - Превращение «125ГР» в заголовке Excel в семантику количества: для каталожных позиций единица всегда из `unit_hint`.
> - Игнор `is_active = 0` / `stop_at` / `stop_until` при матче — остановленные позиции не должны утекать в активные заказы.

## Доказательства в тестах

- `test_orders_export_template_matching.py::test_strict_template_export_writes_totals_and_matches_aliases`
- `test_export_template_distribution_route.py::test_distribution_export_uses_catalog_units_*`
- `test_export_template_distribution_route.py::test_pivot_export_uses_same_catalog_normalization_as_distribution_export`
- `backend/tests/test_real_messages.py::test_ivanovo_customer_price_lines_require_explicit_quantity`
- `backend/tests/test_replay_pipeline_smoke.py::test_worker_admin_catalog_sync_creates_catalog_and_is_idempotent`
- `backend/tests/test_replay_pipeline_smoke.py::test_worker_admin_catalog_creation_backfills_prior_no_catalog_order`
- `test_orders_export_stability.py` (см. [[28-stability-playbook#тестовый-минимум]])
