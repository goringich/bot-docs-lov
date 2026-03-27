# Integration & API Reference

> ⚠️ Исторический документ. Актуальный и проверенный список API смотри в [[14-api-reference]] и в OpenAPI каждого сервиса (`/openapi.json`).

> Для M2M-интеграций используйте раздел `/integrations/*` в `admin_service` и ориентируйтесь на `GET /integrations/capabilities`.

> **Версия**: 2.0 | Полная справка по всем endpoint-ам всех сервисов

## Обзор архитектуры

```
┌─────────────────────────────────────────────────────────────┐
│                    EXTERNAL CLIENTS                         │
│  Telegram │ VK │ Max │ Custom App │ Mobile App │ Web Form   │
└─────┬─────┴──┬──┴──┬──┴─────┬──────┴────┬───────┴──────┬────┘
      │        │     │        │           │              │
      ▼        ▼     ▼        ▼           ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│              ADAPTER LAYER (backend/app/adapters/)           │
│  TelegramAdapter │ VKAdapter │ MaxAdapter │ WebhookAdapter  │
│  IncomingMessage ←→ Core Domain ←→ OutgoingMessage          │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                 CORE DOMAIN (backend/app/)                    │
│  parser/ → order_processing → domain/ → repo/ → DB          │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              ADMIN SERVICE (admin_service/) — 70+ endpoints  │
│  FastAPI REST API — JWT Auth — RBAC — Excel Export           │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              ADMIN WEB (admin-web/) — React + MUI + Vite     │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Admin Service API

**Base URL**: `http://localhost:8010`  
**Auth**: browser/admin flow через cookie session или JWT Bearer; M2M flow через отдельный `X-Service-Key`

### 1.1 Аутентификация

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `POST` | `/auth/login` | — | Email + password → JWT |
| `POST` | `/auth/bootstrap` | Bootstrap-токен | Первый owner аккаунт |
| `POST` | `/auth/one-time-link` | `ADMIN_ONE_TIME_KEY` | Одноразовая ссылка |
| `POST` | `/auth/token-login` | — | Обмен одноразового токена → cookie session |
| `POST` | `/auth/auto-provision` | `ADMIN_ONE_TIME_KEY` | Автосоздание из Telegram |

`POST /auth/login` поднимает `HttpOnly` cookie session; JSON содержит только технический статус cookie-session flow и не отдаёт bearer token в JS.

### 1.2 Пользователи

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/me` | JWT | Текущий пользователь |
| `GET` | `/users` | JWT | Список всех пользователей |
| `GET` | `/users/full` | JWT (owner) | **Полные данные** (password_hash, tg_username, timestamps) |
| `POST` | `/users` | JWT (owner/admin) | Создать пользователя |
| `PUT` | `/users/{id}/roles` | JWT (owner/admin) | Обновить роли |
| `PUT` | `/users/{id}/status` | JWT (owner/admin) | Активировать/деактивировать |
| `PUT` | `/users/{id}/password` | JWT (owner) | Сбросить пароль пользователю |
| `POST` | `/users/confirm-owner-role` | JWT (owner) | Назначить owner (с подтверждением пароля) |
| `DELETE` | `/users/{id}` | JWT (owner/admin) | Удалить пользователя |

**Иерархия ролей:**
- `owner` (level 0) — полный доступ, видит password_hash
- `regional_admin` (level 5) — управление в своих регионах
- `admin` (level 10) — каталоги, заказы, аналитика
- `manager` (level 20) — просмотр заказов, экспорт
- `viewer` (level 50) — только чтение

### 1.3 Роли и разрешения

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/roles` | JWT | Список ролей |
| `POST` | `/roles` | JWT (owner) | Создать роль |
| `GET` | `/permissions` | JWT | Список разрешений |

