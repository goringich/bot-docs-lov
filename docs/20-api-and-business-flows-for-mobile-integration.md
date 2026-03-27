# 20. Интеграционные API и бизнес-процессы

Дата: 2026-03-18

Документ описывает текущий API-контур проекта и последовательности вызовов, которые важны для mobile-клиента, Matrix-адаптера и любых внешних integration-сценариев.

Для provider-agnostic интеграции (Telegram / Matrix / VK / MAX) дополнительно используется:

- [[22-messenger-agnostic-integration-spec]].

## Назначение документа

Документ отвечает на три практических вопроса:

1. какие API уже реализованы и пригодны для интеграции;
2. в какой последовательности вызываются методы в основных бизнес-сценариях;
3. где проходят текущие границы между готовым read/write контуром и будущими расширениями.

## Интеграционное резюме

В проекте уже есть два стабильных backend-контура:

1. `backend` — intake и обработка сообщений/заказов;
2. `admin_service` — основной REST API для управления данными, каталогами, заказами, выдачей, аналитикой, экспортами и доступами.

Для внешних систем также есть M2M-контур:

- `GET /integrations/capabilities`
- `GET /integrations/chats`
- `GET /integrations/catalogs/open`
- `GET /integrations/pickup-places`
- `GET /integrations/orders`

Важно: **на сегодня M2M-контур ориентирован в первую очередь на чтение**, а не на полноценный customer-facing write flow.

Для интеграционной оценки это означает разделение на два независимых направления:

- **bot/provider integration** — подключение Matrix / FluffyChat к существующему message-driven order pipeline;
- **mobile app integration** — вызовы `admin_service` и `/integrations/*` для экранов, форм, очередей, выдачи и справочников.

## Карта сервисов

### 1. `backend`

Назначение:

- приём внешнего event/message потока;
- постановка событий в очередь БД;
- worker-based обработка;
- создание/обновление заказов по текстовым сообщениям.

Замечание: текущий production ingress реализован для Telegram webhook. Для VK/MAX/Matrix ingress нужен отдельный connector layer по контракту `docs/22...`.

Base URL:

- `http://localhost:8000` локально; в deployment — домен/URL вашего backend.

Ключевые endpoints:

- `GET /health`
- `GET /capabilities`
- `POST /telegram/webhook`

### 2. `admin_service`

Назначение:

- JWT auth и RBAC;
- каталоги, чаты, пользователи, роли;
- заказы и строки заказа;
- delivery / pickup admin flows;
- аналитика;
- экспорты;
- M2M `/integrations/*`.

Base URL:

- `http://localhost:8010` локально; в deployment — домен/URL вашего `admin_service`.

## Как читать API-контур по ролям интеграции

### Если задача — подключить внешний мессенджер к текущему order flow

Ключевые документы и ручки:

- [[22-messenger-agnostic-integration-spec]]
- [[23-provider-bridge-runbook]]
- `POST /integrations/{provider}/webhook`
- `GET /capabilities`

Нужно понимать:

- как ingress отдать событие в backend;
- как принять outbound action обратно;
- какие capabilities поддерживаются per provider.

### Если задача — строить mobile UI поверх существующего backend

Ключевые документы и ручки:

- [[14-api-reference]]
- [[17-admin-service-endpoints-manual]]
- `GET /integrations/*`
- `POST /auth/login`, `GET /me`
- `GET/PATCH /orders*`, `GET/POST/PATCH/DELETE /deliveries*`

Нужно понимать:

- где читать каталоги/чаты/точки выдачи;
- где брать operational order queue;
- какие операции требуют JWT, а какие доступны через service key.

## Режимы авторизации

### JWT

Используется для административного и операционного UI/API.

Основные ручки:

- `POST /auth/login`
- `GET /me`

### Service key (`X-Service-Key`)

Используется для внешних систем и межсервисных read-oriented интеграций.

Основные ручки:

- `GET /integrations/capabilities`
- `GET /integrations/chats`
- `GET /integrations/catalogs/open`
- `GET /integrations/pickup-places`
- `GET /integrations/orders`

### Step-up password confirmation

Используется для чувствительных export operations.

Flow:

1. `POST /auth/confirm-password`
2. получить confirmation token;
3. передать его в `X-Password-Confirm-Token` для export route.

## Текущее покрытие API

### Стабильно доступно

- логин операторов / админов;
- читать список чатов;
- читать открытые каталоги;
- читать точки выдачи;
- читать список заказов;
- читать и редактировать заказы в admin JWT-flow;
- выполнять delivery operations;
- строить XLSX exports;
- управлять каталогами, ролями и пользователями.

