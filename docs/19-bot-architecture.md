# 19. Архитектура бота и адаптация под внешние мессенджеры

Дата: 2026-03-18

Документ фиксирует текущую архитектуру Telegram-бота и описывает модель его адаптации под внешние мессенджеры (Matrix / Synapse / Maubot, VK, MAX).

## Краткое резюме

Это не "бот в одном процессе", а связка из трёх контуров с общей БД:

- `backend` принимает входящие события, сохраняет их в очередь обновлений и обрабатывает доменную логику заказа;
- `admin_service` даёт отдельный API для ролей, настроек, заказов, выдачи и экспортов;
- `admin-web` даёт веб-интерфейс для операционной работы команды.

Сильная сторона текущей реализации в том, что Telegram уже не является ядром продукта. Ядро находится в worker-пайплайне, модели заказа и adapter-слое. Это позволяет расширять систему на Matrix/VK/MAX без переписывания бизнес-логики с нуля.

## Назначение документа

Документ отвечает на четыре вопроса:

1. из каких компонентов состоит текущий бот;
2. какие технологии и библиотеки используются;
3. какие модули и пользовательские сценарии уже реализованы;
4. какие части переиспользуются при адаптации под новые провайдеры, а какие требуют нового transport-слоя.

## Архитектурное резюме

Текущий бот уже работает как **domain-first система с отдельным multi-provider integration layer**:

- ядро заказов, парсер, модели данных и административные сервисы переиспользуются независимо от провайдера;
- входящий transport нормализуется в общий envelope и единый worker pipeline;
- исходящие действия маршрутизируются через capability-driven adapter layer;
- Telegram остаётся fully-native production transport, а Matrix / VK / MAX / generic webhook подключаются через provider bridge.

Коротко: **логика переиспользуется, transport реализуется per-provider поверх общего integration contract**.

## Что на чём написано

### Бот и backend

- Язык: `Python`
- HTTP/runtime: `FastAPI`, `uvicorn`
- ORM и SQL-слой: `SQLAlchemy`
- Миграции: `Alembic`
- База данных: `MySQL`-совместимая схема
- Интеграционные HTTP-вызовы: `httpx`
- Excel-экспорт: `openpyxl`
- Конфиг и валидация: `pydantic`, `python-dotenv`
- Опциональный AI-слот: `openai` SDK, но основной production path сейчас не на нём

### Admin API

- Язык: `Python`
- API: `FastAPI`
- ORM: `SQLAlchemy`
- Аутентификация: `python-jose`
- Пароли: `passlib[bcrypt]`, `bcrypt`
- Валидация: `pydantic`

### Admin web

- Язык: `TypeScript`
- UI: `React 18`
- Сборка: `Vite`
- Компоненты: `MUI`, `@emotion/*`
- HTTP-клиент: `axios`
- Графики: `recharts`
- Работа с Excel на фронте: `xlsx`
- Тесты: `Vitest`, `Testing Library`, `Playwright`

## Технологический стек

### Backend бота

- Язык: `Python`
- Web/API слой: `FastAPI`
- ORM / SQL: `SQLAlchemy`
- Миграции: `Alembic`
- База данных: `MySQL`-совместимая схема
- HTTP-клиенты:
  - `httpx` — для части внешних вызовов и межсервисной логики;
  - собственные safe wrappers поверх Telegram Bot API;
  - в safe Telegram client также используется стандартный `urllib`-контур
- Excel-экспорт: `openpyxl`
- Валидация/схемы: `pydantic`
- Runtime: `uvicorn`
- AI-слот: опциональный, по умолчанию выключен; основной production path — regex/domain logic

### Административный контур

- `admin_service`: `FastAPI` + `SQLAlchemy` + `python-jose` + `passlib/bcrypt`
- `admin-web`: `React 18` + `TypeScript` + `Vite` + `MUI` + `axios` + `recharts`

### Краткая таблица «технология → где используется»

| Технология/библиотека | Где используется | Назначение |
|---|---|---|
| `FastAPI` | `backend`, `admin_service` | HTTP API, webhook ingress, операционные endpoint-ы |
| `SQLAlchemy` | `backend`, `admin_service` | ORM/SQL доступ к MySQL-модели |
| `Alembic` | `backend/migrations` | миграции схемы БД |
| `httpx` | `backend` | внешние HTTP-вызовы и bridge dispatch |
| `openpyxl` | `admin_service` | генерация XLSX export-ов |
| `pydantic` | `backend`, `admin_service` | модели запросов/ответов и config |
| `python-jose` | `admin_service` | JWT auth |
| `passlib/bcrypt` | `admin_service` | парольная аутентификация |
| `React + TypeScript + Vite + MUI` | `admin-web` | UI админки и operator workflows |