### 1.4 Заказы

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/orders` | JWT | Список (с фильтрами) |
| `GET` | `/orders/{id}` | JWT | Детали + строки |
| `PATCH` | `/orders/{id}` | JWT | Редактирование |
| `GET` | `/orders/export` | JWT | Операционный XLSX |
| `GET` | `/orders/export-template` | JWT | Экспорт по шаблону |
| `GET` | `/orders/export-history` | JWT | Полная история XLSX |

**Query-параметры `/orders`:**
- `catalog_id`, `chat_id`, `status`, `pickup_place` — фильтры
- `search` — полнотекстовый поиск
- `date_from`, `date_to` — диапазон дат (YYYY-MM-DD)
- `limit`, `offset` — пагинация

**PATCH поля (разрешённые):** `status`, `customer_name`, `phone_last4`, `pickup_place`, `raw_text`, `error_text`

### 1.5 Каталоги

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/catalogs` | JWT | Список каталогов |
| `POST` | `/catalogs` | JWT (owner) | Создать каталог |
| `PATCH` | `/catalogs/{id}` | JWT (owner) | Обновить |
| `POST` | `/catalogs/{id}/open` | JWT (owner) | Открыть приём заказов |
| `POST` | `/catalogs/{id}/close` | JWT (owner) | Закрыть |
| `POST` | `/catalogs/{id}/clone` | JWT (owner) | Клонировать в другой чат |
| `GET` | `/catalogs/{id}/items` | JWT | Позиции каталога |
| `POST` | `/catalogs/{id}/items` | JWT (owner) | Добавить позицию |
| `PATCH` | `/catalogs/{id}/items/{itemId}` | JWT (owner) | Обновить позицию |
| `DELETE` | `/catalogs/{id}/items/{itemId}` | JWT (owner) | Удалить |
| `POST` | `/catalogs/{id}/items/{itemId}/stop` | JWT (owner) | Стоп-лист |
| `POST` | `/catalogs/{id}/items/{itemId}/unstop` | JWT (owner) | Убрать из стопа |

### 1.6 Чаты

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/chats` | JWT | Список чатов |
| `PATCH` | `/chats/{tg_chat_id}` | JWT (owner) | Видимость бота |
| `GET` | `/admin-user-chats` | JWT | Привязки админов к чатам |
| `POST` | `/admin-user-chats` | JWT (owner) | Назначить админа на чат |
| `DELETE` | `/admin-user-chats/{id}` | JWT (owner) | Убрать привязку |

### 1.7 Аналитика

Все поддерживают `?chat_id=` для фильтрации по чату.

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/analytics/dashboard` | JWT | Сводные метрики |
| `GET` | `/analytics/orders-chart` | JWT | Заказы по дням (`days`, `chat_id`) |
| `GET` | `/analytics/top-items` | JWT | Топ позиций (`limit`, `chat_id`) |
| `GET` | `/analytics/status-breakdown` | JWT | По статусам (`chat_id`) |
| `GET` | `/analytics/pickup-stats` | JWT | По точкам выдачи (`limit`, `chat_id`) |
| `GET` | `/analytics/catalog-stats` | JWT | По каталогам |
| `GET` | `/analytics/parser-accuracy` | JWT | Точность парсера |

### 1.8 Настройки

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/settings` | JWT | Все настройки бота |
| `POST` | `/settings` | JWT | Обновить (JSON body) |
| `GET` | `/pickup-places` | JWT | Точки выдачи |
| `POST` | `/pickup-places` | JWT | Создать точку |
| `PATCH` | `/pickup-places/{id}` | JWT | Обновить |
| `DELETE` | `/pickup-places/{id}` | JWT | Удалить |

**Ключи настроек:** `reply_mode`, `welcome_message`, `order_success_message`, `order_error_message`, `enable_notifications`, `notify_on_new_order`, `notify_on_needs_admin`, `auto_unstop_enabled`, `auto_close_hours`, `owner_debug_mode`, `feature_*`

### 1.9 Шаблоны

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/templates/excel` | JWT | Скачать шаблон XLSX |
| `POST` | `/templates/excel` | JWT (owner) | Загрузить новый шаблон |

