# 14. Полный API Reference проекта

Документ фиксирует **фактически реализованные** API всех микросервисов на дату 2026-03-11.

## Карта микросервисов и API

1. `backend` (`backend/app/main.py`)  
   - Назначение: приём Telegram webhook и запись update-ов в очередь БД.
   - Протокол: HTTP/JSON.
2. `admin_service` (`admin_service/app/main.py`)  
   - Назначение: RBAC API для админки, аналитики, экспорта, управления каталогами/заказами.
   - Протокол: HTTP/JSON + XLSX download/upload.
3. `admin-web`  
   - Это UI (не отдельный backend API), клиент к `admin_service`.

---

## Где посмотреть API «вживую»

### Backend

- Swagger: `http://localhost:8000/docs` (локально; в deployment используйте свой домен)
- OpenAPI: `http://localhost:8000/openapi.json` (локально; в deployment используйте свой домен)
- Discovery endpoint: `GET /capabilities`

### Admin service

- Runtime Swagger/OpenAPI сейчас отключены в `admin_service` (`docs_url=None`, `openapi_url=None`) как часть security posture.
- Основной удобный manual для разработчиков: [[17-admin-service-endpoints-manual]].
- Health: `GET /health`
- M2M capabilities: `GET /integrations/capabilities`

---

## 1) Backend API (`backend`)

Base URL: `http://localhost:8000` локально; в deployment — домен/URL вашего backend.

### `GET /health`

Liveness-проба.

Ответ:

```json
{ "ok": true }
```

### `GET /capabilities`

Публичный discovery endpoint сервиса.

Возвращает:
- `service`
- `version`
- `endpoints`
- `messaging.default_provider`
- `messaging.enabled_providers`
- `messaging.supports`
- `docs_enabled`
- `docs_url`
- `openapi_url`

### `POST /telegram/webhook`

Принимает Telegram update и сохраняет его в `tg_updates`.

Особенности:
- при включённом `TELEGRAM_WEBHOOK_SECRET` проверяется заголовок `X-Telegram-Bot-Api-Secret-Token`;
- request сохраняется в provider-agnostic envelope для дальнейшей unified обработки worker-ом;
- при дубликате `update_id` возвращается `ok` (идемпотентность);
- при некорректном payload (`update_id` отсутствует) — `400`.

### `POST /integrations/{provider}/webhook`

Универсальный ingress endpoint для провайдеров `telegram | matrix | vk | max | webhook`.

Особенности:
- проверяет, что provider разрешён в `MESSENGER_ENABLED_PROVIDERS`;
- поддерживает provider-specific secret headers:
   - `telegram` → `X-Telegram-Bot-Api-Secret-Token`
   - `matrix` → `X-Matrix-Webhook-Secret`
   - `vk` → `X-VK-Webhook-Secret`
   - `max` → `X-Max-Webhook-Secret`
   - `webhook` / fallback → `X-Integration-Secret`
- для non-Telegram provider использует per-provider secret (`MATRIX_WEBHOOK_SECRET`, `VK_WEBHOOK_SECRET`, `MAX_WEBHOOK_SECRET`) либо общий `INTEGRATION_WEBHOOK_SECRET` как fallback;
- сохраняет payload в envelope (`__integration_envelope__`) и кладёт в `tg_updates` с вычисленным `update_id`;
- worker нормализует такие события в общий message contract и запускает тот же domain pipeline, что и для Telegram.

### Multi-provider outbound bridge contract

Для non-Telegram provider worker теперь умеет отправлять стандартные действия через bridge endpoint провайдера.

Поддерживаемые действия:
- `send_message`
- `send_document`
- `answer_callback_query`
- `set_message_reaction`
- `delete_message`

Runtime contract:
- URL берётся из provider outbound configuration текущего окружения;
- auth secret берётся из provider bridge configuration текущего окружения;
- запросы отправляются как HTTP JSON с заголовком `X-Bridge-Secret`;
- если outbound URL не настроен, pipeline не падает: действие пропускается как best-effort no-op и логируется как отсутствующий bridge target.

---

## 2) Admin Service API (`admin_service`)

Base URL: `http://localhost:8010` локально; в deployment — домен/URL вашего `admin_service`.

### Авторизация

- JWT flow: `Authorization: Bearer <token>`
- M2M flow: `X-Service-Key: <key>`

### 2.1 Auth

- `POST /auth/login`
- `POST /auth/bootstrap`
- `POST /auth/one-time-link`
- `POST /auth/token-login`
- `POST /auth/auto-provision`
- `POST /auth/confirm-password`
- `POST /auth/logout`
- `GET /auth/sessions`
- `DELETE /auth/sessions/{session_id}`
- `POST /auth/revoke-all-sessions`

