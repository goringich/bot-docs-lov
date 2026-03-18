# Data model (актуально на 2026-01-17)

Эта страница описывает текущую схему данных (SQLAlchemy модели в `backend/app/models.py`).

Цели схемы:
- хранить все апдейты Telegram и быть устойчивыми к перезапускам (идемпотентность);
- хранить “агрегированное” состояние заказа по пользователю в рамках каталога;
- хранить строки заказа с результатом парсинга (`ok|bad_qty|unknown_item|stopped`) и снапшотами товара на момент заказа.

## Сущности

### `TgUpdate`
Raw update от Telegram.

Ключевые поля:
- `status`: `new|processing|done|failed`
- `payload_json`: raw JSON
Назначение: очередь для worker’а + аудит.

Снимок сообщения, чтобы иметь историю/контекст (и будущие реплеи).

### `PickupPlace`
Справочник точек выдачи для конкретного чата.
- `title`
- `aliases` (comma-separated)
- `is_active` (1/0)

### `UserProfile`
Профиль пользователя (по Telegram user_id) для авто‑подстановки данных в заказ:

- `tg_user_id`
- `customer_name`
- `phone_last4`
- `default_pickup`

Профиль заполняется через команды `/set_pickup`, `/set_phone` и при оформлении заказа.
### `Catalog`
Каталог (сбор) в конкретном чате.

Ключевые поля:
- `chat_id`
- `code` (unique per chat)
- `status`: например `open|closed`
- `opened_at`/`closed_at`

Ограничение по бизнес-правилу: в одном чате один активный каталог.

### `CatalogItem`
Позиция прайса в рамках каталога.

Ключевые поля:
- `catalog_id`
- `sku` (unique per catalog)

Текстовые поля:
- `title`
- `aliases` — запятоёнки, все варианты написания для fuzzy matching
- `unit_hint`, `pack_hint`, `price_text`

Стоп-лист:
- `is_active` (1/0)
- `stop_at` (когда поставили)
- `stop_until` (временный стоп до даты/времени)
- `stop_reason`

Для заполнения реального каталога используется `backend/scripts/seed_catalog_real.py` — 45 товаров с rich aliases, 8 пунктов выдачи.

### `Order`
Агрегированный заказ пользователя в рамках каталога.

Ключевые поля:
- `catalog_id`
- `tg_chat_id`
- `tg_user_id`
- `source_message_id` (unique per catalog)

Поля пользователя:
- `customer_name`
- `phone_last4`
- `pickup_place`

Статус:
- `status`: например `active|canceled|needs_admin`

Исходник:
- `raw_text`
- `error_text`

### `OrderLine`
Строка заказа.

Ввод:
- `title_raw`, `qty_raw`

Нормализация:
- `qty_value_text` (нормализованный текст количества)
- `unit` (`kg|g|pcs|pack`)

Привязка к прайсу:
- `catalog_item_id` (nullable)

Снапшоты товара на момент заказа (важно для корректного экспорта и повторяемости):
- `item_title`
- `item_sku`
- `price_text_snapshot`

Статус строки:
- `status`: `ok|stopped|unknown_item|bad_qty`
- `error_text`

### `CommandSnapshot`
Снимок выполнения команд (в т.ч. админских), для идемпотентности и аудита.

## Индексы и уникальность

- `tg_updates.update_id` — unique
- `orders (catalog_id, source_message_id)` — unique
- `catalog_items (catalog_id, sku)` — unique
- дополнительные индексы по статусам (`tg_updates.status`, `orders.status`, `order_lines.status`) для быстрых выборок.

## Где смотреть актуальный код

- Модели: `backend/app/models.py`
- Миграции: `backend/migrations/versions/`

### `Chat` (добавлено 2026-06)
Telegram-чат с привязкой бота.

- `chat_id` (Telegram ID)
- `title`
- `chat_type` (group/private)
- `bot_visible` (1=бот виден, 0=скрытый режим) — управляет видимостью ответов бота

### `AdminUserChats` (добавлено 2026-06)
Связка admin_user ↔ chat для скоупинга данных.

- `admin_user_id`
- `chat_id` (FK → chats.id)
- UNIQUE (admin_user_id, chat_id)

### `OrderDelivery` (добавлено 2026-07)
Запись о выдаче заказа на точке.

- `order_id` (FK → orders.id)
- `delivered_by_admin_id` (FK → admin_users.id, nullable)
- `delivered_by_name`
- `pickup_place_id` (FK → pickup_places.id, nullable)
- `pickup_place_title`
- `recipient_name`, `recipient_phone`
- `items_delivered` (JSON), `items_remaining` (JSON)
- `status`: `delivered|partially_delivered|transferred`
- `notes`
- `transferred_to_name`, `transferred_to_phone` — для передачи остатка другому лицу
- `delivered_at` (datetime)

### `AdminUserPickupPoint` (добавлено 2026-07)
Назначение админа выдачи на конкретную точку.

- `admin_user_id` (FK → admin_users.id)
- `pickup_place_id` (FK → pickup_places.id)
- `assigned_by` (FK → admin_users.id, nullable)
- `assigned_at` (datetime)
- UNIQUE (admin_user_id, pickup_place_id)

### `Order.delivery_status` (добавлено 2026-07)
Новое поле в таблице `orders`:
- `delivery_status`: `pending|delivered|partially_delivered|transferred` — отслеживание выдачи

### [[05-parsing-rules]]