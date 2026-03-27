# 17. Admin service endpoints manual

Дата: 2026-03-16

Это **developer-manual**, а не экран внутри `admin-web`.

Цель документа: дать другим разработчикам один удобный вход в endpoint-ы `admin_service` без live-UI, smoke-run и ручного выполнения запросов из интерфейса сайта.

## Где source of truth

Если нужен актуальный список route-ов и request fields:

- каталог endpoint-ов: `admin-web/src/api/apiCatalog.ts`
- реальный фронтовый клиент: `admin-web/src/api/client.ts`
- расширенное описание API: [[14-api-reference]]
- M2M / external integration flow: [[13-integration-api]]

## Base URL и auth

Локально `admin_service` обычно доступен на `http://localhost:8010`; в deployment base URL задаётся окружением и reverse-proxy слоем.

Основные режимы авторизации:

- browser session через `HttpOnly` cookie `admin_access_token` — основной admin-web flow
- `Authorization: Bearer <jwt>` — совместимый backend/test flow
- `X-Service-Key: <key>` — межсервисные integration route-ы или one-time-link flow, но с разными ключами
- `X-Password-Confirm-Token: <token>` — step-up подтверждение для чувствительных export route-ов

Для cookie-auth mutating запросов backend требует совпадающий `Origin` или `Referer` с `ADMIN_WEB_URL`. Это часть CSRF hardening слоя.

Разделение ключей:

- `ADMIN_ONE_TIME_KEY` — только `/auth/one-time-link` и `/auth/auto-provision`
- `ADMIN_INTEGRATION_SERVICE_KEY` — только `/integrations/*`

## Как читать этот manual

Для каждого endpoint-а важны 4 вещи:

- `method + path`
- кто имеет доступ
- что делает ручка
- где искать поля запроса: в `admin-web/src/api/apiCatalog.ts`

---

## Auth & Sessions

- `POST /auth/login` — вход по email + password, выставляет cookie session и возвращает статус без bearer token
- `POST /auth/one-time-link` — magic link для service-key flow
- `POST /auth/token-login` — обмен one-time token на cookie session, без bearer token в response body
- `POST /auth/auto-provision` — автосоздание/связка admin-user по integration flow
- `POST /auth/confirm-password` — выдать step-up token для чувствительных действий вроде export и password reset
- `POST /auth/logout` — завершить текущую сессию
- `GET /auth/sessions` — список активных сессий
- `DELETE /auth/sessions/{session_id}` — отозвать одну сессию
- `POST /auth/revoke-all-sessions` — завершить все сессии пользователя

## Users / Roles / ACL

- `GET /me` — текущий пользователь, роли, ACL-контекст
- `GET /users` — список пользователей в видимом scope
- `POST /users` — создать пользователя
- `GET /users/{user_id}` — детали пользователя
- `PUT /users/{user_id}/password` — owner-only сброс пароля, требует step-up token
- `PUT /users/{user_id}/status` — активировать/деактивировать пользователя
- `PUT /users/{user_id}/roles` — обновить роли
- `PUT /users/{user_id}/delegate-create` — дать/снять право создавать пользователей
- `GET /users/{user_id}/regions` — получить регионы пользователя
- `PUT /users/{user_id}/regions` — обновить регионы пользователя
- `DELETE /users/{user_id}` — удалить пользователя
- `GET /regions/all` — справочник известных регионов
- `GET /users/full` — расширенный owner-only список пользователей
- `POST /users/{user_id}/invite-link` — создать single-use invite-link
- `GET /roles` — список ролей
- `POST /roles` — создать новую роль
- `GET /permissions` — список permission keys

## Catalogs / Chats / Scoping

### Chats

- `GET /chats` — список чатов
- `PATCH /chats/{tg_chat_id}` — обновить bot visibility / title / настройки чата

### Catalogs

- `GET /catalogs` — список каталогов
- `GET /catalogs/{catalog_id}` — детали каталога
- `POST /catalogs` — создать каталог
- `POST /catalogs/{catalog_id}/clone` — клонировать каталог в другой чат
- `PATCH /catalogs/{catalog_id}` — обновить каталог
- `DELETE /catalogs/{catalog_id}` — удалить каталог

### Catalog items

- `GET /catalogs/{catalog_id}/items` — список товаров каталога
- `GET /catalogs/{catalog_id}/items/{item_id}` — детали товара
- `POST /catalogs/{catalog_id}/items` — создать товар
- `PATCH /catalogs/{catalog_id}/items/{item_id}` — обновить товар
- `POST /catalogs/{catalog_id}/items/{item_id}/stop` — поставить товар на стоп
- `POST /catalogs/{catalog_id}/items/{item_id}/unstop` — снять стоп
- `DELETE /catalogs/{catalog_id}/items/{item_id}` — удалить товар