### 2.2 Users / roles

- `GET /me`
- `GET /users`
- `POST /users`
- `GET /users/{user_id}`
- `PUT /users/{user_id}/roles`
- `PUT /users/{user_id}/delegate-create`
- `GET /users/{user_id}/regions`
- `PUT /users/{user_id}/regions`
- `PUT /users/{user_id}/status`
- `DELETE /users/{user_id}`
- `GET /regions/all`
- `GET /users/full` (owner)
- `PUT /users/{user_id}/password` (owner)
- `POST /users/{user_id}/invite-link`
- `POST /users/confirm-owner-role` (owner)
- `GET /roles`
- `POST /roles` (owner)
- `GET /permissions`

Users / regional-scope notes:
- `GET /me`, `GET /users`, `GET /users/full` возвращают `region_names` для пользователя.
- `GET /users/{user_id}/regions` и `PUT /users/{user_id}/regions` управляют списком городов / регионов пользователя.
- `regional_admin` видит и редактирует только тех менеджеров / операторов, у которых есть пересечение по назначенным регионам.
- При `POST /users` regional-admin может указывать только собственные регионы; если не указать ни одного, backend автоматически назначит регионы текущего regional-admin.

### 2.3 Catalogs / chats

- `GET /chats`
- `PATCH /chats/{tg_chat_id}`
- `GET /catalogs`
- `GET /catalogs/{catalog_id}`
- `POST /catalogs`
- `POST /catalogs/{catalog_id}/clone`
- `PATCH /catalogs/{catalog_id}`
- `DELETE /catalogs/{catalog_id}`

Catalog items:
- `GET /catalogs/{catalog_id}/items`
- `GET /catalogs/{catalog_id}/items/{item_id}`
- `POST /catalogs/{catalog_id}/items`
- `PATCH /catalogs/{catalog_id}/items/{item_id}`
- `POST /catalogs/{catalog_id}/items/{item_id}/stop`
- `POST /catalogs/{catalog_id}/items/{item_id}/unstop`
- `DELETE /catalogs/{catalog_id}/items/{item_id}`

Admin-chat assignments:
- `GET /admin-user-chats`
- `POST /admin-user-chats`
- `DELETE /admin-user-chats/{assignment_id}`

### 2.4 Orders / exports

- `GET /orders`
- `GET /orders/{order_id:int}`
- `PATCH /orders/{order_id:int}`
- `POST /orders/{order_id}/lines` — добавить строку в заказ (owner only)
- `PATCH /orders/{order_id}/lines/{line_id}` — обновить строку заказа (owner only)
- `DELETE /orders/{order_id}/lines/{line_id}` — удалить строку заказа (owner only)
- `GET /orders/export`
- `GET /orders/export-template`
- `GET /orders/export-history`
- `GET /orders/delivered`
- `GET /orders/errors`

### 2.4.1 Order Line CRUD

`POST /orders/{order_id}/lines`:
- Body: `{ title_raw, qty_raw?, qty_value_text?, unit?, item_sku?, item_title?, status? }`
- Создаёт новую строку. `status` по умолчанию `"ok"`.

`PATCH /orders/{order_id}/lines/{line_id}`:
- Body: `{ title_raw?, qty_raw?, qty_value_text?, unit?, status?, item_sku?, item_title?, catalog_item_id? }`
- Обновляет только переданные поля. `status` валидируется: `ok`, `unknown_item`, `bad_qty`, `stopped`.

`DELETE /orders/{order_id}/lines/{line_id}`:
- Удаляет строку заказа.

### 2.4.2 Audit Log

- `GET /audit-log` — журнал действий (owner only)
  - Параметры: `limit`, `offset`, `entity_type`, `entity_id`, `admin_user_id`
  - Возвращает: `{ items: [...], total: N }`

Поддерживаемые фильтры (`/orders` и export endpoints):
- `catalog_id`
- `chat_id`
- `status`
- `pickup_place`
- `search`
- `date_from`
- `date_to`
- `limit` / `offset` (для списка)

RBAC / scope:
- `owner` видит и редактирует все заказы.
- `pickup_admin` видит и редактирует только заказы назначенных точек выдачи.
- Это ограничение применяется не только к списку, но и к `GET /orders/{id}`, `PATCH /orders/{id}` и всем XLSX export route-ам.
- `delivery_status` может менять только `owner` или `pickup_admin`.