### 1.10 Отладка

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `POST` | `/debug/test-parser` | JWT | Тест парсера на тексте |
| `POST` | `/debug/test-match` | JWT | Тест матчинга |
| `GET` | `/debug/system-status` | JWT | Состояние системы |
| `GET` | `/debug/project-state` | JWT | Полная диагностика |
| `GET` | `/admin-settings/dev-mode` | JWT | Режим разработки |
| `PUT` | `/admin-settings/dev-mode` | JWT (owner) | Вкл/выкл dev-mode |

### 1.11 M2M интеграция (Service-to-Service)

Для внешних сервисов используется отдельный `X-Service-Key`, который должен быть равен `ADMIN_INTEGRATION_SERVICE_KEY`.

Важно:

- `ADMIN_INTEGRATION_SERVICE_KEY` не должен использоваться для `/auth/one-time-link`;
- `ADMIN_ONE_TIME_KEY` не должен использоваться для `/integrations/*`.

| Метод | Endpoint | Auth | Описание |
|-------|----------|------|----------|
| `GET` | `/integrations/capabilities` | X-Service-Key | Карта ручек |
| `GET` | `/integrations/chats` | X-Service-Key | Чаты |
| `GET` | `/integrations/catalogs/open` | X-Service-Key | Активные каталоги |
| `GET` | `/integrations/pickup-places` | X-Service-Key | Точки выдачи |
| `GET` | `/integrations/orders` | X-Service-Key | Заказы (`limit`, `status`, `since_id`) |

---

## 2. Adapter Layer (мультимессенджер)

### 2.1 Структура

```
backend/app/adapters/
├── __init__.py          # Реэкспорты
├── base.py              # MessengerAdapter ABC + IncomingMessage/OutgoingMessage
├── telegram_adapter.py  # Telegram Bot API
└── webhook_adapter.py   # Generic HTTP webhook
```

### 2.2 IncomingMessage (единый формат)

```python
@dataclass
class IncomingMessage:
    adapter_id: str        # "telegram", "vk", "webhook"
    chat_id: int | str
    user_id: int | str
    message_id: int | str
    text: str
    user_name: str | None
    chat_title: str | None
    is_group_chat: bool
    raw_event: Any
```

### 2.3 Добавление нового мессенджера

1. Создать `backend/app/adapters/my_adapter.py`
2. Реализовать `MessengerAdapter` с `@register_adapter`
3. Импортировать в `__init__.py`
4. Добавить конфиг в `.env`

### 2.4 Webhook (для кастомных приложений)

```bash
curl -X POST http://localhost:8000/api/v1/webhook/incoming \
  -H "X-Service-Key: your_key" \
  -d '{"chat_id":"my-store","user_id":"c-123","text":"Сёмга 2кг","callback_url":"https://..."}'
```

---

## 3. Database Schema

| Таблица | Описание |
|---------|----------|
| `orders` | Заказы |
| `order_lines` | Строки заказов |
| `catalogs` | Каталоги (раздачи) |
| `catalog_items` | Позиции каталога |
| `chats` | Чаты мессенджеров |
| `pickup_places` | Точки выдачи |
| `user_profiles` | Профили клиентов |
| `bot_settings` | Настройки бота |
| `admin_users` | Админ-панель пользователи |
| `admin_roles` | Роли |
| `admin_user_roles` | M2M пользователи↔роли |
| `admin_permissions` | Разрешения |
| `admin_settings` | Настройки админки |
| `admin_audit_logs` | Аудит-лог |

---

## 4. Docker Deployment

```bash
docker compose -f infra/docker-compose.yml up -d      # production
docker compose -f infra/docker-compose.dev.yml up -d   # dev + hot reload
```

## 5. Security

- JWT HS256 + bcrypt пароли
- CORS с `CORS_ORIGINS`
- Rate limiting 120 req/min
- Security headers (X-Frame-Options, CSP)
- `.env` never committed — use `.env.example`

### [[14-api-reference]]