## Общая схема компонентов

```text
Telegram / Matrix / VK / MAX / Webhook
    ↓
POST /telegram/webhook
или
POST /integrations/{provider}/webhook
    ↓
backend/app/main.py
    ↓
integrations.py + provider parsers
    ↓
таблица tg_updates
    ↓
worker.py
    ↓
parser + order_processing + repo/domain/presenter
    ↓
AdapterManager + capability profiles + provider bridge
    ↓
orders / order_lines / chats / pickup_places / user_profiles
    ↓
admin_service (REST API)
    ↓
admin-web (операционная админка)
```

## Верхнеуровневая внутренняя архитектура

С точки зрения runtime-контуров бот работает следующим образом:

1. Webhook не выполняет тяжёлую бизнес-логику.
Он принимает update, проверяет секрет, нормализует payload и быстро сохраняет его в `tg_updates`.

2. Основная обработка выполняется в worker.
Именно worker разбирает, что это было: команда, callback, обычное сообщение, заказ, edited message, спам или событие от внешнего провайдера.

3. Заказ обрабатывается как доменный объект, а не как Telegram message.
Система выделяет клиента, телефон, точку выдачи, строки заказа, матчинг по каталогу и итоговый статус заказа.

4. Формирование ответа отделено от обработки заказа.
Presenter/UI-слой решает, как показать результат: короткий ответ, детализацию, клавиатуру, реакцию, запрос недостающих данных.

5. Отправка ответа отделена от бизнес-логики.
`AdapterManager` и provider adapters превращают каноническое действие в конкретный вызов Telegram, Matrix, VK, MAX или bridge-транспорта.

По сути это уже не Telegram-specific схема, а order-processing архитектура с несколькими transport-слоями.

## Runtime-компоненты

### 1. Multi-provider webhook gateway

Файл/контур:

- `backend/app/main.py`

Что делает:

- принимает входящий Telegram update по `POST /telegram/webhook`;
- принимает provider-agnostic ingress по `POST /integrations/{provider}/webhook`;
- валидирует provider-specific secret headers;
- вызывает provider-specific parser для Matrix / VK / MAX / webhook payload;
- кладёт normalized envelope в таблицу `tg_updates`;
- быстро отвечает `{"ok": true}`;
- не выполняет тяжёлую бизнес-логику синхронно.

Для Matrix / Synapse / Maubot это означает, что адаптация строится не вокруг переписывания домена, а вокруг connector-а, который отправляет room events в этот ingress-контур.

### 2. Worker / update processor

Файл/контур:

- `backend/app/worker.py`

Что делает:

- выбирает новые записи из `tg_updates`;
- дедуплицирует/маркирует обработку;
- сохраняет `message_snapshots`;
- распознаёт команды, callback events, обычные сообщения и заказы;
- запускает парсер и доменную обработку;
- создаёт/обновляет `orders` и `order_lines`;
- отправляет ответы через provider-aware dispatch;
- применяет антиспам, rate limit, distribution mode, реакции, reply-mode DM/group.

Это центральный оркестратор. Его бизнес-ветки общие, а provider-specific поведение вынесено в integration layer.

### 3. Parser + domain

Основные зоны:

- `backend/app/parser/`
- `backend/app/domain/`
- `backend/app/order_processing.py`

Что делает:

- определяет, похоже ли сообщение на заказ;
- извлекает имя, телефон, точку выдачи, позиции заказа;
- матчит позиции с каталогом;
- классифицирует строки (`ok`, `unknown_item`, `bad_qty`, `stopped` и т.д.);
- собирает итоговый статус заказа (`active`, `partial`, `needs_admin`, `canceled`, ...).

Это **главная reusable-часть** для Matrix-переноса.

### 4. Presenter / bot UI layer

Основные зоны:

- `backend/app/presenter/`
- `backend/app/bot_ui.py`
- `backend/app/handlers/`

Что делает:

- формирует тексты ответов;
- строит inline-клавиатуры и меню;
- поддерживает шаблоны `/help`, `/order_template`, `/profile`, `/pickup_points`;
- читает bot UI settings из БД.

Что можно переиспользовать:

- тексты, шаблоны, бизнес-формулировки;
- часть логики сценариев.

Что нужно переписать для Matrix:

- Telegram inline-keyboards и callback behavior;
- способ отображения меню и быстрых действий;
- возможные реакции и reply mechanics.

### 5. Integration layer / transport layer

Основные зоны:

- `backend/app/telegram_client.py`
- `backend/app/adapters/capabilities.py`
- `backend/app/adapters/bridge_transport.py`
- `backend/app/adapters/manager.py`
- `backend/app/adapters/parsers.py`
- `backend/app/adapters/*_adapter.py`
- `backend/app/provider_bridge.py`

Что делает:

- для Telegram: native `sendMessage`, `sendDocument`, `answerCallbackQuery`, `setMessageReaction`, `deleteMessage`;
- для Matrix / VK / MAX / webhook: bridge-based dispatch тех же canonical actions;
- объявляет capability profile каждого провайдера (reactions, inline actions, edits, deletes, files);
- парсит provider-native ingress payload и приводит его к canonical event shape;
- изолирует vendor-specific transport от order-domain.

Именно этот слой адаптируется под новый провайдер; domain logic при этом остаётся общей.

## Внутренняя модульная архитектура

### Message intake и маршрутизация

- webhook записывает update в БД;
- worker достаёт update;
- worker определяет тип входящего события;
- вызов уходит либо в public/admin command flow, либо в order processing.

### Основные модули

- `main.py` — intake endpoint и discovery;
- `worker.py` — orchestration loop;
- `handlers/public_commands.py` — пользовательские команды;
- `handlers/admin_commands.py` — админские команды;
- `handlers/callback_handler.py` / `admin_actions.py` — callback actions;
- `services/telegram_service.py` — безопасный клиент Telegram API;
- `services/spam_detector.py` — антиспам;
- `services/rate_limiter.py` — ограничение исходящих отправок;
- `order_processing.py` — доменная обработка заказа;
- `domain/` и `parser/` — правила распознавания;
- `repo/` — доступ к данным и matching каталога;
- `presenter/` и `bot_ui.py` — тексты, форматирование, кнопки.

## Что реализовано по модулям

### Контур приёма сообщений

- `backend/app/main.py`
- `backend/app/integrations.py`
- `backend/app/provider_bridge.py`
- `backend/app/adapters/parsers.py`

Реализует:

- Telegram webhook;
- универсальный `/integrations/{provider}/webhook`;
- нормализацию событий от `telegram`, `matrix`, `vk`, `max`, `webhook`;
- capability discovery через `/capabilities`.

### Контур обработки заказов

- `backend/app/worker.py`
- `backend/app/order_processing.py`
- `backend/app/parser/text_parser.py`
- `backend/app/domain/order_domain.py`
- `backend/app/repo/order_repo.py`
- `backend/app/repo/catalog_repo.py`
- `backend/app/repo/pickup_repo.py`

Реализует:

- распознавание заказа из свободного текста;
- выделение клиента, телефона, точки выдачи и товарных строк;
- fuzzy matching по товарам;
- учёт стопов и временных стопов;
- пересборку заказа при edited message;
- агрегацию заказа и его строк в БД.

### Пользовательский bot UX

- `backend/app/handlers/public_commands.py`
- `backend/app/bot_ui.py`
- `backend/app/presenter/order_presenter.py`
- `backend/app/handlers/callback_handler.py`

Реализует:

- публичные команды;
- пользовательское меню и кнопки;
- показ последнего заказа;
- профиль клиента;
- подсказки по формату заказа;
- сценарии добора недостающих данных через callbacks.

### Админский bot UX

- `backend/app/admin_commands.py`
- `backend/app/handlers/admin_actions.py`

Реализует:

- операционные команды по каталогу и точкам выдачи;
- открытие web-админки через `/open_app`;
- ограничение чувствительных команд в Telegram и перенос их в web-контур.

### Внешние провайдеры и egress

