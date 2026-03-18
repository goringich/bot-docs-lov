# Architecture (v2)

## Цели
- надежная обработка большого потока сообщений
- идемпотентность и корректная обработка edited_message
- хранение всей истории
- экспорт Excel по раздаче
- возможность расширения (20+ чатов, админ GUI позже)

## Компоненты

### 1) Multi-provider gateway (webhook)
- принимает updates от Telegram, Matrix, VK, MAX и custom webhook
- валидирует ingress secret (provider-specific headers)
- применяет provider-specific parsers (Matrix events, VK Callback API, MAX Bot API)
- сохраняет normalized envelope в БД (tg_updates)
- отвечает быстро (200 OK)
- не делает тяжелый парсинг синхронно

**Endpoints:**
- `POST /telegram/webhook` — Telegram-specific ingress
- `POST /integrations/{provider}/webhook` — универсальный multi-provider ingress
- `GET /capabilities` — API discovery + provider capability matrix

### 2) Worker (update processor)
- забирает необработанные tg_updates
- дедуплицирует обработку
- определяет провайдера из envelope (Telegram/Matrix/VK/MAX/Webhook)
- устанавливает provider context (contextvars)
- сохраняет снимок сообщения (message_snapshots)
- классифицирует: заказ/не-заказ
- если заказ:
  - парсит поля (имя, номер, пункт выдачи, позиции)
  - применяет стопы
  - обновляет агрегированное состояние заказа
  - отправляет ответ через AdapterManager (автоматический routing по провайдеру)

**Adapter layer (`adapters/`):**
- `AdapterManager` — единая точка dispatch для всех egress-действий
- `BridgeTransport` — HTTP-транспорт для non-Telegram провайдеров
- `ProviderCapabilities` — декларативный профиль возможностей каждого провайдера
- Provider-specific parsers (Matrix, VK, MAX) для ingress-нормализации

### 3) Admin commands (MVP)
- админ управляет через личку с ботом:
  - создание раздачи
  - загрузка каталога
  - старт/закрытие
  - стопы
  - экспорт Excel
- доступ только для whitelisted tg_user_id

### 3.1) Admin API (RBAC)
- отдельный сервис `admin_service`
- JWT‑аутентификация
- роли/уровни доступа (owner/admin/manager и т.д.)
- служит backend’ом для web‑панели
- роутинг разнесён по модулям (`app/api/routers/*`)

### 3.2) Admin Web
- React + Vite + MUI
- интерфейс управления пользователями/ролями/данными
- использует Admin API

### 4) Storage (MySQL-тип)
- хранит:
  - tg_updates (raw updates)
  - message_snapshots (текущее состояние сообщения)
  - round, catalog, stop state
  - вклад сообщений в заказ
  - агрегированные заказы

### 5) Excel exporter
- строит файл Excel по раздаче на основе БД
- колонки товаров формируются из каталога раздачи
- отправляет Excel админу в личку

### 6) Admin web flow
Web‑панель → Admin API → БД → отчёты/данные

## Потоки данных

### Пользовательское сообщение
Telegram -> webhook -> tg_updates(new) -> worker -> parse/aggregate -> ответ в чат

### edited_message
Telegram -> webhook -> tg_updates(new) -> worker -> hash check -> replace contrib -> пересчет -> ответ в чат

### Экспорт
Админ (личка) -> команда /export -> exporter -> файл -> админ

## Идемпотентность
- tg_updates: unique update_id
- message_snapshots: unique (chat_id, message_id) + hash
- вклад сообщения в заказ: unique (round_id, chat_id, message_id)

## Микросервисные границы (v2)

Сервисные границы выделены по зонам ответственности:

1. **backend (Telegram API + Orders)**
  - Webhook приём, запись в `tg_updates`, доменная логика заказа
2. **worker (Update Processor)**
  - Асинхронная обработка, парсинг, создание заказов, ответы пользователю
3. **admin_service (RBAC API)**
  - JWT, роли, настройки бота, CRUD точек выдачи
4. **admin-web (UI)**
  - Интерфейс админов и аналитики

Рекомендуемые документы:
- [[architecture/microservices]]

### Ownership таблиц

- `tg_updates`, `message_snapshots`, `orders`, `order_lines` → backend/worker
- `bot_settings`, `pickup_places` → admin_service
- `catalogs`, `catalog_items` → shared (backend создаёт, admin_service управляет)
- `admin_users`, `roles`, `permissions` → admin_service only

## Новые возможности (v2.1)

### Auto-provision
При первом вызове `/open_app` система автоматически создаёт admin_user:
- Первый пользователь получает роль `owner`
- Последующие — роль `admin`
- Данные берутся из Telegram профиля (username, first_name)

### Auto-unstop
Worker каждые ~60 секунд проверяет `catalog_items` с истёкшим `stop_until`:
- Если `stop_until < NOW()` → `is_active=1`, стоп сбрасывается
- Позиция снова доступна для заказов

### CRUD через UI
Admin-web теперь поддерживает полный CRUD:
- Каталоги: создание, редактирование статуса, удаление
- Товары: добавление, редактирование, stop/unstop
- Пользователи: создание, назначение ролей, деактивация

## Новые возможности (v2.2)

### Fuzzy Matching (Levenshtein)
Парсер заказов поддерживает нечёткое сопоставление товаров:
- Расстояние Левенштейна ≤ 2 символа
- "троска" → "Треска филе"
- "минай" → "Минтай филе"

### Smart Preprocessing
Заказы в формате "всё в одну строку" автоматически разбиваются:
- Текст: "Игорь 2872 бор троска 2кг"
- Разбивается по известным точкам выдачи

### ML Mode (disabled)
- OpenAI API отключен из-за лимитов/стоимости
- Заготовка для локальной ML модели (ML_MODE_LOCAL)
- По умолчанию ML_MODE_DISABLED — только regex

### Debug Tools (owner only)
Admin-web включает страницу Debug:
- **Parser Tester**: тестирование парсера заказов
- **Catalog Matcher**: проверка fuzzy matching
- **System Status**: статус БД, worker, Telegram

## Hardening (v2.3)

### Config Caching
- `load_config()` — загрузка из .env (дорогая операция)
- `get_config()` — кэшированный singleton (используется всеми компонентами)
- Устраняет ~20 перечитываний .env за каждый update

### Security Improvements
- **Webhook**: constant-time `hmac.compare_digest()` для проверки secret token
- **Telegram**: автоматическое обрезание сообщений до 4096 символов
- **Docker**: non-root users, multi-stage builds, parameterized secrets
- **CORS**: явный список методов/заголовков вместо wildcard `"*"`
- **API validation**: allow-list для ключей настроек, 404 для несуществующих ресурсов

### Deprecation Cleanup
- `datetime.utcnow()` → `datetime.now(UTC).replace(tzinfo=None)` (Python 3.12+)
- `asyncio.get_event_loop()` → прямой вызов regex (sync) или `asyncio.run()` (async)
- `@app.on_event("startup")` → `asynccontextmanager` lifespan

### [[04-data-model]]