### Важное ограничение

Сейчас проект **не выделяет отдельный внешний public API для customer-side создания заказа из native mobile-формы** как самостоятельный стабильный контракт.

Сегодня заказ создаётся в production в основном через:

- текстовое сообщение пользователя в боте;
- далее — parser + domain pipeline.

Поэтому при проектировании FluffyChat/mobile сценариев полезно разделять два класса экранов:

- **read + operator screens**, которые уже можно строить на существующем API;
- **customer native order screens**, для которых нужен либо bot-adapter, либо отдельный write contract.

Это означает, что для Matrix / FluffyChat есть два реалистичных варианта.

## Базовые стратегии интеграции mobile / Matrix

Стратегии эквивалентны и для других провайдеров (VK / MAX):

- либо reuse message-driven order pipeline через provider adapter;
- либо отдельный public write API для native customer forms.

### Стратегия A — reuse текущего order pipeline

Мобильный/Matrix слой собирает ввод пользователя и отправляет его в формате, эквивалентном сообщению боту.

Дальше существующий parser/domain:

- извлекает поля;
- создаёт заказ;
- классифицирует строки;
- возвращает статус.

Плюсы:

- быстрее к MVP;
- максимальное переиспользование текущей логики.

Минусы:

- UX формы живёт поверх текстового доменного ввода;
- часть логики по-прежнему опирается на message/event model.

### Стратегия B — отдельный write API для native mobile forms

Нужно спроектировать новый API-контракт для прямого создания/обновления заказа из формы.

Плюсы:

- чище mobile UX;
- меньше зависимости от текстового ввода.

Минусы:

- отдельный backend scope;
- потребуются новые публичные DTO, валидации и security rules.

## Архитектурная интерпретация текущего API-контура

Если речь идёт о Matrix + FluffyChat и нужен быстрый старт, текущий API-контур лучше интерпретировать так:

- **чтение справочников и operational data уже готово**;
- **операторский/admin write flow уже готов через JWT API**;
- **customer/mobile native create-order flow требует либо адаптера, либо нового write API**.

## Основные бизнес-процессы

Ниже — не просто список endpoint-ов, а последовательности, в которых реально вызываются методы.

## Краткая карта сценариев «экран → API»

| Экран/сценарий | Минимальный набор вызовов | Комментарий |
|---|---|---|
| Старт приложения / discovery | `GET /integrations/capabilities`, `GET /integrations/chats` | определяет доступные чаты/города и contract version |
| Каталог / справочники | `GET /integrations/catalogs/open`, `GET /integrations/pickup-places` | read-only данные для mobile UI |
| Login staff-пользователя | `POST /auth/login`, `GET /me` | включает operator/admin scope |
| Очередь заказов | `GET /orders`, `GET /orders/{id}` | JWT-only operational flow |
| Редактирование заказа | `PATCH /orders/{id}`, line CRUD | для модерации и исправлений |
| Выдача | `GET /pickup-admin/my-points`, `POST /deliveries`, `GET/PATCH /deliveries/{id}` | workflow точки выдачи |
| Экспорт / отчёты | `POST /auth/confirm-password`, `GET /orders/export*`, `POST /exports/build` | owner-controlled routes |
| Provider intake | `POST /integrations/{provider}/webhook` | для Matrix/VK/MAX bridge |

## Базовый сценарий мобильной интеграции (пошаговый)

Ниже минимальный практический сценарий, если мобильный клиент стартует с read-интеграции и operator flow.

### Шаг 1. Discovery и справочники

1. `GET /integrations/capabilities`
2. `GET /integrations/chats`
3. `GET /integrations/catalogs/open`
4. `GET /integrations/pickup-places?tg_chat_id=<chat>`

Результат:

- приложение знает, какие города/чаты доступны;
- понимает активные каталоги;
- может отрисовать точки выдачи.

### Шаг 2. Staff-auth (если мобильный клиент для операторов)

1. `POST /auth/login`
2. `GET /me`

Результат:

- получаем роль и scope;
- включаем нужные экраны и доступные действия.

### Шаг 3. Queue и карточки заказов

1. `GET /orders?...` (с фильтрами)
2. `GET /orders/{id}`

Результат:

- список заказов для очереди;
- detail для редактирования/выдачи.

### Шаг 4. Редактирование проблемного заказа

