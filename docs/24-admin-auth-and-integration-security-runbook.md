# 24. Admin Auth & Integration Security Runbook

Дата: 2026-03-26

Этот документ описывает текущую схему авторизации `admin_service` и то, как её правильно настраивать и эксплуатировать после hardening 2026-03-26.

## Что изменилось

Раньше одна и та же схема давала два риска:

- один `X-Service-Key` использовался и для M2M-интеграций, и для выдачи admin magic links;
- one-time token ходил в query string и мог утекать через history / proxy logs / copy-paste.

Теперь это разделено:

- `ADMIN_ONE_TIME_KEY` используется только для выдачи одноразовых ссылок входа;
- `ADMIN_INTEGRATION_SERVICE_KEY` используется только для `/integrations/*`;
- одноразовый токен передаётся во frontend через URL fragment;
- сам вход в админку держится на `HttpOnly` cookie, а не на `localStorage`.

## Переменные окружения

Обязательные переменные для `admin_service`:

```env
ADMIN_API_JWT_SECRET=<strong-random-secret>
ADMIN_API_JWT_EXPIRE_MIN=480
ADMIN_API_BOOTSTRAP_TOKEN=<bootstrap-secret>

# Только для выдачи one-time admin links
ADMIN_ONE_TIME_KEY=<separate-random-secret>

# Только для внешних /integrations/* клиентов
ADMIN_INTEGRATION_SERVICE_KEY=<different-random-secret>

ADMIN_WEB_URL=https://admin.example.com
ADMIN_API_CORS_ORIGINS=https://admin.example.com
```

Требования:

- `ADMIN_ONE_TIME_KEY` и `ADMIN_INTEGRATION_SERVICE_KEY` должны быть разными.
- `ADMIN_WEB_URL` должен указывать на реальный origin `admin-web`.
- На production `ADMIN_WEB_URL` должен быть `https://...`, чтобы auth cookie выставлялась как `Secure`.

## Как теперь устроен вход в админку

### 1. Login по email/password

Поток:

1. `admin-web` вызывает `POST /auth/login`.
2. `admin_service` проверяет пароль, создаёт JWT access token и `ActiveSession`.
3. JWT кладётся в `HttpOnly` cookie `admin_access_token`.
4. JSON-ответ возвращает только технический статус `{"status":"ok","session_type":"cookie"}`.
5. Дальше frontend работает через cookie-сессию.

Важно:

- frontend больше не хранит access token в `localStorage`;
- browser login flow больше не получает bearer token в response body;
- `Authorization: Bearer` всё ещё поддерживается backend-ом как fallback, но основной браузерный путь теперь cookie-based.

### 2. One-time login link из Telegram / backend

Поток:

1. доверенный сервис вызывает `POST /auth/one-time-link` с `X-Service-Key: ADMIN_ONE_TIME_KEY`;
2. backend создаёт запись в `admin_one_time_login_tokens`;
3. backend генерирует JWT с `typ=one_time_login`, `jti`, `exp`;
4. ссылка возвращается в виде:

```text
https://admin.example.com/auth-callback#one_time_token=<jwt>
```

5. `admin-web` читает токен из `location.hash`;
6. frontend отправляет токен в `POST /auth/token-login` в body;
7. backend проверяет JWT и помечает `jti` как consumed;
8. backend поднимает `HttpOnly` cookie session и возвращает только статус;
9. повторный вход по тому же токену получает `401`.

Почему fragment:

- fragment не уходит на сервер как часть HTTP request line;
- он не попадает в обычные access logs reverse proxy;
- его безопаснее шарить, чем query string, хотя ссылку всё равно надо считать чувствительной до истечения TTL.

## Таблицы и сущности

### `admin_active_sessions`

Используется для обычных login-сессий.

Хранит:

- `user_id`
- `token_jti`
- `expires_at`
- `is_revoked`
- `last_activity_at`

### `admin_one_time_login_tokens`

Используется только для one-time login flow.

Хранит:

- `user_id`
- `token_jti`
- `expires_at`
- `consumed_at`
- `consumed_by_ip`

Смысл:

- JWT сам по себе подписан, но одноразовость обеспечивается не только `exp`, а именно server-side consume.

## Какой ключ где использовать

### `ADMIN_ONE_TIME_KEY`

Разрешено:

- `POST /auth/one-time-link`
- `POST /auth/auto-provision`

Не использовать для:

- `/integrations/*`
- внешних read-only интеграций
- мобильных sync-клиентов

### `ADMIN_INTEGRATION_SERVICE_KEY`

Разрешено:

- `GET /integrations/capabilities`
- `GET /integrations/chats`
- `GET /integrations/catalogs/open`
- `GET /integrations/pickup-places`
- `GET /integrations/orders`

Не использовать для:

- admin login
- magic links
- auto-provision flow

## Как деплоить изменения

1. Сгенерировать новый `ADMIN_INTEGRATION_SERVICE_KEY`.
2. Оставить или перевыпустить `ADMIN_ONE_TIME_KEY`.
3. Обновить `.env` на сервере.
4. Обновить клиентов, которые ходят в `/integrations/*`, чтобы они использовали новый integration key.
5. Перезапустить `admin_service`.
6. Проверить:

```bash
curl -H "X-Service-Key: $ADMIN_INTEGRATION_SERVICE_KEY" \
  https://admin.example.com/api/integrations/capabilities
```

И отдельно:

```bash
curl -X POST \
  -H "X-Service-Key: $ADMIN_ONE_TIME_KEY" \
  "https://admin.example.com/api/auth/one-time-link?tg_user_id=123456"
```

## Что проверять после деплоя

- `/integrations/capabilities` отвечает только с `ADMIN_INTEGRATION_SERVICE_KEY`
- `/auth/one-time-link` отвечает только с `ADMIN_ONE_TIME_KEY`
- ссылка логина имеет вид `/auth-callback#one_time_token=...`
- после первого успешного `token-login` повторное использование той же ссылки даёт `401`
- `POST /users/{user_id}/invite-link` тоже выдаёт single-use ссылку, которая не работает повторно
- браузер после login получает `Set-Cookie: admin_access_token=...; HttpOnly`
- `POST /auth/login` и `POST /auth/token-login` не отдают `access_token` в JSON
- в `localStorage` больше нет `admin_token`

## CSRF и Origin-check

Для cookie-auth mutating запросов (`POST`, `PUT`, `PATCH`, `DELETE`) backend теперь делает origin verification:

- если запрос пришёл по `Authorization: Bearer`, дополнительная browser-origin проверка не нужна;
- если запрос использует cookie `admin_access_token`, `Origin` или `Referer` должен совпадать с origin из `ADMIN_WEB_URL`;
- если запрос использует cookie `admin_access_token`, frontend также обязан передавать `X-CSRF-Token`, совпадающий с cookie `admin_csrf_token`;
- при несовпадении backend отвечает `403`.

Это не заменяет `SameSite=Lax`, а добавляет вторую линию защиты.

Для уже активной cookie-сессии frontend может вызвать `GET /auth/csrf`, чтобы переинициализировать `admin_csrf_token` без повторного логина.

## Step-up для сброса пароля

Сброс пароля пользователя теперь считается чувствительной операцией:

- owner сначала вызывает `POST /auth/confirm-password` с `action=user_password_reset`;
- получает краткоживущий `confirmation_token`;
- передаёт его в `X-Password-Confirm-Token` на `PUT /users/{user_id}/password`.

## CORS по умолчанию

Теперь production-safe дефолт такой:

- `ADMIN_API_CORS_ORIGINS` по умолчанию берётся из `ADMIN_WEB_URL`;
- `ADMIN_API_CORS_ORIGIN_REGEX` по умолчанию пустой;
- локальные preview origins нужно явно разрешать через `.env`, а не через permissive regex в коде.

## Что важно разработчикам фронта

- не добавлять обратно хранение access token в `localStorage` или `sessionStorage`;
- не переносить one-time token обратно в query string;
- если нужен session check, ориентироваться на `GET /me`, а не на наличие локального токена.
- для mutating cookie-запросов всегда слать `X-CSRF-Token`.

## Что важно разработчикам backend

- для новых M2M ручек использовать `require_integration_service_key`, а не auth one-time key;
- для новых browser auth flows использовать cookie-сессию как primary path;
- любые новые Excel/XLSX выгрузки должны экранировать строки, начинающиеся с `=`, `+`, `-`, `@`, чтобы не допустить formula injection;
- все “одноразовые” токены, влияющие на вход или privilege escalation, должны иметь server-side consume state.