- `backend/app/adapters/manager.py`
- `backend/app/adapters/telegram_adapter.py`
- `backend/app/adapters/matrix_adapter.py`
- `backend/app/adapters/vk_adapter.py`
- `backend/app/adapters/max_adapter.py`
- `backend/app/adapters/webhook_adapter.py`
- `backend/app/adapters/bridge_transport.py`
- `backend/app/adapters/capabilities.py`

Реализует:

- единый dispatch исходящих действий;
- маршрутизацию в Telegram или bridge-транспорт;
- capability profile для каждого провайдера;
- возможность подключать новые transports без переписывания order domain.

### Карта модулей для быстрой оценки scope

| Зона | Ключевые файлы/пакеты | Что делает |
|---|---|---|
| Ingress | `main.py`, `integrations.py` | приём webhook/event и упаковка в envelope |
| Оркестрация | `worker.py` | routing update/callback/order flow |
| Parser/Domain | `parser/*`, `domain/*`, `order_processing.py` | распознавание заказа и бизнес-статусы |
| Transport | `telegram_client.py`, `services/telegram_service.py`, `adapters/*` | отправка/получение provider сообщений |
| UI/Presenter | `presenter/*`, `bot_ui.py`, `handlers/*` | тексты, команды, кнопки, reply UX |
| Admin API | `admin_service/app/api/routers/*` | роли, каталоги, заказы, выдача, экспорт |

## Функциональные сценарии бота

### Пользовательские команды

Ключевые user-facing сценарии:

- `/start`, `/help`, `/menu`
- `/profile`, `/me`
- `/my_order`
- `/order_template`, `/order_example`, `/order_rules`
- `/pickup_points` / `/pickup_places` / `/where_pickup`
- `/set_pickup <точка>`
- `/set_phone <4 цифры>`
- `/add_to_order <позиции>`
- `/cancel_order`
- `/open_app` — для админского входа в web-панель

Примечание: это список уже реализованных сценариев Telegram-runtime. При переносе на Matrix команды и интерактивность маппятся в provider-native UX, а не копируются 1-в-1 на transport уровне.

### Админские команды в боте

Исторически в боте есть контур команд типа:

- `/catalog_open`, `/catalog_close`
- `/item_add`, `/item_alias`, `/item_list`, `/item_stop`
- `/pickup_add`, `/pickup_list`, `/pickup_disable`
- `/menu_admin`
- `/help`, `/start`, `/whoami`

Важно: по текущей security-политике проекта чувствительные данные, аналитика и mass-export сознательно смещены в `admin-web` / `admin_service`. То есть Telegram-бот больше не считается основным operational-интерфейсом для data-rich действий.

## Вывод для архитектурной постановки

Проект следует рассматривать не как Telegram-бот в узком смысле, а как прикладную платформу для обработки заказов, у которой Telegram был первой точкой входа. Архитектурно в системе уже отделены transport-слой, worker-пайплайн, доменная логика заказа и административный контур. Основной технический фокус на следующем этапе должен быть не в переписывании ядра, а в укреплении границ между `backend`, `admin_service` и `admin-web`, чтобы чувствительные операции не возвращались в чатовый контур и чтобы дальнейшее расширение на новые провайдеры не ломало доменную модель.

## Ответ на вопрос «почему прямой перенос в Matrix невозможен» (коротко)

Потому что Telegram и Matrix различаются одновременно по трём слоям:

1. **Event model**: разные структуры входящих событий, id и жизненный цикл комнат/сообщений.
2. **Interaction model**: Telegram callback/inline UX не эквивалентен Matrix room event relations.
3. **Transport/Auth model**: разные протоколы, подписи/секреты, media/send/delete/reaction semantics.

Поэтому перенос выполняется как «reuse domain core + новый provider adapter/bridge», а не как прямой copy-paste Telegram API-клиента.

## Архитектурная готовность к мультимессенджерности

В проекте уже реализован multi-provider adapter layer:

- `backend/app/adapters/base.py`
- `backend/app/adapters/capabilities.py`
- `backend/app/adapters/bridge_transport.py`
- `backend/app/adapters/manager.py`
- `backend/app/adapters/parsers.py`
- `backend/app/adapters/telegram_adapter.py`
- `backend/app/adapters/matrix_adapter.py`
- `backend/app/adapters/vk_adapter.py`
- `backend/app/adapters/max_adapter.py`
- `backend/app/adapters/webhook_adapter.py`

Там уже описаны и используются в runtime:

- `IncomingMessage`
- `OutgoingMessage`
- `OutgoingReaction`
- `MessengerAdapter`
- `ProviderCapabilities`
- `AdapterManager`
- provider-specific ingress parsers.

Это означает, что архитектурно проект уже готов к появлению нового провайдера без переписывания worker-domain слоя: новый connector добавляется как `adapter + parser + capability profile + bridge mapping`.

## Переиспользование при переносе на внешний провайдер

### Переиспользуется почти без изменений

- схема БД;
- модели заказов и строк заказа;
- каталоги и точки выдачи;
- парсер и матчер каталога;
- доменные правила статусов;
- антиспам-логика (при необходимости адаптации сигнатур событий);
- `admin_service` и `admin-web`;
- экспорт, аналитика, ACL, delivery flow.

### Потребует адаптации

- источник событий (Telegram update → provider event);
- идентификаторы чатов/комнат/пользователей/сообщений;
- команды и UX меню;
- reply / edit / reaction semantics;
- deep-link и one-time login сценарии из чата;
- обработка inline buttons / callbacks;
- DM/group semantics — различаются по платформам.

### Потребует полной новой реализации

- provider-specific authorization / lifecycle;
- membership / room/dialog handling;
- отправка сообщений и media через API провайдера;
- provider-specific action UI вместо Telegram inline-buttons.

## Почему прямой перенос невозможен

Потому что текущая логика взаимодействует не только с текстом сообщения, но и с Telegram-specific primitives:

- webhook payload schema;
- callback queries;
- inline keyboards;
- message reactions;
- reply-to-message semantics;
- deep links и `/command` model;
- Bot API error model и ограничения.

Любой внешний провайдер работает по другой модели:

- свой event model;
- свой lifecycle бота;
- свой способ работы с кнопками, интерактивностью и редактированием;
- свой transport и auth protocol.

Именно поэтому правильная оценка — **не “перенос бота”, а “замена transport/adaptation слоя с сохранением core domain”**.

## Оценка объёма работ по слоям

### Низкий риск / высокий reuse

- parser/domain/order processing;
- DB schema;
- admin_service API;
- admin-web;
- delivery / exports / analytics.

### Средний риск

- нормализация identity model для Matrix users/rooms;
- mapping команд и пользовательских сценариев на Matrix UX;
- переиспользование bot_ui templates в новом interaction model.

### Высокий риск / отдельная реализация

- provider connector layer;
- provider event handling;
- provider-native callback/menu behavior;
- migration path для существующих Telegram-centric assumptions внутри worker.

## Базовые стратегии переноса

### Вариант A — быстрый и прагматичный

Сделать provider adapter, который:

1. получает события провайдера;
2. нормализует их в близкий к `IncomingMessage` формат;
3. вызывает существующий order pipeline;
4. отправляет ответы обратно через API провайдера.

Плюсы:

- максимальный reuse текущего backend;
- быстрее до первого работающего прототипа.

Минусы:

- часть Telegram-centric веток в worker придётся аккуратно изолировать;
- UX будет ограничен тем, насколько удобно конкретный провайдер повторяет Telegram bot flow.

### Вариант B — чище в долгую

Выделить core order engine в более независимый application-service слой, а поверх него делать отдельные bot runtime для каждого провайдера.

Плюсы:

- чище архитектурно;
- проще поддерживать несколько мессенджеров.

Минусы:

- дороже по времени;
- больше рефакторинга на старте.

## Итоговая архитектурная интерпретация

Для оценки и старта работ проект разумно трактовать как:

- **готовое domain ядро**;
- **готовую admin-платформу**;
- **готовый provider-agnostic integration contract**;
- **готовый runtime path для ingress + bridge-based egress**;
- **отдельный vendor-native connector scope**, если нужен полный Matrix/Maubot transport без bridge-слоя.

То есть задача не в переписывании всего продукта, а в выборе конкретного deployment-варианта transport-адаптации:

1. либо использовать уже готовый provider bridge contract;
2. либо реализовать vendor-native connector поверх того же canonical integration layer.

## Связанные документы

- [[20-api-and-business-flows-for-mobile-integration]]
- [[14-api-reference]]
- [[17-admin-service-endpoints-manual]]
- [[03-architecture]]
- [[06-telegram-integration]]
- [[22-messenger-agnostic-integration-spec]]

### [[20-api-and-business-flows-for-mobile-integration]]