1. `PATCH /orders/{id}`
2. при необходимости line CRUD:
   - `POST /orders/{id}/lines`
   - `PATCH /orders/{id}/lines/{line_id}`
   - `DELETE /orders/{id}/lines/{line_id}`

Результат:

- заказ возвращается в рабочий статус без ручного SQL/админки.

### Шаг 5. Выдача

1. `GET /pickup-admin/my-points`
2. `POST /deliveries`
3. `GET /deliveries` / `PATCH /deliveries/{id}` при корректировках

Результат:

- мобильный оператор фиксирует факт выдачи и откаты при ошибках.

---

## Flow 1. Техническое discovery внешней системы

Цель: проверить доступность сервиса и понять, какие integration route-ы доступны.

### Последовательность

1. `GET /integrations/capabilities`
2. `GET /integrations/chats`
3. `GET /integrations/catalogs/open`
4. `GET /integrations/pickup-places?tg_chat_id=...`

### Что получает интеграционный клиент

- версию M2M-контракта;
- список рабочих чатов/городов;
- активные каталоги по чатам;
- точки выдачи по чату.

### Типовые случаи использования

- при старте мобильного клиента;
- при старте Matrix bridge / Maubot plugin;
- для периодического refresh справочников.

---

## Flow 2. Логин администратора / оператора

Цель: дать staff-user доступ к operational API.

### Последовательность

1. `POST /auth/login`
2. принять cookie session
3. `GET /me`
4. в зависимости от роли открыть нужные экраны и endpoints

### Роли, важные для интеграции

- `owner` — полный доступ;
- `regional_admin` — управление в рамках своих регионов;
- `manager` — модерация заказов;
- `pickup_admin` — выдача по назначенным точкам.

### Когда использовать

- если mobile app включает operator/staff сценарии;
- если требуется нативная административная работа не через web.

---

## Flow 3. Загрузка справочников для формы заказа

Цель: показать пользователю каталог и точки выдачи.

### Если используется M2M read API

1. `GET /integrations/chats`
2. `GET /integrations/catalogs/open`
3. `GET /integrations/pickup-places?tg_chat_id=...`

### Если используется JWT admin flow

1. `GET /chats`
2. `GET /catalogs?status=open`
3. `GET /catalogs/{catalog_id}/items`
4. `GET /pickup-places`

### Примечание

M2M-контур уже даёт нужный минимум для внешнего клиента, но если требуется полный состав каталога с item details и сложными операциями, сейчас удобнее использовать JWT admin API.

---

## Flow 4. Показ списка заказов оператору

Цель: отрисовать order queue / moderation list / delivery queue.

### Последовательность

1. `POST /auth/login`
2. `GET /me`
3. `GET /orders?chat_id=...&status=...&pickup_place=...&search=...`
4. при открытии карточки: `GET /orders/{order_id}`

### Важные параметры фильтрации

- `catalog_id`
- `chat_id`
- `status`
- `pickup_place`
- `search`
- `date_from`
- `date_to`
- `limit`
- `offset`

### Что получает клиент

- order list для таблицы/ленты;
- detail с raw text, строками заказа и operational metadata.

---

## Flow 5. Ручная модерация и исправление заказа

Цель: поправить заказ, если парсер не справился идеально.

### Последовательность

1. `GET /orders/{order_id}`
2. `PATCH /orders/{order_id}` — metadata заказа
3. при необходимости:
   - `POST /orders/{order_id}/lines`
   - `PATCH /orders/{order_id}/lines/{line_id}`
   - `DELETE /orders/{order_id}/lines/{line_id}`
4. повторный `GET /orders/{order_id}` или refresh списка

### Что можно редактировать

- статус заказа;
- имя клиента;
- последние 4 цифры телефона;
- точку выдачи;
- raw text;
- error text;
- строки заказа.

---

## Flow 6. Выдача заказа / delivery workflow

Цель: работа оператора на точке выдачи.

### Последовательность

1. `POST /auth/login`
2. `GET /me`
3. `GET /pickup-admin/my-points`
4. `GET /orders?pickup_place=...` или `GET /orders/delivered` / `GET /orders/errors`
5. `POST /deliveries` — создать запись выдачи
6. при необходимости:
   - `GET /deliveries`
   - `GET /deliveries/{id}`
   - `PATCH /deliveries/{id}`
   - `DELETE /deliveries/{id}` — откат выдачи

### Что важно

- `pickup_admin` ограничен только назначенными точками;
- откат выдачи поддерживается;
- есть отдельный flow для проблемных заказов и последних выдач.

---

## Flow 7. Экспорт и отчёты

