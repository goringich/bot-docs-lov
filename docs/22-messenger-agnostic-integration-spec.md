# 22. Универсальный контракт интеграции мессенджеров

Дата: 2026-03-18

Документ задаёт единый provider-agnostic контракт для интеграции любого мессенджера с текущим доменным ядром заказов.

Поддерживаемые целевые провайдеры по этому контракту:

- Telegram
- Matrix (Synapse / Maubot)
- VK
- MAX

## Назначение

Контракт нужен, чтобы любой новый connector работал по одной схеме:

1. принимает event провайдера;
2. нормализует его в единый формат входящего сообщения;
3. передаёт в существующий order/domain pipeline;
4. получает domain-результат;
5. отправляет reply/action обратно через API конкретного провайдера.

## Интеграционные принципы

### 1) Domain-first

Бизнес-логика заказов остаётся общей и не зависит от конкретного мессенджера.

### 2) Adapter-per-provider

Для каждого провайдера добавляется отдельный transport adapter:

- `telegram_adapter`
- `matrix_adapter`
- `vk_adapter`
- `max_adapter`

### 3) Capability-driven behavior

Функциональность (reactions, buttons, edits, files, threads) определяется capability profile провайдера, а не жёстко зашита в сценарии.

В текущем runtime capability profile реализован как отдельный слой и используется для выбора допустимых outbound action-ов без ветвления business logic по provider-specific if/else.

## Единая модель входящего события

Нормализованная сущность должна покрывать минимум:

- `provider_id` — `telegram | matrix | vk | max`
- `chat_id` — id комнаты/чата/диалога
- `user_id` — id пользователя
- `message_id` — id сообщения/ивента
- `text` — текст сообщения
- `message_type` — `text | command | callback | media`
- `timestamp`
- `reply_to_message_id` (опционально)
- `raw_event` — оригинальный payload провайдера

Рекомендуемая совместимость с существующей моделью:

- `IncomingMessage` из `backend/app/adapters/base.py`.

## Единая модель исходящего действия

Минимальный набор действий, который должен поддерживать любой connector:

1. `send_message`
2. `send_document` (если есть в провайдере; иначе graceful fallback)
3. `set_message_reaction` (если есть в провайдере)
4. `edit_message` (если есть в провайдере)
5. `delete_message`
6. `answer_callback_query` (если interaction model провайдера поддерживает callback/ack semantics)

Если capability отсутствует, adapter возвращает безопасный fallback (например, отправка текстового уведомления вместо реакции/редактирования).

## Capability profile провайдера

Для каждого провайдера фиксируется профиль, который как минимум покрывает:

- `send_text`
- `send_photo`
- `send_document`
- `inline_keyboards`
- `reactions`
- `callback_queries`
- `edit_message`
- `delete_message`
- `group_chats`
- `private_chats`
- `html_formatting`
- `markdown_formatting`
- `max_message_length`

Этот профиль используется в runtime для выбора UX-ветки (кнопки, reply style, системные сообщения).

## Mapping-таблица провайдеров

| Провайдер | Входящие события | Исходящие действия | Критичные особенности |
|---|---|---|---|
| Telegram | webhook updates | sendMessage/sendDocument/reactions/callbacks | callback_query, Bot API limits, inline keyboard model |
| Matrix | room events (m.room.message, reactions, edits) | room send, relations/reactions, state events | room/space semantics, event relations, federation latency |
| VK | callback/event API | messages.send, keyboards, attachments | callback confirmation flow, platform throttling |
| MAX | bot event stream / webhook | message send/actions/files | специфика MAX API и ACL модели (определяется по документации провайдера) |

## Нормализация команд и меню

Командный слой должен быть provider-agnostic:

- canonical intent: `menu`, `help`, `my_order`, `set_pickup`, `set_phone`, `add_to_order`, `cancel_order`, `pickup_points`;
- adapter маппит provider-specific формат в canonical intent;
- presenter/bot_ui формирует контент, adapter формирует transport representation (buttons/quick actions/reply tokens).

## Идентификаторы и хранение

Для provider-agnostic интеграции в модели чатов и сообщений рекомендуется использовать ключ вида:

- `provider_id`
- `provider_chat_id`
- `provider_message_id`

Или составной canonical идентификатор:

- `provider_chat_key = <provider_id>:<provider_chat_id>`.

Это предотвращает коллизии между одинаковыми числовыми id разных платформ.

## Безопасность интеграции

### Входящий ingress

- верификация подписи/secret каждого провайдера;
- idempotency key на уровне event id;
- защита от replay;
- rate limiting ingress.

Текущий runtime-контракт для webhook secret:

- `telegram` → `X-Telegram-Bot-Api-Secret-Token` / `TELEGRAM_WEBHOOK_SECRET`
- `matrix` → `X-Matrix-Webhook-Secret` / `MATRIX_WEBHOOK_SECRET`
- `vk` → `X-VK-Webhook-Secret` / `VK_WEBHOOK_SECRET`
- `max` → `X-Max-Webhook-Secret` / `MAX_WEBHOOK_SECRET`
- fallback для generic/webhook connectors → `X-Integration-Secret` / `INTEGRATION_WEBHOOK_SECRET`

### Исходящий egress

- централизованный rate limiting на отправку;
- retry policy с backoff;
- graceful degrade при unavailable API провайдера.

Текущий runtime-контракт для non-Telegram egress:

- worker диспатчит стандартные действия (`send_message`, `send_document`, `answer_callback_query`, `set_message_reaction`, `delete_message`) через provider bridge;
- bridge URL берётся из `MATRIX_OUTBOUND_URL`, `VK_OUTBOUND_URL`, `MAX_OUTBOUND_URL`, `WEBHOOK_OUTBOUND_URL`;
- auth идёт через `X-Bridge-Secret` с provider-specific secret или общим `PROVIDER_BRIDGE_SECRET`;
- если URL не настроен, действие безопасно пропускается без падения domain pipeline.

### Секреты

- отдельные env keys per provider;
- без hardcoded токенов;
- маскирование в логах.

## SLO/SLA ожидания для connector-слоя

- приём входящего event: быстрый ACK без тяжёлой синхронной логики;
- доставка в очередь обработки: at-least-once;
- обработка заказа: идемпотентная;
- отправка ответа: best effort с retry и журналированием ошибок.

## Пошаговый playbook подключения нового провайдера

1. Создать provider adapter (реализация `MessengerAdapter`).
2. Описать capability profile.
3. Реализовать event normalizer → `IncomingMessage`.
4. Реализовать egress client (`send_text`, `send_file`, fallbacks).
5. Настроить ingress endpoint/consumer с signature validation.
6. Подключить provider routing в worker/orchestrator.
7. Добавить интеграционные тесты provider contract:
   - text message
   - command intent
   - partial/rejected order response
   - retry/idempotency
8. Обновить docs:
   - `docs/19...`
   - `docs/20...`
   - [[14-api-reference]] (если меняются публичные route-ы)

## Текущее runtime покрытие (реализовано в коде)

- Универсальный ingress endpoint: `POST /integrations/{provider}/webhook`.
- Legacy-совместимый Telegram endpoint: `POST /telegram/webhook` (тоже упаковывается в envelope).
- В worker добавлена нормализация envelope → unified payload (`provider`, `chat_id`, `user_id`, `message_id`, `text`).
- Для ingress добавлена provider-specific secret validation с fallback на общий `INTEGRATION_WEBHOOK_SECRET`.
- Outbound для non-Telegram теперь работает через bridge-based dispatch:
   - worker формирует action payload и отправляет его на provider-specific outbound URL;
   - если bridge URL не настроен, reply/reaction/delete operations пропускаются как best-effort no-op без падения pipeline.
- Реализованы provider-specific ingress parsers для Matrix / VK / MAX / generic webhook.
- В runtime работает `AdapterManager`, который маршрутизирует outbound действия по provider capability profile.
- Реализованы adapter classes для `telegram`, `matrix`, `vk`, `max`, `webhook`.
- Добавлен adapter bootstrap/validation слой (`adapters/factory.py`) с проверкой provider token readiness.

## Минимальный acceptance checklist

- [ ] Провайдер принимает и нормализует текстовые сообщения
- [ ] Создание/обновление заказа проходит через общий domain pipeline
- [ ] Пользователь получает корректные статусы (`accepted/partial/rejected`)
- [ ] Отсутствующие capabilities обрабатываются fallback-ом без падений
- [ ] Signature/secret validation включена
- [ ] Idempotency для повторных events подтверждена
- [ ] Ограничения rate-limit не приводят к дубликатам заказов
- [ ] Документация и capability profile зафиксированы

## Текущее состояние проекта по этому контракту

- Telegram: production path реализован end-to-end.
- Matrix: ingress parser + bridge-based egress + capability-aware runtime реализованы; при необходимости можно отдельно развивать vendor-native Matrix connector lifecycle.
- VK: ingress parser + bridge-based egress + capability-aware runtime реализованы; vendor-native operational semantics могут отдельно уточняться на bridge-слое.
- MAX: ingress parser + bridge-based egress + capability-aware runtime реализованы; vendor-native operational semantics могут отдельно уточняться на bridge-слое.

Текущий код содержит не только базовую adapter abstraction, но и готовый provider-agnostic runtime path, что снижает стоимость добавления новых провайдеров до уровня `adapter + parser + capability profile + bridge mapping`.

## Связанные документы

- [[18-integration-overview]]
- [[19-bot-architecture]]
- [[20-api-and-business-flows-for-mobile-integration]]
- [[23-provider-bridge-runbook]]
- [[14-api-reference]]
- [[13-integration-api]]