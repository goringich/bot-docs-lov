# 16. Полный справочник endpoint-ов (для отправки команде)

Этот документ сделан в формате «отправил людям и они сразу работают».

- Дата: 2026-03-11
- Источник: фактические route-ы в `backend` и `admin_service`
- Формат: каждый запрос описан (метод, путь, авторизация, параметры, что вернёт)

## Базовые адреса

- Backend API: `http://localhost:8000`
- Admin API: `http://localhost:8010`

## Авторизация

- **JWT**: `Authorization: Bearer <token>`
- **Service key**: `X-Service-Key: <key>`

---

## A) Backend API (`backend/app/main.py`)

### `GET /health`
- Auth: не нужна
- Параметры: нет
- Ответ: `{"ok": true}`
- Назначение: healthcheck

### `GET /capabilities`
- Auth: не нужна
- Параметры: нет
- Ответ: service/version/endpoints/docs flags
- Назначение: discovery для интеграций/ops

### `POST /telegram/webhook`
- Auth: optional secret header `X-Telegram-Bot-Api-Secret-Token`
- Body: Telegram update JSON (должен содержать `update_id`)
- Ответ: `{"ok": true}`
- Назначение: приём webhook и запись в очередь `tg_updates`

---

## B) Admin API — Auth

### `POST /auth/login`
- Auth: не нужна
- Body: `email`, `password`
- Ответ: `access_token`
- Назначение: логин в admin-web

### `POST /auth/bootstrap`
- Auth: `X-Bootstrap-Token`
- Body: `email`, `name`, `password`
- Ответ: созданный первый owner
- Назначение: первичная инициализация owner-аккаунта

### `POST /auth/one-time-link`
- Auth: `X-Service-Key`
- Query: `tg_user_id`
- Ответ: one-time login URL
- Назначение: вход в админку из Telegram

### `POST /auth/token-login`
- Auth: не нужна
- Query: `one_time_token`
- Ответ: `access_token`
- Назначение: обмен one-time токена на JWT

### `POST /auth/auto-provision`
- Auth: `X-Service-Key`
- Query: `tg_user_id`, optional `tg_username`, `tg_first_name`
- Ответ: one-time login URL
- Назначение: авто-провижининг admin по Telegram ID

---

## C) Admin API — Users / Roles

### `GET /me`
- Auth: JWT
- Ответ: профиль текущего пользователя

### `GET /users`
- Auth: JWT
- Ответ: список пользователей

### `POST /users`
- Auth: JWT (owner/regional_admin)
- Body: `email`, `password`, `name`, `role_ids[]`, optional TG поля
- Ответ: созданный пользователь

### `GET /users/{user_id}`
- Auth: JWT
- Ответ: детальная карточка пользователя

### `PUT /users/{user_id}/roles`
- Auth: JWT (owner/regional_admin)
- Body: `role_ids[]`
- Ответ: пользователь с обновлёнными ролями

### `PUT /users/{user_id}/delegate-create`
- Auth: JWT (owner)
- Body: `enabled`
- Ответ: пользователь с обновлённым правом делегирования

### `PUT /users/{user_id}/status`
- Auth: JWT (owner/regional_admin)
- Body: `is_active`
- Ответ: пользователь с новым статусом

### `DELETE /users/{user_id}`
- Auth: JWT (owner/regional_admin)
- Ответ: `{status: deleted}`

### `GET /regions/all`
- Auth: JWT
- Ответ: список доступных регионов

### `GET /users/full`
- Auth: JWT (owner)
- Ответ: полный список пользователей с расширенными полями

### `PUT /users/{user_id}/password`
- Auth: JWT (owner)
- Body: `new_password`
- Ответ: `{status: ok}`

### `POST /users/{user_id}/invite-link`
- Auth: JWT (owner/regional_admin)
- Ответ: one-time login URL

### `POST /users/confirm-owner-role`
- Auth: JWT (owner)
- Body: `target_user_id`, `owner_password`
- Ответ: пользователь с ролью owner

### `GET /roles`
- Auth: JWT
- Ответ: список ролей