Цель: получить XLSX-отчёты или собрать кастомный export package.

### Последовательность для чувствительного экспорта

1. `POST /auth/login`
2. `POST /auth/confirm-password` с `action=export`
3. получить confirmation token
4. вызвать один из endpoint-ов:
   - `GET /orders/export`
   - `GET /orders/export-template`
   - `GET /orders/export-history`
   - `GET /exports/presets`
   - `POST /exports/build`

### Важно

- analytics и exports — owner-controlled контур;
- на некоторых export routes нужен step-up token.

---

## Flow 8. Provider connector поверх текущего домена

Цель: адаптировать внешний мессенджер, не переписывая order engine.

### Рекомендуемая последовательность

1. В connector-слое получить событие провайдера;
2. нормализовать его в структуру, близкую к `IncomingMessage`;
3. определить provider chat/room → internal chat mapping;
4. загрузить активный каталог / pickup places для контекста чата;
5. передать текст в текущий parse/order pipeline;
6. сохранить/обновить заказ;
7. отправить reply через API провайдера.

### Что потребуется дополнительно

- матрица provider chat/user/message mapping;
- правила ответов в DM/group/room semantics провайдера;
- новая реализация menu/actions вместо Telegram callbacks.

## Endpoint shortlist по сценариям

### Для внешнего read-only mobile / integration клиента

- `GET /integrations/capabilities`
- `GET /integrations/chats`
- `GET /integrations/catalogs/open`
- `GET /integrations/pickup-places`
- `GET /integrations/orders`

### Для operator / admin mobile клиента

- `POST /auth/login`
- `GET /me`
- `GET /orders`
- `GET /orders/{order_id}`
- `PATCH /orders/{order_id}`
- `POST/PATCH/DELETE /orders/{order_id}/lines*`
- `GET/POST/PATCH/DELETE /deliveries*`
- `GET /pickup-admin/my-points`

### Для management / owner клиента

- `GET /catalogs`, `GET /catalogs/{id}/items`
- `POST/PATCH/DELETE /catalogs*`
- `GET /analytics/*`
- `GET /orders/export*`
- `GET /exports/presets`, `POST /exports/build`

## Архитектурные границы mobile-интеграции

### Уже закрыто нашей стороной

- модель данных;
- operator/admin API;
- JWT auth;
- delivery flow;
- exports/analytics;
- базовый внешний M2M read API.

### Не закрыто как внешний стабильный mobile contract

- customer-facing create/update/cancel order API для native forms;
- provider-specific messenger adapters (Matrix/VK/MAX);
- готовый SDK/typed client для Flutter.

## Практический вывод

Для интеграционного планирования текущую систему следует рассматривать так:

1. **Provider bot adaptation** — отдельный integration layer project;
2. **операторские/mobile admin сценарии** — можно строить уже сейчас на `admin_service` JWT API;
3. **нативные customer order forms** — требуют дополнительного решения по write contract.

Этот пункт желательно фиксировать как отдельное архитектурное решение, а не предполагать по умолчанию.

## Что уже достаточно, чтобы начать интеграцию без чтения кода

Для старта технической оценки достаточно следующего набора проектных артефактов:

- [[20-api-and-business-flows-for-mobile-integration]] — последовательности вызовов и границы контуров;
- [[14-api-reference]] — полный список endpoint-ов и DTO уровня reference;
- [[17-admin-service-endpoints-manual]] — прикладной manual для разработки клиента;
- [[13-integration-api]] — отдельное описание M2M `/integrations/*`;
- [[22-messenger-agnostic-integration-spec]] и [[23-provider-bridge-runbook]] — transport contract для провайдеров.

Этого достаточно, чтобы:

- оценить сетевой слой Flutter/FluffyChat клиента;
- построить карту экранов и вызовов;
- отделить уже готовые read/operator routes от будущего customer write scope.

## Связанный комплект документации

Обязательный комплект:

- [[14-api-reference]] — полный API;
- [[17-admin-service-endpoints-manual]] — быстрый dev-manual;
- [[13-integration-api]] — M2M `/integrations/*`;
- [[22-messenger-agnostic-integration-spec]] — provider-agnostic контракт;
- [[23-provider-bridge-runbook]] — ingress/egress bridge handoff.

## Связанные документы

- [[14-api-reference]]
- [[17-admin-service-endpoints-manual]]
- [[15-endpoint-consumers-matrix]]
- [[19-bot-architecture]]
- [[22-messenger-agnostic-integration-spec]]
