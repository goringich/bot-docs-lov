# 23. Provider bridge runbook (Matrix / VK / MAX / Webhook)

Дата: 2026-03-18

Документ фиксирует **практический runtime-контракт** для подключения non-Telegram провайдеров к backend через ingress webhook + outbound bridge.

Этот файл нужен интеграторам, которые подключают Matrix, VK, MAX или собственный webhook-bridge без чтения Python-кода.

## Что уже реализовано

Backend уже умеет:

- принимать входящие события через `POST /integrations/{provider}/webhook`;
- проверять provider-specific secret headers;
- упаковывать событие в единый envelope;
- прогонять его через текущий order/domain pipeline;
- отправлять ответные действия через HTTP bridge (`X-Bridge-Secret` + JSON body).

## Поддерживаемые provider IDs

- `telegram`
- `matrix`
- `vk`
- `max`
- `webhook`

Для нового коннектора лучше использовать существующий `provider` id или добавить новый provider целиком через отдельную итерацию в коде.

## 1. Ingress: как отправлять события в backend

Endpoint:

- `POST /integrations/{provider}/webhook`

Примеры:

- `POST /integrations/matrix/webhook`
- `POST /integrations/vk/webhook`
- `POST /integrations/max/webhook`
- `POST /integrations/webhook/webhook`

Body:

- любой JSON payload провайдера;
- backend сохранит оригинальный payload в envelope и в raw snapshot дальнейшей обработки.

### Проверка secret headers

Backend принимает следующие заголовки:

| Provider | Поддерживаемые ingress headers | Secret source |
|---|---|---|
| `telegram` | `X-Telegram-Bot-Api-Secret-Token` | `TELEGRAM_WEBHOOK_SECRET` |
| `matrix` | `X-Matrix-Webhook-Secret`, `X-Matrix-Signature`, `X-Integration-Secret` | `MATRIX_WEBHOOK_SECRET` → fallback `INTEGRATION_WEBHOOK_SECRET` |
| `vk` | `X-VK-Secret`, `X-VK-Webhook-Secret`, `X-Integration-Secret` | `VK_WEBHOOK_SECRET` → fallback `INTEGRATION_WEBHOOK_SECRET` |
| `max` | `X-Max-Webhook-Secret`, `X-Max-Signature`, `X-Integration-Secret` | `MAX_WEBHOOK_SECRET` → fallback `INTEGRATION_WEBHOOK_SECRET` |
| `webhook` | `X-Integration-Secret` | `INTEGRATION_WEBHOOK_SECRET` |

### Рекомендуемая практика

- для production использовать отдельный secret per provider;
- `INTEGRATION_WEBHOOK_SECRET` оставлять как fallback только для generic bridge или dev/stage;
- vendor-native signatures можно проверять на внешнем bridge-слое дополнительно, а в backend передавать уже trusted request.

## 2. Envelope semantics внутри backend

После ingress backend приводит событие к provider-agnostic envelope.

Практически это даёт:

- единый `update_id`;
- единый `provider` context в worker;
- сохранение исходного provider payload без потери полей;
- единый путь обработки заказов и пользовательских reply.

Это значит, что внешнему bridge не нужно повторять бизнес-логику заказов — достаточно корректно передать событие в ingress и принять outbound action обратно.

## 3. Outbound bridge: что backend отправляет наружу

Для non-Telegram провайдеров worker отправляет HTTP POST на bridge endpoint.

Target URL выбирается так:

1. если во входящем envelope/ingress задан `callback_url`, используется он;
2. иначе используется provider-specific env URL:
   - `MATRIX_OUTBOUND_URL`
   - `VK_OUTBOUND_URL`
   - `MAX_OUTBOUND_URL`
   - `WEBHOOK_OUTBOUND_URL`

Если target URL отсутствует, действие не считается ошибкой pipeline: backend делает безопасный best-effort skip.

### Заголовки outbound bridge

Всегда:

- `Content-Type: application/json`