### `POST /roles`
- Auth: JWT (owner)
- Body: `code`, `title`, `level`
- Ответ: созданная роль

### `GET /permissions`
- Auth: JWT
- Ответ: список permissions

---

## D) Admin API — Catalogs / Chats

### `GET /chats`
- Auth: JWT
- Query: `limit`
- Ответ: список чатов

### `PATCH /chats/{tg_chat_id}`
- Auth: JWT (owner)
- Body: optional `bot_visible`, `title`
- Ответ: `{status: ok}`

### `GET /catalogs`
- Auth: JWT
- Query: `status`, `limit`, `offset`
- Ответ: список каталогов

### `GET /catalogs/{catalog_id}`
- Auth: JWT
- Ответ: каталог

### `POST /catalogs`
- Auth: JWT (owner)
- Body: `chat_id`, optional `code`, `title`
- Ответ: созданный каталог

### `POST /catalogs/{catalog_id}/clone`
- Auth: JWT (owner)
- Body: `target_chat_id`, optional `title`, `close_existing_open`
- Ответ: новый каталог-клон

### `PATCH /catalogs/{catalog_id}`
- Auth: JWT (owner)
- Body: optional `title`, `status`
- Ответ: обновлённый каталог

### `DELETE /catalogs/{catalog_id}`
- Auth: JWT (owner)
- Ответ: `{status: deleted}`

---

## E) Admin API — Catalog items

### `GET /catalogs/{catalog_id}/items`
- Auth: JWT
- Query: `include_stopped`
- Ответ: список товаров каталога

### `GET /catalogs/{catalog_id}/items/{item_id}`
- Auth: JWT
- Ответ: товар каталога

### `POST /catalogs/{catalog_id}/items`
- Auth: JWT (owner)
- Body: `sku`, `title`, optional hints/aliases/price
- Ответ: созданный товар

### `PATCH /catalogs/{catalog_id}/items/{item_id}`
- Auth: JWT (owner)
- Body: partial update
- Ответ: обновлённый товар

### `POST /catalogs/{catalog_id}/items/{item_id}/stop`
- Auth: JWT (owner)
- Body: optional `stop_reason`, `stop_until`
- Ответ: товар в стопе

### `POST /catalogs/{catalog_id}/items/{item_id}/unstop`
- Auth: JWT (owner)
- Body: нет
- Ответ: товар активирован

### `DELETE /catalogs/{catalog_id}/items/{item_id}`
- Auth: JWT (owner)
- Ответ: `{status: deleted}`

---

## F) Admin API — Admin/user chat assignments

### `GET /admin-user-chats`
- Auth: JWT
- Query: optional `admin_user_id`
- Ответ: список назначений

### `POST /admin-user-chats`
- Auth: JWT (owner)
- Body: `admin_user_id`, `chat_id`
- Ответ: `{status: assigned}`

### `DELETE /admin-user-chats/{assignment_id}`
- Auth: JWT (owner)
- Ответ: `{status: removed}`

---

## G) Admin API — Orders / Export

### `GET /orders`
- Auth: JWT
- Query: `catalog_id`, `chat_id`, `status`, `pickup_place`, `search`, `date_from`, `date_to`, `limit`, `offset`
- Ответ: список заказов

### `GET /orders/{order_id:int}`
- Auth: JWT
- Ответ: заказ + строки

### `PATCH /orders/{order_id:int}`
- Auth: JWT
- Body: allowlist полей (`status`, `customer_name`, `phone_last4`, `pickup_place`, `raw_text`, `error_text`)
- Ответ: `{success: true, ...updated_fields}`

### `GET /orders/export`
- Auth: JWT
- Query: те же фильтры, что у `/orders`
- Ответ: XLSX (операционный экспорт)

### `GET /orders/export-template`
- Auth: JWT
- Query: те же фильтры
- Ответ: XLSX по шаблону

### `GET /orders/export-history`
- Auth: JWT
- Query: те же фильтры
- Ответ: XLSX для аудита/истории

---

## H) Admin API — Analytics

