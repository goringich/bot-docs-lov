# Decisions log

## 2026-07-25 — completion is evidence-based, not a handoff

**Context**: Локальный тест или готовый diff иногда ошибочно выдавались за
завершение задачи, хотя пользователь просил изменить работающий runtime или
данные, а deploy и проверка результата не были выполнены.

### Решение

- Добавлен единый [[33-task-completion-gate]] для Codex, Claude и Copilot.
- Терминальный статус строго двоичный: `COMPLETE` только с доказательствами
  всех применимых слоёв; при недоступном runtime/data/deploy — `BLOCKED` с
  точным недостающим доступом.
- `make check-agent-contract` защищает сам контракт от случайного удаления и
  входит в стандартные `fast`/`all` check runners.

**Consequences**:
- Команда или handoff, которые не были выполнены, больше не считаются
  доказательством результата.
- Задачи, меняющие production runtime, не могут быть закрыты только по
  unit-тестам: требуется подтверждение deploy и требуемого бизнес-эффекта.
- Локальные FastAPI `TestClient` suites сначала проходят короткий timeout probe;
  при несовместимом интерпретаторе они fail-fast, а admin-service CI закреплён
  на production-matched Python 3.12.
- Для live catalog proof добавлен безопасный verifier
  `backend/scripts/verify_catalog_lifecycle.py`: он оценивает только
  sanitized `id/status/closed_at` и не открывает config/DB самостоятельно.
- Lifecycle reconciliation выполняется перед list/detail/bootstrap catalog
  reads, поэтому просроченный cutoff не остаётся визуально `open` между
  периодическими worker-проходами.

## 2026-06-22 — orders list filters source provider with TG/MAX buttons

**Context**: Оператору нужен быстрый operational способ смотреть ленту заказов по источнику соцсети (`telegram` vs `max`) прямо в `OrdersPage`, без owner-only scope-фильтров.

### Решение

- В backend `orders` router расширен фильтром `source_provider=telegram|max`.
- В `admin-web` добавлены отдельные кнопки-фильтры `Все / TG / MAX`.
- Каталог остался отдельным фильтром выбора `catalog_id`, не частью сортировки.
- Видимость кнопок источника управляется sysadmin-настройкой `orders_filter_show_source_provider`.

**Consequences**:
- Любая роль с доступом к разделу `Заказы` может быстро отфильтровать список по источнику (`TG`/`MAX`) без owner-only scope-режима.
- Диагностика «почему не вижу MAX-заказы» стала проще: сначала сортировка/фильтр в UI, затем проверка фактического intake по `source_provider`.

## 2026-06-21 — deferred reorder columns are not enough for inserts; migrations must be applied before live intake

**Context**: Runtime verification с реальным MAX `message_created` показала, что `deferred`-поля в ORM защищают только read-path (`SELECT`), но не insert-path: при создании заказа worker всё равно пишет `orders.parent_order_id` / `orders.is_reorder`. На инстансе без миграции это даёт `OperationalError: Unknown column 'parent_order_id'` и блокирует создание заказов.

### Решение

- Зафиксирован operational-invariant: перед live intake обязателен `alembic upgrade head` для `backend`.
- На текущем инстансе применена миграция reorder-полей (`l8m9n0o1p2q3`) до `head`, после чего MAX update обработан в `done`, заказ создан.

**Consequences**:
- Legacy-совместимость через `deferred` остаётся полезной для чтения, но не заменяет миграции схемы.
- Runbook для инсталляций должен трактовать `alembic upgrade head` как обязательный шаг перед запуском ingestion worker.

## 2026-06-21 — new chats default to hidden bot visibility

**Context**: Требование оператора — в любых новых чатах бот должен быть полностью скрыт по умолчанию при первом появлении чата в системе (Telegram/MAX/admin bootstrap).

### Решение

- Дефолт `Chat.bot_visible` в модели установлен в `hidden`.
- При создании чата в runtime путях значение задаётся явно:
  - `backend/app/repo/order_repo.py::ensure_chat`
  - `backend/app/admin_commands.py::_ensure_chat`
  - `backend/app/handlers/admin_commands.py::_handle_catalog_open`
  - `admin_service/app/api/routers/catalogs.py::_ensure_chat` (через column-existence guard для legacy-схем)
- Fallback-нормализация в worker (`normalize_bot_visible_mode`, `get_chat_bot_visible`) также возвращает `hidden` для невалидных/пустых значений.

**Consequences**:
- Новые чаты не получают “full” режим случайно из fallback/legacy defaults.
- Для старых чатов поведение управляется существующим значением в БД (без принудительной массовой миграции).

## 2026-06-21 — global JSON exception handling for REST stability

**Context**: Нужно повысить устойчивость REST API, чтобы неожиданные исключения не приводили к “ломающим” HTML/traceback ответам и были трассируемы по request id.

### Решение

- В `backend/app/main.py` и `admin_service/app/main.py` добавлены:
  - `RequestIdMiddleware` (генерация/проброс `X-Request-ID`)
  - handler `RequestValidationError` → структурированный JSON (`detail`, `errors`, `request_id`)
  - handler `Exception` → стабилизированный JSON 500 (`detail`, `request_id`) + `logger.exception`.

**Consequences**:
- Клиенты получают предсказуемый JSON-контракт для 422/500.
- Любой сбой коррелируется по `X-Request-ID` в логах без раскрытия внутренних деталей наружу.

## 2026-06-17 — legacy-schema compatibility: defer reorder columns in `Order`

**Context**: На инстансе без миграции reorder-полей (`orders.parent_order_id`, `orders.is_reorder`) startup catalog heal падал на `SELECT Order` с ошибкой `Unknown column ... in field list`.

### Решение

- В `backend/app/models.py` поля `Order.parent_order_id` и `Order.is_reorder` переведены в `deferred(mapped_column(...))`.
- Это выравнивает поведение с уже deferred архивными полями и позволяет `select(Order)` работать на legacy-схемах, где эти колонки ещё отсутствуют.

**Consequences**:
- Worker startup heal больше не падает на старой схеме `orders`.
- Reorder-поля подгружаются только при явном доступе; на legacy-БД такой доступ нужно избегать до применения миграций.

## 2026-06-17 — provider-aware replay from `tg_updates` is required for MAX catalog recovery

**Context**: Когда MAX-сообщения прилетают до открытия каталога или когда нужно догнать уже сохранённые webhook events, sysadmin использует replay из `tg_updates`. До этого `backend/scripts/collect_orders_from_tg_updates.py` умел читать только Telegram-shaped payload (`message` / `channel_post`) и пропускал provider envelopes (`__integration_envelope__`) для MAX.

### Решение

- `backend/scripts/collect_orders_from_tg_updates.py` теперь разворачивает provider envelope через `envelope_to_worker_payload(...)` перед фильтрацией и парсингом.
- При создании заказа replay теперь сохраняет `source_provider`, `source_chat_key` и `source_user_key`, чтобы MAX replay был консистентен с обычным worker path.
- Добавлен regression test `backend/tests/test_sync_catalog_from_tg_updates.py::test_collect_orders_from_tg_updates_supports_max_provider_envelope`.

**Consequences**:
- MAX/VK/Matrix/Webhook replay из `tg_updates` проходит тем же нормализованным путём, что и live worker processing.
- Для каталога MAX можно догонять старые provider messages без ручной конверсии payload в Telegram shape.

## 2026-06-16 — chats are synced on any inbound provider message (including empty/service events)

**Context**: В multi-messenger режиме (MAX/TG) часть чатов не появлялась в UI до первого «текстового заказа». Если бот уже добавлен в чат, но прилетали только service/empty события (например onboarding), запись в `chats` могла не появиться вовремя.

### Решение

- В `backend/app/worker/loop.py` добавлен ранний вызов `ensure_chat(...)` внутри ветки обработки входящего `message` (после извлечения `chat_id/chat_type/title/provider`), до фильтрации по `text`.
- Для non-Telegram провайдеров используется `__provider` + `__provider_raw.chat_id`, чтобы корректно заполнять `messenger_provider/provider_chat_id/provider_chat_key`.
- Добавлен regression test `backend/tests/test_worker_chat_sync.py::test_handle_update_syncs_chat_even_for_empty_text`.

**Consequences**:
- Чат теперь появляется в `/chats` и в `CatalogsPage`/UI сразу после любого валидного входящего сообщения от провайдера, а не только после первой непустой текстовой строки.
- Поток обработки заказов не изменён; это только ранняя синхронизация метаданных чата.

## 2026-06-16 — MAX (Max.ru) direct Bot API integration

**Context**: Добавлен токен `MAX_BOT_TOKEN` для бота в мессенджере MAX (ex-TamTam, Mail.ru). Ранее MAX-адаптер работал только через внешний bridge-сервер (`MAX_OUTBOUND_URL`), который никогда не был запущен. Пользователь хочет собирать заказы из MAX чатов, где добавлен бот, так же как из Telegram.

### Решение

- Создан `backend/app/max_client.py` — прямой клиент MAX Bot API (`platform-api.max.ru`), аналог `telegram_client.py`. Auth через `Authorization: <token>` header (query-param больше не поддерживается с 2026).
- `AdapterManager` обновлён: `send_message` и `send_document` для провайдера `max` теперь идут через `max_client` напрямую, минуя bridge transport. Только Matrix/VK/Webhook по-прежнему используют bridge.
- `main.py` lifespan регистрирует MAX webhook через `POST /subscriptions` при старте (так же как для Telegram через ngrok). Webhook URL: `{ngrok}/integrations/max/webhook`.
- `provider_bridge.py`: исправлено имя заголовка для верификации входящих MAX webhooks: `X-Max-Bot-Api-Secret` (официальный заголовок MAX API) вместо `X-Max-Webhook-Secret`.
- `.env.example`: обновлены комментарии к `MAX_BOT_TOKEN` и `MAX_WEBHOOK_SECRET`.

**Invariants**:
- Весь порядок обработки сообщений (parse → update → TgUpdate → worker → parser → order_handler) не изменился — MAX события проходят через тот же путь через `/integrations/max/webhook` → `build_ingest_envelope` → `parse_max_event` → worker.
- `MAX_OUTBOUND_URL` / `MAX_OUTBOUND_SECRET` (bridge) по-прежнему читаются из конфига, но при наличии `MAX_BOT_TOKEN` bridge не используется для исходящих сообщений.

**Требования MAX API**:
- Вебхук должен быть на HTTPS порту 443 с валидным TLS-сертификатом (ngrok обеспечивает это автоматически).
- Токен — в `Authorization: <token>` header.

## 2026-06-03 — empty catalogs stay empty, and bulk clear is explicit + sysadmin-gated

**Context**: Оператор создавал «пустой» каталог для нового чата/сбора, но UI + backend молча подхватывали позиции из предыдущего каталога. В результате source-of-truth для нового каталога подменялся историческим ассортиментом ещё до явного импорта текста. Отдельно не хватало быстрого способа очистить ошибочно наполненный каталог целиком.

**Root cause**:
- `CatalogsPage` при смене чата сбрасывал форму создания на последний подходящий `source_catalog_id`, а не на truly-empty состояние.
- `POST /catalogs` допускал copy-flow без явного `source_catalog_id` и fallback-ом выбирал последний каталог чата.
- Для bulk-reset каталога не было отдельного безопасного endpoint-а.

### Решение

- `CatalogsPage` теперь сбрасывает создание на `source_catalog_id=0` и `copy_items_from_source=false`.
- `POST /catalogs` копирует позиции только при явно заданном источнике; без него новый каталог остаётся пустым.
- Добавлен `DELETE /catalogs/{catalog_id}/items`: перед удалением сервис снимает связи `order_lines.catalog_item_id`, чтобы история заказа и `raw_text` оставались нетронутыми.
- В sysadmin-настройки добавлен toggle `feature_catalog_delete_all_items` для показа/скрытия массовой кнопки очистки.

**Consequences**:
- Пустой каталог теперь действительно пустой по умолчанию.
- Copy/clone стали явными операциями, а не скрытым side-effect при создании.
- Owner может быстро очистить каталог, а sysadmin может скрыть опасное массовое действие в UI.

## 2026-06-03 — pivot export shares distribution semantics; missing qty is no longer invented

**Context**: Операторы сравнивали `export-template-distribution` и `export-pivot` («Книга 1») для одного и того же каталога и видели расхождение по количествам. Параллельно parser скрывал часть проблемных строк, когда matched-позиция без явного qty автоматически превращалась в `1 шт/1 уп/1 банка`.

**Root cause**:
- `export_orders_pivot_xlsx` использовал отдельный raw qty parse-path вместо общего catalog-aware расчёта количества.
- `_infer_missing_qty_for_catalog()` в `backend/app/domain/order_domain.py` синтезировала qty по `unit_hint`, даже когда пользователь его не писал.

### Решение

- Pivot export переведён на shared catalog-aware path (`_resolve_catalog_item_for_line` + `_extract_line_qty_for_template`), тот же по смыслу, что и distribution/template.
- Дата каталога в шапке pivot normalizes to naive datetime перед записью в workbook, чтобы Excel/openpyxl не падал на timezone-aware значениях.
- Parser больше не придумывает qty для matched-строки без явного количества: такие кейсы остаются `bad_qty` и не попадают в числовые totals.

**Consequences**:
- `export-pivot` Book 1 и `export-template-distribution` синхронизированы по семантике количества.
- Missing qty теперь виден оператору как реальная проблема, а не маскируется под псевдо-валидный `1 шт`.

## 2026-05-22 — export-template-strict: pre-создание колонок для всех позиций каталога

**Context**: Операторы замечали, что `export-template-strict` и `export-template-distribution` дают разные результаты для одного каталога. В distribution всегда есть колонка для каждой позиции (прямой маппинг `catalog_item_id → column`). В strict позиции, которые не смогли fuzzy-матчиться к шаблонному заголовку, появлялись только через overflow при обработке строк — что порождало два edge-case:
1. Если у разных заказов одна и та же позиция была сохранена с разными `item_title`, могло создать несколько дублирующих overflow-колонок.
2. Позиции, не матчащиеся к шаблону, получали колонку с `item_title` из первой встреченной строки, а не из `catalog_items.title`, что нарушало инвариант «заголовок = catalog title».

**Root cause**: `_build_catalog_item_template_column_map` при `strict=True` пропускал (`continue`) позиции без шаблонного матча, не добавляя их в `catalog_item_column_map`. Заполнение шло через `append_overflow_column=True` уже при обработке строк — поздно, с возможными коллизиями кэша.

### Решение

В `_build_catalog_item_template_column_map` добавлен параметр `append_missing: bool = False`. Когда True, позиция без матча получает новую колонку сразу (через `_append_template_column`) с точным `catalog_items.title`, в порядке `position_order → id`. В `export_orders_template_strict_xlsx` вызов использует `append_missing=True`.

**Следствие**:
- Все позиции каталога гарантированно имеют колонку в strict экспорте (как и в distribution).
- Строки с одной и той же позицией (даже с разными item_title) всегда попадают в одну колонку — lookup идёт по `catalog_item_id`, а не по тексту.
- Заголовки новых колонок = `catalog_items.title` (точно), а не `item_title` из конкретного заказа.
- `template-distribution` не затронут (default `append_missing=False`).

**Pickup-place уточнение**: значение берётся напрямую из `orders.pickup_place` без вывода или dodумывания — это инвариант для всех трёх strict-ориентированных режимов.



**Context**: После добавления sysadmin-переключателя `merge_reorders` UI сохранение падало с `Invalid settings keys`, потому что ключа не было в backend allowlist (`ALLOWED_SETTINGS_KEYS` через `SYSADMIN_UI_DEFAULTS`).

Параллельно потребовался owner-only режим просмотра заказов по скоупам: города / chat_group_key / конкретные чаты / провайдеры (TG/MAX), включаемый через sysadmin.

### Решение

1. В `admin_service` ключи зарегистрированы в `SYSADMIN_UI_DEFAULTS`:
  - `merge_reorders=false`
  - `orders_owner_scope_view=false`

2. В `admin-web`:
  - `DebugPage` получил переключатель `orders_owner_scope_view`.
  - `OrdersPage` получил owner-only блок scope-фильтров (provider/city/group/chat) и scoped-рендер списка.

3. Добавлен тест-контроль ключей в `admin_service/tests/test_settings_keys.py` (`ORDERS_BEHAVIOR_KEYS`).

**Consequences**:
- Сохранение sysadmin-настроек больше не падает на `merge_reorders`.
- Owner может включать/выключать расширенный scope-view централизованно через sysadmin.
- Scope-view не доступен не-owner ролям даже при наличии ключа (owner-only guard в UI).

## 2026-05-18 — export-template-strict: сортировка position_order консистентна с distribution

**Context**: Функция `export_orders_template_strict_xlsx` не сортировала `catalog_items` при подготовке экспорта, в отличие от `export_orders_template_distribution_xlsx`, которая применяла сортировку по `position_order` → `id`. Это привело к тому, что товарные колонки в strict экспорте могли отображаться в другом порядке, чем в distribution экспорте для одного и того же каталога, вызывая confusion.

**Root cause**: В strict экспорте запрос `select(catalog_items).where(catalog_items.c.catalog_id.in_(catalog_ids))` выполнялся без `.order_by()`. В distribution экспорте была полная логика сортировки.

### Решение

В `export_orders_template_strict_xlsx` (line ~2768) добавлена идентичная сортировка:

```python
order_cols = [catalog_items.c.id.asc()]
if _table_has_column(catalog_items, "position_order"):
    order_cols = [
        catalog_items.c.position_order.is_(None),
        catalog_items.c.position_order.asc(),
        catalog_items.c.id.asc(),
    ]
catalog_item_rows = db.execute(
    select(catalog_items)
    .where(catalog_items.c.catalog_id.in_(catalog_ids))
    .order_by(*order_cols)
).mappings().all()
```

**Результат**: обе функции теперь гарантированно выдают товарные позиции в одинаковом порядке (по position_order, затем по id). Это исключает иллюзию разницы между strict и distribution экспортами и упрощает оператору сравнение структуры выгрузок.

## 2026-05-XX — reverse pack_hint: вес пользователя → количество упаковок

**Context**: Пользователь писал "картошка фри 2.5 кг" для товара с `unit_hint="уп"` и `pack_hint="2.5 кг"`. Парсер видел `unit="кг"` и не применял никакой конвертации (прямая конвертация требует `unit in ("уп","шт")`), что приводило к "2.5 кг" в экспорте при ожидаемом "1 уп".

**Root cause**: `_normalize_qty_for_catalog()` имела только прямую конвертацию (уп/шт × pack_weight → weight). Обратная (weight / pack_weight → count) отсутствовала.

### Решение

Добавлен `elif`-блок в `_normalize_qty_for_catalog()` (`backend/app/domain/order_domain.py`):
- Срабатывает когда `unit in ("кг","г","л","мл")` (пользователь явно написал вес/объём) **И** `unit_hint in _DEFAULT_SINGLE_QTY_UNITS` ("уп","шт","банка") **И** `catalog_item.pack_hint` задан.
- Оба значения нормализуются в базовые единицы (г или мл) через `to_base_unit()`, затем `count = user_weight / pack_weight`.
- Примеры: "2.5 кг" при pack_hint "2.5 кг" → "1 уп"; "5 кг" → "2 уп"; "200 г" при pack_hint "200 гр" → "1 уп".

**Ограничение**: голое число без единицы (напр. "2.5") при `unit_hint="уп"` по-прежнему интерпретируется как "2.5 уп" — это недетерминированный случай (может означать и кол-во паков, и вес).

## 2026-05-XX — respect_user_aliases: пользователь может удалять авто-алиасы через UI

**Context**: При сохранении товара в каталоге через PATCH `/catalogs/{id}/items/{item_id}` с явно указанным полем `aliases` функция `_merge_aliases()` всегда добавляла автоматически сгенерированные алиасы из `_build_auto_aliases(title)`. Это означало, что пользователь не мог удалить широкий alias (например, "нерки" у позиции "ИКРА НЕРКИ с/м 0.2"), т.к. система его тут же возвращала.

**Root cause**: `_build_auto_aliases("ИКРА НЕРКИ с/м 0.2")` возвращает список слов из title, включая "нерки". `_merge_aliases` всегда добавлял auto_parts поверх user_parts.

### Решения

1. **`_merge_aliases(respect_user_aliases=True)`**: добавлен параметр. Когда `True` и `user_aliases is not None`, auto_parts не добавляются. Пользователь получает ровно то, что написал (плюс исторические алиасы).

2. **PATCH endpoint**: передаёт `respect_user_aliases=(payload.aliases is not None)`. Если пользователь явно поставил aliases = "...", auto-aliases не добавляются. Если aliases не пришёл в PATCH (пользователь обновлял только price/unit), поведение прежнее.

3. **Data fix**: `ИКРА НЕРКИ с/м 0.2` (catalog_id=9, id=1850) — alias "нерки" удалён напрямую из DB. Алиас был корнем бага "6 штук нерки" в заказе 3342: три разные строки ("Φиле нерки 4 б", "Нерка слабосоленая 1 шт", "Икра нерки 1 б") одновременно матчились в один item через слишком широкий единственный alias, давая агрегат 4+1+1=6 при показе в admin-ui.

### Consequences

- Пользователь может убрать авто-генерированный alias через UI — он ОСТАЁТСЯ убранным.
- Предыдущее поведение (auto-aliases) сохраняется, если пользователь не трогает поле aliases в PATCH.
- Для CREATE endpoint поведение не изменилось (там не передаётся `respect_user_aliases`).

## 2026-05-XX — price visibility: showPrice=false по умолчанию; gate в LinkAliasDialog

**Context**: `ui_show_price = "false"` в `bot_settings`, но цена отображалась в каталогах из-за двух ошибок:
1. `useState(true)` в CatalogsPage → flash показа цены до загрузки settings
2. В `LinkAliasDialog` `option.price_text` рендерился без проверки `showPrice`

### Решения

- `CatalogsPage`: `useState(false)` для `showPrice` — безопасный дефолт (скрыто до подтверждения из settings).
- `LinkAliasDialog`: добавлен проп `showPrice?: boolean` (дефолт `false`); `option.price_text` под guard.
- `.catch(() => {})` заменён на `.catch((err) => console.warn(...))` чтобы ошибки загрузки settings были видимы.

## 2026-05-07 — roe-class guard: "Нерка сл.сол" и "Нерка" больше не матчатся в "ИКРА НЕРКИ"

**Context**: Три живых заказа: 505 (Ирина 2261), 514 (Татьяна 4076) и 513. Все три содержали строки с нерки, которые матчились неверно. В предыдущей итерации был добавлен title-class conflict-check (`fillet vs roe` → block), что починило `Φиле нерки → ИКРА НЕРКИ`. Но оставался кейс: `Нерка сл.сол` → тоже матчилась в ИКРА НЕРКИ через alias `нерки` (species-name без продуктового класса) в alias-loop и через step-5 word-overlap.

### Решения

1. **Alias loop (step 0)**: добавлен roe-специфический guard — если title продукта имеет класс `roe` ("икра"), а fragment и alias не несут `roe` класс, alias-match блокируется при score < 900. Намеренно узкий: только `roe` требует явного упоминания "икра" в запросе; другие классы (`fillet`, `canned`, `steak`) — это лишь способ приготовления, клиент вправе писать только вид рыбы.

2. **Step-5 word-overlap**: тот же guard применён в дешёвом fuzzy-scoring. `Нерка сл.сол` больше не накапливала score через shared word `нерка` с `Икра нерки`.

3. **Title-check refinement**: существующий `_has_match_conflict(frag, title)` чек теперь override-ается когда alias несёт тот же класс, что и fragment. Это устраняло ложный конфликт `Φиле нерки с палтусом малосольное` → `Ассорти нерка + палтус м/с` (fillet ≠ assorti → false conflict), который был pre-existing регрессией. Логика: если alias говорит то же, что и fragment, title′s класс не должен перебивать.

4. **Regression tests**: добавлен `test_nerka_slabosol_does_not_match_ikra_nerki_via_species_alias` в `test_parser_user_reported_regressions.py`.

### Consequences

- `Нерка сл.сол`, `Нерка свежая`, generic `нерка` — больше не попадают в `ИКРА НЕРКИ`.
- `Φиле нерки`, `икра нерки` — продолжают матчиться верно (через product-class check `roe` == `roe`).
- `Горбуша в с/с`, `Φиле нерки с палтусом малосольное` через ассорти — работают корректно (guards не применяются для non-roe классов).

## 2026-05-05 — parser ignores address shards like `д. 6`, and alias matching respects item class conflicts

**Context**: На живых заказах всплыли два неприятных silent-bad-parse кейса. Во-первых, строка заголовка вроде `Инесса 1921, Белякова, д. 6` разбивалась по запятым, и фрагмент `д. 6` ошибочно проходил как товарная строка (`title_raw="д"`, `qty=6`) с дальнейшим fuzzy-match в каталог. Во-вторых, alias-path мог матчить `Филе нерки` в товар `Икра нерки`, если у позиции икры был слишком общий alias вроде `нерки`: conflict-check смотрел только на alias-токены, но не на product class самого title товара.

### Решения

- В `backend/app/domain/order_domain.py::_is_meta_fragment()` добавлен явный guard для адресных осколков после comma-split: `д. 6`, `кв. 3`, `корп. 2`, `стр. 1`, `офис 5` и подобные фрагменты теперь всегда считаются metadata, а не товаром.
- В `backend/app/repo/catalog_repo.py::match_catalog_item_from_items()` alias-match теперь проверяет конфликт не только между fragment и alias, но и между fragment и `CatalogItem.title`. Это не даёт generic alias вроде `нерки` переопределять product class (`филе` vs `икра`).
- Добавлены регрессионные тесты на оба кейса в `backend/tests/test_parser_user_reported_regressions.py`.

### Consequences

- Адресные куски из header-line больше не протекают в `order_lines` как фейковые товары.
- Матчинг по alias остаётся гибким, но перестаёт склеивать разные продуктовые классы только из-за общего корня слова.

## 2026-05-01 — fuzzy template matching: 3-char abbreviations + empty-export guard

**Context**: Экспорт возвращал пустой файл по нескольким причинам:
1. Matcher требовал оба слова ≥ 4 символа для prefix-сравнения. Заголовки шаблона «СОС», «СИР», «КЕД» (3 символа) не матчились с «сосновом», «сиропе», «кедровый». В итоге «Брусника в сосновом сиропе» и «Кедровый орех в кедровом сиропе» давали score 0 и не попадали ни в одну колонку.
2. Generic-word guard (`bool(cg) != bool(hg) and shared_non_generic ≤ 1 → 0.0`) блокировал «Кедровый орех» ← «КЕД ОР В КЕД СИР шт» потому что «орех» ∈ GENERIC_WORDS, а header-аббревиатура «ОР» (2 символа) вообще не попадала в match_words.
3. Пустой экспорт (0 заказов под фильтр) возвращал валидный XLSX с одной строкой заголовков — оператор не понимал, баг это или пустой результат.

### Решения

- **`_template_word_similarity`**: добавлена ветка для 3-char prefix-matching (shorter ≥ 3, longer ≥ 5, `longer.startswith(shorter)`) → score 0.88.
- **`_template_similarity_score`**: generic-word guard теперь lazy-вычисляет fuzzy overlap для non-generic слов; если хотя бы одна пара слов fuzzy-матчится ≥ 0.78, guard не блокирует.
- **Penalty за size-variant headers**: заголовки с числами (0.5, 125 и т.п.), когда кандидат их не содержит, штрафуются ×0.85 в fuzzy-ветке — «ОРЕХ КЕДР 0.5 шт» больше не бьёт «КЕД ОР В КЕД СИР шт» для «Кедровый орех в кедровом сиропе».
- **Empty-export guard**: все три template-export эндпоинта (`/export-template`, `/export-template-strict`, `/export-template-distribution`) теперь возвращают HTTP 404 с понятным сообщением вместо пустого XLSX.

### Tests

- `test_template_header_matching_covers_reference_headers`: теперь проходит без падения для «Брусника» и компании (ранее 1 fail).
- Новые тесты: `test_template_similarity_3char_abbreviated_headers`, `test_abbreviated_header_beats_size_variant_for_same_product`, `test_empty_export_returns_404_not_empty_xlsx`.
- Переименован `test_distribution_export_with_catalog_and_chat_filters_returns_valid_empty_xlsx` → `*_returns_404_for_empty_result`.
- Итого: 55 passed, 0 failed.

### Consequences

- Больше продуктов с аббревиатурами в заголовках шаблона попадают в нужные колонки при экспорте.
- Пустой экспорт сразу виден как ошибка, а не как «прозрачный» пустой файл.
- Производительность не пострадала: fuzzy overlap считается только один раз на пару (candidate, header), computationally bounded O(n*m) где n,m ≤ 10 слов.



**Context**: На живых данных создавалось ощущение, что заказы за 06–20 апреля «пропали». Данные в БД были на месте, но UI страницы `Заказы` строил пагинацию эвристикой `data.length === PAGE_SIZE ? page + 1 : page`, а backend убирал технические `rejected`-строки уже после `limit/offset`. В итоге список мог показывать неполные страницы и не давать оператору ясного количества доступных заказов.

### Решения

- `GET /orders` теперь считает реальный total после всех бизнес-фильтров и отдаёт его в header `X-Total-Count`.
- Технические `rejected`-записи вида `message no longer qualifies as order` исключаются на SQL-уровне **до** расчёта total и до пагинации.
- `OrdersPage` переключён на серверный total и больше не угадывает число страниц по размеру текущего чанка.

### Consequences

- Список заказов показывает предсказуемую пагинацию даже на больших исторических диапазонах.
- Служебные rejected-сообщения больше не «съедают» места на страницах и не создают ложное впечатление потери заказов.

## 2026-04-28 — runtime parser no longer mutates catalog from admin-feed messages

**Context**: В runtime worker оставалась нарушающая архитектуру связка: startup/online catalog-heal читали admin-feed сообщения, находили «недостающие» товары и автоматически вставляли их в `catalog_items`. Это ломало ключевой инвариант проекта: каталог должен быть источником истины для парсера, а не объектом, который парсер/worker переписывает по сообщениям.

### Решения

- В `backend/app/worker/catalog_heal.py` и legacy worker убрана runtime-вставка товаров в каталог из admin-feed.
- Startup reparse сохраняется, но теперь он только перепарсивает сохранённые заказы по **текущему** каталогу (`reprocess_problem_orders(..., reprocess_all=True)`).
- Online admin-feed сообщения по-прежнему помечаются как служебные и не проходят как клиентские заказы, но больше не могут автоматически расширять каталог.
- Ручной sync/catalog maintenance остаётся отдельной операцией; parser использует уже существующие `CatalogItem` / aliases как source of truth.

### Consequences

- Runtime больше не создаёт скрытых каталожных drift-ов из чата.
- Если товар отсутствует в каталоге, заказ остаётся `partial`/`unknown_item` до явного обновления каталога, вместо тихого автодобавления позиции.
- Поведение parser/replay/export становится предсказуемым: сначала обновляется каталог, потом orders перепарсиваются относительно него.
- Debug/admin reprocess-инструменты тоже переведены в order-only режим: они больше не умеют досоздавать каталог из `tg_updates`, а manual `import-text` стал каноническим способом обновить каталог и сразу перепарсить заказы.

## 2026-04-28 — template exports keep original order message in the summary column

**Context**: В шаблонных XLSX-экспортах (`export-template`, `export-template-strict`, `export-template-distribution`) четвёртая колонка визуально выглядела как «Заказ / Состав заказа», но фактически туда записывался пересобранный summary по распарсенным `order_lines`. Для операторов это ломало ожидание: при ручном текстовом вводе или исторических alias-совпадениях экспорт показывал не исходное сообщение клиента, а синтетическую интерпретацию.

### Решения

- Для всех template-based export режимов колонка `Заказ / Состав заказа` теперь заполняется из `orders.raw_text`, если он есть.
- Синтетическая сборка по `order_lines` оставлена только как fallback на случай старых/неполных записей без `raw_text`.
- Товарные числовые колонки остаются catalog-driven и по-прежнему строятся по распарсенным строкам заказа, чтобы не потерять операционную раскладку по шаблону.

### Consequences

- XLSX показывает оператору ровно то сообщение, которое пришло от клиента, вместо «пересказа» системы.
- Ручные текстовые кейсы и исторические заказы стали понятнее при сверке с оригиналом, без потери catalog-driven числового экспорта.

## 2026-04-28 — catalog-backed template export no longer derives units from message/header text