`GET /orders/export-template`:
- использует только активный лист загруженного Excel-шаблона;
- удаляет лишние листы из итогового файла;
- кроме состава заказа может подставлять delivery metadata: `Статус заказа`, `Статус выдачи`, `Получатель`, `Телефон получателя`, `Кто отметил`, `Что выдано`, `Что осталось`, `Заметки`, `Ссылка на сообщение`;
- остаётся ограниченным pickup-scope для `pickup_admin`.

`PATCH /orders/{order_id}` allow-list:
- `status`
- `customer_name`
- `phone_last4`
- `pickup_place`
- `raw_text`
- `error_text`
- `delivery_status`

Для `pickup_admin` смена `pickup_place` разрешена только в рамках собственных назначенных точек.

### 2.5 Analytics

- `GET /analytics/dashboard`
- `GET /analytics/orders-chart`
- `GET /analytics/top-items`
- `GET /analytics/status-breakdown`
- `GET /analytics/pickup-stats`
- `GET /analytics/catalog-stats`
- `GET /analytics/parser-accuracy`

### 2.6 Settings / templates / debug

Settings:
- `GET /settings`
- `POST /settings`
- `GET /pickup-places`
- `POST /pickup-places`
- `PATCH /pickup-places/{place_id}`
- `DELETE /pickup-places/{place_id}`

Settings / pickup-points notes:
- `POST /settings` — owner-only, allow-listed keys only.
- Ключи режима раздачи: `distribution_mode`, `distribution_mode_chat_ids`, `distribution_message`.
- `POST/PATCH/DELETE /pickup-places*` — owner-only.
- `chat_id` в pickup-place API может передаваться как internal chat PK, так и как публичный `tg_chat_id`.

Delivery / pickup-admin:
- `GET /deliveries`
- `POST /deliveries`
- `GET /deliveries/{id}`
- `PATCH /deliveries/{id}`
- `GET /pickup-admin-assignments`
- `POST /pickup-admin-assignments`
- `DELETE /pickup-admin-assignments/{admin_user_id}/{pickup_place_id}`
- `GET /pickup-admin/my-points`

Практический UI-flow:
- `/delivery` в `admin-web` открывает рабочее место смены с вкладками `К выдаче / Журнал / Отмеченные / Проблемы`.
- `POST /deliveries` и `PATCH /deliveries/{id}` допускают как обычные line-based элементы с `line_id`, так и ручные позиции через `custom_title`.
- `GET /deliveries` возвращает `raw_text` заказа через JOIN с таблицей orders.
- `GET /orders/delivered` возвращает `raw_text` заказа.
- `GET /orders/errors` возвращает `raw_text` заказа (было всегда).

Owner dev-mode:
- `GET /settings/dev-mode`
- `PUT /settings/dev-mode`

Templates:
- `GET /templates/excel`
- `POST /templates/excel`

Debug (owner):
- `POST /debug/test-parser`
- `POST /debug/test-match`
- `GET /debug/status`
- `GET /debug/project-state`

### 2.7 Integration (M2M)

- `GET /integrations/capabilities`
- `GET /integrations/chats`
- `GET /integrations/catalogs/open`
- `GET /integrations/pickup-places`
- `GET /integrations/orders`

Подробности: [[13-integration-api]].

Примечание: список точек выдачи для JWT API (`GET /pickup-places`) и service-key API (`GET /integrations/pickup-places`) теперь собирается через общий helper, чтобы не расходилась логика chat-id/title/aliases.

---

## 3) Admin-web как API-клиент

Реальный список используемых фронтом endpoint-ов: `admin-web/src/api/client.ts`.

Это удобный «практический контракт» между UI и `admin_service`.

Отдельный in-app экран для endpoint-ов убран из `admin-web`: API-каталог теперь поддерживается как документация для разработчиков, а не как UI-фича продукта.

Используйте:

- [[17-admin-service-endpoints-manual]] — быстрый dev-manual
- `admin-web/src/api/apiCatalog.ts` — source of truth по route-ам и request fields
- `admin-web/src/api/client.ts` — практический клиентский контракт

---

## Проверка консистентности API (чек-лист)

1. Открыть `backend` и `admin_service` OpenAPI (`/openapi.json`).
2. Сверить с этим документом.
3. Проверить клиентские вызовы в `admin-web/src/api/client.ts`.
4. При изменении route-ов обязательно обновить:
   - [[14-api-reference]]
   - [[13-integration-api]] (если затронуты M2M ручки)
   - `README.md` (раздел навигации по документации)

### [[15-endpoint-consumers-matrix]]