### `GET /analytics/dashboard`
- Auth: JWT
- Query: optional `chat_id`
- Ответ: сводные KPI

### `GET /analytics/orders-chart`
- Auth: JWT
- Query: `days`, optional `chat_id`
- Ответ: series labels/data

### `GET /analytics/top-items`
- Auth: JWT
- Query: `limit`, optional `chat_id`
- Ответ: топ товаров

### `GET /analytics/status-breakdown`
- Auth: JWT
- Query: optional `chat_id`
- Ответ: разрез по статусам

### `GET /analytics/pickup-stats`
- Auth: JWT
- Query: `limit`, optional `chat_id`
- Ответ: статистика по точкам выдачи

### `GET /analytics/catalog-stats`
- Auth: JWT
- Query: нет
- Ответ: статистика по каталогам

### `GET /analytics/parser-accuracy`
- Auth: JWT
- Query: нет
- Ответ: метрики точности парсера

---

## I) Admin API — Settings / Pickup places

### `GET /settings`
- Auth: JWT
- Ответ: key-value настройки бота

### `POST /settings`
- Auth: JWT
- Body: JSON ключи из allowlist
- Ответ: `{status: ok, updated_keys: [...]}`

### `GET /pickup-places`
- Auth: JWT
- Query: нет
- Ответ: список точек выдачи

### `POST /pickup-places`
- Auth: JWT
- Body: `title`, optional `chat_id`, `aliases`, `is_active`
- Ответ: созданная точка

### `PATCH /pickup-places/{place_id}`
- Auth: JWT
- Body: partial (`title`, `is_active`)
- Ответ: `{status: ok}`

### `DELETE /pickup-places/{place_id}`
- Auth: JWT
- Body: нет
- Ответ: `{status: ok, deleted_id: ...}`

---

## J) Admin API — Templates

### `GET /templates/excel`
- Auth: JWT
- Ответ: файл template.xlsx

### `POST /templates/excel`
- Auth: JWT (owner)
- Body: multipart file `.xlsx`
- Ответ: `{success: true, ...}`

---

## K) Admin API — Debug

### `POST /debug/test-parser`
- Auth: JWT (owner)
- Body: `{text: ...}`
- Ответ: структура теста парсинга

### `POST /debug/test-match`
- Auth: JWT (owner)
- Body: `{query: ..., catalog_id?: ...}`
- Ответ: exact/fuzzy match + suggestions

### `GET /debug/status`
- Auth: JWT (owner)
- Ответ: краткий статус системы

### `GET /debug/project-state`
- Auth: JWT (owner)
- Ответ: расширенный статус (worker/webhook/schema/chats)

---

## L) Admin API — Dev mode

### `GET /settings/dev-mode`
- Auth: JWT
- Ответ: текущее значение `dev_mode`

### `PUT /settings/dev-mode`
- Auth: JWT (owner)
- Body: `{value: "on"|"off"}`
- Ответ: новое значение `dev_mode`

---

## M) M2M Integration API (`/integrations/*`)

### `GET /integrations/capabilities`
- Auth: `X-Service-Key`
- Ответ: список интеграционных endpoint-ов

### `GET /integrations/chats`
- Auth: `X-Service-Key`
- Query: `limit`
- Ответ: чаты

### `GET /integrations/catalogs/open`
- Auth: `X-Service-Key`
- Ответ: открытые каталоги

### `GET /integrations/pickup-places`
- Auth: `X-Service-Key`
- Query: `tg_chat_id`, `only_active`
- Ответ: точки выдачи

### `GET /integrations/orders`
- Auth: `X-Service-Key`
- Query: `limit`, `status`, `since_id`
- Ответ: orders feed (`count`, `server_time`, `items[]`)

---

## Как быстро проверить, что всё доступно

1. Открыть Swagger:
   - `http://localhost:8000/docs`
   - `http://localhost:8010/docs`
2. Проверить discovery:
   - `GET /capabilities`
   - `GET /integrations/capabilities`
3. Проверить auth:
   - JWT endpoint `/me`
   - service key endpoint `/integrations/chats`

### [[17-admin-service-endpoints-manual]]