**Context**: После ужесточения правил экспорта оставалась переходная ветка, где template/distribution export для catalog-backed строки всё ещё мог смотреть на внешний header (`1200 Г`) или формулировку сообщения и трактовать количество в этих единицах вместо единицы каталога.

### Решения

- Для любой строки, связанной с `catalog_item` / SKU, экспортное количество теперь считается только через каталог (`unit_hint`, `pack_hint`) и нормализованное значение строки заказа.
- Внешний Excel header и текст сообщения могут помогать только найти нужную колонку, но больше не участвуют в выборе единицы измерения в export.
- Старую header-aware qty conversion для catalog-backed строк убрали, чтобы исключить возврат к `1200` вместо `1.2 кг` и похожим искажениям.

### Consequences

- Шаблонный export стал единообразным: одинаковый SKU всегда попадает в XLSX в одной и той же единице каталога.
- Числовые фрагменты в названии товара и в заголовке шаблона перестали влиять на семантику количества.

## 2026-04-28 — exact-title distribution export now follows catalog units, not weight fragments in header text

**Context**: Повторная проверка реального distribution XLSX по апрельским каталогам показала смешение двух разных источников правды. Заголовки уже совпадали с каталогом 1:1, но export всё ещё местами ориентировался на числовые куски текста в header (`125ГР`, `1200 Г`) и мог отдавать несовместимые значения для дискретных позиций вроде `ФИЛЕ УГРЯ` (`уп`) как `1.5` или `0.5`.

### Решения

- Для template/distribution export закреплено правило: если товарная колонка совпадает с каталожным заголовком, источником истины считается каталог (`title`, `unit_hint`, `pack_hint`), а не весовые фрагменты, встроенные в текст названия.
- Нормализация количества в `admin_service/app/api/routers/orders.py` больше не должна silently переименовывать несовместимое mass-количество в дискретную единицу каталога.
- Для дискретных каталожных единиц (`шт/уп/банка`) export пишет только безопасно совместимые целые значения; несовместимые или дробные псевдо-упаковки не должны загрязнять числовую колонку.
- В проектных инструкциях и docs дополнительно зафиксировано, что source of truth для каталога — вставленный текст импорта, а после создания каталога вручную допускается расширять только алиасы.

### Consequences

- Distribution XLSX остаётся синхронным с каталогом не только по заголовкам, но и по единицам учёта.
- Вес/фасовка внутри названия товара остаются частью naming, а не скрытой командой переписать export в граммы.
- Операторы больше не получают ложные значения вроде `1.5` в колонке дискретной позиции, если безопасного преобразования к единице каталога нет.

## 2026-04-28 — April historical order audit hardened parser against inline metadata drift and descriptor splits

**Context**: Повторный аудит реальных заказов за 2026-04-06..2026-04-12 показал, что после базовых parser-fix ещё оставались «тихие» misparse-кейсы: typo в строке точки выдачи мог превращаться в товар, inline `дозаказ` без явного qty тянул в `title_raw` имя/телефон, а строка вида `Кета, мороженая 1 шт. Скумбрия ...` рвала descriptor-хвост от первой позиции и загрязняла вторую.

### Решения

- В `backend/app/domain/order_domain.py` добавлен fuzzy guard для строк точки выдачи, чтобы typo-адреса не проходили как товарные фрагменты.
- Inline-header cleanup расширен на `дозаказ`-строки, где после метаданных идёт только товарный хвост без явного количества.
- Merge-логика order fragments теперь умеет приклеивать descriptor + qty (`мороженая 1 шт`) обратно к предыдущему товару, если после этого сразу начинается следующая товарная строка.
- В `backend/app/repo/catalog_repo.py` закреплена нормализация сокращённых каталожных заголовков (`сем`, `медаль`, `заб`, `североатл`, и т.д.) против нормальной клиентской формулировки.

### Consequences

- Исторические «silent bad parse» кейсы перестают маскироваться под `active` заказы с неправильными строками.
- Реальный апрельский аудит заметно очищается: меньше `partial/unknown_item`, больше корректно собранных `active` заказов без ручного добора.

## 2026-04-28 — order classifier no longer accepts identity-only messages

**Context**: В живом чате обычные сообщения с одними метаданными клиента (`имя + последние 4 цифры`, иногда ещё и точка выдачи) могли проходить эвристику `looks_like_order(...)` без товарной части. Из-за этого нейтральный чат вроде `Елена 2063` ошибочно сохранялся как заказ.

### Решения

- В `backend/app/domain/order_domain.py` добавлен ранний guard: metadata без product-signal и без price-list структуры больше не считается заказом.
- Существующее исключение для коротких customer price-list сообщений оставлено рабочим: там по-прежнему нужен полноценный identity + продуктовые строки.
- Добавлены регрессионные тесты для `Елена 2063`, `Елена 2063 ТЦ Марс` и `Елена 2063 там сообщение обычное`.

### Consequences

- Обычные сообщения больше не попадают в `orders` только из-за имени/телефона.
- Реальные заказы с товарами, количеством и price-only customer list не теряют поддержку.

## 2026-04-28 — catalog heal now reparses active orders, not only problematic ones

**Context**: Исправления parser/matcher сами по себе не меняют уже сохранённые `order_lines`. В реальном сценарии это давало ложное ощущение «ничего не поменялось»: код уже умел лучше распознавать строки, но старые `active` заказы оставались со старой разметкой, а export продолжал опираться на устаревшие данные.

### Решения

- Startup catalog-heal в `backend/app/worker/catalog_heal.py` и legacy worker теперь вызывает `reprocess_problem_orders(..., reprocess_all=True)`, то есть перепарсивает и `active` заказы тоже.
- Online catalog-heal при появлении новых товаров из admin-feed расширяет репроцесс до `active` заказов, если каталог реально дополнился новыми позициями.
- Статистика `reprocess_problem_orders()` теперь считает `updated` по фактическому изменению состояния заказа/строк, а не только по смене верхнеуровневого статуса.

### Consequences

- После деплоя parser-fix или появления новых каталожных позиций корректировки действительно доходят до уже сохранённых заказов и экспорта.
- Логи heal-flow больше не занижают объём реально изменённых заказов, если статус остался `active`, но `order_lines` были переписаны.

## 2026-04-28 — parser now hardens OCR/typo edge-cases and keeps discrete catalog units in exports

**Context**: В живых заказах появились повторяющиеся проблемы: товарные строки с опечатками (`сельдб`, `укгря`, `вяленная`) не совпадали с каталогом, OCR-вариант `o.5кг` трактовался как `5 кг`, а дискретные позиции с `pack_hint` (например, икра) иногда конвертировались в граммы, что искажало выдачу и экспорт.

### Решения

- В parser normalizer (`backend/app/parser/text_parser.py`) расширена OCR-нормализация `o/о` перед десятичным разделителем: теперь поддерживаются и `,` и `.` (`o,5` / `o.5` → `0,5` / `0.5`).
- В catalog typo-map (`backend/app/repo/catalog_repo.py`) добавлены рабочие коррекции из живых кейсов: `сельдб→сельдь`, `укгря→угря`, `вяленная→вяленая`.
- В `_normalize_qty_for_catalog()` (`backend/app/domain/order_domain.py`) pack-конверсия (`уп/шт` × `pack_hint`) ограничена только весовыми/объёмными unit_hint (`кг/г/л/мл`). Для дискретных позиций (`уп/шт/банка`) количество сохраняется дискретным и не уходит в граммы.
- Базовый словарь product keywords дополнен частыми позициями (`омуль`, `угорь/угря`, `чука`) как fallback при слабом каталожном контексте.

### Consequences

- Снижен процент «тихих» промахов по живым опечаткам в заказах.
- Дробные веса из OCR не раздуваются в 10 раз.
- В Excel/выдаче дискретные позиции больше не искажаются в граммы из-за `pack_hint`.

## 2026-04-27 — operational export excludes technical non-order rejected messages

**Context**: В рабочем диапазоне 06–12 апреля в XLSX попадали сервисные записи со статусом `rejected` и ошибкой `message no longer qualifies as order`. Это не клиентские заказы, а технические сообщения из чата (например, отзывы/комментарии), которые не должны загрязнять операционный экспорт.

### Решения

- В `admin_service/app/api/routers/orders.py` добавлен фильтр `_is_technical_rejected_non_order(...)`.
- `GET /orders/export` исключает такие строки из выборки перед построением листов `Заказы`, `Позиции`, `Сводка по товарам`, `По точкам`.
- Для уже сохранённых проблемных заказов выполнен целевой `catalog-reprocess` и точечная нормализация алиасов каталога (кейс `Балык из карельс. форели`) для корректной повторной классификации.

### Consequences

- В операционном XLSX остаются только реальные заказы; технические rejected-сообщения не мешают выдаче.
- После репроцесса снижается доля `partial` из-за устаревшего парсинга старых сообщений.

## 2026-04-27 — unit normalization unified: catalog.unit_hint stores lowercase to match backend parser

**Context**: Catalog items imported from text (e.g., "КИЖУЧ с/м ШТ") now correctly normalize unit into backend-compatible lowercase format. Previously, units were stored uppercase ("КГ", "ШТ", "УП"), causing mismatch when evaluating parsed order quantities against catalog: backend parser normalizes to lowercase ("кг", "шт", "уп"), leading to unit_hint comparison failures in `_normalize_qty_for_catalog()`.

### Solution

- Changed `_UNIT_PATTERNS` in `admin_service/app/api/routers/catalogs.py` to return lowercase units:
  - "КГ" → normalize to "кг" (not "КГ")
  - "ШТ" → normalize to "шт" (not "ШТ")
  - "УП" → normalize to "уп" (not "УП")  
  - "Ж/Б" → "шт", "КАПС" → "уп"
- API returns uppercase display units (`_display_unit`) for UI/export; internally stored lowercase for consistency with parser.

### Consequences

- Catalog items now correctly match parsed orders regardless of quantity specification (implicit, explicit, from unit_hint);
- Products like Кижуч, Горбуша, etc. with missing qty now infer "1 шт" correctly;
- Excel export shows proper unit display (ШТ, КГ, etc.) via `_display_unit()` filter.

## 2026-04-17 — prebuild guard now blocks compose build/restart when catalogs or POST preflight regressions fail

**Context**: Повторяющиеся `500` на каталожных endpoint-ах (`/catalogs`, `/catalogs/{id}/items`) показали, что точечный фикс без обязательного prebuild-gate недостаточен: при следующем изменении несовместимость схемы могла вернуться и всплыть уже в UI.

### Решения

- В `scripts/run_checks.sh` добавлен новый режим `catalogs-preflight`:
  - `py_compile` для `admin_service/app/api/routers/catalogs.py`;
  - lifecycle regression `admin_service/tests/test_catalog_lifecycle.py` и
    legacy-schema regression `admin_service/tests/test_catalogs_legacy_schema_compat.py`.
- В `Makefile` добавлены:
  - `make check-catalogs-preflight`;
  - `make check-prebuild` (`check-post-preflight` + `check-catalogs-preflight`).
- `compose-build` теперь зависит от `check-prebuild`, поэтому `make compose-build`, `make compose-restart` и `make sync` не проходят при регрессиях preflight.
- Процедура закреплена в проектных инструкциях (`.github/copilot-instructions.md`) и эксплуатационной документации (`README.md`, `docs/08-admin-flows.md`).

### Consequences

- Регрессии по POST-контракту и legacy-schema каталогов ловятся до сборки/рестарта, а не после в UI.
- Сборочный процесс стал fail-fast по критичным runtime-сценариям.

## 2026-04-17 — catalogs read-path supports legacy `chats.tg_chat_id` and minimal `catalog_items` shape

**Context**: В боевых legacy-инсталляциях часть схем оставалась в старом формате: `chats` хранил публичный чат как `tg_chat_id` (без `chat_id`), а `catalog_items` мог не иметь `is_active` и stop-полей. Это приводило к runtime `500` на `GET /catalogs` и `GET /catalogs/{id}/items` при загрузке админ-страницы каталогов.

### Решения

- В `catalogs` router добавлен runtime fallback для публичного chat-id: сначала `chats.chat_id`, затем `chats.tg_chat_id`, и только потом fallback на `catalogs.chat_id`.
- Для read-path каталогов (`list/get/bootstrap`) добавлены безопасные optional-column выражения, чтобы отсутствие отдельных legacy-колонок не роняло endpoint.
- Для `GET /catalogs/{id}/items` добавлен fallback: если `is_active` отсутствует, API возвращает `is_active=1` и не добавляет фильтр `is_active` в SQL.
- Добавлен regression test на «very legacy» форму таблиц (`chats.tg_chat_id` + `catalog_items` без `is_active`), покрывающий оба endpoint-а.

### Consequences

- Страница каталогов перестаёт падать с `500` на старых схемах до ручной миграции.
- API-контракт остаётся стабильным: фронтенд продолжает получать `chat_id` и `is_active` даже на legacy БД.

## 2026-04-15 — catalog copy auto-deduplicates item codes to prevent create/clone runtime failures

**Context**: `POST /catalogs` (и clone flow) могут копировать позиции из прошлого каталога. В реальных данных встречаются дубли `sku/item_code` внутри source-каталога (legacy/import/manual drift), что приводило к runtime-падениям на вставке в новый каталог и к `500` в UI.

### Решения

- В `_copy_catalog_items` перед вставкой каждая позиция теперь проходит через `_ensure_unique_item_code(...)` для target-каталога.
- Если исходный код пустой/грязный, база для кода строится из title (`_build_item_code_base`), затем уникализируется suffix-логикой (`-01`, `-02`, ...).
- Добавлен regression test: при копировании source-каталога с дублями кодов новый каталог создаётся успешно, а коды в target становятся уникальными.

### Consequences

- Создание/клонирование каталога перестаёт падать на data edge-cases с дублями item code.
- Поведение остаётся backward-compatible для нормальных source-данных и legacy-схем.

## 2026-04-14 — catalog creation/stop flows ignore optional stop columns on legacy `catalog_items`

**Context**: После фикса чтения и text-import для старых схем оставался ещё один runtime-edge-case: `POST /catalogs` мог падать на legacy-таблицах `catalog_items`, где нет `stop_reason` и/или `stop_until`. Причина — copy/stop write-path всё ещё пытался писать в отсутствующие столбцы.

### Решения

- `catalogs` router теперь добавляет `stop_reason` и `stop_until` в insert/update payload только если эти колонки реально есть в отражённой таблице.
- Тот же guard применён к `POST /catalogs`, clone/copy flow и stop/unstop item flow.
- Добавлен regression test на `POST /catalogs` с legacy `catalog_items` (`item_code`, без `position_order`, без `stop_reason/stop_until`).

### Consequences

- Создание нового каталога с автокопированием из предыдущего перестаёт падать на старых БД.
- Stop-поля остаются backward-compatible до ручной миграции инсталляции.

## 2026-04-14 — Orders/Delivery Problems tabs become independently sysadmin-toggleable; catalog items API made legacy-schema compatible

**Context**: Операторы просили, чтобы «Проблемы» в первую очередь относились к разделу заказов, а в выдаче оставались опцией. Параллельно в runtime всплыли `500` на `GET /catalogs/{id}/items` и `POST /catalogs/{id}/import-text` на инсталляциях со старой схемой `catalog_items` (без `position_order` и/или с `item_code` вместо `sku`).

### Решения

- Добавлен отдельный sysadmin-toggle `orders_show_problems_tab` для раздела `Orders`.
- Существующий toggle `delivery_show_problems_tab` сохранён отдельно для `Delivery`.
- `catalogs` router в `admin_service` переведён на runtime-совместимость со старой схемой `catalog_items`:
  - fallback между `sku` и `item_code`;
  - fallback при отсутствии `position_order`;
  - fallback при отсутствии `stop_until`.
- Добавлен regression test на legacy-форму `catalog_items` для `GET /catalogs/{id}/items` и `POST /catalogs/{id}/import-text`.

### Consequences

- Sysadmin может отдельно управлять вкладкой проблем в заказах и в выдаче.
- Импорт и чтение позиций каталога перестали падать на старых БД до ручной миграции столбцов.

## 2026-04-14 — Settings allowlist synchronized with frontend sysadmin toggles (export modes + people visibility)

**Context**: В `DebugPage` и связанных view-model фронтенд отправляет системные toggle-ключи (`export_mode_*`, `people_show_*`) в `POST /settings`. Бэкенд-allowlist не содержал часть этих ключей, из-за чего реальные сохранения падали с `400 Invalid settings keys`.

### Решения

- Ключи `export_mode_template`, `export_mode_strict`, `export_mode_distribution`, `export_mode_flexible` и `people_show_*` добавлены в `SYSADMIN_UI_DEFAULTS`.
- За счёт существующей композиции это автоматически синхронизировало:
  - server allowlist (`ALLOWED_SETTINGS_KEYS`),
  - safe-набор для non-owner чтения (`NON_OWNER_SETTINGS_KEYS`),
  - дефолтные значения при `GET /settings`.
- Добавлен regression test на `POST /settings`, который проверяет сохранение полного набора этих toggle-ключей.

### Consequences

- Сохранение sysadmin toggle-настроек из текущего frontend-контракта больше не ломается на валидации ключей.
- Контракт frontend ↔ backend по системным toggle-ключам стал явным и покрыт автотестом.

## 2026-04-14 — POST preflight becomes mandatory for risky auth/mutation changes; device fingerprint binding gets deterministic fallback

**Context**: В реальных сессиях админки повторялась регрессия вида «POST снова падает после изменений» (особенно заметно на `/api/auth/login` и мутациях после cookie-login). Требовалась архитектурная защита не только кодом, но и процессом: чтобы risky изменения не проходили без обязательной preflight-проверки POST-контракта.

### Решения

- В `admin_service` auth-flow усилен fallback-механизмом для device binding:
  - если `X-Device-Fingerprint` отсутствует, сервер детерминированно выводит fingerprint из request metadata (IP + User-Agent);
  - та же логика используется при последующей проверке cookie-сессии, чтобы POST-поток не ломался из-за отсутствующего заголовка.
- Добавлен регрессионный тест на сценарий cookie-auth POST-мутации без явного `X-Device-Fingerprint`.
- В едином раннере проверок добавлен режим `post-preflight`, и в `Makefile` — команда `make check-post-preflight`.
- Процесс зафиксирован в агентных/проектных инструкциях: любые изменения в `login/auth/cookie/CSRF/middleware` и mutating API считаются risky и требуют обязательного preflight перед завершением.
- Добавлены `.llmignore` и `.copilotignore`, чтобы LLM/Copilot не тратил контекст на runtime-мусор и не затрагивал лишние артефакты.

### Consequences

- Риск повторных «тихих» поломок POST-контракта после локальных правок снижен за счёт обязательного узкого preflight-гейта.
- Авторизация стала устойчивее в окружениях, где `X-Device-Fingerprint` может не приходить стабильно.
- Проверка теперь встроена в стандартный operational flow (`make check-post-preflight`), а не зависит от ручной дисциплины.

## 2026-04-14 — catalog text import and admin visibility toggles are treated as operational controls, not one-off UI hacks

**Context**: Пользовательский сценарий для админки требует, чтобы оператор мог вставить сырой список позиций в каталог без ручного заведения каждой строки, при этом порядок позиций должен сохраняться строго как в исходном сообщении. Параллельно служебные поля (`SKU`, `ID каталога`, цена, исходный текст заказа, дата, проблемы, linked code) должны реально скрываться через настройки, а не просто существовать в конфиге без применения в боевых экранах.

### Решения

- `CatalogsPage` теперь монтирует `ImportTextDialog` и даёт явное действие **«Вставить текстом»** прямо в секции товаров каталога.
- Импорт позиций из текста закреплён как order-preserving flow: сортировка в каталоге по умолчанию использует `position_order`, а backend сохраняет `position_order` и при ручном создании, и при clone/import в другой чат.
- Скрытие служебных полей оформлено как реальные runtime-toggle правила:
  - `SettingsPage` загружает и сохраняет `orders_show_source`, `orders_show_order_id`, `orders_show_linked_code`, `ui_show_sku`, `ui_show_price`, `ui_show_order_date`, `ui_show_order_problem`, `ui_show_catalog_id`, `delivery_show_order_status`;
  - `CatalogsPage` и `OrdersPage` применяют эти флаги в live UI.
- Сортировка заказов по точке выдачи реализована серверно (`sort_by=pickup_place`) и прокинута во frontend-фильтры, чтобы оператор работал не только через группировку по чату.

### Consequences

- Создание каталога из pasted text стало штатным операционным флоу, а не скрытой заготовкой в коде.
- Порядок каталога стабилен между импортом, ручным добавлением и клонированием в другой чат.
- Sysadmin/owner может реально убирать внутренние поля из повседневной работы операторов без кастомных правок кода.
- Детали заказа и XLSX-выгрузки показывают единицы в source-style нотации (`КГ`, `Г`, `ШТ`, `УП`), чтобы оператор видел тот же формат, что и в исходных сообщениях.

## 2026-04-07 — distribution export switched to catalog-driven completeness over strict 1:1 template columns *(later superseded for distribution mode)*

**Context**: Пользователь подтвердил, что источник бизнес-данных для export — каталог/БД, а Excel-шаблон нужен только для внешнего вида. При режиме `export-template-distribution` часть позиций терялась, если шаблонная колонка не находилась 1:1.

### Решения

- Для `GET /orders/export-template-distribution` включён overflow fallback: если позиция из заказа/каталога не сматчилась в существующие template-колонки, справа добавляется новая колонка с названием позиции.
- Это поведение закрепляет правило «данные из каталога/БД важнее ограничений шаблонной сетки» и убирает тихие потери строк.
- Regression-тесты обновлены: теперь в distribution проверяется добавление overflow-колонки вместо `None` в несматченных кейсах.

### Consequences

- Distribution-файл может иметь дополнительные товарные колонки справа относительно исходного шаблона.
- При этом позиции больше не пропадают из выгрузки из-за несовпадения заголовков шаблона.

> Позже это решение было **отменено именно для `export-template-distribution`**: текущий инвариант — раздачный export не добавляет новые товарные колонки вне каталога и не должен выводить не-каталожные позиции.

## 2026-04-07 — strict/distribution template matching combines catalog and line candidates

**Context**: В выгрузках `export-template-strict` и `export-template-distribution` часть позиций могла не попадать в шаблонные колонки, хотя визуально заголовок в шаблоне соответствовал строке заказа. Кейс пользователя: `Сельдь олюторская с/м 5 кг` и `Филе хека (Аргентина) 1200 г` не записывались, когда каноническое название в каталоге имело другое обозначение фасовки/веса.

### Решения

- В strict/distribution подбор колонки теперь учитывает не только catalog-кандидаты, но и line-кандидаты (`title_raw/item_title/SKU`) в едином списке.
- Приоритет catalog-driven matching сохранён, но line-текст больше не отбрасывается, если canonical-нотация каталога и header-нотация шаблона отличаются.
- Добавлен regression test на сценарий различий в нотации (`... 1.2 кг` в каталоге vs `... 1200 г` в header).

### Consequences

- Меньше «тихих потерь» строк в distribution-экспорте.
- Позиции из каталога-раздачи надёжнее попадают в целевые колонки даже при разных вариантах записи веса/фасовки.

## 2026-04-07 — template-based Excel exports are flattened to a single clean worksheet

**Context**: Пользовательский экспорт из живого Excel-шаблона всё ещё унаследовал визуальный и структурный шум оригинального файла: в выдаче оставались лишние листы (`СТОП,ПРИЗЫ`, `Траты`, conflict-листы), цветные template-заливки в header/body и хвостовые куски данных/таблиц далеко за рабочей областью листа (например, в районе `AL207`). Пользовательское ожидание — один аккуратный лист без цвета и без мусора, но с корректными данными заказа и итоговой строкой.

### Решения

- Для template-based export добавлен workbook-pruning helper, который оставляет только первый worksheet перед финальной выдачей файла.
- После заполнения шаблонного листа вычисляется фактически используемая область и worksheet принудительно обрезается по реальным `last_row/last_col`.
- Финальная стадия экспортов (`template`, `strict`, `distribution`) теперь сбрасывает template-derived fills/conditional formatting/table-noise в plain-оформление:
  - без цветных header/body заливок,
  - с единым тонким border,
  - с читаемым bold header и plain body alignment.
- Добавлен regression test на сценарий: цветной шаблон + лишние листы + хвостовой мусор в `AL207`.

### Consequences

- Пользователь получает один чистый Excel-лист вместо набора наследованных template-sheet'ов.
- Цветовой шум исходного шаблона больше не протекает в экспорт.
- Хвостовые таблицы/пустые колонки вне реальной рабочей области удаляются до сохранения файла.

## 2026-04-07 — distribution template export keeps first-sheet layout strict and normalizes qty to catalog units

**Context**: В рабочем `ExportPage` пользователи формируют раздачный шаблон через `export-template-distribution`, где первая страница должна оставаться 1:1 с загруженным Excel-шаблоном. Фактически в этом режиме могли появляться лишние товарные колонки (overflow), а количества записывались без приведения к единице каталога (`unit_hint`), из-за чего в ячейках оказывались «чужие» единицы относительно каталожного учёта.

### Решения

- В шаблонной записи количества добавлено приведение к единице каталога через `_normalize_qty_for_catalog_item(...)` перед заполнением ячеек.
- Приведение учитывает `pack_hint`, поэтому сценарии вида `2 шт` при фасовке `500 г` и `unit_hint=кг` в выгрузке дают `1 кг`.
- Для `GET /orders/export-template-distribution` отключено авто-добавление overflow-колонок: первая страница теперь сохраняет 1:1 раскладку шаблона.
- Для `GET /orders/export-template-strict` поведение overflow сохранено (fallback-колонки допустимы), чтобы не терять позиции в strict-режиме, где это ожидаемый fail-safe.

### Consequences

- Раздачный шаблон перестал «расползаться» по колонкам на первой странице.
- Значения в товарных колонках синхронизированы с единицами каталога, что упрощает сверку и итоговую выдачу.

## 2026-04-07 — ExportPage переведена на view-model композицию и защищена от частичного API-контракта

**Context**: В `admin-web` страница экспорта снова росла как orchestration-монолит: загрузка пресетов/справочников, sysadmin visibility-режимы, валидация owner password, запуск двух export-flow и большой JSX жили в одном файле. Дополнительно в тестовом/частично деградирующем runtime-контуре падал рендер при отсутствии `getSettings()` на API-клиенте (TypeError в `loadPresets`).

### Решения

- Введён `pages/export/useExportPageModel.ts` как единый stateful слой страницы:
  - загрузка данных (`presets`, справочники, settings),
  - производные состояния (selected sheets, mode options),
  - бизнес-валидация и запуск `flexible`/`template` экспортов,
  - toggles групп/фильтров и update-функции конфигов листов.
- В `pages/export/components.tsx` выделен `TemplateExportCard`, чтобы убрать крупный inline-блок шаблонного экспорта из `ExportPage.tsx`.
- `ExportPage.tsx` оставлена как тонкий view-composition слой с wiring extracted-компонентов и hook-модели.
- `loadPresets` получил безопасную деградацию: если `apiClient.getSettings` недоступен (частичный mock/контракт), страница использует fallback `{}` вместо аварии.
- Добавлен регрессионный тест на сценарий отсутствующего `getSettings`.

### Consequences

- Экспортный экран проще сопровождать локальными изменениями без роста page-монолита.
- Падение страницы из-за частичного API-клиента устранено.
- Тестовый контур стал устойчивее к неполным мокам и фиксирует этот edge-case явно.

## 2026-04-07 — mobile operational cards should prefer calm hierarchy over bright status noise

**Context**: В мобильных карточках `OrdersPage`, `DeliveryPage`, `CatalogsPage` и `UsersPage` стало слишком много ярких сигналов одновременно: цветные status/meta-блоки спорили за внимание, а часть операторских полей (`статус заказа`, `связка`) нельзя было гибко скрыть через sysadmin-настройки. Пользовательский фидбек прямо указал, что в мобильной работе «не очень сразу понятно что куда», а статусы `открыт/закрыт` и пёстрые карточки «бросаются в глаза» сильнее, чем нужно.

### Решения

- Для `DeliveryPage` добавлен отдельный sysadmin-toggle `delivery_show_order_status`, чтобы показывать статус заказа только там, где он реально нужен оператору.
- Для `OrdersPage` добавлен отдельный sysadmin-toggle `orders_show_linked_code`, чтобы связка / код объединения не шумела в mobile-карточках, если сценарий этого не требует.
- Mobile-card presentation в `CatalogsPage` и `UsersPage` переведена на более спокойную visual language: outline-чипы, мягкие tinted surfaces вместо плотных цветных заливок, приглушённые акцентные карточки.
- В мобильных карточках заказов усилена иерархия: точка выдачи вынесена в отдельный meta-блок вместо смешивания с остальными полями.

### Consequences

- Sysadmin получил точнее управляемый mobile UX без необходимости прятать целые страницы или сценарии.
- На телефоне ключевые данные считываются быстрее, потому что цвет теперь помогает, а не перекрикивает структуру.
- UI стал ближе к operator-first модели: сначала ясность и рабочий фокус, потом декоративная сигнализация.

## 2026-04-06 — strict template export keeps 1:1 layout but appends overflow item columns

**Context**: Пользовательский reference-файл `samples/excel/template_headers_reference_2026-04-06.txt` показал, что живые шаблоны используют много сокращённых товарных заголовков (`БРУСНИКА В СОС СИР`, `КЕД ОР В КЕД СИР`, `ШИШКА МАРМ`, `ИКРА ДОЙ`, `СЕЛЬДЬ ОЛЮТ ...`). В режимах `export-template-strict` и `export-template-distribution` строки заказа, которые не находили точную или fuzzy-колонку, тихо пропадали из товарной части таблицы. Параллельно ширина всех товарных колонок была избыточной для листов 1:1, где в ячейках обычно лежат только короткие числа.

### Решения

- Вынесено общее разрешение товарной колонки в helper `_resolve_template_line_column()`.
- Для strict/distribution режимов включён безопасный fallback: если позиция не сматчилась с существующими заголовками, справа добавляется overflow-колонка с названием позиции вместо молчаливой потери данных.
- `_append_template_column()` теперь регистрирует новый заголовок через ту же normalization/match-схему, что и исходный шаблон, и копирует стили не только заголовка, но и уже подготовленных строк тела.
- В `_apply_template_layout(strict=True)` ширина товарных колонок после первых четырёх снижена до компактного numeric-friendly формата.

### Consequences

- 1:1-экспорт перестал терять заказанные позиции даже при неполном или устаревшем шаблоне.
- Добавленные на лету товарные колонки корректно участвуют в дальнейших match/fill и суммарной строке.
- Лист distribution/strict заметно компактнее по горизонтали и лучше подходит для ручной раздачи/сверки.

## 2026-04-07 — Security hardening, sysadmin-configurable UI, Excel export reliability

**Context**: Комплексный аудит выявил уязвимости безопасности и архитектурные слабости: SQL-инъекция через f-string в debug endpoint, timing-атаки на сравнение секретов, утечка памяти в rate limiter, LIKE wildcard injection, header injection в Content-Disposition. Также «Инструменты» жили в общих настройках вместо sysadmin-режима, экспортные режимы и поля в разделе «Люди» были жёстко захардкожены без возможности конфигурации, а шаблонный Excel-экспорт молча пропускал позиции без fuzzy-совпадения.

### Решения