Опционально:

- `X-Bridge-Secret: <secret>`

Secret выбирается так:

| Provider | Primary secret | Fallback |
|---|---|---|
| `matrix` | `MATRIX_OUTBOUND_SECRET` | `PROVIDER_BRIDGE_SECRET` |
| `vk` | `VK_OUTBOUND_SECRET` | `PROVIDER_BRIDGE_SECRET` |
| `max` | `MAX_OUTBOUND_SECRET` | `PROVIDER_BRIDGE_SECRET` |
| `webhook` | `WEBHOOK_OUTBOUND_SECRET` | `PROVIDER_BRIDGE_SECRET` |

Timeout:

- `PROVIDER_BRIDGE_TIMEOUT_SEC`, по умолчанию `6.0` сек.

## 4. Формат outbound JSON

Backend отправляет JSON строго такого вида:

```json
{
  "provider": "matrix",
  "action": "send_message",
  "payload": {
    "chat_id": 123,
    "text": "✅ Заказ принят"
  }
}
```

### Поддерживаемые actions

- `send_message`
- `send_document`
- `answer_callback_query`
- `set_message_reaction`
- `delete_message`

## 5. Payload schema по action

### `send_message`

```json
{
  "provider": "matrix",
  "action": "send_message",
  "payload": {
    "chat_id": 123,
    "text": "✅ Заказ принят",
    "reply_to_message_id": 456,
    "reply_markup": null,
    "disable_web_page_preview": true
  }
}
```

Поля:

- `chat_id` — canonical chat id, который backend использует в message pipeline;
- `text` — готовый текст ответа;
- `reply_to_message_id` — optional reference на исходное сообщение;
- `reply_markup` — transport-agnostic структура клавиатуры/кнопок, bridge сам маппит её в vendor format;
- `disable_web_page_preview` — hint, можно игнорировать, если провайдер не поддерживает.

### `send_document`

```json
{
  "provider": "vk",
  "action": "send_document",
  "payload": {
    "chat_id": 123,
    "file_path": "/app/exports/orders_2026.xlsx",
    "caption": "Excel export",
    "reply_to_message_id": 456
  }
}
```

Поля:

- `file_path` — локальный путь файла на стороне backend-container/runtime;
- bridge сам решает, как прочитать файл и загрузить его в vendor API;
- если провайдер не поддерживает upload напрямую, допустим fallback в текстовое уведомление.

### `answer_callback_query`

```json
{
  "provider": "max",
  "action": "answer_callback_query",
  "payload": {
    "callback_query_id": "cb-123",
    "text": "Готово"
  }
}
```

Используется для снятия loading state/ack на inline interaction, если провайдер поддерживает такой UX.

### `set_message_reaction`

```json
{
  "provider": "matrix",
  "action": "set_message_reaction",
  "payload": {
    "chat_id": 123,
    "message_id": 456,
    "emoji": "👍"
  }
}
```

Если reactions не поддерживаются, bridge может:

- тихо проигнорировать действие;
- или заменить его на lightweight text/system message на своей стороне.

### `delete_message`

```json
{
  "provider": "vk",
  "action": "delete_message",
  "payload": {
    "chat_id": 123,
    "message_id": 456
  }
}
```

## 6. Что должен вернуть bridge

Backend считает доставку успешной, если HTTP response имеет статус `2xx`.

Минимальное требование:

- вернуть `200 OK`, `202 Accepted` или другой успешный `2xx`.

Тело ответа backend сейчас не использует, поэтому bridge может отвечать кратко:

```json
{ "ok": true }
```

Если bridge вернёт `4xx/5xx`, backend зафиксирует ошибку outbound dispatch в логах, но не должен ломать уже вычисленный domain result заказа.

## 7. Provider-specific рекомендации

### Matrix

Рекомендуемый внешний bridge:

- принимает `m.room.message`, reaction/edit events;
- нормализует room/event sender data в ingress payload;
- в outbound преобразует:
  - `send_message` → `m.room.message`
  - `set_message_reaction` → annotation/relation event
  - `delete_message` → redaction

Практический совет:

- хранить mapping `backend chat_id ↔ matrix room_id` и `backend message_id ↔ event_id` в bridge-слое;
- если нужен federation-safe retry, реализовывать его именно в bridge, не в backend order pipeline.

### VK

Рекомендуемый внешний bridge:

- принимает callback API события от VK;
- проходит confirmation/signature на своей стороне;
- в backend отправляет уже trusted webhook JSON;
- в outbound преобразует:
  - `send_message` → `messages.send`
  - `delete_message` → доступный VK delete flow
  - `answer_callback_query` — optional no-op, если нативного аналога нет

Практический совет:

- для callback confirmation/secret лучше оставить vendor-native handshake на bridge-слое;
- backend-secret использовать как второй внутренний trust boundary.

### MAX

Так как vendor contract MAX может отличаться по ACL и action semantics, bridge рекомендуется строить по тем же принципам:

- внешний ingress понимает native webhook/event stream MAX;
- backend получает унифицированный trusted JSON;
- outbound action map делается на стороне bridge без изменения domain logic.

## 8. Рекомендуемый deployment pattern

### Вариант A — shared bridge per provider

- `matrix-bridge` обслуживает только Matrix;
- `vk-bridge` обслуживает только VK;
- `max-bridge` обслуживает только MAX.

Плюсы:

- проще vendor-specific отладка;
- проще ротация секретов;
- проще логирование и алерты.

### Вариант B — universal bridge gateway

Один сервис принимает actions для всех provder'ов:

- читает `provider` из JSON body;
- роутит в vendor-specific client внутри себя.

Подходит, если команда хочет один deployment и единый observability stack.

## 9. Минимальный rollout checklist

### Matrix / VK / MAX

- [ ] Настроен ingress secret
- [ ] Настроен outbound URL
- [ ] Настроен outbound bridge secret
- [ ] Bridge отвечает `2xx` на health/test action
- [ ] Текстовое сообщение проходит end-to-end
- [ ] Unknown-item / partial-order reply приходит обратно пользователю
- [ ] Реакция или её graceful fallback работают
- [ ] Логи не содержат raw secrets

## 10. Пример env-конфига

```dotenv
MESSENGER_ENABLED_PROVIDERS=telegram,matrix,vk,max,webhook
INTEGRATION_WEBHOOK_SECRET=shared-dev-secret

MATRIX_WEBHOOK_SECRET=matrix-ingress-secret
MATRIX_OUTBOUND_URL=https://<matrix-bridge-host>/api/bridge
MATRIX_OUTBOUND_SECRET=matrix-egress-secret

VK_WEBHOOK_SECRET=vk-ingress-secret
VK_OUTBOUND_URL=https://<vk-bridge-host>/api/bridge
VK_OUTBOUND_SECRET=vk-egress-secret

MAX_WEBHOOK_SECRET=max-ingress-secret
MAX_OUTBOUND_URL=https://<max-bridge-host>/api/bridge
MAX_OUTBOUND_SECRET=max-egress-secret

PROVIDER_BRIDGE_TIMEOUT_SEC=6.0
```

## 11. Границы текущей готовности

Что уже production-safe на уровне backend:

- единый ingress endpoint;
- idempotent envelope path;
- provider-specific secret validation;
- best-effort outbound bridge dispatch.

Что ещё остаётся на стороне интеграции/bridge:

- vendor-native OAuth/token lifecycle;
- точные signature/checksum semantics конкретного провайдера;
- room/dialog/message mapping storage;
- retry/backoff policy против реального API провайдера;
- observability по vendor-specific delivery errors.

## Связанные документы

- [[14-api-reference]]
- [[18-integration-overview]]
- [[22-messenger-agnostic-integration-spec]]
- [[changelog/decisions-log]]
- `.env.example`