### Admin ↔ chat assignments

- `GET /admin-user-chats` — список назначений admin-user ↔ chat
- `POST /admin-user-chats` — создать назначение
- `DELETE /admin-user-chats/{assignment_id}` — удалить назначение

## Orders / Lines / Delivery / Audit

### Orders

- `GET /orders` — главный список заказов с фильтрами
- `GET /orders/{order_id}` — детали заказа
- `PATCH /orders/{order_id}` — обновить заказ
- `DELETE /orders/{order_id}` — удалить заказ

### Order lines

- `POST /orders/{order_id}/lines` — добавить строку
- `PATCH /orders/{order_id}/lines/{line_id}` — обновить строку
- `DELETE /orders/{order_id}/lines/{line_id}` — удалить строку

### Operational / XLSX exports

- `GET /orders/export` — операционный XLSX-экспорт
- `GET /orders/export-history` — исторический XLSX-экспорт
- `GET /orders/export-template` — экспорт по Excel-шаблону

### Delivery flow

- `GET /orders/{order_id}/delivery/latest` — последняя выдача по заказу
- `GET /orders/delivered` — список отмеченных/выданных заказов
- `GET /orders/errors` — проблемные заказы
- `GET /deliveries` — список выдач
- `POST /deliveries` — создать запись выдачи
- `GET /deliveries/{id}` — детали выдачи
- `PATCH /deliveries/{id}` — обновить выдачу
- `DELETE /deliveries/{id}` — откатить выдачу

### Pickup admin assignment

- `GET /pickup-admin-assignments` — матрица назначений pickup-admin
- `POST /pickup-admin-assignments` — назначить админа на точку выдачи
- `DELETE /pickup-admin-assignments/{admin_user_id}/{pickup_place_id}` — снять назначение
- `GET /pickup-admin/my-points` — точки текущего pickup-admin

### Audit / security feeds

- `GET /audit-log` — журнал действий
- `GET /login-attempts` — журнал попыток входа
- `GET /security-summary` — агрегированная security summary

## Analytics / Export Builder / Settings / Templates / Debug

### Analytics

- `GET /analytics/dashboard`
- `GET /analytics/orders-chart`
- `GET /analytics/top-items`
- `GET /analytics/status-breakdown`
- `GET /analytics/pickup-stats`
- `GET /analytics/catalog-stats`
- `GET /analytics/parser-accuracy`
- `GET /analytics/pickup-detailed`
- `GET /analytics/chat-stats`
- `GET /analytics/delivery-stats`

### Export builder metadata

- `GET /exports/presets` — список sheet presets и template presets
- `POST /exports/build` — собрать multi-sheet XLSX

### Settings

- `GET /settings` — все настройки
- `POST /settings` — bulk update allow-listed settings
- `GET /settings/{key}` — получить одну настройку
- `PUT /settings/{key}` — обновить одну настройку

### Pickup places

- `GET /pickup-places`
- `POST /pickup-places`
- `PATCH /pickup-places/{id}`
- `DELETE /pickup-places/{id}`

### Templates

- `GET /templates/excel` — скачать текущий server-side Excel template
- `POST /templates/excel` — загрузить новый Excel template

### Debug / diagnostics

- `POST /debug/test-parser`
- `POST /debug/test-match`
- `GET /debug/status`
- `GET /debug/project-state`

### Spam / moderation

- `GET /spam/flagged`
- `GET /spam/stats`
- `POST /spam/dismiss`
- `POST /spam/delete`

## Integration / External (service-key)

- `GET /integrations/capabilities` — discovery endpoint для внешних сервисов
- `GET /integrations/chats` — список чатов
- `GET /integrations/catalogs/open` — открытые каталоги
- `GET /integrations/pickup-places` — точки выдачи
- `GET /integrations/orders` — заказы для внешней синхронизации

## Что делать разработчику дальше

Если нужно быстро понять endpoint:

1. Открой этот manual.
2. Найди нужную группу route-ов.
3. Перейди в `admin-web/src/api/apiCatalog.ts` и найди descriptor по `id` / `path`.
4. Если нужен практический пример фронтового вызова — смотри `admin-web/src/api/client.ts`.
5. Если нужен расширенный контекст, ограничения RBAC и edge cases — смотри [[14-api-reference]].

Для auth/integration hardening и операционного запуска смотри также [[24-admin-auth-and-integration-security-runbook]].

## Что здесь намеренно НЕ делается

- нет live-вызовов запросов из UI;
- нет smoke-run внутри интерфейса сайта;
- нет зависимости от owner-экрана для чтения API другими разработчиками.

Это именно документация, а не рабочий экран продукта.

### [[PRODUCTION_ROADMAP]]
### [[18-integration-overview]]