**Security:**
- SQL injection fix: `debug.py test_catalog_match` переведён на параметризованные запросы (`:catalog_id`).
- Timing attack fixes: `deps.py` — `hmac.compare_digest` для service key, device fingerprint и CSRF token (было `!=`).
- Content-Disposition header injection: `orders.py _build_xlsx_response` — filename санитизируется, оборачивается в кавычки.
- Rate limiter memory leak: `main.py RateLimitMiddleware` — добавлена периодическая чистка stale buckets (каждые 5 мин).
- LIKE wildcard injection: `deliveries.py` — user input экранируется (`%`, `_`, `\`).
- Filesystem path leak: `debug.py _run_catalog_reprocess_script` — ошибка больше не раскрывает внутренний путь клиенту.

**UI Architecture:**
- «Инструменты» перенесены из `SettingsPage` (tab 5) в `DebugPage` (tab 5) с гейтом `sysadmin-only`.
- `AppRoutes.tsx`: `/templates` и `/excel-preview` теперь ведут на `/debug?tab=5`.

**Sysadmin Configurability:**
- `DebugPage SysadminControls` расширен 17 новыми полями: 4 export mode toggles (`export_mode_template`, `export_mode_strict`, `export_mode_distribution`, `export_mode_flexible`) + 7 people field toggles.
- `ExportPage` загружает настройки и динамически показывает/скрывает режимы шаблонного экспорта и flexible builder.
- `UsersPage` загружает настройки и скрывает кнопки (password reset, promote owner, delegate toggle, full details, regions) по конфигурации sysadmin.

**Excel Export:**
- `_fill_template_row`: при отсутствии fuzzy-совпадения вместо `continue` создаётся fallback-колонка из `catalog_item.title` / `item_title` / `title_raw` через `_append_template_column`.

### Consequences

- Закрыты 6 уязвимостей безопасности (1 HIGH, 3 MEDIUM, 2 LOW).
- Sysadmin получил полный контроль над видимостью экспортных режимов и полей в разделе «Люди» через единый UI в Debug > Sysadmin.
- Excel-экспорт больше не теряет позиции молча при отсутствии fuzzy-совпадения.
- Шаблонный экспорт и flexible builder можно полностью отключить через настройки.

## 2026-04-06 — Admin debug parser switched to production parser pipeline; parser analytics now respect chat scope

**Context**: В админке `DebugPage` endpoint `/debug/test-parser` жил на упрощённой самописной логике (regex + substring/fuzzy-lite), тогда как реальный runtime-парсинг заказов шёл через `backend/app/domain/order_domain.py` и DB-aware rules активного каталога / pickup places. Из-за этого debug-инструмент мог показывать другой результат, чем production worker. Дополнительно `AnalyticsPage` уже фильтровала дашборд по чату, но `/analytics/parser-accuracy` оставался глобальным и не совпадал с выбранным chat scope.

### Решения

- `admin_service /debug/test-parser` переведён на production parsing stack:
  - использует `backend.app.domain.order_domain.parse_order_text`, `looks_like_order`, `evaluate_order_lines`, `build_catalog_keywords`;
  - подмешивает реальные `pickup_places` и `catalog_items` из БД вместо локальной упрощённой эвристики;
  - умеет работать в scope выбранного каталога/чата, но сохраняет прежний response shape для `admin-web`.
- В `DebugPage` добавлен выбор открытого каталога для parser-теста, чтобы оператор тестировал тот же каталог, что участвует в боевом match/evaluation.
- `/analytics/parser-accuracy` расширен фильтрами `chat_id` и `catalog_id`; `AnalyticsPage` теперь передаёт выбранный `chat_id`, чтобы parser-метрики соответствовали текущему фильтру дашборда.
- Добавлены API-регрессии на:
  - production-backed parser test с alias pickup place + catalog scope;
  - chat-scoped parser accuracy.

### Consequences

- Debug UI больше не показывает «альтернативную реальность» по парсингу: администратор видит почти тот же pipeline, что и production runtime.
- Аналитика качества парсера стала пригодной для разборов по конкретному чату, а не только по всей базе целиком.
- Любые дальнейшие улучшения parser-domain автоматически становятся видны и в debug-инструментах админки без дублирования логики.

## 2026-04-06 — Worker decomposed into package modules; backend security middleware aligned; follow-up backlog fixed in docs

**Context**: Исторический `backend/app/worker.py` вырос в orchestration-монолит, где в одном файле смешивались polling loop, update dispatch, spam gating, callback flow, order processing, catalog healing и messaging/runtime helpers. Это повышало стоимость любых правок в runtime и делало worker слишком хрупким для дальнейшего развития. Параллельно основной backend-контур в `backend/app/main.py` отставал от уже усиленного admin-side security baseline и нуждался хотя бы в минимальном защитном middleware-слое.

### Решения

- `app.worker` переведён из одного файла в package `backend/app/worker/` с разбиением по зонам ответственности:
  - `loop.py` — polling loop, batch processing, update routing;
  - `order_handler.py` — order processing, smart recognition fallback, supplement/reply logic;
  - `command_router.py` — public/admin command routing;
  - `catalog_heal.py` — startup/online catalog healing и reparse problem orders;
  - `messaging.py`, `helpers.py`, `spam_handler.py` — transport/runtime helper слой.
- Legacy-монолит сохранён как `backend/app/_worker_legacy.py` как reference/archive, а публичный import path `app.worker` теперь обслуживается через совместимый package facade.
- В `backend/app/main.py` добавлен минимальный backend security middleware baseline:
  - per-IP rate limiting;
  - security headers middleware;
  - suspicious request blocking;
  - webhook payload size validation.
- Следующие крупные кандидаты на декомпозицию зафиксированы явно, чтобы не потерять architectural follow-up после worker-refactor:
  - `admin_service/app/api/routers/orders.py` (~3219 LOC);
  - `backend/app/domain/order_domain.py` (~1990 LOC).
- Следующие security follow-ups также зафиксированы как обязательный backlog следующей волны hardening:
  - добавить явные Pydantic request validation schemas для webhook payloads;
  - маскировать пароль/credentials в DB connection URL logging;
  - заменить broad `except Exception` на более узкие exception types там, где это влияет на observability и security reviewability.

### Consequences

- Worker runtime стал модульнее и безопаснее для локальных изменений: loop, order handling, commands, spam и catalog-heal теперь можно развивать отдельно.
- Backward compatibility для `app.worker` imports сохранена через re-export surface, что уменьшает риск поломки тестов и legacy call sites.
- Backend security posture выровнен лучше прежнего, но validation / secret masking / exception hygiene остаются отдельной следующей фазой, а не считаются уже закрытыми.
- Архитектурный backlog теперь закреплён в документации, а не живёт только в контексте текущей сессии.

## 2026-03-26 — Order parser policy: no chat-specific hacks, only reusable learning from admin data

**Context**: Пользователь зафиксировал архитектурное требование для бота: парсинг заказов не должен чиниться точечными чат-специфичными костылями под отдельные сообщения. Источник истины для доступных товаров — активный каталог, собранный из admin-side сообщений в чате. Источник истины для точек выдачи — только точки, заведённые нами в `pickup_places`. Распознавание пользовательских заказов должно улучшаться через общие механизмы нормализации, fuzzy matching и alias derivation внутри этих справочников, а не через ручные if/else под конкретный кейс.

### Решения

- Любые улучшения парсера должны быть переносимыми между чатами:
  - использовать активный каталог и вручную заведённые pickup points как единственный source of truth;
  - расширять общую нормализацию текста, similarity scoring и alias derivation;
  - избегать жёстко пришитых chat-specific/product-specific веток, если их нельзя объяснить как reusable rule.
- Пользовательские сообщения не должны создавать новые точки выдачи или новые товары сами по себе:
  - если текст не подтверждается `catalog_items`, позиция остаётся unmatched;
  - если текст не подтверждается `pickup_places`, точка выдачи остаётся unset.
- Реальные новые ошибки проверяются по live БД и свежим сообщениям, а не только по историческим sample-файлам.
- Если в данных обнаруживаются «мусорные» сущности (например, служебный текст, попавший в pickup places), парсер должен уметь их отфильтровывать как класс, а не добавлять встречный костыль.

### Consequences

- Парсер развивается как доменный слой, а не как набор исключений под текущую выборку сообщений.
- Новые чаты и новые каталоги должны выигрывать от тех же улучшений без ручной донастройки.

## 2026-03-25 — PeopleAnalyticsPanel: починка данных аналитики и полнота ролей

**Context**: Панель «Аналитика людей» (`PeopleAnalyticsPanel`) визуально рендерилась, но все пользователи отображались как «Нет входа», потому что backend-эндпоинт `/users` не возвращал поля `created_at` и `last_login`. Кроме того, `ROLE_LABELS` не содержал роли `sysadmin` и `viewer`, и они показывались как raw code-строки.

### Решения

- Backend:
  - `UserResponse` в `schemas.py` расширен полями `created_at: str | None` и `last_login: str | None`.
  - `_build_user_response()` теперь заполняет `created_at` из `AdminUser.created_at` и `last_login` из `admin_login_attempts` (последний успешный логин).
  - Добавлен `_load_last_logins()` для batch-запроса `MAX(created_at)` по `LoginAttempt`, что избавляет от N+1 в `list_users`.
- Frontend:
  - `PeopleAnalyticsPanel.tsx`: `ROLE_LABELS` дополнен `sysadmin: "Сисадмин"` и `viewer: "Наблюдатель"`; цвет pie-chart для sysadmin — `error.dark`.
  - `WorkforcePage.tsx`: `ROLE_LABELS` синхронизирован с тем же набором ролей.

### Consequences

- Графики логинов (7д/30д/90д/Старше/Нет входа) теперь корректно распределяют пользователей.
- Список «Недавняя активность» показывает реальные даты входа.
- Sysadmin и viewer отображаются с человекочитаемыми подписями.

## 2026-03-24 — Sysadmin elevated to real owner-superset RBAC

**Context**: Первая итерация роли `sysadmin` покрыла settings/UI и добавила owner-side view mode, но в реальном RBAC система оставалась перекошенной: многие backend-роуты и page-local frontend guards всё ещё проверяли только `owner`. В результате sysadmin видел часть интерфейса, но не получал фактический доступ ко всем owner-сценариям.

### Решения

- Backend:
  - `is_owner()` в `admin_service/app/api/common.py` теперь трактует `sysadmin` как owner-equivalent роль для всех legacy owner-checks.
  - `can_manage_deliveries()` расширен до `owner/sysadmin/pickup_admin`.
  - `sysadmin` добавлен в `DEFAULT_ROLES` и в `_has_admin_access()` внутри `admin_sync.py`.
- Frontend:
  - `AppRoutes.tsx` больше не вычитает `owner` из effective roles при owner→sysadmin view mode; режим теперь **добавляет** sysadmin-возможности вместо их симуляции ценой потери owner-доступа.
  - Маршрут `/export` и shell-навигация (`MainLayout`) теперь используют секционные permissions с `sysadmin` наравне с `owner`.
  - `OrdersPage`, `UsersPage`, `CatalogsPage` переведены с локальных `isOwner`-ограничений на owner/sysadmin access для delete/export/full-details/catalog management сценариев.
  - Targeted tests обновлены под новую модель доступа.

### Consequences

- `sysadmin` стал реальной superset-ролью по отношению к `owner`, а не только UI-режимом просмотра.
- Legacy owner-only backend guards продолжают работать без массового переписывания роутеров, потому что shared helper теперь отражает актуальную бизнес-модель.
- Owner view mode остаётся полезным как UX-инструмент, но больше не конфликтует с фактической иерархией прав.

## 2026-03-24 — Sysadmin role, UI toggle system, DeliveryPage customization, scroll-to-top & error report

**Context**: Система ролей не поддерживала разделение между владельцем бизнеса (owner) и техническим администратором. Системные настройки были доступны только owner'у, операторам delivery-страницы не хватало гибкости в интерфейсе, а глобальная навигация не имела «наверх» и «сообщить об ошибке».

### Решения

- **Роль `sysadmin`**:
  - Backend: добавлены хелперы `is_sysadmin()`, `is_owner_or_sysadmin()` в `common.py`; `is_admin_or_above()` включает `sysadmin`.
  - Settings router: endpoint `PUT /settings` теперь принимает `is_owner_or_sysadmin`.
  - Frontend: `sysadmin` добавлен в `ACTIVE_ROLE_CODES`, `ROLE_LABELS`, `ROLE_DESCRIPTIONS`, `DEFAULT_SECTION_PERMS`.
  - Добавлен client-side `role view mode`: owner может переключиться в `sysadmin`-режим из user menu / Settings, чтобы видеть интерфейс и доступы глазами sysadmin без изменения своих реальных ролей в БД.

- **UI Toggle Settings (управляются sysadmin)**:
  - 13 новых ключей: `delivery_show_transferred_btn`, `delivery_show_not_delivered_btn`, `delivery_show_partial_btn`, `delivery_show_raw_text`, `delivery_show_problems_tab`, `delivery_show_composition`, `delivery_show_order_id`, `delivery_show_order_date`, `delivery_simplified_mode`, `settings_tab_vis_app`, `settings_tab_vis_interface`, `settings_tab_vis_system`, `settings_tab_vis_tools`.
  - SettingsPage: 8-я вкладка «Сисадмин» с двумя карточками: UI выдачи + видимость вкладок настроек.
  - Видимость вкладок управляется sysadmin через `TAB_VIS_MAP`; сам sysadmin видит все вкладки.

- **DeliveryPage кастомизация**:
  - Удалён текст описания под заголовком; кнопка «Обновить» перенесена в панель вкладок.
  - Вкладка «Проблемы» стала опциональной (`showProblemsTab`).
  - Колонки (ID, состав, raw text, дата) и кнопки (частично/передан/не выдан) теперь переключаемые через `uiSettings`.
  - `simplifiedMode` скрывает дополнительные столбцы для упрощённого интерфейса.

- **Scroll-to-top + Сообщить об ошибке** (MainLayout):
  - Floating Fab «наверх» появляется при скролле > 300px, плавно скроллит вверх.
  - Fab «Сообщить об ошибке» открывает Dialog с TextField; отправляет POST `/feedback` (пока placeholder, будет подключен к Telegram).

- **Каталог Балашиха/Железнодорожный**:
  - Создан `backend/scripts/seed_catalog_balashiha.py` с 36 позициями из chat export.
  - Точки выдачи: Белякова 6, ТЦ Марс (Железнодорожный).
  - Бот настроен в режиме `hidden` (молчаливый сбор заказов).

### Consequences

- Owner'ы освобождены от системных настроек; sysadmin получает полный контроль над UI и системой.
- Owner может безопасно проверить sysadmin-only сценарии без ручной переназначки ролей и без риска потерять owner-доступ.
- DeliveryPage стала гибкой: операторы видят только нужные столбцы и кнопки.
- Глобальная навигация улучшена scroll-to-top и формой обратной связи.
- Новый рынок (Балашиха) подготовлен для запуска с полным каталогом и точками выдачи.

## 2026-03-23 — Smart item add flow with templates, auto item-code, and clear SKU wording

**Context**: После smart-create каталога ручной труд оставался в самой частой операции — добавлении новой позиции. Оператору приходилось заново набивать карточку товара (код, единица, фасовка, цена, алиасы), хотя в истории чата уже были похожие позиции. Параллельно пользовательская формулировка `SKU` была слишком технической и вызывала лишние вопросы у операторов.

### Решения

- Backend (`catalogs` router + schemas):
  - добавлен endpoint `GET /catalogs/{catalog_id}/item-templates` для выдачи шаблонов позиций из каталогов того же чата;
  - create/update позиций теперь принимают совместимые поля `sku` и `item_code`;
  - добавлена генерация/нормализация уникального кода товара при пустом ручном вводе;
  - в ответах сохранена обратная совместимость: возвращаются и `sku`, и `item_code`.
- Frontend (`CatalogsPage` + dialogs + utils):
  - в диалоге «Добавить товар» добавлен автокомплит по шаблонам прошлых каталогов;
  - при выборе шаблона автоматически подставляются `title`, `unit_hint`, `pack_hint`, `price_text`, `aliases` и код товара;
  - добавлены `buildItemCodePreview()` и `suggestAliases()` для автогенерации кода и «умных алиасов»;
  - пользовательские подписи в формах/таблицах переведены на термин «Код товара (артикул)» вместо `SKU`.
- UX order editing:
  - в `OrderEditDialog` подсказки и placeholders синхронизированы с новой терминологией (код товара вместо SKU), чтобы поведение было единым между «Каталогами» и «Заказами».

### Consequences

- Добавление новой позиции ускорено: оператор чаще выбирает «шаблон + правка исключений», а не вводит всё с нуля.
- Единая формулировка «Код товара» снижает когнитивную нагрузку для не-технических пользователей.
- API остаётся совместимым с существующими клиентами благодаря dual-field контракту `sku`/`item_code`.

## 2026-03-23 — Smart catalog bootstrap + one-click distribution launch + auto-alias hardening

**Context**: Перед каждой новой раздачей оператору приходилось повторно вручную создавать каталог, переносить ассортимент и отдельно вручную выбирать чаты для `distribution_mode`. При постоянном обновлении ассортимента это увеличивало операционную нагрузку и вероятность ошибок, а parser требовал ручного ввода большого списка aliases для новых/вариативно называемых позиций.

### Решения

- `CatalogsPage` переведена на smart-create flow:
  - выбор основы из предыдущих каталогов текущего чата;
  - автокопирование позиций из source-каталога;
  - авто-закрытие существующего open-каталога в целевом чате;
  - предзаполнение полей create-диалога и preview количества копируемых позиций.
- `POST /catalogs` расширен опциями `source_catalog_id`, `copy_items_from_source`, `close_existing_open`.
- Добавлен `GET /catalogs/bootstrap-options` для получения рекомендуемой базы каталога по чату.
- `SettingsPage → Точки` получила one-click автоподготовку раздачи: чаты с открытыми каталогами автоматически подставляются в `distribution_mode_chat_ids`, а `distribution_mode` включается одним действием.
- В `catalogs` router включена авто-нормализация/автогенерация aliases при create/update позиции, включая распространённые сокращения (`с/м`, `м/с`, `х/к`, `г/к`, `с/с`) и keyword-обогащение для parser-слоя.

### Consequences

- Новый каталог и запуск выдачи теперь требуют существенно меньше ручных шагов и меньше подвержены operator-error.
- Повторное использование исторических каталогов стало встроенным сценарием, а не отдельной ручной процедурой.
- Parser устойчивее к пользовательским сокращениям и вариативным названиям новых товаров без обязательного ручного заполнения каждого alias.

## 2026-03-23 — AnalyticsPage декомпозирована в модуль `pages/analytics/*`

**Context**: `AnalyticsPage.tsx` накапливала mixed-слой: owner-only orchestration (загрузка и фильтрация данных по чатам) смешивалась с крупными inline UI-хелперами (`StatCard`, fallback bar chart, status-distribution блок, collapsible section wrapper) и статусными цветами. Это увеличивало размер page-файла и усложняло локальные правки аналитического UI.

### Решения

- Добавлен модуль `admin-web/src/pages/analytics/*`:
  - `types.ts` — типы карточек и распределения статусов;
  - `constants.ts` — `STATUS_COLORS` и `PIE_COLORS`;
  - `components.tsx` — `StatCard`, `SimpleBarChart`, `StatusDistribution`, `AnalyticsSection`.
- `AnalyticsPage.tsx` оставлена orchestration-страницей: запросы API, фильтрация по чату, вычисления summary/series и wiring графиков/секций.

### Consequences

- Owner-only аналитика стала проще для сопровождения: UI-компоненты и константы меняются локально без перегрузки page-файла.
- Снижен риск регрессий при следующих итерациях по аналитическим виджетам и layout-рефактору.

## 2026-03-23 — PickupAdminPage: старт модульной декомпозиции (`pages/pickupAdmin/*`)

**Context**: `PickupAdminPage.tsx` всё ещё содержит крупные mobile/desktop блоки для назначений и точек выдачи, плюс inline-диалог назначения и UI-утилиты. Для поэтапного безопасного рефактора выбран incremental-подход: сначала вынести low-risk слой без изменения поведения таблиц.

### Решения

- Добавлен модуль `admin-web/src/pages/pickupAdmin/*`:
  - `types.ts` — типы страницы и диалога назначения;
  - `utils.tsx` — `renderMobileMetaCard` и helper подсчёта назначений по точке;
  - `dialogs.tsx` — `AssignPickupAdminDialog`.
- `PickupAdminPage.tsx` переведена на импорт этих модулей, сохранив текущий UX и API-контракт.

### Consequences

- Страница стала тоньше и менее связной без рискованной массовой переклейки таблиц.
- Подготовлен безопасный фундамент для следующей волны extraction (assignments/pickup places mobile+desktop blocks в components-layer).

## 2026-03-23 — PickupAdminPage: extraction tab-блоков в `pickupAdmin/components.tsx`

**Context**: После первого шага декомпозиции (`types/utils/dialogs`) основной вес `PickupAdminPage` оставался в inline-табах: mobile/desktop рендер назначений и точек выдачи по-прежнему жили в page-файле.

### Решения

- Добавлен `admin-web/src/pages/pickupAdmin/components.tsx` с extracted UI-блоками:
  - `PickupAdminSummaryCards`;
  - `AssignmentsSection` (mobile + desktop);
  - `PickupPlacesSection` (форма добавления + mobile + desktop).
- `PickupAdminPage.tsx` переведена на orchestration-подключение этих компонентов, при сохранении существующего поведения handler-ов и confirm-flow.

### Consequences

- Основной page-файл стал значительно компактнее и проще для поддержки.
- Крупные UI-блоки вкладок теперь можно менять отдельно от data-loading/permission-логики.
- Следующие итерации (например, unit-тесты и дополнительные sub-components) можно делать локально в `pickupAdmin/components.tsx`.

## 2026-03-23 — ExportPage декомпозирована в модуль `pages/export/*`

**Context**: `ExportPage.tsx` содержала в одном файле шаблонные карточки, группировку листов, per-sheet фильтры, глобальные фильтры, preview выбранных листов и orchestration export-flow (password confirmation + build). Такой слой рос как UI-heavy монолит и усложнял точечные правки в builder-части.

### Решения

- Добавлен модуль `admin-web/src/pages/export/*`:
  - `types.ts` — типы конфигурации листа и групп листов;
  - `constants.ts` — иконки/порядок групп export-листов;
  - `utils.ts` — группировка листов и helper скачивания blob;
  - `components.tsx` — шаблонные карточки, selector листов, карточка глобальных фильтров и preview выбранных листов.
- `ExportPage.tsx` оставлена orchestration-страницей: загрузка пресетов/справочников, состояние фильтров и запуск export-build.

### Consequences

- Export builder проще расширять без правок в длинном page-файле.
- UI-блоки выбора/фильтров стали переиспользуемыми и удобнее для локальных изменений.
- Структура страницы согласована с уже внедрённым паттерном feature-модулей (`settings`, `orders`, `delivery`, `catalogs`, `users`).

## 2026-03-23 — UsersPage декомпозирована в модуль `pages/users/*`

**Context**: `UsersPage.tsx` оставалась следующим крупным монолитом после `CatalogsPage`: в одном файле жили role-константы, поиск/сортировка, mobile/desktop представления, редактирование ролей, регионы, password reset, owner promotion и create-user flow. Любая локальная правка цепляла сразу и data wiring, и JSX-heavy UI.

### Решения

- Добавлен модуль `admin-web/src/pages/users/*`:
  - `types.ts` — типы ролей, формы создания пользователя и props extracted-блоков;
  - `constants.ts` — assignable role set и display/color-конфиги ролей;
  - `utils.tsx` — поиск, сортировка, форматирование дат и mobile meta-card helper;
  - `components.tsx` — мобильная сетка карточек и desktop-таблица пользователей;
  - `dialogs.tsx` — диалоги create user, edit roles, regions, password reset и owner promotion.
- `UsersPage.tsx` оставлена orchestration-страницей: загрузка данных, state/handlers, permission guards и wiring extracted UI-модулей.
- Публичный контракт сохранён: `UsersPage` по-прежнему поддерживает `embedded`-режим для `PeoplePage`.

### Consequences

- Изменения в user-management UI теперь локализуются по feature-слоям, а не в одном 1000+ строковом файле.
- Mobile/desktop surface и диалоги можно развивать независимо от API-загрузки и permission-логики.
- Риск повторного повреждения orchestration-файла при следующих refactor-итерациях заметно снижается.

## 2026-03-23 — CatalogsPage декомпозирована в модуль `pages/catalogs/*`

**Context**: `CatalogsPage.tsx` оставалась крупным orchestration-монолитом: список каталогов, рендер mobile/desktop представлений, CRUD товаров, импорт каталога в чат и диалоги жили в одном файле. Это усложняло точечные изменения и делало страницу хрупкой при дальнейших UI-итерациях.

### Решения

- Добавлен модуль `admin-web/src/pages/catalogs/*`:
  - `types.ts` — типы каталогов, товаров и формовых состояний;
  - `constants.ts` — статусные конфиги каталогов;
  - `utils.ts` — форматирование дат и preview генерации кода каталога;
  - `components.tsx` — mobile/desktop render-блоки и секция товаров каталога;
  - `dialogs.tsx` — диалоги создания каталога, CRUD товара, stop-листа и импорта в другой чат.
- `CatalogsPage.tsx` пересобрана как тонкая orchestration-страница: загрузка данных, state/handlers и wiring extracted-компонентов без изменения текущего UX и API-контракта.

### Consequences

- Каталожный экран стал устойчивее к локальным правкам и проще для дальнейшего дробления по feature-компонентам.
- CRUD/clone-диалоги и таблицы теперь можно менять отдельно от загрузки данных и permission-логики.
- Проверки после восстановления страницы: целевой `eslint` и production build `admin-web` проходят.

## 2026-03-23 — DeliveryPage декомпозирована в модуль `pages/delivery/*`

**Context**: `DeliveryPage.tsx` оставалась крупным монолитом: в одном файле смешивались статусные конфиги, типы форм/черновиков, JSON-парсинг выдачи, форматтеры, построение payload и UI-хелперы таблиц/мобильных meta-card. Это повышало риск регрессий при локальных изменениях выдачи.

### Решения

- Добавлен модуль `admin-web/src/pages/delivery/*`:
  - `types.ts` — типы draft-структур и delivery status;
  - `constants.ts` — статусные конфиги (`DELIVERY_STATUS_CONFIG`, `ORDER_ERROR_STATUS`) и `PAGE_SIZE`;
  - `utils.ts` — pure-утилиты для парсинга/сборки payload/форматирования/summary;
  - `components.tsx` — UI-хелперы `SyncedTableFrame` и `renderMobileMetaCard`.
- `DeliveryPage.tsx` переведена на импорты из `pages/delivery/*` без изменения текущего UX и API-поведения.

### Consequences

- Страница стала заметно тоньше и проще для сопровождения.
- Изолированные delivery-утилиты теперь можно развивать и тестировать отдельно от JSX-слоя.
- Проверки после рефактора: `eslint` (целевые файлы) + targeted Vitest (`MainLayout`, `OrdersPage`, `SettingsPage`) проходят.

## 2026-03-23 — OrdersPage декомпозирована в модуль `pages/orders/*`

**Context**: После декомпозиции app-shell и `SettingsPage` страница `OrdersPage` всё ещё содержала смешанный слой: UI + export-типы + статусные конфиги + утилиты формирования параметров/имени выгрузки в одном файле. Это усложняло поддержку и делало изменение экспортного flow рискованным.

### Решения

- Добавлен модуль `admin-web/src/pages/orders/*`:
  - `types.ts` — типы export flow (`ExportMode`, `ExportFilterState`, defaults);
  - `constants.tsx` — статусные конфиги заказа/выдачи и `PAGE_SIZE`;
  - `utils.ts` — `formatDate`, `buildExportParams`, `buildExportFilename`.
- `OrdersPage.tsx` переведена на импорты из `pages/orders/*`, с сохранением текущего UI/поведения.

### Consequences

- Логика страницы стала более слоистой, а экспортный поток — легче для сопровождения и тестирования.
- Локальные изменения в форматах/правилах экспорта теперь меньше затрагивают JSX-слой.
- Проверки после рефактора: `eslint src` + targeted Vitest (`OrdersPage`, `SettingsPage`, `MainLayout`) проходят.

## 2026-03-23 — SettingsPage выделена в модульный слой `pages/settings/*`

**Context**: После декомпозиции app-shell `SettingsPage.tsx` оставалась самым тяжёлым фронтовым экраном: в одном файле одновременно жили типы, дефолтные настройки, role normalization, preview-логика и tab-layout утилиты. Это усложняло поддержку и создавало высокий риск регрессий при локальных правках.

### Решения

- Создан модульный слой `admin-web/src/pages/settings/*`:
  - `types.ts` — типы `BotSettings`, `PickupPlace`, `PreviewMessageKind`, `SettingsTabLayoutItem`;
  - `constants.ts` — `DEFAULT_SETTINGS`, `BOT_TEXT_CONFIG`, `BOT_BUTTON_CONFIG`, role labels/descriptions и `normalizeRoleList`;
  - `preview.ts` — генерация live-preview сообщения и клавиатуры бота;
  - `tabLayout.ts` — утилиты нормализации/чтения layout вкладок с явными параметрами.
- `SettingsPage.tsx` переведена на импорты из этих модулей без изменения UX и API-контракта.

### Consequences

- Логика страницы стала слоистой и переиспользуемой: настройки/preview/tab layout теперь можно тестировать и менять отдельно.
- Файл страницы заметно упрощён для дальнейшего компонентного дробления.
- Валидация после изменений: `eslint src` + targeted Vitest (`SettingsPage`, `MainLayout`) проходят.

## 2026-03-23 — Admin-web shell декомпозирован на app-модули (contexts / permissions / routes)

**Context**: `admin-web/src/App.tsx` одновременно держал theme provider, auth callback flow, session-guard bootstrap, секционные права, feature flags и роутинг защищённой зоны. Это повышало связность: `MainLayout`, `PeoplePage`, `OrdersPage`, `SettingsPage` и тесты зависели от экспортов из `App.tsx`, из-за чего любой рефактор shell-логики затрагивал полпроекта.

### Решения

- Введён слой `admin-web/src/app/*` для app-level модулей:
  - `contexts.tsx` — `ThemeContext`, `OwnerDebugContext`, `FeatureFlagsContext`, общие типы;
  - `permissions.ts` — `DEFAULT_SECTION_PERMS`, `expandRoleAliases`, `useSectionPermissions`, `resolveHomePath`;
  - `AppRoutes.tsx` — `OneTimeAuthHandler`, `ProtectedRoutes`, `RootWrapper`.
- `App.tsx` оставлен как тонкий composition-root: theme + error/snackbar providers + верхнеуровневые routes.
- Компоненты и страницы (`MainLayout`, `AppearanceSettingsPanel`, `PeoplePage`, `OrdersPage`, `SettingsPage`) переведены на прямые импорты из `app/*`, без зависимости от `App.tsx`.
- Тесты `MainLayout` и `SettingsPage` синхронизированы с текущей IA меню/вкладок.

### Consequences

- Фронтенд-shell стал масштабируемее: изменения auth/ACL/routes больше не требуют массовой переклейки импортов от `App.tsx`.
- Переиспользование app-level state (permissions/context hooks) стало явным и предсказуемым.
- Регрессионные проверки после декомпозиции: `eslint src` + targeted Vitest по `MainLayout` и `SettingsPage` проходят.

## 2026-03-18 — Интеграционный комплект документации выровнен под самостоятельное техническое чтение

**Context**: После реализации полного multi-provider integration layer часть документов уже содержала правильную структуру, но местами ещё описывала старое состояние системы (`stubs`, слишком Telegram-centric трактовка, коммуникационно-прикладные формулировки вместо neutral doc map).

### Решения

- Синхронизированы `docs/19`, `docs/20`, `docs/21`, `docs/22` с текущим runtime состоянием: adapter manager, capability profiles, ingress parsers, bridge-based egress.
- [[24-integration-docs-map]] зафиксирован как нейтральная карта интеграционного комплекта документации.
- Убраны остаточные формулировки, ориентированные на обсуждение/созвон, и заменены на language уровня project documentation.

### Consequences

- Пакет документов читается как самостоятельный architectural/reference set.
- Архитектура бота, API choreography и scope UI-refactor отражают текущее состояние кода без необходимости заглядывать в исходники.

## 2026-03-18 — Полная реализация мульти-провайдерного integration layer

**Context**: Система имела двойную абстракцию: adapter ABC (async, с заглушками для Matrix/VK/MAX) и bridge pattern (sync, в worker.py через if/else). Адаптеры были зарегистрированы, но не подключены к runtime. Worker содержал 6 proxy-функций с дублированием логики маршрутизации.

### Решения

- Создана подсистема **capabilities** (`adapters/capabilities.py`) — декларативный профиль каждого провайдера (20+ полей: send_text, reactions, inline_keyboards, delete_message, max_message_length, и т.д.).
- Реализован **BridgeTransport** (`adapters/bridge_transport.py`) — синхронный HTTP-транспорт для non-Telegram провайдеров, оборачивающий `provider_bridge.dispatch_provider_action()`.
- Созданы **провайдер-специфичные парсеры** (`adapters/parsers.py`) — понимают нативные форматы: Matrix events (m.room.message, m.reaction, appservice batches), VK Callback API (message_new, message_event), MAX Bot API (message_created, message_callback, bot_started).
- Реализован **AdapterManager** (`adapters/manager.py`) — единая точка dispatch, заменяет if/else в worker. Проверяет capabilities перед отправкой (reactions не отправляются в VK, callback_query не отправляется в Matrix).
- Все адаптеры **переписаны с заглушек на bridge-backed реализации**: MatrixAdapter, VkAdapter, MaxAdapter используют BridgeTransport. WebhookAdapter убрана fragile `_callback_url` зависимость.
- Worker proxy-функции (send_message, send_document, answer_callback_query, set_message_reaction, delete_message) **делегируют в AdapterManager** — ~130 строк if/else заменены на ~40 строк делегирования.
- `/capabilities` endpoint расширен: возвращает `provider_capabilities` — детальную матрицу возможностей каждого включённого провайдера.
- `build_ingest_envelope()` в integrations.py вызывает parser перед обёрткой — нативные payload форматы провайдеров автоматически нормализуются до canonical dict.

### Consequences

- Подключение нового провайдера = 1 adapter файл + 1 parser функция + 1 capabilities профиль. Нет изменений в worker.
- Система автоматически пропускает неподдерживаемые action'ы (VK reactions, Matrix callback queries).
- 76 тестов покрывают capabilities, parsers, manager, adapters, bridge, integrations.
- Backward-compatible: все call sites в worker остались без изменений.

## 2026-03-18 — Интеграционная документация структурирована как самостоятельный архитектурный пакет

**Context**: Разделы по архитектуре бота, API choreography и FluffyChat theme-refactor содержали нужную информацию, но часть формулировок была ориентирована на контекст коммуникации, а не на роль каноничной документации проекта.

### Решения

- Обновлён [[18-integration-overview]] как нейтральная тематическая карта раздела без коммуникационно-специфичных формулировок.
- Уточнены `docs/19`, `docs/20`, `docs/21` в сторону product/architecture-first подачи (модули, контуры, последовательности вызовов, границы scope).
- Из навигации убраны коммуникационные артефакты, чтобы интеграционный раздел читался как самостоятельная проектная документация.

### Consequences

- Интеграционный пакет стал устойчивым source of truth для архитектурной и технической оценки.
- Документы можно использовать независимо от истории переписки и организационного контекста.

## 2026-03-18 — HTTP provider bridge contract зафиксирован как каноничный integration handoff для non-Telegram egress

**Context**: После добавления bridge-based outbound в runtime стало важно зафиксировать не только идею, но и точный handoff-контракт для внешних интеграторов: какие headers принимает ingress, какой JSON отправляется наружу, какие action names считаются каноничными и где проходит граница между backend domain logic и vendor-specific transport.

### Решения

- Добавлен отдельный runbook [[23-provider-bridge-runbook]] как практический handoff для Matrix/VK/MAX/Webhook bridge.
- Каноничным outbound форматом зафиксирован JSON `provider + action + payload` с auth через `X-Bridge-Secret`.
- Зафиксирован приоритет маршрутизации outbound target: `callback_url` → provider-specific `*_OUTBOUND_URL`.
- Vendor-specific signature/OAuth/storage concerns оставлены на стороне внешнего bridge, а не ядра backend.

### Consequences

- Внешняя команда может подключать bridge без чтения `worker.py` и `provider_bridge.py`.
- Граница ответственности между backend и transport bridge стала формальной и проверяемой.
- Дальнейшая реализация Matrix/VK/MAX клиентов может идти параллельно без изменений в order pipeline.

## 2026-03-18 — Non-Telegram outbound переведён с fallback-mode на provider bridge contract

**Context**: После введения provider-agnostic ingress backend уже умел принимать события от `telegram|matrix|vk|max|webhook`, но egress для non-Telegram оставался слишком условным: callback reply или no-op. Для реальной интеграционной готовности нужен был единый безопасный контракт исходящих действий без привязки доменного pipeline к vendor SDK.

### Решения

- Добавлен модуль `backend/app/provider_bridge.py` как единая точка provider-specific ingress/egress contract.
- Ingress-проверка секретов переведена на per-provider headers с fallback на общий `INTEGRATION_WEBHOOK_SECRET`.
- Worker для non-Telegram провайдеров теперь диспатчит стандартные действия (`send_message`, `send_document`, `answer_callback_query`, `set_message_reaction`, `delete_message`) на provider-specific outbound bridge URL.
- Конфигурация расширена env-параметрами `*_OUTBOUND_URL`, `*_OUTBOUND_SECRET`, `PROVIDER_BRIDGE_SECRET`, `*_WEBHOOK_SECRET`.
- Если provider bridge не настроен, pipeline остаётся безопасным: действие пропускается как best-effort no-op и не ломает обработку заказа.

### Consequences

- Backend теперь закрывает не только общий ingress, но и унифицированный egress-контур для Matrix/VK/MAX/Webhook.
- Telegram runtime не меняется и остаётся основным production path.
- Для полной production готовности вне Telegram всё ещё нужны vendor-native transport clients, signature nuances и интеграционные тесты против конкретных API провайдеров.

## 2026-03-18 — Backend переведён на provider-agnostic ingress runtime (Telegram / Matrix / VK / MAX / Webhook)

**Context**: Документация уже фиксировала provider-agnostic архитектуру, но runtime backend принимал только `POST /telegram/webhook`. Для практической интеграционной готовности нужен был единый ingress-контур и безопасное поведение worker-а для не-Telegram источников без ломки доменного пайплайна.

### Решения

- Добавлен универсальный endpoint `POST /integrations/{provider}/webhook` с поддержкой `telegram|matrix|vk|max|webhook`.
- В `backend/app/main.py` внедрён единый envelope-формат входящих событий (`__integration_envelope__`) с вычисляемым idempotent `update_id`.
- Worker научен нормализовать envelope в общий payload и продолжать обработку через текущий order pipeline.
- Для non-Telegram провайдеров внедрён safe fallback outbound: callback reply (если есть `callback_url`) либо no-op с логированием, без аварийного падения обработки.
- Добавлены provider stubs (`matrix_adapter`, `vk_adapter`, `max_adapter`) и bootstrap/validation фабрика адаптеров (`adapters/factory.py`).
- Конфигурация расширена env-параметрами `MESSENGER_PROVIDER`, `MESSENGER_ENABLED_PROVIDERS`, `INTEGRATION_WEBHOOK_SECRET`, `MATRIX_BOT_TOKEN`, `VK_BOT_TOKEN`, `MAX_BOT_TOKEN`.

### Consequences

- Backend теперь готов к подключению внешних messenger интеграций через единый HTTP ingress контракт без переписывания доменной логики.
- Telegram runtime остаётся полностью совместимым с текущим webhook потоком.
- Для Matrix/VK/MAX остаётся отдельная задача production transport egress/signature hardening, но ingress + unified processing уже функционируют.

## 2026-03-18 — Интеграционный контур зафиксирован как provider-agnostic контракт (Telegram / Matrix / VK / MAX)

**Context**: Интеграционная документация уже покрывала архитектуру бота и API, но в формулировках оставалась заметная Matrix-centric трактовка. Для долгоживущей архитектуры нужен единый контракт, который одинаково применим к Telegram, Matrix, VK и MAX.

### Решения

- Добавлен документ [[22-messenger-agnostic-integration-spec]] как каноничный provider-agnostic контракт.
- Зафиксирован единый подход `adapter-per-provider` поверх общего domain pipeline.
- В интеграционных документах (`docs/18`, `docs/19`, `docs/20`) закреплена модель capability-driven интеграции и общий playbook подключения нового провайдера.
- Навигация [[Отличный улов/docs/README]] и `README.md` обновлена ссылкой на универсальный контракт.

### Consequences

- Документация интеграции больше не привязана к одному вендору.
- Любой новый провайдер подключается через один и тот же architectural шаблон: normalize → domain → provider egress.
- Оценка работ по VK/MAX/Matrix становится сопоставимой и проверяемой по единому checklist.

## 2026-03-18 — Внешний handover по Matrix/mobile оформляется как отдельный пакет docs, а не как разрозненные ссылки

**Context**: Внешняя команда запросила три вещи одновременно: верхнеуровневую архитектуру текущего бота, API + последовательности бизнес-процессов для интеграции мобильного клиента и явное описание объёма работ по theme/refactor во FluffyChat. Нужная информация уже была в репозитории, но была размазана по `README`, API reference, admin flows и Telegram docs, что повышало риск недопонимания на оценке.

### Решения

- Добавлен отдельный handover-пакет документации:
  - [[18-integration-overview]]
  - [[19-bot-architecture]]
  - [[20-api-and-business-flows-for-mobile-integration]]
  - [[21-fluffychat-theming-and-ui-refactor-scope]]
- Зафиксировано явно, что текущий бот имеет reusable domain core, но Telegram-specific transport layer не переносится напрямую на Matrix.
- Зафиксировано явно, что внешний `/integrations/*` контур сейчас read-oriented и не является готовым публичным customer write API для нативных mobile-форм заказа.
- Навигация по новым материалам добавлена в [[Отличный улов/docs/README]] и корневой `README.md`.

### Consequences

- Внешней команде можно отправить готовый набор файлов без дополнительных устных пояснений по базовой архитектуре.
- Граница между reusable domain логикой и новым Matrix adapter scope стала прозрачной ещё до старта оценки.
- Ограничение по отсутствию отдельного customer-facing write contract больше не спрятано между строк и может быть осознанно зафиксировано на созвоне.

## 2026-03-18 — API smoke остаётся явной командой, а не скрытым prebuild-hook

**Context**: В репозитории уже появился `admin-web/scripts/api-smoke.mjs` и корневой `make check-api-smoke`, но без явных npm-скриптов и README-инструкций эта проверка оставалась полувстроенной: о ней знали docs/changelog и часть shell-обвязки, но не основной developer flow.

### Решения

- В `admin-web/package.json` добавлены явные команды `api:smoke` и `api:smoke:strict`.
- README и testing docs обновлены так, чтобы live-проверка контракта `admin-web` ↔ `admin_service` была discoverable без чтения исходников.
- Smoke не возвращается в автоматический `prebuild`: запуск остаётся только по явной команде разработчика или CI/ops-сценарию.

### Consequences

- Проверка контракта стала доступной из обычного developer workflow.
- Сборка фронтенда не делает неожиданных сетевых вызовов и не зависит от живого owner-аккаунта по умолчанию.

## 2026-03-18 — `admin_web` dev compose использует builder-stage, а не псевдо-image override

**Context**: `infra/docker-compose.dev.yml` пытался запустить `admin_web` как `image: node:20-alpine`, но при merge с базовым `docker-compose.yml` compose одновременно сохранял `build`, из-за чего собирал frontend image и тегировал его именем `node:20-alpine`. В результате dev-контейнер стартовал не на чистом node-образе и падал с `npm: not found`.

### Решения

- В dev override `admin_web` теперь явно использует `target: builder` из `admin-web/Dockerfile`.
- Dev-режим сохраняет `npm run dev`, volume-mount исходников и hot reload, но больше не конфликтует с production stage на `nginx`.

### Consequences

- `docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build` снова корректно поднимает frontend в dev-режиме.
- Официальный образ `node:20-alpine` больше не перетирается локальной сборкой по ошибке.

## 2026-03-16 — Персональные настройки внешнего вида и выравнивание прав admin-web

- Настройки темы, плотности и внешнего вида переведены на персональное хранение по пользователю, чтобы общий браузер/рабочее место не перетирали внешний вид у коллег.
- В `admin-web` добавлен отдельный пользовательский вход в настройки внешнего вида без захода в owner-only системные настройки.
- Раздел экспорта в UI дополнительно выровнен с backend-ограничением owner-only, чтобы не генерировать лишние 403/429 у региональных админов.
- Для регионального администратора без назначенных регионов список пользователей больше не падает с 403: интерфейс остаётся рабочим и показывает хотя бы собственную запись.

## 2026-03-16 — API Center как единый контракт UI + prebuild smoke-check

**Context**: Репозиторий уже содержал актуальную документацию по API, но перед живым запуском и особенно перед build не было единой operational-точки, где можно одновременно: увидеть полный список используемых endpoint-ов, вручную проверить проблемный запрос, прогнать живой smoke-run и удобно отдать контракт внешней интеграции.

### Решения

- В `admin-web` добавлен owner-only экран `API Center`, который служит живым каталогом endpoint-ов `admin_service` и integration routes.
- Каталог фиксирует method/path/auth/automation/used-by и поля запроса, чтобы UI-контракт не был спрятан только в `client.ts` и markdown-доках.
- Для предсборочной валидации добавлен `admin-web/scripts/api-smoke.mjs`; он логинится owner-учёткой, проверяет automatable route-ы и пишет отчёт в `.artifacts/api-smoke-report.json`.
- Сборка фронтенда получила optional prebuild smoke-hook: если в окружении заданы `API_SMOKE_*`, build прогоняет API-проверку автоматически; strict-режим вынесен в `npm run build:verified`.
- Отчёты и каталог можно экспортировать в `JSON / CSV / Markdown`, копировать и шарить наружу без отдельной ручной подготовки.

### Consequences

- Перед релизом или демо можно быстро увидеть не только «что сломано», но и какой именно endpoint и с каким auth-контрактом.
- Внешним интеграциям проще отдавать живой и компактный контракт без ручной сборки ссылок по нескольким docs-файлам.
- Smoke-проверка становится частью normal operational-flow, а не только ad-hoc отладки через DevTools.

## 2026-03-16 — Перевод API-каталога из UI в developer-docs manual

**Context**: Практика показала, что in-app экран с каталогом endpoint-ов, ручным запуском запросов и smoke-run перегружает `admin-web`, портит рабочий UX и выглядит как продуктовая фича, хотя реальная потребность — просто удобный manual для разработчиков.

### Решения

- Убран отдельный экран endpoint-ов из `admin-web` и удалены маршруты/пункт меню, связанные с ним.
- Сборка `admin-web` больше не выполняет автоматические live API-запросы через prebuild hook.
- Для разработчиков добавлен docs-first manual: [[17-admin-service-endpoints-manual]].
- Source of truth по route-ам и request fields сохранён в `admin-web/src/api/apiCatalog.ts`, а расширенный reference остаётся в [[14-api-reference]].

### Consequences

- Продуктовый интерфейс остаётся рабочим и не несёт лишнего технического UI.
- Другие разработчики получают отдельную документацию, которую можно читать без запуска сайта и без owner-доступа.
- Live-запросы больше не маскируются под documentation flow.

## 2026-03-16: Admin service self-heals legacy security schema drift on startup

**Context**: После security hardening часть инсталляций осталась на старой схеме `admin_service`: таблицы уже существовали, но `Base.metadata.create_all()` не добавляет новые колонки в существующие таблицы. В результате сервис мог подняться, но `POST /auth/login` падал на runtime-вставках в `admin_audit_logs` / `admin_login_attempts`, а снаружи это выглядело как `500` или `502` при входе.

### Решения

- В startup `admin_service` добавлен best-effort compatibility pass, который проверяет legacy admin-таблицы и автоматически добавляет недостающие security/audit/session колонки (`ip_address`, `user_agent`, `device_fingerprint`, `severity`, и связанные session/login поля).
- Audit/session helper'ы изолированы через nested transaction, чтобы сбой записи служебного лога не отравлял основной request transaction и не ломал сам login flow.
- Middleware security headers исправлен так, чтобы удаление служебных заголовков не падало на `MutableHeaders`.

### Consequences

- Старые окружения восстанавливаются без ручного SQL в типовом случае: после пересборки `admin_service` логин снова начинает отвечать штатно.
- Ошибки во вторичных security side-effects теперь меньше рискуют превращать нормальный `401/403` в неожиданный `500`.

## 2026-03-16: Edited Telegram messages stop auto-mutating orders once delivery starts

**Context**: Пользователь может отредактировать исходное Telegram-сообщение уже после того, как оператор начал или завершил выдачу. Старый flow перепарсивает заказ по `source_message_id`, что хорошо для ещё “живых” заказов, но опасно для уже выданных: редактирование могло изменить состав задним числом и сломать operational truth.

### Решения

- В `worker._maybe_process_order()` добавлена защита: если update пришёл как `edited_message` и у найденного заказа уже есть `delivery_status`, worker больше не переписывает строки и статусы автоматически.
- Для таких заказов сохраняется только новый `raw_text`, чтобы оператор видел фактический отредактированный текст в истории.
- Пользователю отправляется явное сообщение, что заказ уже в выдаче/закрыт и дальнейшие изменения нужно решать через оператора.
- В `DeliveryPage` кнопка `Полностью` теперь сначала открывает компактную confirm-модалку; если оператор отменяет быстрое подтверждение, система сразу переводит его в полную форму выдачи.

### Consequences

- История Telegram-редакций сохраняется, но не ломает уже зафиксированные процессы выдачи.
- Быстрый сценарий “заказ просто отдан целиком” стал быстрее, а более сложные кейсы (частично / перенос / комментарии) по-прежнему уходят в полную форму.

## 2026-03-15: Comprehensive security hardening (v0.5.0)

**Context**: Админ-панель и API требовали комплексного усиления безопасности: защита от brute-force, расширенный аудит, управление сессиями, защита фронтенда от инспекции и копирования данных, а также инструменты для owner по мониторингу безопасности.

### Решения

**Backend (admin_service):**
- Добавлены модели `LoginAttempt` и `ActiveSession` для отслеживания всех попыток входа и активных сессий
- `AuditLog` расширен полями `ip_address`, `user_agent`, `device_fingerprint`, `severity`
- Brute-force protection: account lockout после 5 неудач за 15 мин (блок 30 мин), IP block после 20 попыток за 60 мин
- Все JWT привязаны к `ActiveSession` через JTI; `deps.py` проверяет валидность сессии при каждом запросе
- Rate limiting пересмотрен: login=5/min, auth=10/min, export=3/min, global=100/min
- `SuspiciousRequestMiddleware` — regex блокировка path traversal, XSS, SQL injection, code injection, sensitive file access
- Security headers расширены: CSP, COOP, COEP, CORP, Referrer-Policy, Cache-Control no-store
- Swagger/ReDoc/OpenAPI отключены в production
- Все token/secret сравнения через `hmac.compare_digest()`
- Новые owner-only endpoints: `POST /auth/logout`, `POST /auth/revoke-all-sessions`, `GET /auth/sessions`, `DELETE /auth/sessions/{id}`, `GET /login-attempts`, `GET /security-summary`
- Audit-log endpoint расширен фильтрами `action_filter`, `severity_filter`, `actor_filter`

**Frontend (admin-web):**
- `securityShield.ts` — блокировка DevTools (F12, Ctrl+Shift+I/J/C), right-click, copy, print, drag; console override; CSS user-select:none
- `sessionGuard.ts` — auto-logout при неактивности > 30 мин; cross-tab синхронизация через localStorage
- `SecurityDashboard` — новая вкладка «Безопасность» в настройках (owner-only): threat level, failed logins, active sessions management, panic button, stealth mode, top attacking IPs
- `AuditLogPanel` расширен: фильтры по severity/action, новые колонки IP/severity, expandable device info
- `client.ts` — новые методы: `logout()`, `getActiveSessions()`, `revokeSession()`, `revokeAllSessions()`, `getLoginAttempts()`, `getSecuritySummary()`
- Logout flow обновлён: server-side revocation + cross-tab broadcast
- `index.html` — мета-теги `noindex`, `nofollow`, `noarchive`, `no-referrer`

### Consequences

- Любая brute-force атака блокируется автоматически на уровне аккаунта и IP
- Сессии можно отзывать мгновенно; owner видит все активные сессии в реальном времени
- Frontend-защита делает инспекцию и копирование данных значительно сложнее
- Panic button позволяет мгновенно разлогинить всех при компрометации
- Расширенный аудит с IP/UA/fingerprint позволяет отслеживать подозрительную активность

## 2026-03-14: Delivery and Catalog mobile surfaces aligned with the new order cards

**Context**: После переработки мобильного `OrdersPage` соседние operational-экраны всё ещё воспринимались как из разных приложений: `DeliveryPage` и `CatalogsPage` формально уже имели card-based layout, но на телефоне сохраняли много chip-шума, плотные action-строки и менее предсказуемые full-screen диалоги.

### Решения

- Для `DeliveryPage` мобильные карточки очереди, журнала и проблемных заказов приведены к общему паттерну: короткий identity-блок, сетка mini-info cards и затем action-grid вместо длинных wrap-строк кнопок.
- Для `CatalogsPage` карточки каталогов и раскрытых товаров переведены на более спокойную mobile-композицию с meta-cards вместо длинных лент chip-ов.
- Основные диалоги `Выдачи` и `Каталогов` на мобильных оставлены full-screen, но их нижние action-зоны сделаны более явными: primary и secondary actions теперь нормально раскладываются под one-hand usage.
- Breakpoint detection в этих экранах переведён в client-only режим (`useMediaQuery(..., { noSsr: true })`), чтобы уменьшить риск первого рендера в старом desktop-подобном виде.

### Consequences

- Переход между `Заказы` → `Выдача` → `Каталоги` на телефоне больше не ощущается как прыжок между разными поколениями UI.
- Telegram browser получает более стабильный и предсказуемый layout без лишней визуальной мелочи.
- Операционные кнопки и диалоги стали ближе к реальному рабочему сценарию “смотрю с телефона и жму большим пальцем”, а не к desktop-таблице, ужатой до узкой колонки.

## 2026-03-14: User and pickup-admin mobile screens follow the same operational pattern

**Context**: После выравнивания `Заказов`, `Выдачи` и `Каталогов` в мобильной админке ещё оставались два экрана с более “служебным” и визуально разнородным поведением: `UsersPage` и `PickupAdminPage`. Функционально они были рабочими, но mobile scanning и one-hand actions там ощущались менее предсказуемо.

### Решения

- Для `UsersPage` карточки пользователей разделены на понятные зоны: identity, meta-info, роли/регионы и grid действий.
- Для `PickupAdminPage` карточки назначений и точек выдачи тоже переведены на info-card pattern вместо набора разнородных текстовых блоков и chip-лент.
- Breakpoint detection на обоих экранах переведён в `useMediaQuery(..., { noSsr: true })`.
- Диалоги этих экранов приведены к тому же mobile footer pattern, что уже используется в `OrdersPage`, `DeliveryPage` и `CatalogsPage`.

### Consequences

- Вся operational-навигация admin-web теперь выглядит ближе к единой mobile design system, а не к набору отдельно улучшенных экранов.
- Пользовательские и owner-only сценарии на телефоне стали спокойнее и последовательнее: меньше визуальных скачков, больше предсказуемости в действиях.

## 2026-03-14: Global mobile spacing is centralized instead of page-by-page

**Context**: После нескольких волн UI-доработок mobile spacing начал жить в двух мирах одновременно: часть ритма задавалась темой, а часть — вручную внутри отдельных страниц. Из-за этого на телефоне интерфейс местами выглядел слишком плотным, а местами наоборот “разъехавшимся”.

### Решения

- В theme overrides зафиксирован единый responsive spacing scale для `MuiCardContent`, `MuiDialogTitle`, `MuiDialogContent`, `MuiDialogActions`, `MuiToolbar` и `MuiAlert`.
- В `MainLayout` выровнен внешний mobile padding контента и нижний safe-area reserve под bottom navigation.
- Ключевые operational-страницы (`Orders`, `Delivery`, `Catalogs`, `Users`, `PickupAdmin`, `Settings`) переведены на согласованные mobile paddings вместо разрозненных локальных значений.

### Consequences

- На мобильных переход между разделами больше не ощущается как смена разных spacing-систем.
- Карточки, фильтры и full-screen диалоги выглядят цельнее и лучше держат визуальный ритм в Telegram browser.
- Дальнейшие UI-правки проще делать без повторного “расползания” отступов по страницам.

## 2026-03-14: Large-radius styles also receive larger spacing

**Context**: После централизации mobile spacing стало видно ещё одно UX-правило: стили с большим `borderRadius` (`minimal`, `elegant`) визуально требуют больше внутреннего воздуха. При одинаковых паддингах они выглядели теснее, чем более “угловатые” темы.

### Решения

- В `applyStylePresetToTheme()` spacing теперь зависит не только от `density`, но и от `borderRadius`.
- Для более скруглённых стилей автоматически увеличиваются paddings у `CardContent`, `Toolbar`, `DialogTitle`, `DialogContent`, `DialogActions` и `Alert`.
- Бонус к spacing ограничен сверху, чтобы темы с большим радиусом стали свободнее, но не «распухали» чрезмерно.

### Consequences

- `minimal` и `elegant` выглядят более естественно: крупные скругления больше не конфликтуют с тесным внутренним контентом.
- Переключение между стилями меняет не только форму, но и общий ритм воздуха — без ручной подгонки по страницам.

## 2026-03-14: Sensitive analytics and exports become owner-only with step-up confirmation

**Context**: После mobile/UX доработок стало понятно, что главная оставшаяся дыра — не в кнопках, а в data-rich surfaces: аналитика, XLSX-выгрузки и bot-side admin commands. UI уже умел прятать разделы, но backend во многих местах всё ещё полагался на “не показывать кнопку”, а не на жёсткую серверную политику.

### Решения

- Для admin-web введён **step-up password confirmation** на чувствительные export-операции: owner повторно вводит пароль, backend выдаёт краткоживущий confirmation token, который используется только для action `export`.
- Confirmation token дополнительно может быть привязан к `X-Device-Fingerprint`, чтобы step-up нельзя было безболезненно переиспользовать на другом устройстве.
- Все analytics endpoint-ы переведены в policy **owner-only на backend**.
- Все export endpoint-ы (`/exports/build`, `/orders/export`, `/orders/export-template`, `/orders/export-history`) переведены в policy **owner-only + password confirmation**.
- Telegram admin bot перестал быть каналом доступа к статистике, заказам, диагностике и экспортам: эти сценарии намеренно выведены только в web-админку.
- Frontend defaults синхронизированы с новой политикой: секция `export` по умолчанию owner-only.

### Consequences

- Чувствительные данные больше не зависят от видимости кнопок или дисциплины операторов.
- Даже при компрометации обычного admin/pickup-admin аккаунта объём доступной аналитики и выгрузок резко уменьшается.
- Бот больше не работает как “теневой backdoor” для статистики и массовых выгрузок.

## 2026-03-14: Delivery workspace becomes operator-first on mobile

**Context**: После первой волны mobile-adaptation `DeliveryPage` уже перестал быть просто широкой таблицей, но у операторов выдачи оставалась практическая проблема: сверху висели большие summary-блоки, а до самой очереди приходилось дополнительно проматывать экран. Параллельно прочерки в карточках без пояснений выглядели как визуальный шум, а соседние operational-разделы (`UsersPage`, `PickupAdminPage`, `CatalogsPage`) на телефоне всё ещё в основном оставались desktop-таблицами.

### Решения

- В `DeliveryPage` крупные summary-карточки заменены на **компактную мини-сводку** из chip-метрик.
- Для роли `pickup_admin` без owner/regional scope рабочая очередь остаётся главным первым контентом экрана: вкладки и фильтры ведут сразу к заказам, а статистика больше не оттесняет их вниз.
- В мобильных карточках выдачи UI больше не показывает бесполезные прочерки там, где данных нет; вместо этого скрываются пустые поля или используются более понятные текстовые fallback'и.
- В мобильной карточке очереди убрано время создания заказа как слабополезный шум для сценария живой выдачи.
- `UsersPage`, `PickupAdminPage` и `CatalogsPage` переведены на **card-based mobile layout** с full-screen диалогами для основных операций.

### Consequences

- `pickup_admin` на телефоне быстрее доходит до целевого действия: найти заказ и отметить выдачу.
- Мобильная версия админки стала восприниматься как отдельный рабочий сценарий, а не как сжатый desktop.
- Соседние разделы теперь меньше ломают поток работы при переходе между выдачей, пользователями, точками и каталогами.

## 2026-03-14: Trusted-device login via browser credential manager

**Context**: Пользователю нужен сценарий “вошёл один раз на своём устройстве и больше не набираю пароль каждый раз”, но прямое хранение пароля в `localStorage` или `IndexedDB` приложения создаёт слишком большой риск при XSS, отладочных дампах браузера и shared-device сценариях.

### Решения

- Для `admin-web` выбран путь **trusted-device login через browser-managed credential storage**, а не кастомное хранение пароля в приложении.
- На `LoginPage` добавлен opt-in переключатель **`Запомнить это устройство`**.
- После успешного логина пароль может быть сохранён только через `Credential Management API` / password manager браузера, если браузер поддерживает secure storage.
- Само приложение сохраняет локально только:
  - флаг разрешения trusted-device login,
  - последний email,
  - одноразовый suppress-флаг, чтобы после ручного logout не происходил немедленный auto-login.
- При ручном `Выйти` / `Сменить аккаунт` выполняется suppress silent login, чтобы пользователь действительно видел экран логина.

### Consequences

- Реализован UX “устройство помнит вход”, но без хранения сырого пароля в открытом storage приложения.
- Поведение остаётся безопаснее для shared-device и incident-response сценариев.
- Logout и switch-account больше не конфликтуют с автологином на доверенном устройстве.

## 2026-03-14: Mobile-first admin shell and operational card views

**Context**: Админка уже была функциональной на desktop, но на телефоне рабочие сценарии оставались тяжёлыми: длинные таблицы в `Заказы` и `Выдача`, перегруженные tabs в `Настройки`, отсутствие быстрого перехода между разделами одной рукой.

### Решения

- В `MainLayout` добавлена **нижняя мобильная навигация** с основными operational-разделами и отображение названия текущего экрана в header.
- Для `OrdersPage` реализован **card-based mobile layout** вместо горизонтального скролла таблицы.
- Для `DeliveryPage` реализованы отдельные **мобильные карточные представления** очереди, журнала выдач и проблемных заказов.
- Критичные диалоги в `OrdersPage`, `DeliveryPage` и `SettingsPage` на мобильных теперь открываются в **full-screen** режиме.
- `SettingsPage` переведён на **селектор секций** вместо узких scroll-tabs на телефонах, а сохранение вынесено в sticky bottom action.

### Consequences

- Админка стала заметно ближе к реальному mobile operational flow: просмотреть, найти, подтвердить и поправить заказ теперь можно без постоянного горизонтального скролла.
- Самые частые действия доступны одной рукой, особенно для операторов выдачи и owner, которые часто работают “на бегу”.
- Мобильный UI теперь развивается как отдельный продуктовый сценарий, а не как уменьшенная копия desktop-таблиц.

## 2026-03-14: Bot chat inactivity mode and unified delivery journal

**Context**: В живом чате возникла операционная путаница: режим `hidden` визуально выглядел как «бот умер», хотя на самом деле worker продолжал читать сообщения и обрабатывать заказы. Параллельно в рабочем месте выдачи разделы `Журнал` и `Отмеченные` дублировали друг друга, а ошибочно отмеченную выдачу нельзя было удобно откатить назад.

### Решения

**Bot chat modes:**
- Зафиксировано явное различие между `hidden` и новым режимом `inactive`.
- `hidden` теперь трактуется как «бот читает и обрабатывает, но не отвечает в чат».
- `inactive` означает полную паузу по конкретному чату: worker не обрабатывает сообщения, не создаёт заказы, не отвечает и не ведёт callback-flow.
- При `my_chat_member` событиях выход/кик бота из чата переводит чат в `inactive`; при повторном добавлении из состояния `left/kicked` режим может автоматически вернуться в `full`.

**Delivery workspace:**
- В `DeliveryPage` удалено дублирование между вкладками `Журнал` и `Отмеченные`: остаётся единый список отмеченных выдач с составом, raw text, статусом, точкой и действиями.
- Для записей выдачи добавлен явный rollback-flow: оператор может отменить выдачу, после чего заказ возвращается в обычную очередь через пересчёт `orders.delivery_status`.
- Ответы delivery API обогащаются читаемыми названиями строк и order-метаданными (`customer_name`, `phone_last4`, `pickup_place`), чтобы UI не показывал мусор вида `Строка #123` там, где есть известные позиции.

**Bot usability:**
- Добавлена новая публичная команда с алиасами `/pickup_points`, `/pickup_places`, `/where_pickup`, чтобы пользователь мог быстро увидеть доступные точки выдачи прямо в чате.

### Consequences
- У owner/операторов появилось ясное различие между «бот скрыт» и «бот полностью выключен для этого чата».
- Ошибочно оформленную выдачу теперь можно откатить без ручной чистки БД.
- Delivery workspace стал компактнее и ближе к реальному operational flow смены.
- Пользователям проще самообслуживаться по точкам выдачи без лишних вопросов в чате.

## 2026-03-14: Bot UX becomes admin-configurable from Settings

**Context**: Настройки бота в админке покрывали только базовые параметры (`reply_mode`, 2-3 сообщения и несколько switches), тогда как реальный UX бота — меню `/start`, `/menu`, `/help`, шаблоны заказа, тексты профиля, подсказки `/set_pickup`, тексты ошибок и сами inline-кнопки — оставался захардкоженным в `worker.py`, `public_commands.py` и `order_presenter.py`.

### Решения

- Введён единый слой `backend/app/bot_ui.py`, который читает bot UI settings из `bot_settings`, держит дефолты и строит:
  - пользовательское меню,
  - inline-клавиатуры,
  - шаблоны `/order_template`, `/order_example`, `/order_rules`, `/help`,
  - тексты профиля и служебных подсказок.
- `admin_service` расширен новыми allow-listed keys без новой миграции: bot_settings остаётся универсальным storage, но теперь официально поддерживает десятки bot-UX ключей.
- `SettingsPage` получил полноценный owner-пульт управления ботом:
  - настройка текстов ответов,
  - настройка шаблонов и справки,
  - включение/скрытие кнопок,
  - переименование кнопок прямо из UI.
- Клавиатуры бота теперь собираются из сохранённых настроек, включая owner/admin-only кнопки.

### Consequences
- Owner может реально настраивать, *как бот выглядит и говорит*, без редактирования Python-кода.
- UX бота стал управляемым как продуктовый слой, а не как набор случайных строк по проекту.
- Будущие изменения bot-flow проще расширять: новые кнопки и тексты можно добавлять через единый bot UI registry.

## 2026-03-14: Live preview for bot messages in Settings

**Context**: После того как тексты и кнопки бота стали управляться из админки, оставалась UX-проблема: owner менял шаблоны “вслепую” и всё равно был вынужден сохранять, идти в Telegram и проверять глазами, как именно выглядят `/menu`, `/help` и клавиатуры.

### Решение
- В `SettingsPage` добавлен live preview блока `Настройки → Бот`.
- Preview умеет показывать:
  - `/menu` и `/start`,
  - `/help`,
  - `/order_template`, `/order_example`, `/order_rules`,
  - профиль пользователя,
  - сообщение с точками выдачи,
  - сценарий “нет активной раздачи”.
- Owner может переключать preview:
  - по выбранному чату (для подстановки точек выдачи),
  - в режиме обычного пользователя или администратора.
- Кнопки меню визуализируются рядом с сообщением, чтобы сразу видеть реальную раскладку inline keyboard.

### Consequences
- Настройка бота стала WYSIWYG-подобной: “изменил → сразу увидел”.
- Снизился риск сломать tone of voice или испортить шаблон текста незаметным плейсхолдером.

## 2026-03-14: Admin auth hardening, delegated user creators, catalog-aware order editing

**Context**: Появился набор живых operational требований: модератор должен полностью убирать stale error text после исправления заказа, статусы выдачи должны отображаться полными человеческими фразами, строки заказа нужно редактировать через каталог с автоподстановкой SKU/единиц, а владелец — безопасно делегировать создание пользователей без раздачи полного user-management. Параллельно нужно уменьшить связность admin auth с Telegram и закрыть сценарий «кто-то левый вошёл в админку по чужому токену/ссылке».

### Решения

**Auth / security:**
- Добавлен helper `admin_service/app/auth_identity.py` с нормализацией auth provider (`local_password`, `telegram`, `external`) и единым построением one-time login URL.
- В config добавлены `ADMIN_AUTH_PRIMARY_PROVIDER`, `ADMIN_AUTH_ALLOWED_PROVIDERS`, `ADMIN_AUTH_REQUIRE_DEVICE_BINDING`.
- Access token admin panel теперь может содержать fingerprint hash устройства; frontend отправляет постоянный `X-Device-Fingerprint`, backend валидирует совпадение при каждом запросе.
- Проверка сложности пароля теперь применяется не только при reset, но и в bootstrap/create flows.

**Users / ACL:**
- Введена settings-based capability `user_create_delegate_ids` без немедленной DB-миграции.
- Только `owner` может включать делегированное право создавать пользователей.
- Делегат может создавать только пользователей двух operational-уровней: `regional_admin` (админ) и `manager` (модератор), без назначения регионов.
- UI `UsersPage` адаптирован: owner видит delegate toggle, делегаты получают доступ к созданию, но не к полному управлению существующими аккаунтами.

**Order editing / delivery UX:**
- `OrderEditDialog` теперь поддерживает выбор товара из каталога с автоподстановкой `title`, `SKU`, `unit`.
- Backend line normalization использует `catalog_item_id`, `unit_hint`, `pack_hint` и понимает соотношения `г ↔ кг`, а также `шт` для позиций с упаковочным весом.
- При выводе заказа из `needs_admin` пустой `error_text` принудительно нормализуется в `NULL`, а UI перестаёт показывать старый текст ошибки.
- Для delivery статусов зафиксированы полные подписи: `Полностью выдан`, `Частично выдан`, `Передан другому лицу`, `Ожидает выдачи`, `Не выдан`.

### Consequences
- Admin auth больше не привязан архитектурно только к Telegram и готов к будущему внешнему/собственному identity provider.
- Украденный одноразовый токен или access token становится менее полезным без того же браузерного fingerprint.
- Владелец может масштабировать операционную команду без раздачи полного owner/regional scope.
- Редактирование заказов становится ближе к реальному каталогу и заметно уменьшает ручные опечатки в SKU/единицах.

## 2026-03-14: Comprehensive export builder — multi-sheet XLSX с шаблонами и кастомизацией

**Context**: Владельцу нужен гибкий экспорт данных, где в одном XLSX-файле можно получить произвольную комбинацию из 16 типов листов: каталоги, товары, статистику по точкам/чатам/регионам, заказы по позициям, выданные/оставшиеся заказы, остаток vs забрано, сводки выдач, топ товаров, точность парсера, полный дамп данных. Docker-сборка при этом ломалась из-за `samples/` в `.dockerignore`.

### Решения

**Backend export engine** (`admin_service/app/api/routers/exports.py`):
- Новый `GET /exports/presets` — возвращает каталог 16 типов листов, сгруппированных по категориям (Справочники, Статистика, Аналитика, Выдача, Данные), + 5 готовых шаблонов.
- Новый `POST /exports/build` — принимает JSON с массивом sheet-запросов (key + per-sheet фильтры), генерирует multi-sheet XLSX с профессиональным оформлением (colour-coded headers по группам, status fills, autofilter, freeze panes, auto-width).
- 5 шаблонов по умолчанию: Полный отчёт (9 листов), Отчёт по выдаче (4 листа), Аналитика (6 листов), Каталоги и товары (3 листа), Полный дамп (3 листа).
- Каждый лист поддерживает фильтры: catalog_id, chat_id, pickup_place, date_from/date_to, limit.

**Frontend Export Page** (`admin-web/src/pages/ExportPage.tsx`):
- Полностью новая страница «📥 Экспорт данных» в навигации.
- Блок готовых шаблонов (карточки с описаниями, один клик → выбор листов).
- Покомпонентный выбор листов по группам с expand/collapse.
- Глобальные фильтры (каталог, чат, даты, имя файла) в правой панели.
- Per-sheet фильтры через кнопку ⚙️ на каждом листе.
- Preview выбранных листов с нумерацией.
- Доступ: owner + regional_admin.

**Docker fix:**
- Убран `samples/` из `.dockerignore` — `admin_service/Dockerfile` копирует `samples/` в образ для шаблонного экспорта.

### Consequences
- Владелец получает единый интерфейс для любых отчётных нужд — от быстрой сводки до полного дампа.
- Шаблоны позволяют в один клик получить отчёт нужного типа без ручного выбора.
- Docker-сборка всех сервисов теперь проходит успешно (admin_service, admin_web, api).

## 2026-03-14: Delivery runtime hardening — template asset, not_delivered и выбор позиций из текущей раздачи

**Context**: После завершения мега-спринта всплыли runtime-проблемы уже в живом operational flow: `GET /orders/export-template` отдавал `404` в docker-среде, в выдаче не хватало явного статуса «Не выдан», а ручные позиции в форме выдачи выбирались только свободным текстом без привязки к текущему каталогу/раздаче.

### Решения

**Backend / runtime packaging:**
- В `admin_service/Dockerfile` добавлен `COPY samples /app/samples`.
- Это закрепляет контракт `_resolve_template_path()` для `samples/excel/template.xlsx` и устраняет расхождение между локальным repo и содержимым контейнера.

**Delivery status contract:**
- В `orders.py` статус `not_delivered` добавлен в `_VALID_DELIVERY_STATUSES` и `_DELIVERY_STATUS_LABELS`.
- В `schemas.py` обновлена документационная подсказка по допустимым delivery status.
- Добавлен backend regression-тест, чтобы статус и человекочитаемая подпись не потерялись при следующих правках.

**DeliveryPage UX:**
- В `DeliveryPage.tsx` добавлена отдельная action-кнопка `Не выдан` и возможность выставлять этот статус как при создании, так и при редактировании выдачи.
- Ручные позиции теперь выбираются через `Autocomplete` из объединённого списка: строки текущего заказа + товары текущего каталога (`getCatalogItems(order.catalog_id)`).
- Разрешён `freeSolo`, чтобы оператор всё ещё мог ввести нестандартную служебную пометку, если её нет в каталоге.

**Accessibility / browser noise:**
- У мобильного `Drawer` удалён `ModalProps.keepMounted`, чтобы скрытый контент не удерживал фокус и не провоцировал предупреждения про `aria-hidden`.
- В frontend добавлен favicon (`public/favicon.svg` + link в `index.html`).

### Consequences
- Шаблонный экспорт теперь работает в docker не только «по коду», но и фактически по наличию шаблона в runtime image.
- У операторов появился явный operational outcome «Не выдан» без подмены его частичной выдачей или заметками.
- Выбор ручных позиций стал быстрее и ближе к реальной текущей раздаче, что снижает риск опечаток в журнале.

## 2026-03-14: Mega-sprint — все задачи завершены

**Context**: Мега-спринт из 18 задач полностью завершён. Ниже ключевые архитектурные решения.

### Export by roles (section_vis_export)
- Добавлен ключ `section_vis_export` в ALLOWED_SETTINGS_KEYS (backend) и DEFAULT_SECTION_PERMS (frontend)
- Кнопка экспорта в OrdersPage защищена проверкой `useSectionPermissions()` → `canExport`
- Владелец может менять видимость через SettingsPage → Доступ

### Analytics chat_id filter
- Три расширенных аналитических эндпоинта (pickup-detailed, chat-stats, delivery-stats) теперь принимают optional `chat_id: int | None = Query(None)`
- Frontend передаёт `chatParam` во все вызовы (основные + расширенные)

### Project-state expansion (Tasks 14 + 16)
- Backend `_get_project_state_payload` возвращает `error_orders_count` и `awaiting_delivery_count`
- Frontend System tab показывает 3 новых карточки: ошибки, ожидающие выдачи, всего заказов
- Панель «Текущие проблемы» расширена: включает ошибочные заказы

### "Кем выдан" autofill (Task 2)
- Backend уже автозаполнял `delivered_by_name` из JWT. Frontend теперь отображает имя администратора в диалогах создания и редактирования выдачи.

### awaiting_delivery action (Task 6)
- Кнопка «📦 Ожидает выдачи» добавлена в диалог деталей заказа (OrdersPage)
- Активна для заказов в статусах active/needs_admin/partial, если delivery_status ещё не awaiting_delivery/delivered

### Excel export filename (Task 9)
- Имя файла теперь включает контекст: чат, точку выдачи, даты
- Формат: `{type}_{chat}_{pickup}_{dates}.xlsx`

---

## 2026-03-14: raw_text во всех вкладках DeliveryPage + RawTextCell компонент

**Context**: Исходный текст заказа (raw_text) был виден только во вкладке «К выдаче». Для операторов выдачи важно видеть его и при проверке журнала, отмеченных заказов и ошибок.

### Решения

**Backend — deliveries.py:**
- `list_deliveries`: SQL изменён с `SELECT * FROM order_deliveries` на `SELECT od.*, o.raw_text AS order_raw_text FROM order_deliveries od LEFT JOIN orders o ON o.id = od.order_id` — теперь каждая запись журнала возвращает raw_text заказа.
- `list_delivered_orders`: добавлен `raw_text` в маппинг ответа (данные уже были в `SELECT o.*`).
- `ErrorOrderResponse` уже содержал raw_text — изменений не требовалось.

**Backend — schemas.py:**
- `OrderDeliveryResponse`: добавлено поле `raw_text: str | None = None`.

**Frontend — client.ts:**
- `DeliveryRow`: добавлено `raw_text?: string | null`.
- `DeliveredOrderRow`: добавлено `raw_text?: string | null`.

**Frontend — RawTextCell.tsx (новый компонент):**
- Переиспользуемый компонент для отображения raw_text.
- Короткие тексты (<80 символов) показываются сразу.
- Длинные тексты — сворачиваемый блок (2 строки + «показать всё» / «свернуть»).
- Monospace шрифт, компактный размер.

**Frontend — DeliveryPage.tsx:**
- Все 4 вкладки теперь показывают колонку «Исходный текст» через `<RawTextCell>`.
- Вкладка «К выдаче» — заменён inline tooltip на `RawTextCell`.
- Вкладки «Журнал», «Отмеченные», «Ошибки» — добавлена новая колонка.
- colSpan для пустых состояний обновлён.

**Альтернативы:** Tooltip (плохая UX для длинных текстов), модальное окно (слишком тяжело). Сворачиваемый inline — оптимальный баланс.

---

## 2026-07-17: Comprehensive admin panel enhancements — analytics, delivery, orders, ACL

**Context**: Серия запросов на улучшение admin panel: расширенная аналитика, новые статусы заказов, удаление заказов, авто-очистка ошибок, owner-only каталоги, raw_text в очереди выдачи, детальная аналитика по точкам/чатам/выдаче, панель текущих ошибок в системе.

### Решения

**Backend — orders.py:**
- Новый статус `awaiting_delivery` ("Ожидает выдачи") добавлен в `_VALID_ORDER_STATUSES` и `_ORDER_STATUS_LABELS`.
- Новый endpoint `DELETE /orders/{order_id}` — owner-only, каскадное удаление (order_deliveries → order_lines → order), с audit log.
- Улучшенный `update_order`: при смене статуса `needs_admin → active` автоматически очищается `error_text`; при пустом `error_text` + все строки ОК — авто-промоция в `active`.
- Исправлен bug: в `export_orders_template_xlsx` отсутствовала переменная `order_deliveries`.

**Backend — analytics.py:**
- Новый endpoint `GET /analytics/pickup-detailed` — детальная аналитика по точкам выдачи: total, delivered, partial, errors, active, delivery_rate.
- Новый endpoint `GET /analytics/chat-stats` — аналитика по чатам: total, delivered, errors, active, delivery_rate. С резолвом названий чатов.
- Новый endpoint `GET /analytics/delivery-stats` — статистика выдачи: delivery_breakdown по статусам, top_issuers по количеству.

**Frontend — client.ts:**
- Новые типы: `PickupDetailedStats`, `ChatStats`, `DeliveryStats`.
- Новые методы: `deleteOrder()`, `getPickupDetailedStats()`, `getChatStats()`, `getDeliveryStats()`.
- `awaiting_delivery` добавлен в `TEMPLATE_STATUS_LABELS`.

**Frontend — OrdersPage.tsx:**
- Добавлен `awaiting_delivery` в STATUS_CONFIG и DELIVERY_STATUS_CONFIG.
- Новый фильтр "Ожидают выдачи" в обоих dropdown'ах (основной + экспорт).
- Кнопка удаления заказа (owner-only) с диалогом подтверждения.
- Определение роли owner через `getMe()`.

**Frontend — DeliveryPage.tsx:**
- Колонка "Исходный текст" (raw_text) в очереди выдачи — с tooltip для полного текста.
- `awaiting_delivery` добавлен в DELIVERY_STATUS_CONFIG.

**Frontend — AnalyticsPage.tsx:**
- Новая секция "Детальная аналитика по точкам" — stacked bar chart + delivery_rate карточки.
- Новая секция "Аналитика по чатам" — карточки с LinearProgress.
- Новая секция "Статистика выдачи" — breakdown + top issuers.
- Lazy loading новых данных (Promise.allSettled, не блокирует основной загрузку).

**Frontend — CatalogsPage.tsx:**
- Кнопки создания/редактирования/удаления каталогов и товаров скрыты для не-owner.
- `isOwner` state определяется через `getMe()`.

**Frontend — SettingsPage.tsx:**
- Панель "Текущие проблемы" в System tab — показывает все обнаруженные проблемы (БД, Worker, Telegram, Schema, Failed updates, Webhook errors).

### Consequences
- 3 новых backend endpoint'а, 1 новый DELETE endpoint
- 6 frontend страниц обновлено
- Owner-only guard на всех мутационных операциях каталогов (backend + frontend)
- Детальная аналитика по 3 измерениям: точки, чаты, выдача
- TypeScript компиляция — 0 ошибок

## 2026-07-16: Parser hardening, audit logging, UX regions autocomplete

**Context**: Парсер допускал ошибки при обработке реальных сообщений из samples/telegram: сокращения с точкой ("с.", "мор.") ломали разбиение строк; телефон "43 80" (разделённый пробелом) не извлекался; единица "ст.б." не распознавалась; числа-характеристики продукта (как "160" в "Омега 160 капс 3 шт") блокировали нахождение реального количества. Кроме того: отсутствовало логирование 36+ mutation endpoints, регионы пользователей редактировались свободным текстом, на странице выдачи не отображались числа при монтировании.

### Решения

**Parser (backend/app/domain/order_domain.py, backend/app/parser/text_parser.py):**
- Защита сокращений с точкой: "с.", "мор.", "св.", "свеж.", "х." защищаются плейсхолдером перед подсчётом точек-разделителей. Единицы (кг., шт., уп.) НЕ защищаются, т.к. обозначают границы товаров.
- Распознавание разделённого пробелом телефона: "43 80" → 4380 в `_strip_inline_order_header()` и `_is_meta_fragment()`.
- **Sanity check внутри finditer loop**: вместо выхода с `None` при первом insane qty (160), парсер продолжает искать следующее совпадение ("3 шт"). Критичный fix для "Омега 160 капс 3 шт".
- **Range sanity**: диапазоны где start >> end*10 (напр. "160 -1шт") пропускаются, позволяя стандартному qty найти "1шт".
- **Единица "ст.б."** (стеклянная банка) добавлена в _UNIT_PAT и маппинг на "банка".
- **Расширены _COMMON_NAMES**: зуля, лера, римма, алевтина, катерина, эмма, роза, флора, гюльнара, зульфия.
- **Расширены _PLACE_NAMES_LOWER**: героев, миля, межевая, бигам, кирова, лебедева.
- **Skip-next для адресных префиксов**: после "пр", "ул", "наб" и т.д. следующее слово пропускается при поиске имени (fix: "пр.Героев" → "Героев" больше не определяется как имя).

**Audit logging (admin_service):**
- Создан `admin_service/app/audit.py` с `write_audit()` helper function.
- Добавлено 36+ вызовов `write_audit()` во все 9 файлов роутеров:
  - orders.py (4): order.update, order_line.create/update/delete
  - users.py (7): user.create, roles_update, status_update, delete, regions_update, password_reset, promote_owner
  - deliveries.py (4): delivery.create/update, pickup_assignment.create/delete
  - catalogs.py (12): chat.update, catalog CRUD, catalog_item CRUD, admin_chat assign/unassign
  - settings.py (4): settings.update, pickup_place.create/update/delete
  - admin_settings.py (1): settings.dev_mode
  - auth.py (2): auth.bootstrap, auth.login
  - spam.py (2): spam.dismiss, spam.delete

**UX: Regions autocomplete (backend + frontend):**
- Новый endpoint `GET /regions/all` в users.py — возвращает дедуплицированный список всех регионов из admin_user_regions + pickup_places.title.
- Новый метод `getAllRegions()` в API client.
- UsersPage: TextField для регионов заменён на MUI `Autocomplete` (multiple + freeSolo) в обоих местах:
  - Диалог редактирования регионов пользователя
  - Форма создания нового пользователя
- Удалён parseRegionInput, состояние regionInput изменено с string на string[].

**DeliveryPage mount fix:**
- Добавлен отдельный useEffect на mount, загружающий данные всех 4 табов.
- Теперь при открытии страницы сразу отображаются числа в stat-картах и вкладках.

**Validation fix:**
- `le=200` → `le=500` в orders.py для параметра limit (frontend отправляет 500).

### Consequences
- Парсер корректно обрабатывает "Елена 43 80 горбуша с. мор. 1 шт", "Омега 160 капс 3 шт", "1 ст.б.".
- Все 268 parser-тестов проходят.
- Каждое мутирующее действие в admin panel записывается в audit log.
- Регионы выбираются из единого списка с автокомплитом, а не вводятся вручную.
- При открытии страницы выдачи числа видны сразу.

---

## 2026-03-14: Full Order Line CRUD + OrderEditDialog + UX infrastructure overhaul

**Context**: До этого заказы можно было редактировать только на уровне metadata (customer_name, phone_last4, pickup_place, status) через простой raw-edit dialog. Строки заказов (order_lines) были read-only: нельзя было ни добавить, ни изменить, ни удалить позицию. Также отсутствовала единая система для подтверждений (везде native `confirm()`), уведомлений и глобальной обработки ошибок.

### Решения

**Backend (admin_service):**
- Добавлены 3 новых CRUD-эндпоинта для order lines:
  - `POST /orders/{id}/lines` — добавить строку в заказ
  - `PATCH /orders/{id}/lines/{line_id}` — обновить поля строки
  - `DELETE /orders/{id}/lines/{line_id}` — удалить строку
- Добавлен `GET /audit-log` — просмотр журнала действий с пагинацией и фильтрацией
- Добавлены Pydantic-модели `OrderLineCreateRequest` и `OrderLineUpdateRequest`
- Все операции защищены owner-only доступом и pickup scope

**Frontend (admin-web):**
- **OrderEditDialog** — полноценный редактор заказа:
  - Редактирование metadata (имя, телефон, точка выдачи, статус, статус доставки)
  - Inline-таблица строк заказа с редактированием каждого поля
  - Добавление новых строк в заказ
  - Удаление строк с диалогом подтверждения
  - Просмотр исходного текста сообщения (collapsible)
  - Ссылка на оригинал в Telegram
- **ConfirmDialog** — красивый MUI Dialog вместо native `confirm()`:
  - Severity-стилизация (warning/error/info)
  - Иконки по типу действия
  - Loading state support
  - Применён в 6 местах: CatalogsPage, UsersPage, PickupAdminPage (3 случая)
- **ErrorBoundary** — глобальная обработка ошибок React:
  - Ловит ошибки рендеринга и показывает fallback
  - Кнопки «На главную» и «Перезагрузить»
- **SnackbarProvider** — контекстная система toast-уведомлений:
  - `showSuccess()`, `showError()`, `showInfo()`, `showWarning()`
  - Очередь сообщений, auto-dismiss
- **AuditLogPanel** — компонент просмотра журнала действий:
  - Фильтрация по типу объекта
  - Expandable details
  - Relative timestamps
  - Пагинация
  - Интегрирован как 9-я вкладка в Настройках
- **API client** — 4 новых типизированных метода + `AuditLogEntry` тип

### Consequences

- Заказы теперь полностью управляемы через UI: от metadata до каждой отдельной строки.
- Все деструктивные действия имеют подтверждение через красивые диалоги.
- Глобальная обработка ошибок предотвращает "белый экран смерти".
- Тосты дают мгновенную обратную связь на каждое действие.
- Audit log позволяет отслеживать кто что менял.

## 2026-03-13: Backup/restore moved into project-level operational scripts

**Context**: До этого backup БД оставался в категории "ну сделайте как-нибудь через mysqldump". Для реальной эксплуатации этого мало: процедура должна быть воспроизводимой, задокументированной и одинаковой для локального/Docker-сценария.

### Решения

- Добавлены `scripts/backup_db.sh` и `scripts/restore_db.sh`.
- Поддержаны два operational режима:
  - `docker` — через `docker compose exec mysql`
  - `native` — через локальные `mysqldump` / `mysql`
  - `auto` — выбрать доступный режим автоматически
- В `Makefile` добавлены `make backup-db` и `make restore-db FILE=...`.
- Backup по умолчанию складывается в `backups/db/` как `.sql.gz`.
- Restore требует явного подтверждения, чтобы снизить риск случайного перезаписывания базы.

### Consequences

- Backup/restore теперь входит в нормальный operational toolbox проекта.
- Перед миграциями и рискованными ручными операциями можно использовать единый сценарий, а не импровизацию.
- Следующий логичный шаг — добавить scheduled execution и внешнее хранилище, но baseline уже существует.

## 2026-03-13: Canonical local Python bootstrap moved to project scripts instead of manual venv folklore

**Context**: В репозитории накопились разные способы локального запуска Python-контуров: кто-то пользовался системным `python`, кто-то старой `.venv`, кто-то отдельно создавал `.venv-admin`. Это делало локальные тесты и сервисные launcher'ы зависимыми от «исторической удачи» конкретной машины.

### Решения

- Добавлен `scripts/setup_python_envs.sh` как каноничный bootstrap для:
  - `./.venv` (`backend` + shared dev tooling)
  - `./.venv-admin` (`admin_service`)
- В `Makefile` добавлены entrypoints:
  - `make py-envs`
  - `make py-env-backend`
  - `make py-env-admin-service`
- `scripts/admin_service.sh` теперь явно подсказывает, как восстановить `.venv-admin`, если она отсутствует.
- Документация обновлена так, чтобы manual venv creation больше не был основным рекомендуемым сценарием.

### Consequences

- Локальный setup становится воспроизводимее и ближе к CI-подходу.
- Снижается зависимость от системного Python и случайно «живых» старых venv.
- Старые вручную собранные env могут всё ещё существовать, но больше не считаются source of truth.

## 2026-03-13: Frontend lint added as a src-only ESLint baseline, not a repo-wide cleanup bomb

**Context**: После Python lint baseline следующий логичный шаг — дать admin-web такой же автоматический gate. Но запускать строгий ESLint по всему frontend-контенту (`src`, config, Playwright, misc files) сразу означало бы получить много инфраструктурного шума и отвлечься от реальных ошибок.

### Решения

- В `admin-web` добавлен ESLint baseline (`eslint.config.js`).
- Базовый scope ограничен `admin-web/src`.
- Линтер включён как low-noise gate без style-правил и без forcing cleanup старого кода.
- Добавлены команды:
  - `npm run lint`
  - `make check-lint-frontend`
  - `bash scripts/run_checks.sh admin-web-lint`
- CI теперь запускает frontend lint перед build + tests.

### Consequences

- Admin-web получает автоматическую проверку, но без резкого роста ложного шума.
- E2E/config-файлы можно подключать к lint позднее отдельной итерацией.
- Quality gate для frontend теперь ближе по философии к Python baseline: сначала low-noise safety, потом постепенное усиление.

## 2026-03-13: Python lint introduced as a low-noise ruff baseline, not a style crusade

**Context**: После появления CI и unified local checks проекту нужен был автоматический Python lint gate. Но включать строгий style-набор сразу на историческом коде означало бы получить лавину нерелевантных замечаний и превратить полезную проверку в шум.

### Решения

- Добавлен `.ruff.toml` как минимальный baseline-конфиг.
- На старте включены только low-noise правила, ориентированные на реальные ошибки:
  - `E9`
  - `F63`
  - `F7`
  - `F82`
- Добавлен `requirements-dev.txt` с `ruff`.
- Локальный запуск стандартизирован через `scripts/run_checks.sh lint-python` и `make check-lint-python`.
- В CI добавлен отдельный job `Python lint (ruff baseline)`.

### Consequences

- Проект получает автоматическую проверку Python-кода на действительно опасные ошибки без массового style-шторма.
- Линтер теперь можно усиливать итеративно, когда кодовая база и команда к этому готовы.
- Локальный и CI-контур используют один и тот же минимальный quality gate.

## 2026-03-13: Added baseline GitHub Actions CI instead of waiting for a "perfect" pipeline

**Context**: В репозитории уже были pytest / Vitest / Playwright сценарии, но они жили только локально. Из-за этого проект зависел от ручного запуска и от состояния локального окружения разработчика.

### Решения

- Добавлен базовый pipeline `.github/workflows/ci.yml`.
- В CI включены только те проверки, которые уже отражают реальные критичные зоны и могут работать без подъёма полного production-стека:
  - targeted backend pytest;
  - `admin_service/tests`;
  - `admin-web` build + Vitest;
  - Playwright smoke на ACL navigation flow.
- Принято решение не ждать полного enterprise-пайплайна, а сначала зафиксировать **рабочий baseline**, который можно постепенно усиливать.
- Локальные каноничные команды запусков зафиксированы в `README.md` и [[09-testing-strategy]].

### Consequences

- Репозиторий получает минимальную автоматическую защиту от регрессий на backend/admin_service/admin-web.
- Дальнейшие улучшения CI можно делать итеративно: lint, Docker build sanity, расширенные regression suites.
- Локальная рассинхронизация env всё ещё остаётся отдельной задачей, но уже не блокирует наличие базового CI-контра.

## 2026-03-13: AI stays optional, regex-first, with an offline/local provider slot

**Context**: Продуктовый приоритет — бот должен продолжать работать даже при блокировке или недоступности внешних AI-сервисов. При этом архитектуру надо сохранить такой, чтобы позже можно было подключить лёгкий локальный AI-слой без переделки всего пайплайна распознавания.

### Решения

- `backend/app/ai/order_recognizer.py`: зафиксирован режим `regex-first` как основной production path.
- `AI_MODE=disabled` остаётся дефолтом.
- `AI_MODE=local` объявлен как **offline/local provider slot** — архитектурное место для будущего локального распознавания без SaaS-зависимостей.
- `AI_MODE=openai` оставлен только как **явно включаемая** внешняя опция; без `OPENAI_API_KEY` recognizer больше не пытается «магически» уйти во внешний API и откатывается в local slot.
- В коде убраны устаревшие вызовы `use_ml=False`; recognizer теперь явно создаётся в `ai_mode="disabled"` там, где нужен deterministic/offline-safe путь.

### Consequences

- Базовый сценарий работы бота не зависит от внешних AI-провайдеров.
- Переход на локальный AI в будущем можно делать через уже существующий `AI_MODE=local`, не ломая внешний контракт recognizer'а.
- Внешний AI остаётся опциональным инструментом для экспериментов, а не скрытой runtime-зависимостью.

## 2026-03-13: In-memory rate limiting is intentional for single-instance Telegram send protection

**Context**: Термин `in-memory rate limiting` звучит подозрительно, будто это временный костыль. Но для Telegram-бота в single-instance режиме он решает вполне конкретную задачу: не дать процессу превысить outbound Bot API limits и не утонуть в burst-нагрузке от больших чатов.

### Решения

- `backend/app/services/rate_limiter.py`: лимитер зафиксирован как локальный singleton в памяти процесса для **защиты исходящих сообщений**.
- Лимитер трактуется не как защита «от количества участников», а как защита от **скорости отправки**.
- Добавлено разделение bucket-политик:
  - group/supergroup/channel (`chat_id < 0`) — более строгий per-chat лимит;
  - private chats (`chat_id > 0`) — более свободный лимит.
- Параметры лимитов вынесены в env (`TELEGRAM_SEND_GLOBAL_*`, `TELEGRAM_SEND_GROUP_*`, `TELEGRAM_SEND_PRIVATE_*`), чтобы single-instance high-load можно было тюнить без правки кода.
- Shared/Redis-backed limiter остаётся не обязательным «на сейчас», а следующей ступенью только при горизонтальном масштабировании на несколько worker/process instances.

### Consequences

- Для одного процесса/одного bot worker лимитер остаётся простым, быстрым и независимым от внешней инфраструктуры.
- Нагрузка вида `20-30` чатов по `300k` участников не является проблемой сама по себе, пока фактическая скорость исходящих ответов укладывается в лимиты Telegram.
- Если понадобится несколько параллельных bot worker-инстансов, текущее in-memory состояние уже не будет достаточно согласованным — тогда нужен Redis/shared state.

## 2026-03-13: Regional-admin user management finalized with explicit region assignments

**Context**: После упрощения роли `regional_admin` продукт уже знал, что супервайзер должен работать "в рамках своих городов", но это оставалось скорее обещанием в docs, чем завершённым operational flow. Таблица `admin_user_regions` уже существовала, а UI и API всё ещё не доводили её до реального использования.

### Решения

- `admin_service/app/api/routers/users.py`: добавлены region-aware ответы `/me`, `/users`, `/users/full`, новые ручки `GET/PUT /users/{id}/regions` и backend-scope для `regional_admin` по пересечению регионов.
- `admin_service/app/core_tables.py`: `admin_user_regions` включена в reflected core tables, чтобы feature работала без новой ORM-модели.
- `admin-web/src/pages/UsersPage.tsx`: в таблицу пользователей добавлена колонка `Города / регионы` и отдельный UI для редактирования назначений.
- Создание пользователя через `regional_admin` теперь может автоматически наследовать регионы текущего супервайзера, если их не указали явно.
- `admin-web/src/pages/OrdersPage.tsx`: quick-actions выдачи расширены на `needs_admin` / `partial`, но одношаговый `transferred` убран, чтобы не обходить обязательный ввод новых данных получателя.

### Consequences

- Региональный админ теперь не просто "есть в системе", а реально ограничен и полезен в operational flow.
- Назначение локальной команды стало прозрачным прямо в UI.
- Быстрые действия по выдаче больше не нарушают собственные же бизнес-правила про `transferred`.

## 2026-03-13: UI role model simplified to owner / regional_admin / manager / pickup_admin

**Context**: В продукте накопились лишние сущности `admin` и `viewer`, из-за чего настройки прав стали путать реальные бизнес-роли. Параллельно owner-инструменты (`Шаблон выгрузки`, `Excel Превью`) жили отдельными пунктами меню, хотя логически это часть раздела настроек.

### Решения

- `admin-web/src/pages/SettingsPage.tsx`: настройки перегруппированы по смыслу — `Приложение`, `Бот`, `Команда и точки`, `Интерфейс`, `Инструменты`, `Безопасность`, `Доступ`, `Система`.
- `TemplatesPage` и `ExcelPreviewPage` встроены в `Настройки → Инструменты`, а отдельные маршруты `/templates` и `/excel-preview` теперь просто ведут в Settings.
- В UI больше не предлагаются роли `admin` и `viewer`; legacy `admin` трактуется как совместимый `manager`, `viewer` скрыт из новых сценариев.
- `admin_service/app/admin_sync.py` и `api/common.py`: backend начал считать `manager` основной operational-ролью для заказа, сохранив legacy `admin` только для обратной совместимости.
- `backend/app/domain/order_domain.py` и `admin_service/app/api/routers/deliveries.py`: добавлены сопутствующие guards — split-phone `43 80 → 4380` и обязательные новые данные получателя для статуса `transferred`.

### Consequences

- Настройки стали ближе к реальной оргструктуре команды и меньше зависят от исторических терминов.
- Owner-инструменты больше не засоряют основное меню.
- Система мягко переживает старые роли, но ведёт новых пользователей уже по новой модели.

## 2026-03-13: Delivery workspace moved from dashboard-style browsing to pickup-day operations

**Context**: Реальный флоу точки выдачи оказался неудобным: оператор попадал на owner-dashboard, delivery-раздел больше выглядел как журнал, а частичная выдача жила только в рамках исходных строк заказа. Template export ещё и тянул лишние листы из заготовки.

### Решения

- `admin-web/src/App.tsx` и `MainLayout.tsx`: корневой маршрут теперь ведёт в operational section (`/orders` или `/delivery`), dashboard убран из основного меню и остаётся только как скрытый owner-only route `/dashboard`.
- `admin-web/src/pages/DeliveryPage.tsx`: delivery-раздел превращён в рабочее место смены с вкладками `К выдаче`, `Журнал`, `Отмеченные`, `Проблемы`, фильтрами по точке/поиску и быстрыми действиями по выдаче.
- Частичная выдача теперь поддерживает не только line-based split, но и ручные позиции через `custom_title`, чтобы фиксировать реальную выдачу, а не только идеальный состав заказа.
- `admin_service/app/api/routers/orders.py`: template export теперь использует один активный лист, убирает лишние страницы и добавляет delivery metadata в итоговый Excel.

### Consequences

- Pickup-admin заходит сразу в рабочий operational flow точки.
- Частичные и нестандартные выдачи фиксируются ближе к реальной картине смены.
- Шаблонный экспорт перестал тянуть лишние листы и стал полезнее для офлайн-работы на выдаче.

## 2026-03-13: Pickup-admin scope hardened on orders/exports and pickup-point operations centralized

**Context**: После расширения delivery-флоу роль `pickup_admin` уже присутствовала в UI, но backend order route-ы и XLSX exports оставались слишком широкими. Параллельно управление точками выдачи оказалось размазанным между `Settings` и отдельным разделом назначений, а distribution mode работал только как глобальный флаг.

### Решения

- `admin_service/app/api/routers/orders.py`: `pickup_admin` теперь ограничен назначенными точками на `GET /orders`, `GET /orders/{id}`, `PATCH /orders/{id}`, `GET /orders/export`, `GET /orders/export-template`, `GET /orders/export-history`.
- `admin_service/app/api/common.py`: вынесены общие helper-ы для pickup scope, title/chat-id resolution и единой выдачи списка точек.
- `admin_service/app/api/routers/settings.py`: запись настроек и CRUD точек выдачи зафиксированы как `owner`-only, добавлены ключи `distribution_mode_chat_ids` и `distribution_message`.
- `backend/app/worker.py`: режим раздачи расширен до chat-scoped сценария, чтобы можно было остановить приём заказов не глобально, а только для выбранных городов/чатов.
- `admin-web/src/pages/PickupAdminPage.tsx` и `MainLayout.tsx`: основной operational flow по точкам выдачи перенесён в раздел `Админы точек`.

### Consequences

- Pickup-admin больше не может случайно увидеть чужие заказы или экспортировать не свои точки.
- Owner получает единое место для управления и назначениями, и самими точками выдачи.
- Distribution mode стал пригоден для реальной поэтапной раздачи по городам.
- Документация и UI теперь согласованы с backend-ограничениями, а не живут в параллельных вселенных.

## 2026-03-13: Operational commands switched from `alembic upgrade head` to `heads`

**Context**: Полный restart через `make sync` упёрся в `Multiple head revisions are present`, потому что в репозитории одновременно существуют несколько Alembic head-веток. База при этом может быть уже актуальной, но старые operational команды всё равно падают на `upgrade head`.

### Решения

- `Makefile` переведён на `docker compose exec -T api alembic upgrade heads`.
- `README.md`, [[10-ops-runbook]], [[11-restart]] и связанные upgrade-документы обновлены на `upgrade heads`.
- В restart docs явно зафиксировано, почему `heads` — это не случайность, а корректное поведение при нескольких независимых миграционных вершинах.

### Consequences

- `make sync` снова соответствует реальному состоянию миграционного дерева и не падает на стандартном full-restart path.
- Операционные инструкции в репозитории перестают противоречить друг другу.

## 2026-03-13: Parser no longer accepts qty-only feedback as orders

**Context**: Сообщения-отзывы без товара и без клиентских метаданных вроде `Рыбина на 3.850 кг. с бонусом` могли проходить `looks_like_order()` только за счёт одной распознанной qty-строки. Это создавало ложные `needs_admin` / шум в заказах.

### Решение

- В `backend/app/domain/order_domain.py` добавлен ранний guard: если в тексте нет product keyword и нет ни имени, ни телефона, ни точки выдачи, такое сообщение не считается заказом.
- Добавлен регресс-тест в `backend/tests/test_parser_upgrade.py` на реальный feedback-case с `с бонусом`.
- Обновлены parsing docs, чтобы правило было видно не только в коде.

### Consequences

- Отзывы с весом/бонусом перестают создавать ложные заказы.
- Нормальные сообщения вроде `Форель 3.850 кг` не страдают, потому что содержат товарный keyword.

## 2026-03-13: Delivery permissions hardened, pickup-admin workflow expanded, password-hash exposure removed

**Context**: После добавления delivery-трекинга появилось смешение ролей: обычный `admin` мог работать с выдачей, хотя бизнес-логика требует owner + админов выдачи (`pickup_admin`). Выданные заказы показывались отдельно, но без нормального редактирования состава выдачи. Также owner UI показывал password hash пользователей, что небезопасно даже для закрытой админки.

### Решения

- **Delivery access** (`admin_service/app/api/common.py`, `admin-web/src/App.tsx`): управление выдачей ограничено ролями `owner` и `pickup_admin`. Обычный order-admin / manager больше не считается delivery-оператором.
- **Orders visibility for pickup_admin** (`admin-web/src/App.tsx`): админ выдачи получил доступ к разделу `Заказы`, чтобы исправлять `needs_admin` / проблемные заказы перед выдачей.
- **Order PATCH guard** (`admin_service/app/api/routers/orders.py`): поле `delivery_status` теперь можно менять только owner/pickup_admin.
- **Delivery status fidelity** (`admin_service/app/api/routers/deliveries.py`): статус заказа теперь сохраняет `transferred` как отдельное значение, а не схлопывает его в `partially_delivered`.
- **Delivered orders UX** (`admin_service/app/api/routers/deliveries.py`, `admin-web/src/pages/DeliveryPage.tsx`): список выданных заказов теперь показывает, кто именно выдал заказ, кому фактически выдали, и позволяет открыть последнюю выдачу на редактирование.
- **Per-line delivery split** (`admin-web/src/pages/DeliveryPage.tsx`): модалки создания/редактирования выдачи поддерживают разбиение по строкам заказа на «выдано» и «осталось».
- **Error orders workflow** (`admin-web/src/pages/DeliveryPage.tsx`, `admin_service/app/api/routers/deliveries.py`): отдельный список ошибочных заказов ограничивается назначенными точками для `pickup_admin` и получил inline-исправление заказа.
- **Security hardening** (`admin_service/app/api/routers/users.py`, `admin-web/src/pages/UsersPage.tsx`): endpoint `/users/full` и owner UI больше не отдают/не показывают `password_hash`.
- **Section visibility config** (`settings.py`, `SettingsPage.tsx`): роль `pickup_admin` и секция `section_vis_pickup_admin` добавлены в управляемую конфигурацию видимости разделов.

### Consequences

- Owner сохраняет полный operational control над выдачей и назначениями.
- Админы выдачи работают только в рамках назначенных пунктов и видят только свои проблемные/выданные заказы.
- Обычные модераторы заказов больше не могут случайно менять delivery state.
- UI owner'а перестал светить секретоподобные данные, что снижает blast radius при shoulder-surfing/скриншотах.

## 2026-07-15 (v2): Delivery status flow — editable orders + quick delivery actions

**Context**: Заказы со статусом `needs_admin` не могли быть отредактированы админами точек выдачи. Поле `delivery_status` (добавленное ранее в модель) не было доступно для редактирования через PATCH endpoint и не отображалось в интерфейсе управления заказами.

### Решения

- **Backend PATCH `/orders/{id}`** (`admin_service/app/api/routers/orders.py`):
  - Добавлен `_VALID_DELIVERY_STATUSES = {"pending", "delivered", "partially_delivered", "transferred"}`.
  - `delivery_status` добавлен в `_EDITABLE_FIELDS`.
  - Валидация `delivery_status` выполняется отдельно от `status` заказа.
- **OrdersPage — колонка «Выдача»**: В таблицу заказов добавлена колонка с Chip-индикатором статуса выдачи.
- **OrdersPage — кнопки выдачи в модалке**: В детальной карточке заказа добавлены быстрые кнопки «✅ Выдан», «⚠️ Частично», «🔄 Передан» (отображаются для активных заказов без статуса `delivered`).
- **OrdersPage — редактирование delivery_status**: В диалоге редактирования заказа добавлен Select для статуса выдачи.
- **Cleanup**: Удалён мёртвый код `history export` (тип `ExportMode`, `historyExportLoading`, ветки в `handleExecuteExport`, `HistoryIcon`), остававшийся после предыдущей сессии.
- **Build fix**: Исправлена ошибка `TS2304: Cannot find name 'HistoryIcon'` в OrdersPage.tsx.

### Delivery Flow

```
active (no delivery_status)
  → ✅ delivered           — полностью выдан
  → ⚠️ partially_delivered — выдан частично
  → 🔄 transferred        — передан другому покупателю
```

Pickup-админы и модераторы могут устанавливать статус выдачи через:
1. Быстрые кнопки в детальной карточке заказа
2. Select-поле в диалоге редактирования
3. PATCH API endpoint напрямую

### Rationale

- Pickup-админы работают «на месте» и должны отмечать выдачу без участия главных админов.
- Delivery status ортогонален order status: заказ остаётся `active`, но `delivery_status` фиксирует этап выдачи.
- Быстрые кнопки в модалке — один клик для самого частого действия (выдача).

## 2026-07-15: Parser sanity limits, distribution mode, stop-reason replies, Telegram deep links, UX improvements

**Context**: Парсер мог интерпретировать телефонные номера как количество (напр. «Елена 4424 Иваново 2» → qty 4424000). Бот не сообщал причину стопа при отклонённых товарах. Не было режима выдачи, блокирующего новые заказы. Фронтенд не давал прямой ссылки на исходное сообщение в Telegram.

### Решения

- **Parser qty sanity** (`backend/app/parser/text_parser.py`): Добавлен `_MAX_SANE_QTY` dict с порогами по единицам (кг:100, г:50000, шт:200, уп:100, банка:100, л:50, мл:50000, без единицы:500). Функция `_is_sane_qty(value, unit)` вызывается во ВСЕХ ветках парсинга qty (стандартный, комбо кг+г, упаковка, дробь, диапазон). Значение сверх порога → `(None, None, None)`.
- **Implicit single-unit fix**: Предотвращён баг, при котором «3 банки» парсились как «1 банка» из-за того, что `_IMPLICIT_SINGLE_UNIT_RE` срабатывал до стандартного `_QTY_RE`. Добавлен guard `if not _NUMBER_RE.search(frag_norm)`.
- **Distribution mode** (`worker.py`): Добавлен хелпер `_get_bot_setting(db, key, default)`. При `distribution_mode=true` новые заказы отклоняются с сообщением «📦 Сбор заказов завершён. Идёт выдача.». Настройка управляется через админ-панель (Settings).
- **Stop-reason в ответах бота** (`order_processing.py`, `order_presenter.py`): При rebuild order lines для stopped-товаров теперь сохраняются `item_title`/`item_sku` и передаётся `stop_reason`. Бот показывает «🛑 стоп (причина)» вместо generic отклонения.
- **Telegram deep link** (`admin_service/app/api/routers/orders.py`, `schemas.py`): В `OrderDetailResponse` добавлено поле `source_message_url`. Для supergroup чатов (chat_id с префиксом `-100`) генерируется ссылка `https://t.me/c/{chat_id}/{message_id}`.
- **Edited message note** (`worker.py`): При обработке отредактированного сообщения добавляется auto_note «✏️ Заказ обновлён (сообщение отредактировано)».
- **Pickup admin assignment fix** (`deliveries.py`): `db.get(AdminUser, id)` заменён на `select(AdminUser).options(selectinload(AdminUser.roles))` для корректной загрузки ролей.
- **Frontend**: Удалена кнопка экспорта истории, добавлена кнопка «Открыть в Telegram» в карточке заказа, «Точки выдачи» вынесены в отдельный пункт навигации (`/settings?tab=2`), Settings поддерживает `?tab=N` deep link.

### Rationale

- Sanity limits — защита от мусорных данных, которые портят статистику и Excel-экспорт.
- Distribution mode — оператор может приостановить приём заказов на время выдачи, не отключая бота.
- Stop-reason feedback — пользователь понимает, почему товар не принят, и может скорректировать заказ.
- Telegram deep link — админ быстро находит исходное сообщение для контекста.

## 2026-03-12: Self-healing admin schema refresh and robust admin_service launcher

**Context**: После добавления delivery-функционала реальные `503` оказались вызваны не только отсутствующей миграцией, но и хрупким operational path: `admin_service` кэшировал список core tables до рестарта, а ручной запуск через `./.venv-admin/bin/uvicorn` мог падать на устаревшем shebang после пересоздания venv.

**Decision**:
- `admin_service/app/api/common.py`: `get_tables(*required)` теперь при необходимости сам перечитывает reflected core tables и только потом возвращает `503`.
- Delivery endpoints переведены на `get_tables(...required tables...)`, чтобы сервис автоматически подхватывал свежеприменённые backend-миграции без отдельного «пустого» 503 до ручного refresh cache.
- Добавлен единый запускатель `scripts/admin_service.sh`, который использует `.venv-admin/bin/python -m uvicorn ...` вместо прямого shebang-скрипта `uvicorn`.
- Парсер заказов теперь понимает shorthand вида `карт.фри -упак.` как `1 уп`, что покрывает реальные краткие заказы по упаковочным позициям.

**Consequences**:
- После применения backend-миграций delivery/pickup-admin endpoints быстрее возвращаются в строй и меньше зависят от ручного рестарта cache.
- Локальный/серверный старт `admin_service` устойчивее к пересозданию venv.
- Короткие заказы без явной цифры для упаковочных товаров распознаются стабильнее.

## 2026-03-13: Make auth throttling less brittle and surface login cooldown in UI

**Context**: После нескольких неудачных или случайно задвоенных логин- попыток админка продолжала бомбить `/auth/login`, а backend rate limiter считал все auth-запросы подряд. В результате пользователь видел каскад одинаковых `429 Too Many Requests` без понятного объяснения и без защиты от повторных кликов.

**Decision**:
- `RateLimitMiddleware` теперь ведёт отдельную корзину именно для `/auth/login` и считает в ней только неуспешные логин-попытки; успешный вход сбрасывает накопленный счётчик ошибок.
- `Retry-After` для `429` считается по реальному оставшемуся времени до освобождения окна, а не всегда равен полному размеру окна.
- `admin-web/src/pages/LoginPage.tsx` получил submit-lock, disabled state для полей/кнопки во время запроса и понятный cooldown-текст при `429`.

**Consequences**:
- Случайные двойные клики и повторный Enter больше не размножают одинаковые login-запросы.
- Пользователь видит, сколько реально ждать до следующей попытки.
- Лимитер по логину остаётся защитным, но перестаёт быть «сам себе DOS». 

## 2026-03-13: Stabilize delivery role flows and surface order errors on main orders page

**Context**: После добавления функционала delivery/pickup-admin часть запросов стала хрупкой: новая роль могла отсутствовать после старта сервиса, новые backend-таблицы не отражались в `CORE_TABLES`, а ошибки заказов были видны только в отдельном delivery-флоу.

### Решения

- В `admin_service` системные роли теперь сидируются всегда, даже если `ADMIN_USER_IDS` пуст.
- В `CORE_TABLES` добавлены `order_deliveries` и `admin_user_pickup_points`, чтобы новые ручки не работали «в обход» состояния схемы.
- Назначение на точку теперь разрешено только пользователям, у которых уже есть роль `pickup_admin` (при этом роль может сочетаться с другими ролями).
- На странице `Заказы` ошибки теперь отображаются прямо в основном списке: есть quick-toggle `Только с ошибками`, признак проблем и краткий текст ошибки.
- Во фронтенд-клиент добавлен мягкий retry для временных ошибок сети/502/503/504 на безопасных GET-запросах.

### Rationale

- Новая роль должна существовать стабильно, а не «если повезёт со сценарием старта».
- Пользователь должен видеть проблемные заказы в основном рабочем экране, не переключаясь между разделами.
- Временные сетевые сбои не должны превращаться в ощущение, что «система снова сломалась».

## 2026-07-14: Delivery tracking, pickup_admin role, error orders, security hardening

**Context**: Необходимо отслеживать выдачу заказов на точках, ввести роль «админ точки выдачи» (pickup_admin), отображать ошибочные заказы отдельно, усилить безопасность, улучшить парсер и Excel-экспорт.

### Решения

- **Роль `pickup_admin`** (level=15): Новая роль между admin(10) и manager. Может отмечать выдачу заказов только на назначенных точках. Создаётся автоматически через `admin_sync.py`.

- **Модель `OrderDelivery`**: Таблица для отслеживания выдачи заказов — кто выдал, кому, когда, что выдано/осталось, передача другому лицу. Привязка к order + pickup_place.

- **Модель `AdminUserPickupPoint`**: Назначение pickup_admin на конкретные точки выдачи. Owner управляет назначениями через UI.

- **Delivery API** (`admin_service/app/api/routers/deliveries.py`): 10 эндпоинтов — CRUD выдач, список выданных заказов, ошибочные заказы, управление назначениями, «мои точки» для pickup_admin.

- **Security hardening**:
  - Валидация паролей (min 8 chars, uppercase+lowercase+digit)
  - Input sanitization (strip HTML/script tags)
  - CSRF token generation
  - JWT `jti` для уникальности токенов
  - Tiered rate limiting (15/min для auth, 120/min для остальных, Retry-After header)
  - Audit logging middleware для mutation запросов
  - Security headers: Permissions-Policy, Content-Security-Policy, HSTS

- **Excel export**: Новая колонка «Статус выдачи» с human-readable лейблами.

- **Parser improvements**:
  - Дроби: «1/2 кг» → 0.5 кг
  - Приблизительные: «около 2 кг» → 2 кг
  - Слово «по» (как цена) пропускается и не ломает парсинг

- **Frontend**:
  - DeliveryPage: 3 вкладки (выдачи, выданные заказы, ошибки) с модалками создания/редактирования выдачи, поддержка передачи другому лицу
  - PickupAdminPage: управление назначениями админов на точки
  - Обновлены App.tsx (routing + section permissions), MainLayout (nav items + pickup_admin chip)

### Rationale

- Pickup-admin = делегация рутины. Owner/admin не должны лично отмечать выдачу на каждой точке.
- Scoped access: pickup_admin видит только свои точки → минимальные привилегии.
- Error orders отдельно → быстрее находить и исправлять проблемы.
- Security hardening — подготовка к продакшену.

## 2026-03-11: Frontend test baseline for ACL/Settings regressions (Vitest + RTL)

**Context**: После серии UI-изменений (owner-only секции, динамическая `section_vis_*` видимость, 9 вкладок Settings, антиспам-действия) потребовалось закрыть фронтенд регрессии автоматическими тестами, особенно в местах, где уже были пользовательские жалобы.

### Решения

- В `admin-web` добавлен тестовый стек `Vitest + jsdom + @testing-library/react + @testing-library/user-event`.
- Добавлена базовая тест-конфигурация (`admin-web/vitest.config.ts`, `admin-web/src/test/setup.ts`) с полифилами `matchMedia` и `ResizeObserver`.
- Добавлены тесты на критичные UX/ACL зоны:
  - `src/components/MainLayout.test.tsx` — owner-only/role-based видимость меню + динамические section permissions + debug visibility.
  - `src/pages/SettingsPage.test.tsx` — 9 вкладок, антиспам load/delete/dismiss flow, сохранение изменений в «Доступ».
  - `src/theme/index.test.ts` — registry/persistence fallback для темы.
- В `admin-web/package.json` добавлены скрипты `test`/`test:watch`.

### Rationale

- Основные регрессии в админке возникали не в чистом бизнес-слое, а на UI-стыках: ACL, переключатели видимости и экраны настроек.
- Локальные быстрые frontend-тесты закрывают эти сценарии до ручной проверки и снижают риск повторных падений в owner-флоу.

## 2026-03-11: Add Playwright e2e for ACL navigation and spam moderation flow

**Context**: Unit-тесты закрывают логику компонент, но часть жалоб воспроизводится только в сквозном UI-потоке (авторизация → роутинг → навигация → действия в Settings).

### Решения

- В `admin-web` добавлен Playwright-контур (`playwright.config.ts`) с автостартом Vite dev server.
- Добавлены e2e-сценарии с API route-mocking:
  - `e2e/acl-navigation.spec.ts` — owner/admin видимость разделов, dynamic section visibility.
- Антиспам-флоу оставлен в детерминированном RTL-покрытии (`SettingsPage.test.tsx`) для стабильности CI.
- Добавлен helper `e2e/mockApi.ts` для детерминированных mock-ответов `/api/*` и состояния spam-списка.
- Добавлены npm-скрипты `e2e` и `e2e:headed`.

### Rationale

- Сквозные UI-тесты ловят регрессии на стыке routing/context/state, где unit-тесты часто «зелёные», а пользовательский флоу уже сломан.
- Mock API в e2e сохраняет скорость и повторяемость прогонов без зависимости от живого backend.

## 2026-06-18: High-load architecture, spam detection, section visibility, Users→owner

**Context**: Бот планируется на 50 чатов × 300к участников (~300 msg/min). Требовалась high-load архитектура, защита от спама/рекламы в чатах, пер-ролевое управление разделами админки, и ограничение раздела Пользователи до owner-only.

### Решения

- **Rate Limiter** (`backend/app/services/rate_limiter.py`): Thread-safe token-bucket с двумя уровнями — global (25 tok/sec, burst 30) + per-chat (1 tok/sec, burst 3). Singleton `get_rate_limiter()`. Stats tracking (sent/limited/tracked_chats). Периодическая очистка stale buckets каждые 10 мин. Интегрирован в `_send_user_reply()` worker.

- **Spam Detector** (`backend/app/services/spam_detector.py`): Score-based система (0–10 баллов). 7 индикаторов: внешние URL (+2–4), спам-фразы 20+ паттернов (+5), forwarded из канала (+3), промо-телефон (+3), CAPS (+1.5), emoji (+1), короткое сообщение+ссылка (+2). Консервативный порог 6.0 — обычное общение, конкурсы, заказы, t.me ссылки НЕ флагятся. Админы всегда trusted. Настройки через bot_settings (`spam_detection_enabled`, `spam_auto_delete`, `spam_notify_admins`, `spam_score_threshold`).

- **Concurrent Worker** (`worker.py`): ThreadPoolExecutor-режим через `WORKER_MAX_THREADS` env var (default: 1 = последовательный). Каждый thread получает свою DB session. Optimistic locking на уровне update (`TgUpdate.status == "new"` → `"processing"`).

- **Telegram deleteMessage** (`telegram_client.py`): Новая функция `delete_message()` для удаления спам-сообщений. Используется в worker (auto-delete) и admin_service (manual delete).

- **Spam Management API** (`admin_service/app/api/routers/spam.py`): 4 эндпоинта — `GET /spam/flagged` (список помеченных сообщений), `GET /spam/stats` (счётчики), `POST /spam/dismiss` (снять пометку), `POST /spam/delete` (удалить через Telegram API). Owner-only. Использует `message_snapshots.kind = "spam_flagged" | "spam_deleted"`.

- **Section Visibility** (фронтенд): Новый `SectionPermissionsContext` в App.tsx. Настройки `section_vis_*` (10 ключей, CSV ролей) загружаются в ProtectedRoutes и применяются к Routes и MainLayout меню. Владелец настраивает через Settings → вкладку "Доступ" (toggle chips per section per role).

- **Users → Owner** (фронтенд): Разделы Пользователи и Регистрация ограничены до `owner` (App.tsx + MainLayout.tsx).

- **SettingsPage**: 9 вкладок (+ Антиспам + Доступ). Антиспам: toggle'ы, ползунок порога, список помеченных сообщений с кнопками "Удалить" / "Не спам". Доступ: визуальный редактор per-role chips для каждого раздела.

- **Тесты**: 36 новых — 26 для spam_detector (normal/spam/threshold/edge cases), 10 для rate_limiter (TokenBucket, RateLimiter, threading, cleanup, singleton). Общий счёт: 387 passed.

### Rationale
- Token bucket — стандарт для Telegram Bot API rate limiting (30 msg/sec global + 1 msg/sec per chat group).
- Score-based spam вместо binary rules — меньше false positives на чатах с смешанным контентом (заказы + общение + конкурсы).
- ThreadPoolExecutor с оптимистичным locking — простой путь к параллельности без перехода на asyncio.
- Section visibility через bot_settings — не требует миграций и использует существующую инфраструктуру настроек.

## 2026-06-18: UI/UX overhaul — Excel rewrite, dark-mode fixes, Settings tabs, analytics ACL

**Context**: Комплексный аудит UI/UX выявил ряд проблем: Excel-экспорт формировался без профессионального оформления (нет заголовков, стилей, авто-ширины); в тёмной теме белый текст на белом фоне (raw text, Excel preview, шаблоны, дашборд); Settings — хаотичный бесконечный скролл; аналитика была видна всем админам (избыточно); layout overflow в CatalogsPage при resize.

### Решения
- **`excel_export.py`** (полная перезапись): 4 листа с русскими заголовками ("Заказы", "Позиции", "Сводка товаров", "По точкам"); title row с названием каталога и датой; синий header (PatternFill #4472C4 + белый Font); zebra-striping; auto-width; freeze panes; auto-filter; emoji статусы; форматированные даты (dd.MM.yyyy HH:mm); 3 режима (full/pickup/assembly); sanity-пороги сохранены.
- **Dark-mode fixes** (4 файла): OrdersPage raw text, ExcelPreviewPage header, TemplatesPage column cards, DashboardPage QuickStat — все hardcoded цвета заменены на theme-aware palette tokens.
- **SettingsPage** (реорганизация): монолитный скролл → 7 вкладок (Общие, Сообщения, Точки выдачи, Уведомления, Тема, Функционал, Система) с иконками.
- **Analytics ACL**: Route и sidebar ограничены `isOwner` / `roles: ["owner"]` — админы больше не видят аналитику.
- **Layout fix**: CatalogsPage `calc(100vw - 320px)` → `100%`; MainLayout active menu `primary.light` → `primary.main` (лучший контраст).
- **Тесты**: 20 новых тестов в `test_excel_export.py` (unit + integration). Общий счёт: 351 passed.

### Rationale
- Профессиональный Excel — ключевое требование для операционной работы (закупка/сборка/раздача).
- Dark-mode — основной режим работы для многих админов, белый текст на белом фоне = нечитаемо.
- 7 вкладок Settings масштабируемы: добавление новых настроек не превращает страницу в хаос.
- Аналитика содержит бизнес-метрики — ограничение до owner соответствует принципу минимальных привилегий.

## 2026-06-17: Massive parser upgrade — smart preprocessing, typo tolerance, pack computation

**Context**: Анализ реальных Telegram-сообщений (result.json, result-2.json — 30+ разнообразных заказов) выявил множество edge cases: слитный телефон+место (`8086Ванеева`), имя+телефон (`Ольга8493`), emoji как bullets, `=` как разделитель, опечатки в названиях рыб, формат фасовок `2×250г`, словесные числа (три-десять), отзывы с весом, отмены заказов. Парсер был значительно прокачан для обработки всех этих случаев.

### Решения
- **`text_parser.py`**: общий `_UNIT_PAT` фрагмент для всех regex (добавлены коробка, баночка, штука, палка, стакан, ведро, пластина, кусок); emoji bullet stripping (`_EMOJI_BULLET_RE`); `=` как разделитель (`_EQ_SEP_RE`); вычисление итогов pack qty (`2×250г → 500 г`, авто г→кг при ≥1000г); словесные числа три-десять; расширение `_normalize_unit()` на ~60 маппингов; strip скобочных комментариев из цен.
- **`order_domain.py`**: `_smart_preprocess_text()` — разделение слитых полей (имя+телефон, телефон+место, имя+место через known_places); `_CANCEL_PATTERNS` (7 паттернов: отказ, отменить, убрать, замена и т.д.); `_PHONE_GLUED_RE` + `_PHONE_5D_RE` для 3-проходного извлечения телефона; расширены `_PRODUCT_KEYWORDS` (~30 новых, включая опечатки) и `_COMMON_NAMES` (~25 новых); `looks_like_order()` отвергает отмены и запросы цен.
- **`catalog_repo.py`**: `_TYPO_MAP` (10 частых опечаток: комбала→камбала, троска→треска и т.д.) + `_fix_common_typos()` перед fuzzy matching.
- **`excel_export.py`**: `_MAX_SANE_QTY` словарь с per-unit порогами — строки с абсурдными qty исключаются из итоговых сумм.
- **`AnalyticsPage.tsx`**: `isMockData` state + `<Alert>` баннер при недоступности API.
- **Тесты**: новый `test_parser_upgrade.py` (58 тестов, 11 классов), обновлены pack-format тесты в `test_parser_comprehensive.py` и `test_order_processing_mvp.py`.

### Rationale
- Regex-based парсер — единственный слой распознавания; каждый неперехваченный edge case = потерянный заказ.
- Реальные сообщения показали, что пользователи активно пишут слитно, с опечатками, emoji и нестандартным форматированием.
- Sanity-пороги в Excel защищают от исторических parser-ошибок в итоговых суммах.
- Баннер mock-данных — UX requirement, чтобы админ не принимал фейковые данные за реальные.

## 2026-06-16: Codebase audit — critical bug fixes, /cancel_order, excel optimization

**Context**: Полный аудит кодовой базы выявил ряд критических и некритических проблем: модуль `handlers/` использовал неверное значение статуса каталога (`"active"` вместо `"open"`), что делало все handler-команды нерабочими; конструктор `CatalogItem` в handler-ах использовал несуществующее поле `status`; в `order_presenter.py` сломанный Unicode emoji; в `admin_commands.py` ошибка отступа; мёртвый код в `worker.py`; отсутствовала команда `/cancel_order`.

### Решения
- **10 исправлений Catalog.status**: `"active"` → `"open"` в `handlers/admin_actions.py` (3) и `handlers/admin_commands.py` (7), включая создание каталогов.
- **CatalogItem constructor fix**: удалено несуществующее поле `status`, добавлены `sku` (обязательное) и `is_active` (default=1).
- **Emoji fix**: U+FFFD → 👤 в `order_presenter.py`.
- **Indentation fix**: `aliases=aliases,` в `admin_commands.py`.
- **Dead code removal**: `_maybe_handle_admin_command()` из `worker.py`.
- **`/cancel_order` command**: полная реализация в `worker.py` — находит последний незавершённый заказ пользователя, устанавливает `status="canceled"`, отправляет подтверждение. Добавлен в `PUBLIC_COMMANDS`, в help-текст, и кнопка «🚫 Отменить» в оба варианта клавиатур (`user_inline_keyboard`, `user_inline_keyboard_v2`).
- **Excel pickup optimization**: в `excel_export.py` при `mode="pickup"` пропускается загрузка `order_lines` и `catalog_items` — экономит N+1 SQL-запросов.

### Rationale
- Handlers-модуль был полностью неработоспособен из-за бага со статусом — критическое исправление для будущей интеграции.
- `/cancel_order` — одна из самых запрашиваемых пользовательских функций из roadmap.
- Pickup export обрабатывает только агрегированную информацию по заказам, order_lines не нужны.

## 2026-03-11: Unified API surface documentation + backend capabilities endpoint

**Context**: Документация по API была распределена по нескольким файлам и частично смешивала актуальные и устаревшие контуры (особенно для интеграций). Потребовалась единая «точка входа», где видно, какие API реально реализованы у каждого микросервиса и где их проверять в runtime.

### Решения
- Добавлен [[14-api-reference]] как единый исчерпывающий реестр API по сервисам `backend` и `admin_service`, включая ссылки на OpenAPI/Swagger и карту endpoint-ов.
- В `backend` добавлен discovery endpoint `GET /capabilities` (`backend/app/main.py`) для машинно-читаемого перечисления публичных ручек и статуса доступности docs (`/docs`, `/openapi.json`).
- Добавлен [[Отличный улов/docs/README]] как индекс документации.
- Обновлены точки навигации в `README.md` и `admin_service/README.md` на новый API reference.

### Rationale
- Сокращает время онбординга и аудита API.
- Упрощает проверку «какие API уже есть, где смотреть и чем пользоваться» без чтения исходников.
- Делает integration/discovery более предсказуемыми для внешних сервисов и для команды эксплуатации.

## 2026-03-11: Allow localhost preview ports in admin_service CORS

**Context**: Во время локальной отладки `admin-web` регулярно поднимается не только на `4173`, но и на временных Vite preview/dev портах вроде `4174`. Жёсткий список `ADMIN_API_CORS_ORIGINS` ломал login preflight к `POST /auth/login`, и owner физически не мог войти в админку с нового preview-адреса.

### Решения
- `admin_service/app/main.py`: для CORS добавлен `allow_origin_regex`, который разрешает локальные origins вида `http://localhost:<port>` и `http://127.0.0.1:<port>`.
- `.env.example` и `admin_service/README.md` обновлены так, чтобы явные локальные origins по умолчанию включали `4173` и `4174`.
- `admin-web/src/api/client.ts` и `admin-web/vite.config.ts`: локальный frontend по умолчанию ходит в backend через same-origin `/api`, а dev/preview proxy Vite пробрасывает эти запросы на `http://localhost:8010`, чтобы login не зависел от browser CORS при временно stale `admin_service`.

### Rationale
- Preview-порты в локальной разработке не должны ронять авторизацию и создавать ложное впечатление, что сломан пароль или backend.
- Regex оставляет правило узким и дев-ориентированным: только localhost/127.0.0.1, а не произвольные внешние origins.

## 2026-03-11: Add built-in fallback Excel template #3 to admin-web

**Context**: В реальной локальной среде `admin_service` мог оставаться на старом процессе и отдавать `404/500` для `GET/POST /templates/excel` и `GET /orders/export-template`. Из-за этого owner не мог даже скачать эталонный шаблон, чтобы продолжить работу вручную.

### Решения
- В `admin-web/public/templates/template-3.xlsx` добавлен встроенный fallback-шаблон.
- Файл скопирован 1-в-1 из `samples/excel/template.xlsx`.
- В `admin-web/src/api/client.ts` добавлен client-side fallback для `exportOrdersTemplateXlsx()`: если backend возвращает `404/500` или временно недоступен, UI собирает template-export локально на основе встроенного `template-3.xlsx` и данных `GET /orders` + `GET /orders/{id}`.
- Отдельные UI-действия для ручного скачивания встроенного шаблона убраны, чтобы не смешивать service-template management с реальным экспортом заказов.
- Дополнительно в owner-страницах убраны download/view действия для server template: шаблон теперь позиционируется только как внутренняя основа формирования выгрузки, а CTA в `OrdersPage` явно обещают скачать готовые заказы по шаблону.
- `orders.py` и frontend fallback больше не дописывают данные в хвост sample-файла: body шаблона очищается, а реальные заказы раскладываются по товарным колонкам шаблона, чтобы результатом был новый Excel по заказам, а не визуально тот же исходный образец.

### Rationale
- Excel-шаблон — это операционный артефакт, который должен быть доступен даже при временно сломанном backend flow.
- Локальный static asset в `admin-web` позволяет owner-у продолжить работу и сверку формата, не дожидаясь рестарта сервиса.

## 2026-03-11: Admin UI regression fixes for template path, catalog filter limit, and raw order payload visibility

**Context**: После расширения админки всплыли три неприятных regressions в реальном UI. Excel-шаблон искался не в корневой `samples/excel`, а в ошибочном относительном пути внутри `admin_service`, из-за чего download/upload работали нестабильно. Страница заказов запрашивала `GET /catalogs?limit=500`, но backend разрешал только `200`, что давало `422` при загрузке фильтров. И, наконец, `GET /orders/{id}` не возвращал `raw_text` и `error_text`, поэтому raw-сообщение не отображалось в карточке и диалоге редактирования заказа.

### Решения
- `templates.py`: путь к шаблону переведён на вычисление от корня репозитория через `Path(__file__).resolve().parents[4]`, чтобы локальная разработка и файловые операции шли в реальный `samples/excel/template.xlsx`.
- `orders.py`: `GET /orders/export-template` больше не ищет шаблон своим отдельным путём; endpoint переиспользует общий resolver из `templates.py`, чтобы download/upload/export-template смотрели в один и тот же файл.
- `catalogs.py`: лимит для `GET /catalogs` расширен до `500`, чтобы соответствовать фактическим запросам UI для фильтров и массового выбора каталогов.
- `schemas.py` + `orders.py`: `OrderDetailResponse` снова включает `tg_chat_id`, `order_no`, `raw_text`, `error_text`, `updated_at`; detail endpoint заполняет эти поля из таблицы заказов.
- `schemas.py` + `admin_sync.py`: `UserResponse.email` перестал валидироваться через `EmailStr`, а автосоздаваемые Telegram-админы получают домен `@otlichniy-ulov.app`, чтобы живые служебные аккаунты не роняли `/users` response validation.
- `users.py`: добавлен совместимый endpoint `POST /users/{user_id}/invite-link`, который генерирует one-time login URL для уже созданного пользователя и закрывает ожидания `RegisterPage`.

### Rationale
- UI не должен ломаться на «служебных» limit-значениях, если сам интерфейс ожидает до 500 сущностей.
- Raw payload — это не optional cosmetic detail, а рабочий инструмент диагностики и ручной коррекции заказов.
- Для шаблонов особенно важен предсказуемый путь: иначе владелец видит 404/500 там, где должен быть обычный upload/download flow.

## 2026-03-12: Major UI/UX upgrade — responsive design, recharts analytics, feature toggles, raw order editing

**Context**: Админ-панель страдала от ряда UX-проблем: раскрывающиеся элементы в таблице каталогов «разъезжались» на узких экранах (контент пропадал, оставался только фон), аналитика была примитивной (текстовые счётчики без графиков), экспорт не поддерживал шаблоны, редактирование заказов ограничивалось только сменой статуса, и не было возможности управлять фичами из UI.

### Решения

**Frontend:**
- `MainLayout.tsx`: корневой контейнер получил `overflow: hidden`, `maxWidth: 100%`; Drawer — `keepMounted: true` + `overflowX: hidden`; main content — flex column с `width: 0` (force shrink).
- `CatalogsPage.tsx`: все вложенные таблицы переведены на responsive minWidth `{ xs: 600, md: 980 }`, Collapse-контейнер — `overflow: hidden`, `maxWidth: 100%`.
- `AnalyticsPage.tsx`: полностью переписана с использованием `recharts` (AreaChart для заказов по дням, PieChart для статусов, горизонтальный/вертикальный BarChart для товаров и точек выдачи), добавлены сворачиваемые секции `AnalyticsSection` и фильтр по чатам.
- `OrdersPage.tsx`: добавлены фильтрация по чатам, raw-редактирование заказов (Dialog с полями customer_name, phone_last4, pickup_place, status, raw_text, error_text), экспорт по шаблону template.xlsx.
- `SettingsPage.tsx`: новая карточка «Управление функционалом» с 5 Switch-тогглами для feature flags (analytics_charts, excel_preview, raw_order_edit, chat_grouping, registration).
- Новые страницы: `ExcelPreviewPage.tsx` (drag-n-drop загрузка, xlsx парсинг на клиенте, multi-sheet tabs), `RegisterPage.tsx` (3-шаговый Stepper регистрации с валидацией пароля и выбором ролей).
- `client.ts`: добавлены методы `exportOrdersTemplateXlsx()` и `updateOrderRaw()`.
- `App.tsx`: новые routes `/register` и `/excel-preview`.

**Backend:**
- `PATCH /orders/{id}`: расширен до allowlisted полей (customer_name, phone_last4, pickup_place, status, raw_text, error_text) вместо только status.
- `GET /orders/export-template`: новый endpoint — загружает `samples/excel/template.xlsx` как базовый шаблон, заполняет данными заказов.
- Все analytics endpoints (`/dashboard`, `/orders-chart`, `/top-items`, `/status-breakdown`, `/pickup-stats`) получили опциональный `chat_id` query param для фильтрации по чатам.
- `ALLOWED_SETTINGS_KEYS` расширен 5 feature-toggle ключами.

### Rationale
- Responsive design критичен: админы работают и с телефона, и с десктопа.
- Collapsible bug был в том, что вложенные таблицы с `minWidth: 980px` внутри Collapse ломали layout — всё содержимое уходило за viewport.
- recharts даёт визуальную аналитику без серверных зависимостей (данные уже были, не хватало визуализации).
- Feature toggles позволяют owner-у контролировать видимость фич без деплоя.
- Raw order editing нужен для ручной коррекции ошибок парсера без прямого доступа к БД.

## 2026-03-11: Fix /orders/export route collision and harden parser on live edge cases

**Context**: В админке Excel-экспорт падал с `422`, потому что `GET /orders/export` и `GET /orders/export-history` перехватывались динамическим маршрутом `GET /orders/{order_id}`. Параллельно в живых данных остались реальные edge-cases: stop-анонсы ошибочно создавали заказы, price-list сообщения с ценами без qty проходили эвристику заказа, `1бан` не распознавался как количество, а `Креветки 250+ 1уп` рвались по `+`.

### Решения
- `admin_service` маршруты деталей/patch заказа переведены на path converter `/{order_id:int}`, чтобы статические `export` и `export-history` больше не коллидировали.
- В parser rules добавлены дополнительные guard'ы против stop/announcement сообщений и price-list постов без количества.
- В `text_parser.py` добавлена поддержка `бан/бан.` и диапазонов вида `0,5-1 шт`.
- Multi-item split больше не режет size markers вида `250+` внутри названия позиции.
- Исторически сломанные live-заказы были перепарсены по новой логике: несколько `needs_admin/partial` заказов стали `active`, а 2 ложных срабатывания переведены в `canceled`.

### Rationale
- Экспорт должен быть доступен из UI без скрытой зависимости от порядка объявления маршрутов.
- Стоп-посты и прайс-листы не должны загрязнять очередь заказов.
- Реальные живые форматы пользователей важнее красивой теории: parser hardening идёт от production data.

## 2026-03-11: Distinguish customer price-list orders from admin price posts

**Context**: После первого жёсткого фикса price-only сообщений живой заказ Юлии (`Иваново 3 / Юлия 4818 / ...`) оказался отменён вместе с действительно лишним админским стоп-постом. У клиентского сообщения были полные метаданные и короткий список товаров, но без явного `qty`; вдобавок matcher ошибочно уводил `Филе нерки мало солёное` в `Филе горбуши`.

### Решения
- `parse_line()` теперь очищает `title_raw` от цены даже если qty не найдено.
- `match_catalog_item_from_items()` нормализует строки (`ё/е`, пробелы, punctuation) и добавляет phrase-similarity fallback, чтобы `филе нерки мало солёное` корректно мэтчилось в `Филе нерки малосольное`, а не в первый попавшийся `Филе ...`.
- Для коротких клиентских сообщений с именем + телефоном + точкой выдачи и распознанными товарами price-only строки больше не отбрасываются автоматически как не-заказ.
- Для каталожных позиций с `unit_hint` из `шт/уп/банка` отсутствие qty теперь интерпретируется как заказ `1` единицы.

### Rationale
- В живых чатах клиенты иногда просто копируют товар и цену из прайса, подразумевая `1` штуку/банку/упаковку.
- Это поведение безопасно только для дискретных единиц; для `кг/л` автоподстановка не делается.
- Стоп-посты админа по-прежнему отсекаются отдельными guard'ами и не возвращаются в очередь заказов.

## 2026-03-09: Orders export dialog with multi-criteria filters + layout hardening

**Context**: На странице заказов обычные кнопки экспорта были слишком ограничены: нельзя было удобно выгружать по датам, раздачам и точкам выдачи из одного окна. Параллельно в `Каталогах` на некоторых размерах окна UX визуально схлопывался при работе с блоком режимов бота.

### Решения
- В `OrdersPage` быстрые кнопки заменены на сценарий через единый диалог гибкого экспорта.
- Для `GET /orders/export` и `GET /orders/export-history` добавлены фильтры: `chat_id`, `catalog_id`, `pickup_place`, `status`, `date_from`, `date_to`, `search`.
- В `list_orders` добавлены поддержка `offset` и `search`, чтобы фронтенд-фильтры работали консистентно с экспортом.
- В `settings` API точки выдачи теперь возвращают публичный Telegram `chat_id`, а не внутренний PK, чтобы фильтрация по чатам в UI не ломалась.
- В `CatalogsPage` блок управления видимостью бота переведён в более устойчивый card/grid layout, а в `MainLayout` убрано лишнее схлопывание контента по ширине.

## 2026-03-14: Minimalist mobile admin mode + first-render stabilization + numeric pickup alias header fix

**Context**: В Telegram browser админка всё ещё выглядела слишком тесной: safe-area и нижняя навигация местами ощущались криво, а на первом открытии с телефона до перезагрузки мог всплывать старый desktop-ish UI. Параллельно parser ошибочно трактовал сообщения вида `Наталья Иваново 2. Горбуша...` — цифра из alias точки выдачи попадала в количество первой позиции.

### Решения
- В `admin-web` добавлен новый style preset `Минималистичный`, который даёт более свободную композицию, мягкие карточки и аккуратный mobile-friendly ритм без потери возможности мгновенно вернуться к прежнему виду.
- В `index.html`, theme overrides и `MainLayout` усилена mobile-safe подача: `viewport-fit=cover`, safe-area padding, более устойчивые AppBar / BottomNavigation и менее дёрганый вид в Telegram browser.
- В `admin-web/src/main.tsx` убран legacy outer `ThemeProvider/CssBaseline`, а ключевые `useMediaQuery(...)` переведены в `noSsr` client-mode, чтобы первый mobile render сразу использовал актуальный layout вместо промежуточного старого вида.
- `OrdersPage` оставлен чисто order-management разделом: статус выдачи можно видеть для контекста, но менять его теперь нужно только через `DeliveryPage`.
- Mobile cards в `OrdersPage` и full-screen detail dialog дополнительно уплотнены под Telegram browser: вместо россыпи мелких chip-ов используются компактные info-blocks и более предсказуемая сетка действий.
- В `order_domain.py` inline-header strip расширен на сценарии без телефона, где точка выдачи содержит цифру (`Иваново 2`), а имя клиента извлекается из префикса до товарной части.
- Добавлен regression-test на кейс `Наталья Иваново 2. Горбуша свежемороженая-1шт., Срезки филе трески-1шт.` — теперь парсер сохраняет 2 корректные товарные строки и не превращает `2` из alias точки в количество.

## 2026-03-09: Live Ivanovo Telegram orders — inline header parsing + specific alias priority

**Context**: В живом чате Иваново один заказ уходил в `needs_admin`, а сообщение Вахида вообще не создавалось как полноценный заказ. Причины оказались двойными: в чате не были досеяны ивановские `pickup_places`/товары, а парсер и matcher слишком рано хватали общие совпадения (`горбуша`, `корюшка`) и плохо обрабатывали однострочный формат `Имя 3430 Бигам Товар ...`.

### Решения
- В `order_domain.py` добавлен strip inline-header логики для фрагментов вида `Имя 3430 Точка Товар qty`, чтобы товарная часть начиналась с первого product keyword.
- В `order_processing.py` pickup, найденный по alias (`Иваново 2`), теперь канонизируется до `PickupPlace.title` (`Миля`) ещё на этапе DB-aware parsing, чтобы заказы не зависали на alias-значениях.
- В `text_parser.py` line splitting теперь защищает десятичные количества с запятой (`0,5-1 шт.`), чтобы они не распадались на ложные пустые/unknown fragments.
- В `_clean_fragment()` добавлена очистка голого `Дозаказ.`/`До заказ.` после удаления префикса, чтобы такой маркер не создавал пустую `unknown_item` строку.
- В `catalog_repo.py` alias/title matching переведён с "первое попавшееся совпадение" на более специфичное scoring-сопоставление.
  - `горбуша в с/с` теперь приоритетно матчится в консерву, а не в обычную горбушу.
  - `корюшка с/м` теперь приоритетно матчится в свежемороженую корюшку, а не во вяленую.
- В live БД для чата `-1002453744799` досеяны ивановские pickup places и недостающие catalog items через `seed_catalog_ivanovo.py`.
- Сообщения `5641`, `5652`, `5653` перепроиграны через актуальную логику: оба заказа Натальи и заказ Вахида стали `active`.
- После дополнительных парсерных правок перепроиграны и последние сообщения `5654` и `5655`: оба заказа перешли в `active` без ручного вмешательства.

## 2026-03-09: Move admin-web from 5173 to 4173

**Context**: На пользовательской машине `localhost:5173` оказался загрязнён старыми browser site-data/service worker, из-за чего вместо админки открывался пустой экран. Чистый профиль Chrome рендерил приложение нормально, значит проблема была в origin-specific browser state, а не в самом bundle.

### Решения
- `admin-web` dev/prod переведён на порт `4173`.
- Дефолты `ADMIN_WEB_URL` и `ADMIN_API_CORS_ORIGINS` в `admin_service` тоже переведены на `http://localhost:4173`.
- Docker compose и документация синхронизированы с новым портом.

## 2026-03-09: Portable local Node.js for admin-web

**Context**: На Linux-машине пользователя не было системных `node`/`npm`, из-за чего `admin-web` не запускался вообще (`npm run dev` → exit code 127). Docker и sudo тоже отсутствовали, значит нужен self-contained способ запуска фронта прямо из репозитория.

### Решения
- Добавлен `scripts/setup_local_node.sh`, который скачивает portable Node.js 20.x в `.tools/`.
- Добавлен `scripts/admin_web.sh` для `install/build/dev` без системного Node.
- В `Makefile` добавлены команды `front-install`, `front-build`, `front-dev`.
- Обёртка умеет удалять не-writable `admin-web/node_modules`, если они были созданы контейнером/чужим uid, и ставить зависимости заново.

## 2026-03-09: Chat analysis tool + Ivanovo onboarding + UX fixes

**Context**: Нужен инструмент для быстрого анализа новых чатов и автоматического извлечения каталога/точек выдачи из Telegram-экспорта. Также найдены UX баги в OrdersPage и CatalogsPage.

### Решения
- Создан `scripts/analyze_chat_export.py` — универсальный анализатор Telegram-экспортов:
  - Извлекает продукты, цены, единицы из admin-постов (оба формата: один продукт / список)
  - Распознаёт точки выдачи из сообщений с адресами
  - Считает упоминания продуктов в заказах пользователей
  - Поддерживает: `--json`, `--gen-seed`, `--seed-db`, `--scan-dir`
- Создан `scripts/seed_catalog_ivanovo.py` — 42 товара + 3 Ивановские точки выдачи (Бигам, Миля, Межевая)
- **Fix OrdersPage**: `handleExportHistoryXlsx` был вложен внутрь `handleExportXlsx` → кнопка «📜 История» не работала. Извлечён как отдельная функция. Исправлен orphan import `HistoryIcon`.
- **Fix CatalogsPage UX**: `handleToggleBotVisibility` вызывал `loadChats()` → полный re-render → аккордеоны схлопывались, Select флickерил. Заменён на optimistic local state update с rollback при ошибке.

## 2026-03-09: Auto chat onboarding + catalog clone + pickup autodiscovery

**Context**: Нужно убрать ручные операции при подключении новых Telegram-чатов и ускорить запуск раздачи в новых регионах.

### Решения
- Worker теперь синхронизирует `chats` при `my_chat_member/chat_member` событиях: новый чат автоматически появляется в админке.
- Добавлен endpoint `POST /catalogs/{catalog_id}/clone` для импорта существующего каталога в другой чат.
- В UI (`Каталоги`) добавлен сценарий импорта каталога в выбранный чат, с опцией закрыть текущий open-каталог в целевом чате.
- Worker автоматически добавляет новые точки выдачи в `pickup_places` по распознанным сообщениям заказов (per-chat), чтобы шаблоны и подсказки показывали актуальные точки.

## 2026-03-09: Added order history export (audit XLSX)

**Context**: Нужен отдельный экспорт для истории и сверки с ручной «конечной таблицей», не только операционный XLSX для сборки/выдачи.

### Решения
- Добавлен endpoint `GET /orders/export-history`.
- Добавлен UI action `📜 История (Excel)` на странице заказов.
- Формат включает 3 листа:
  - `orders_history` (полные заказы),
  - `lines_history` (полные строки),
  - `audit_summary` (агрегаты по статусам).
- Экспорт используется для аудита и расхождений, а не вместо стандартного операционного экспорта.

## 2026-03-09: External Integration API via service key

**Context**: Нужна интеграция админки не только с Telegram, но и с внешними сервисами/будущим приложением без UI-авторизации по JWT.

### Решения
- Добавлен M2M-контур `/integrations/*` в `admin_service`.
- Аутентификация по `X-Service-Key` (reuse `ADMIN_ONE_TIME_KEY`) для машинных клиентов.
- Добавлены read-oriented endpoints для синхронизации:
  - capabilities, chats, open catalogs, pickup places, orders.
- Документация вынесена в [[13-integration-api]].

## 2026-03-09: User safety guards in admin management

**Context**: В UI и API нужны предохранители от самоудаления/самодеактивации и удаления последнего owner.

### Решения
- Запрещены self-deactivate и self-delete.
- Запрещено удалять/деактивировать последнего активного owner.
- Запрещено удалять owner-аккаунты через обычный flow.
- Для owner запрещено снимать у себя роль owner через редактирование ролей.

## 2026-03-09: Owner debug mode instead of new developer role

**Context**: Нужен был режим разработчика для owner, но без размножения ролей и без постоянного засорения интерфейса служебными деталями.

### Решения
- Новая роль не добавляется.
- Используется owner-only toggle `owner_debug_mode` в `bot_settings`.
- При `owner_debug_mode=false` owner видит стандартный UI без системных логов и debug-раздела.
- При `owner_debug_mode=true` owner получает доступ к странице `Debug`, project state и расширенной диагностике.
- Project state выведен в UI: schema/migrations, worker status, webhook status, queue size, failed updates, readiness по чатам.
- Для эксплуатации добавлена единая команда `make sync` (build + alembic upgrade heads + force-recreate stack).

## 2026-03-09: Diagnose-first fix for "bot does not react in chat"

**Context**: Внешне бот "не реагировал", хотя API был доступен по `/health`. Фактическая причина оказалась двухслойной: БД отставала по Alembic-миграциям, а для нового чата не было открытого каталога и pickup places.

### Решения
- Для диагностики использовали не только healthcheck, но и `Telegram getWebhookInfo` + состояние очереди `tg_updates`.
- Принято правило: если webhook принимает update, но реакции нет — сначала проверяем `alembic_version`, затем `tg_updates.status`, затем наличие open catalog для конкретного chat.
- Миграция `b2c3d4e5f6a7` сделана idempotent/retry-safe для MySQL: проверки существования таблицы и индексов выполняются явно, без зависимости от `IF NOT EXISTS` в DDL.
- После восстановления схемы обязательно seed/open catalog для нового chat, иначе worker корректно обработает update, но не создаст order.

### Практический итог
- Схема БД обновлена до `b2c3d4e5f6a7`
- Для chat `-1001234567890` создан open catalog `20260309` и 8 pickup places
- Проверка E2E подтверждена: webhook `200` → `tg_updates.status=done` → order `active` с распознанной строкой товара

## 2026-06-15: Real catalog seed, parser hardening, comprehensive tests, analytics & stealth docs

**Context**: Переход от демо-каталога (5 позиций) к полному реальному (45 позиций). Парсер должен корректно обрабатывать все 6+ форматов заказов из реальных Telegram-сообщений. Analytics нуждается в реальных данных вместо mock.

### 1. Реальный каталог (seed_catalog_real.py)
- 45 товаров из 9 категорий (рыба, филе, копчёная, консервы, икра, морепродукты, мёд, добавки, маринады)
- Каждый товар: SKU, title, unit_hint, price_text, pack_hint, rich aliases (все варианты написания)
- 8 пунктов выдачи с aliases (Автозавод, Ванеева, Сормово, Щербинки, Канавино, Печёры, Лысогорская, Бор)
- CLI: `--chat-id`, `--chat-title`, `--code`, `--pickup-only` для гибкого использования
- Авто-определение существующего каталога, дозагрузка отсутствующих позиций

### 2. Улучшения парсера (text_parser.py)
- QTY_RE: добавлены единицы — `брусок`, `брикет`, `пласт`, `тушка`, `рыбка`, `вес`
- _normalize_unit: все нестандартные единицы → `шт` (брусков→шт, пластов→шт, тушку→шт, рыбку→шт)
- parse_line: улучшена очистка title — strip trailing dashes/dots/spaces, collapse внутренних множественных пробелов

### 3. Улучшения order_domain.py
- _PRODUCT_KEYWORDS: +20 ключевых слов (василёк, донник, печень, лойны, ассорти, малосольн, ванамей, краб, угольная, вялен, морож, св/м, собственном, собств., с/с)
- _smart_preprocess_text: фикс dot-split regex `[А-ЯЁа-яё]` (ранее пропускал lowercase после точки)
- _looks_like_recipe_or_post: сильные маркеры рецептов (рецепт/ингредиент) всегда отклоняют, даже при наличии qty паттернов

### 4. Комплексные тесты (test_real_messages.py)
- 63 новых теста из реальных Telegram-сообщений
- 10 классов: Standard, DotSeparated, CommaSeparated, PhoneAtEnd, Dozakaz, AltUnits, NameExtraction, PickupExtraction, LooksLikeOrder, NonOrders, FullPipeline, CatalogKeywords, EdgeCases
- Full pipeline тесты с реальным 45-item каталогом в SQLite
- Итого: 242 теста, все зелёные

### 5. Аналитика: real data + parser accuracy
- Analytics: status breakdown загружается через API (ранее — mock percentages)
- Analytics: pickup place distribution chart с реальными данными
- Новый API endpoint: `/analytics/parser-accuracy` — match_rate, auto_processing_rate, top_unknown_items
- Frontend: секция «🎯 Точность парсера» с цветовой индикацией (green/yellow/red) и списком нераспознанных товаров

### 6. Документация: Stealth Mode Setup
- [[STEALTH-MODE-SETUP]]: пошаговая инструкция подключения бота в скрытом режиме
- 8 шагов: BotFather → добавление в чат → .env → запуск → chat_id → hidden mode → каталог → проверка
- Чеклист перед запуском, troubleshooting таблица, архитектурная схема скрытого режима

---

## 2026-06-14: bot_visible 3-mode, fuzzy pickup, regional_admin, security hardening

**Context**: Бот в скрытом режиме всё равно ставил 👍 — нужен 3-уровневый режим видимости. Пользователи ошибаются в названиях точек раздачи. Нужна промежуточная роль между owner и admin для управления региональными админами.

### 1. Трёхрежимная видимость бота (bot_visible)
- Миграция `b2c3d4e5f6a7`: `bot_visible` INT → VARCHAR(20)
- Три режима: `hidden` (полностью невидим), `reactions_only` (👍 + ошибки), `full` (все ответы)
- `worker.py._send_user_reply()`: новый параметр `is_error` — в mode `reactions_only` отправляются только error-ответы
- Реакция 👍 НЕ ставится в режиме `hidden` (полная невидимость)
- UI: выпадающий Select вместо Switch с описанием каждого режима
- Обратная совместимость: старые значения 0→hidden, 1→full

### 2. Нечёткий поиск точек раздачи
- `pickup_repo.py`: добавлен `_levenshtein()` для расстояния Левенштейна
- `find_pickup_place()`: Pass 1 — точное substring совпадение; Pass 2 — fuzzy по словам/биграммам/триграммам
- Порог: `max(1, len(name) // 4)` — 25% допуск на опечатки
- Минимум 3 символа в имени для fuzzy (исключает ложные срабатывания)
- 18 новых тестов (Levenshtein, кириллица, per-chat изоляция, fuzzy matching)

### 3. Роль regional_admin
- Миграция: seed `admin_roles` с code=`regional_admin`, level=50
- Новая таблица `admin_user_regions` (admin_user_id, region_name)
- `users.py`: regional_admin может создавать/редактировать/удалять пользователей
- Ограничение: не может назначать роли owner/regional_admin
- Ограничение: не может модифицировать пользователей с ролью owner/regional_admin
- UI: роль отображается как `secondary` color, видна в sidebar

### 4. Исправление бага добавления точек раздачи
- Баг: фронтенд не отправлял `chat_id` при создании pickup place
- Фикс API: авто-определение chat_id если в системе 1 чат
- Фикс API: резолвинг tg_chat_id → internal PK для больших/отрицательных ID
- Фикс UI: добавлен Chat selector в диалог создания, поле aliases

### 5. Per-chat управление в UI
- SettingsPage: фильтр pickup places по чату
- SettingsPage: Chat selector при создании новой точки
- CatalogsPage: 3-mode selector для каждого чата
- MainLayout: regional_admin видит Analytics + Users, но не owner-only разделы

### 6. Безопасность
- Rate limiting: 120 req/min per IP (in-memory, middleware)
- Security headers: X-Content-Type-Options, X-Frame-Options, X-XSS-Protection, Referrer-Policy, Cache-Control
- Input validation: bot_visible проверяется против enum перед записью
- SQL: все запросы параметризованы (подтверждено аудитом)

### 7. High-load оптимизация
- Миграция: 4 новых составных индекса
- `ix_orders_tg_user_catalog_status` — быстрый поиск заказов по пользователю
- `ix_pickup_places_chat_active` — быстрый lookup активных точек
- `ix_user_profiles_tg_user_id` — профили пользователей
- `ix_message_snapshots_chat_msg` — дедупликация сообщений

---

## 2026-06-14: Adaptive parser, bot reactions, stealth mode, Excel export, admin-chat scoping

**Context**: Продукция, имена, локации и точки раздач постоянно меняются — тысячи наименований. Парсер не должен полагаться только на захардкоженные ключевые слова. Бот должен молча тестироваться в реальных чатах, а подтверждённые заказы — получать 👍.

### 1. Адаптивный парсер (catalog_keywords)
- `build_catalog_keywords()` в `order_domain.py` — извлекает ключевые слова из названий и aliases товаров каталога
- `looks_like_order()`, `_has_product_keyword()`, `parse_order_text()` принимают `catalog_keywords` — динамическое множество
- Захардкоженный `_PRODUCT_KEYWORDS` остаётся как базовый набор, catalog_keywords его дополняет
- `worker.py` строит `catalog_keywords` из активного каталога при каждом сообщении

### 2. Реакции бота (Telegram Bot API 7.0+)
- `set_message_reaction()` в `telegram_client.py` — ставит 👍 через `setMessageReaction`
- На подтверждённые заказы (status=active) бот ставит 👍 — работает даже в скрытом режиме
- Ошибки реакций игнорируются (non-critical, не все чаты поддерживают)

### 3. Скрытый режим бота (bot_visible)
- Миграция `a1b2c3d4e5f6`: добавлен `chats.bot_visible` (INT, default=1)
- `worker.py` → `_send_user_reply()` проверяет `bot_visible` — если 0, текстовые ответы не отправляются
- Реакции 👍 ставятся всегда (даже в скрытом режиме)
- API: `PATCH /chats/{tg_chat_id}` — toggle visibility (owner only)
- UI: переключатель в CatalogsPage с визуальным отображением статуса

### 4. Admin-Chat scoping
- Миграция `a1b2c3d4e5f6`: создана таблица `admin_user_chats` (admin_user_id, chat_id, UNIQUE)
- API: `GET/POST/DELETE /admin-user-chats` — назначение админов к чатам
- Owner видит все чаты, обычные админы — только назначенные

### 5. Excel экспорт из UI
- `GET /orders/export` — генерация XLSX с 4 листами (Заказы, Позиции, Сводка по товарам, По точкам)
- Цветовое кодирование статусов, авто-ширина колонок
- Кнопка «📥 Скачать Excel» на странице заказов (вместо CSV)

### 6. Owner-only UI секции
- Боковое меню: owner-only пункты отмечены 👑, жёлтой полосой слева, разделителем
- CatalogsPage: секция «Видимость бота» обрамлена warning-бордером с badge "👑 Owner"

**Тесты**: все 161 тест проходят без регрессий.

## 2026-06-13: Parser v2 — scoring-based order detection, trained on real data

**Context**: Парсер заказов пропускал/неправильно распознавал многие реальные форматы заказов. Обучение на данных из `samples/telegram/result.json` (NN-группа, 139K строк) и `result-2.json` (Arzamas, 30K строк) выявило 30+ форматов.

**Изменения в `text_parser.py`:**
- Добавлены алиасы единиц: `тушка/рыбка → шт`, `пачка → уп`
- Добавлено удаление цен (`_PRICE_RE`): `950р`, `485руб`, `370р/кг`, `по 450`, `950 за кг`
- Добавлена очистка словесных чисел в скобках: `3 (три)кг → 3кг`

**Изменения в `order_domain.py`:**
- `_PRODUCT_KEYWORDS`: набор ~80 ключевых слов товаров (рыба, морепродукты, мёд) для scoring
- `_NON_ORDER_PATTERNS`: компилированные regex для фильтрации не-заказов (вопросы, приветствия, жалобы, админ-посты)
- `looks_like_order()`: переписан на scoring-based подход (порог ≥ 3 баллов) вместо жёстких правил
- `extract_customer_name()`: двухпроходный алгоритм — (1) поиск по `_COMMON_NAMES`, (2) capitalized слово без мест/товаров
- `extract_phone_last4()`: поиск телефона во ВСЕХ строках (поддержка phone-at-end)
- `_smart_preprocess_text()`: переупорядочивание phone-at-end, разбиение нумерованных списков, multi-product single lines
- `_looks_like_recipe_or_post()`: расширена детекция админ-постов (❗️❗️❗️, «свободной продаже»)
- `_is_meta_fragment()`: учитывает product keywords — фрагмент с товаром+количеством не помечается как метаданные

**Тесты**: `tests/test_parser_comprehensive.py` — 133 теста (32 реальных заказа + edge cases). Все 161 тест проходят без регрессий.

**Rationale**: Scoring-подход гибче жёстких правил: каждый сигнал (телефон, место, товар, количество) вносит вклад в итоговый балл. Это позволяет распознавать неполные заказы (без телефона, но с товаром+местом) и отсеивать вопросы с упоминанием товаров.

## 2026-01-12
- В одном чате одновременно одна активная раздача.
- Каталог вводится админами вручную, прайс в чате не источник истины.
- Поля заказа обязательны: номер, имя, пункт выдачи, хотя бы 1 позиция.

---

## 2026-03-14: Settings IA — bot visibility + антиспам под «Бот», режим выдачи под «Точки»

**Context**: В интерфейсе накопились owner-only настройки «про бота», но они были разнесены по разным экранам (часть в `Каталоги`, часть в `Настройки → Безопасность`, часть в `Настройки → Бот`). Это мешало операционной работе и приводило к «я не нашёл где это менять».

### Decision
- Перенести управление `bot_visible` из `Каталоги` в `Настройки → Бот`.
- Убрать отдельную вкладку `Настройки → Безопасность`: антиспам остаётся функционально тем же, но находится в `Настройки → Бот`.
- Перенести `distribution_mode` (режим «сбор завершён / выдача») в `Настройки → Точки`.
- Перенести справочный блок «Роли команды» в `Настройки → Доступ` рядом с настройками видимости разделов.

### Consequences
- `CatalogsPage` больше не содержит секцию «Видимость бота».
- `SettingsPage` теперь имеет 8 вкладок вместо 9 (без отдельной «Безопасности»).
- Ссылки со старыми `?tab=` остаются работоспособными (индекс вкладки ограничивается диапазоном).
- Стопы ставит только админ, действуют с момента установки.
- Частичное принятие заказа при стопах по товарам.
- edited_message пересчитывает вклад сообщения без дублей.
- Excel: главное занести все данные, сортировка не критична.
- Листы "участники", "Траты", "призы", "conflicts" игнорируются в MVP.

## 2026-03-03
- В UI создания каталога убран ручной ввод `chat_id`: вместо этого добавлен выбор из доступных чатов (`GET /chats`).
- Генерация `code` для каталога перенесена в backend (если код не передан, он создаётся автоматически и уникализируется).
- Решение принято для снижения ошибок ручного ввода и ускорения работы админов.
- Для Docker-режима зафиксировано: обычный restart не гарантирует применение новых image/layer.
  `make compose-restart` переведён на `up -d --build --force-recreate`, добавлен `docker-compose.dev.yml`
  с hot-reload для `api`, `admin_service` и `admin-web`.


## 2026-01-12 (stage 1 done)
- webhook /telegram/webhook принимает update и пишет его в MySQL (tg_updates)
- включена проверка X-Telegram-Bot-Api-Secret-Token при TELEGRAM_WEBHOOK_SECRET


## 2026-01-12 (stage 1 done)
- webhook /telegram/webhook принимает update и пишет его в MySQL (tg_updates)
- включена проверка X-Telegram-Bot-Api-Secret-Token при TELEGRAM_WEBHOOK_SECRET


## 2026-01-12 (stage 2 done)
- добавлен worker: tg_updates (new -> processing -> done)
- добавлено сохранение message_snapshots из message/edited_message
- время хранится в UTC (timezone-aware)

## 2026-01-17
- В `order_lines` храним снапшоты позиции на момент заказа (`item_sku`, `item_title`, `price_text_snapshot`) для воспроизводимого экспорта.
- Excel экспорт поддерживает несколько режимов (full/assembly/pickup) вместо одного универсального файла.
- Добавлен `user_profiles` для хранения дефолтных данных пользователя (телефон/точка/имя).
- Админ‑панель выделена в отдельные компоненты: `admin_service` (RBAC API) и `admin-web` (UI).
- В админ‑панели введён dev‑mode, доступный только роли `owner`.
- Docker Compose используется как рекомендованный быстрый способ запуска всех сервисов.
- В admin_service добавлены read-only эндпоинты для каталогов и заказов.
- Добавлен Makefile с командами compose-* для удобного запуска.

## 2026-01-21
- Исправлены emoji в админской клавиатуре (кнопки "Сводка" и "Перезапуск" отображались некорректно из-за кодировки).
- Добавлен сервис ngrok в docker-compose для автоматического создания туннеля к локальному API (требуется для работы Telegram webhook).
- Настроена передача `.env` файла в контейнер ngrok для корректной инициализации authtoken.
- Реализован механизм очистки очереди tg_updates при rate-limit ошибках от Telegram API (обновления помечаются как `done` с пометкой `skipped`).
- Docker healthcheck для MySQL предотвращает race condition при старте зависимых сервисов.

## 2026-01-22
- Исправлена ошибка создания magic link: добавлена недостающая зависимость `cryptography` и `httpx` в `admin_service/requirements.txt`.
- Пересобран Docker образ `admin_service` с новыми зависимостями.
- Проверено создание admin_users записи в БД для работы эндпоинта `/auth/one-time-link`.
- Создан [[roadmap]] с планом развития проекта на 1-3 месяца вперёд.
- ~~OpenAI API отключен в пользу собственной ML-системы~~ **Восстановлен как опция**.
- Добавлен Levenshtein distance для fuzzy matching товаров (опечатки до 2 символов).
- Добавлен `_smart_preprocess_text()` для разбиения "всё в одну строку" заказов по известным точкам выдачи.

## 2026-01-22: OpenAI Toggle Restored

**Context**: OpenAI был полностью удалён, но пользователь попросил вернуть с toggle

**Decision**: Добавлен AI_MODE с тремя режимами:
- `disabled` (default) — только regex, без внешних API
- `openai` — regex + OpenAI для сложных случаев
- `local` — regex + локальная ML (future, stub)

**Implementation**:
- `backend/app/config.py` — добавлены `ai_mode` и `openai_api_key`
- `backend/app/ai/order_recognizer.py` — восстановлен `_openai_extract()`
- `.env.example` — документация переменных окружения
- Модель: `gpt-4o-mini` (быстрая и дешёвая)

**Rationale**: Дать выбор пользователю — по умолчанию без внешних зависимостей, но с возможностью включить AI при необходимости.

## 2026-01-22 (architecture cleanup)
- Зафиксированы микросервисные границы и ownership таблиц (см. [[architecture/microservices]]).
- Вся отчётная документация перенесена в `docs/upgrade/` для чистого корня репозитория.
- Роуты `admin_service` разнесены по модульным APIRouter (слой `app/api/routers/*`).

## 2026-01-22: Improved Order Parser (based on real data)

**Context**: Анализ реальных заказов из `samples/telegram/result.json` выявил новые паттерны

**Discovered Formats**:
- "Имя XXXX Точка\nТовар1\nТовар2" — стандартный многострочный
- "Имя XXXX.Точка.Товар1 Xкг.Товар2 Xшт." — точки как разделители (all-in-one)
- "Точка, XXXX, Имя, товар1, товар2" — обратный порядок, запятые
- "0124 Ольга Ванеева 93, мойва 10кг" — телефон в начале
- "Татьяна 8086Ванеева" — без пробела между телефоном и точкой
- Нумерованные списки: "1) Товар 2) Товар"

**Parser Improvements**:
- `_smart_preprocess_text()` теперь обрабатывает dot-separated формат
- `extract_pickup_place()` улучшен для встроенных мест ("Марина 6503 Автозавод")
- `extract_customer_name()` поддерживает comma-separated и reversed order
- `_is_meta_fragment()` лучше фильтрует header-строки
- Расширен список `_COMMON_NAMES` русскими именами из реальных данных

**Migration Chain Fix**:
- Исправлена цепочка миграций (были конфликтующие down_revision)
- Последовательность теперь линейная: ...→9c0f2c9e6c4b→36978f59dcc3→...→c9f8a1234567

**Test Samples**:
- Добавлен `backend/tests/test_samples.py` с 25 реальными заказами для regression testing

## 2026-01-22: v2.0 Production Upgrade

**Context**: Комплексное улучшение всех компонентов системы для production-готовности

**Bot UX Improvements**:
- Минималистичные inline-кнопки (4 базовых → 2 после успешного заказа)
- Улучшенные сообщения с emoji и Markdown форматированием
- Компактные ответы без информационного спама
- Новая функция `help_text()` с полной справкой
- Контекстные кнопки для исправления ошибок

**Order Recognition**:
- Улучшенная эвристика `looks_like_order()` с фильтрацией рецептов
- Защита от ложных срабатываний (медиа-ссылки, длинные тексты >500 символов)
- Распознавание кулинарных инструкций

**Database Schema**:
- Добавлена таблица `bot_settings` для хранения конфигурации бота
- Добавлена таблица `pickup_places` для управления точками выдачи
- Миграция: `c9f8a1234567_add_bot_settings_and_pickup_places.py`

**Admin API Extensions**:
- `GET/POST /settings` — управление настройками бота
- `GET/POST/PATCH/DELETE /pickup-places` — CRUD для точек выдачи
- Все настройки теперь управляются через UI без правки кода

**Admin Web UI**:
- Полная поддержка тёмной темы с переключателем в header
- Страница настроек (`SettingsPage`) с управлением:
  - Режим ответов (DM/chat/both)
  - Кастомизация текстов сообщений бота
  - Управление точками выдачи
  - Настройки уведомлений
- CSV экспорт заказов с фильтрацией
- Продвинутая аналитика с bar-charts

**Decision**: Админы управляют всеми аспектами бота через UI без необходимости редактировать код или переменные окружения

**Rationale**: Максимальная гибкость и простота для пользователей без технических навыков

## 2026-01-22 (refactoring)
**Major refactoring to modular architecture:**

- Создан новый пакет `app/handlers/` с модульными обработчиками:
  - `public_commands.py` — пользовательские команды
  - `admin_commands.py` — админ-команды
  - `admin_actions.py` — обработка callback кнопок
  - `callback_handler.py` — роутер callbacks

- Создан пакет `app/services/` с бизнес-логикой:
  - `telegram_service.py` — безопасный wrapper для Telegram API с retry, rate limit handling

- Исправлена обработка callback_query:
  - `answer_callback_query` теперь вызывается СРАЗУ (убирает loading indicator)
  - Проверка is_admin теперь работает и по config, и по таблице admin_users

- Подробности: [[changelog/refactoring-jan2026]]
- Обновлена документация с учётом текущих исправлений.

## 2026-01-22 (production upgrade)
**Production-ready features implementation:**

### AI Order Recognition
- Создан модуль `backend/app/ai/` с гибридным подходом regex + LLM
- `OrderRecognizer` — основной класс распознавания заказов
- `RecognitionResult` — типизированный результат с confidence scoring
- Промпты оптимизированы для русского языка (рыба/морепродукты)
- Graceful degradation если OpenAI API недоступен

### Order Service
- `backend/app/services/order_service.py` — интеграция AI с БД
- `OrderService.process_message()` — полный pipeline обработки
- Автоматический matching против каталога

### Admin Web UI Overhaul
- `MainLayout` — responsive sidebar с role-based меню
- `OrdersPage` — управление заказами с фильтрами и поиском
- `AnalyticsPage` — дашборд со статистикой и графиками
- `TemplatesPage` — управление Excel шаблонами
- `DashboardPage` — редизайн с quick stats

### Admin API Enhancements
- `GET /analytics/dashboard` — статистика для дашборда
- `GET /analytics/orders-chart` — данные для графика заказов
- `GET /analytics/top-items` — топ популярных товаров
- `GET /templates/excel` — скачивание шаблона
- `POST /templates/excel` — загрузка нового шаблона
- `PATCH /orders/{id}` — обновление статуса заказа

### Architecture Decisions
- Hybrid AI: regex first (fast), LLM fallback (accurate)
- Confidence threshold: 0.7 for AI trigger
- Role-based UI visibility (owner > admin > viewer)
- Charts без внешних библиотек (simple bar chart)

Подробности: [[changelog/implementation-status]]

## 2026-03-11: Self-healing sync for Telegram admins in admin_service

**Context**: Права доступа в админ‑панель могли «слетать», когда Telegram `user_id` оставался в `ADMIN_USER_IDS`, но запись в `admin_users` теряла `tg_user_id`, роль или активный статус. Бот продолжал считать пользователя админом, а web‑админка уже нет.

**Decision**:
- `admin_service` при старте синхронизирует `ADMIN_USER_IDS` в `admin_users`, если база уже инициализирована хотя бы одним пользователем.
- `/auth/auto-provision` и `/auth/one-time-link` самовосстанавливают аккаунт по `tg_user_id`: реактивируют пользователя и гарантируют роль не ниже `admin`.
- Автопровижининг ограничен только `tg_user_id` из `ADMIN_USER_IDS` или уже существующими связанными аккаунтами.

**Rationale**:
- Делает Telegram-вход в админку устойчивым к рестартам и ручным правкам БД.
- Устраняет рассинхрон между backend-проверкой админа и RBAC в `admin_service`.
- Одновременно убирает лишнюю дыру: сервис больше не автосоздаёт произвольный `tg_user_id` только по service key.

## 2026-01-23 (UX Improvements)
**Bot UX and Admin UI Enhancements:**

### Bot UX Improvements
- Упрощены inline-клавиатуры: минимум кнопок для обычных пользователей
- `user_inline_keyboard()` — 4 кнопки вместо 8
- `user_inline_keyboard(compact=True)` — 2 кнопки после успешного заказа
- `build_order_reply()` — улучшенные сообщения с emoji и Markdown
- `help_text()` — полная справка по формату заказа
- `order_public_message()` — поддержка /rules, /example, /help

### Order Recognition Improvements
- `looks_like_order()` — улучшенная эвристика
- Фильтрация рецептов и медиа-постов (не триггерят бота)
- Защита от ложных срабатываний на длинных текстах
- Pattern-based detection для кулинарных инструкций

### Admin UI
- **Тёмная тема**: `ThemeContext` + localStorage persistence
- Кнопка переключения темы в header
- `SettingsPage` — страница настроек бота:
  - Режим ответов (DM/chat/both)
  - Кастомизация сообщений бота
  - Управление точками выдачи
  - Настройки уведомлений
- `AnalyticsPage` — улучшения:
  - StatusDistribution — распределение заказов по статусам
  - Quick Stats — блок быстрой статистики
  - Улучшенный bar chart с показателем среднего

### API Client
- `getSettings()` — получение всех настроек
- `updateSettings()` — обновление настроек пачкой

### Architecture
- ThemeContext для глобального состояния темы
- Separation of concerns в presenter layer
- Improved error messages with emoji formatting

## 2026-01-22 (complete system improvements)
**Comprehensive system improvements to ideal state:**

### Tech Debt Closure
1. **Auto-provision admin** (`/auth/auto-provision`):
   - Автоматически создаёт admin_user при первом входе через Telegram
   - Первый пользователь получает роль `owner`, последующие — `admin`
   - Больше не нужен ручной INSERT в БД

2. **Auto-unstop expired items** (`services/auto_unstop.py`):
   - Worker проверяет stop_until каждые ~60 секунд
   - Автоматически снимает просроченные стопы (is_active=1)
   - Логирует количество авто-разблокированных позиций

### Admin API Expansion
- **Catalogs CRUD**: `POST/PATCH/DELETE /catalogs`
- **Catalog Items CRUD**: `POST/PATCH/DELETE /catalogs/{id}/items`
- **Stop/Unstop items**: `POST /catalogs/{id}/items/{id}/stop|unstop`
- **Extended Analytics**:
  - `GET /analytics/status-breakdown` — распределение по статусам
  - `GET /analytics/pickup-stats` — статистика по точкам выдачи
  - `GET /analytics/catalog-stats` — статистика по каталогам

### Admin Web UI
- **CatalogsPage** — полноценное управление каталогами:
  - CRUD операции через UI
  - Expandable row для просмотра товаров
  - Stop/unstop товаров с указанием причины
- **UsersPage** — управление пользователями:
  - Создание новых админов
  - Редактирование ролей
  - Активация/деактивация
- **Updated routes**: `/catalogs`, `/users` добавлены в MainLayout

### Backend Improvements
- `handle_public_command()` теперь передаёт Telegram username/first_name
- `_get_admin_link()` использует auto-provision вместо old one-time-link

### Updated Schemas (admin_service)
- `CatalogItemResponse`, `CatalogItemCreateRequest`, `CatalogItemUpdateRequest`
- `CatalogItemStopRequest`, `CatalogCreateRequest`, `CatalogUpdateRequest`
- `PickupPlaceResponse`, `PickupPlaceCreateRequest`, `PickupPlaceUpdateRequest`
- `DashboardStatsResponse`, `OrdersChartResponse`, `TopItemResponse`
- `AdminUserCreateRequest`, `AdminUserUpdateRequest`

## 2026-01-23 (Security & Final Polish)

### Security Documentation
- Добавлен [[11-security]] с полной документацией по безопасности
- Обновлён `.github/copilot-instructions.md` с Security Guidelines:
  - Never commit/expose checklist
  - Sensitive data handling rules
  - Code review security checkpoints
  - Docker production security

### Bot UX Polish
- Исправлены сломанные emoji в `order_presenter.py` (📝 вместо garbled)
- Унифицированы клавиатуры `user_inline_keyboard()` и `user_inline_keyboard_v2()`
- Callback-handler корректно отвечает на все кнопки

### TypeScript Improvements
- Все компоненты admin-web используют строгую типизацию
- Типы для API responses экспортируются из `client.ts`
- Исправлены implicit any warnings

### Documentation Updates
- `known-issues.md` — закрыты все ранее открытые issues
- `decisions-log.md` — задокументированы все архитектурные решения
- Security checklist добавлен в workflow

## 2026-01-22 (bot improvement analysis)

- Добавлен документ [[12-bot-improvements-analysis]] с приоритизированным планом улучшений бота.
- Решение: улучшения делаем "без фанатизма" — сначала детерминированные эвристики/нормализация/UX и quality-метрики в админке; LLM (если нужен) — только как опциональный fallback для edge-cases.

## 2026-01-22 (bot text processing fix)

### Диагностика проблемы "бот не отвечает на сообщения"

**Причины молчания бота:**
1. В групповом чате нет открытого каталога → бот молча выходил из `_maybe_process_order()`
2. Открытый каталог (в личке) был пустой (0 товаров) → нет совпадений
3. Эвристика `looks_like_order()` была слишком строгой (требовала 3+ цифр и 6+ букв)

**Исправления:**

1. **worker.py: Уведомление при отсутствии каталога**
   - Если нет активного каталога И сообщение похоже на заказ → бот отвечает "📭 Нет активной раздачи"
   - Ранее: молчал полностью

2. **order_domain.py: Улучшенная эвристика `looks_like_order()`**
   - Добавлена проверка `title_raw` (если есть распознанная товарная строка → заказ)
   - Добавлен паттерн количества (кг/г/шт/уп/л/мл и т.д.)
   - Снижен порог: 2+ цифр и 4+ букв (было 3+/6+)
   - Короткие сообщения (<100 символов, ≤3 строк, 4+ букв) теперь распознаются как возможные заказы

**Доступ к админским функциям:**
- Уже было правильно реализовано:
  - Callback `admin:` и `cmd:` проверяют `is_admin` перед выполнением
  - `user_inline_keyboard()` (старая) НЕ показывает админские кнопки
  - `user_inline_keyboard_v2(is_admin)` показывает админ-кнопки только если `is_admin=True`

**Следующие шаги:**
- Для работы бота админу нужно: открыть каталог в групповом чате и добавить товары

## 2026-01-22: Improved bot responsiveness and added "Forel"

### Context
Users in the group were sending orders like "Форель 1шт" which were being ignored by the bot.
Analysis showed:
1. "Forel" was missing from the catalog (only Gorbusha, Semga, Treska).
2. The `looks_like_order` heuristic was too strict for short messages without phone/pickup.
3. The bot silently ignored messages if no active catalog was found (or if item matching failed and heuristic returned false).

### Decisions
1. **Added "Forel" to Catalog 6**: To satisfy immediate demand (Filet Trout).
2. **Relaxed `looks_like_order`**: Now accepts short messages with qty patterns (e.g., "1шт") or simply short text lines (e.g., "Форель 1").
3. **Bot Feedback**: Added explicit reply "📭 Нет активной раздачи" if a message looks like an order but no catalog is open.
4. **Validation**: Verified against manual test cases `Форель 1шт`, `Игорь, 2322, Бор, форель 1шт`.

## 2026-01-22: Major bot improvements - smart order processing

### Context
Bot needed to handle "dumb" orders better - incomplete data, typos, unclear items. Users expected the bot to ask clarifying questions.

### Changes
1. **Fuzzy item matching** (`catalog_repo.py`):
   - Added prefix matching for first word (e.g., "форел" → "Форель филе")
   - Improved suggestion algorithm with better scoring
   - Now returns up to 3 suggestions instead of 2
   
2. **Smart order replies** (`order_presenter.py`):
   - Always shows full order summary: what's accepted, what's rejected, what's missing
   - Shows customer info (name, phone, pickup) at the top
   - Detailed per-item breakdown with suggestions for unknown items
   - Lists available catalog items when all items rejected
   
3. **Order supplement via reply** (`worker.py`):
   - When user replies to bot's "missing data" message, bot extracts the missing info
   - Recognizes: phone (4 digits), pickup place (from known list or short text), name (1-3 words)
   - Auto-updates user profile for future orders
   - Changes order status from "needs_admin" to "active" when all data collected
   
4. **Catalog re-open fix** (`admin_commands.py`):
   - `/open` command now handles duplicate codes gracefully
   - Re-opens closed catalog with same code instead of failing
   - Falls back to unique code (e.g., "20260122_1") if needed


## 2026-01-XX — Full Project Refactoring (Production Hardening)

### Architecture & Performance
- **Config caching**: Replaced 20+ per-update `load_config()` calls in worker.py with `get_config()` singleton.
- **Deprecated asyncio pattern**: Replaced `asyncio.get_event_loop().run_until_complete()` in worker's `_smart_recognize_order` with direct regex call (sync path) and `asyncio.run()` for AI path.
- **`recognize_order_sync()`**: Fast-path for regex-only avoids asyncio overhead.

### Bug Fixes
- **`_format_number` crash**: `text_parser._format_number` → `text_parser.format_number` (private name didn't exist).
- **`models.py` corruption**: Repaired `TgUpdate.processed_at` and `PickupPlace` class definitions broken by sed.
- **`datetime.utcnow` deprecation**: Replaced ~25 occurrences across all services with `datetime.now(UTC).replace(tzinfo=None)`. Added `_utcnow()` helper for ORM defaults.
- **Analytics `total_users` misleading**: Renamed to `total_customers` (counts TG users from user_profiles) + `admin_users` (counts admin accounts).
- **False success on mutations**: Added 404 existence checks to PATCH/DELETE for orders, catalogs, catalog items, pickup places.
- **Settings key injection**: Added `ALLOWED_SETTINGS_KEYS` frozenset validation for POST /settings.
- **`_open_app_link` wrong port**: Fixed default from 8000→8010, added rstrip('/').
- **Test fixture**: Updated `test_replay_pipeline_smoke.py` to use `get_config()` instead of removed `load_config`.

### Security
- **Webhook secret**: Replaced `!=` with `hmac.compare_digest()` to prevent timing attacks.
- **Telegram message overflow**: Added `_truncate_text()` at 4096 char limit.
- **Docker hardening**: Non-root users in all Dockerfiles, multi-stage nginx build for admin-web, parameterized MySQL credentials.
- **Hardcoded secrets removed**: `ADMIN_ONE_TIME_KEY=dev-secret` no longer in docker-compose (sourced from .env only).
- **CORS restricted**: Admin service now uses explicit method/header lists instead of `"*"`.
- **`.env.example` aligned**: Fixed env var names to match actual code.

### Type Safety
- **`admin_commands.py`**: Added `Config` type annotation for `cfg` parameter.
- **Frontend**: Updated `DashboardStats` interface for renamed analytics fields.

## 2026-03-02 — Admin Login UX

- Добавлен переключатель видимости пароля на странице входа `admin-web/src/pages/LoginPage.tsx`.
- Политика безопасности не менялась: пароль по‑прежнему не хранится/не отображается сервером, только вводится пользователем в форме.

## 2026-07-17: Hard-lock owner-only routes for Export and API Center

**Context**: Backend уже режет `GET /exports/presets`, `POST /exports/build` и owner tooling в `API Center` по роли `owner`, но клиентский роутинг ещё мог пустить non-owner на `/export` или `/api-center`, если visibility-настройки когда-то были расширены. В результате пользователь видел не полезный redirect, а расплывчатую ошибку загрузки конфигурации экспорта.

### Решения

- `admin-web/src/App.tsx`: маршруты `/export` и `/api-center` теперь монтируются только для `owner`, независимо от динамических section visibility.
- `admin-web/src/pages/ExportPage.tsx`: первичная ошибка загрузки пресетов теперь показывает исходную backend/network причину через `getApiErrorMessage(...)`, чтобы вместо generic-фразы было видно, это `owner role required`, `Network Error` или другая конкретика.

### Consequences

- Прямой URL больше не открывает owner-only экраны пользователям без owner-роли.
- Диагностика проблем экспорта стала быстрее: UI показывает реальную причину отказа, а не только общий текст.

## 2026-07-17: Make API catalog a clearly standalone screen

**Context**: Хотя каталог endpoint-ов уже существовал отдельно от `SettingsPage`, в навигации и текстах он читался как внутренний owner-tooling экран `API Center`. Пользовательский запрос был проще: отдельная понятная страница со всеми endpoint-ами, а не ощущение, что нужная функция «где-то в настройках». 

### Решения

- `admin-web/src/components/MainLayout.tsx`: пункт меню переименован в `API и эндпоинты`.
- `admin-web/src/App.tsx`: основным маршрутом страницы сделан `/api-endpoints`; старый `/api-center` сохранён как redirect-alias.
- `admin-web/src/pages/ApiWorkbenchPage.tsx`: заголовок и описание переписаны в терминах standalone-каталога endpoint-ов, добавлена явная info-подсказка про отдельный маршрут.

### Consequences

- Экран легче найти в боковом меню и проще воспринимать как отдельную страницу.
- Старые ссылки не ломаются благодаря redirect с `/api-center`.

## 2026-04-07 — runtime export matching hardened: repeated-line dedupe + header-aware qty normalization

**Context**: В живых выгрузках distribution/strict шаблонов обнаружились три связанных дефекта: (1) внутри одного заказа повторялся полный блок строк (`A,B,A,B`), что удваивало и summary, и количества; (2) фасовочные позиции вроде `Чука` в колонке `... уп 1кг` писались как сырые граммы (`1000`) вместо количества упаковок; (3) при fuzzy-сопоставлении короткий товар `Кижуч` мог попадать в колонку `Икра кижуча ...`.

### Решения

- Добавлено схлопывание полностью повторённой последовательности строк заказа (`_collapse_repeated_line_sequence`) и применено в:
  - `_fetch_order_lines_grouped(...)` (экспортные флоу),
  - `get_order(...)` (деталка заказа в API/UI).
- Добавлена header-aware нормализация количества для шаблонного экспорта:
  - `_parse_template_header_qty_hint(...)`,
  - `_normalize_qty_for_template_header(...)`,
  - `_extract_line_qty_for_template(...)`.
- Оба заполнителя шаблонов (`_fill_template_row`, `_fill_strict_template_row`) переведены на единый extractor, учитывающий фактический заголовок колонки и unit каталога.
- В `_normalize_qty_for_catalog_item(...)` добавлена конверсия mass→package для `unit_hint in {шт, уп, банка}` при наличии `pack_hint` (например, `1000 г` -> `1 уп` при `pack_hint=1 кг`).
- Усилен template matching:
  - `_find_matching_template_column(...)` учитывает числовые токены заголовков как guard,
  - `_template_similarity_score(...)` дополнительно штрафует конфликтные/ложные substring-совпадения и повышает вес содержательного core-overlap.

### Consequences

- Дубли строк `A,B,A,B` больше не раздувают выдачу и экспорт.
- Фасовочные колонки (`уп/шт/банка`) получают количество упаковок вместо сырых граммов при корректном `pack_hint`.
- Снижен риск ложных попаданий в «родственные», но неверные товарные колонки (fish vs caviar).
