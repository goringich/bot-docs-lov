# Быстрый перезапуск после изменений

Ниже — короткая памятка, как применить изменения в коде и конфиге.

## В Telegram (админ)

Команда `/restart` пришлёт краткую инструкцию перезапуска прямо в чат.

## 1) Обновил код (Python файлы)

Перезапусти **API** и **worker**.

### Остановить процессы и запустить заново

```bash
pkill -f "uvicorn app.main" || true
pkill -f "python -m app.worker" || true

set -a && source .env && set +a
PYTHONPATH="$(pwd)/backend" ./.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-level info \
  > .api.log 2>&1 &

set -a && source .env && set +a
PYTHONPATH="$(pwd)/backend" ./.venv/bin/python -m app.worker > .worker.log 2>&1 &
```

Проверка:

```bash
tail -n 50 .api.log
tail -n 50 .worker.log
curl http://127.0.0.1:8000/health
```

### Docker Compose (быстро)

Если проект запущен через docker compose:

```bash
cd infra
docker compose up -d --build api worker admin_service admin_web
```

Или через Makefile:

```bash
make sync
make compose-restart
```

`make compose-restart` пересоздаёт контейнеры (`--force-recreate`),
поэтому изменения в коде и Dockerfile не «залипают» в старом контейнере.

Если нужен **полный безопасный цикл после обновления кода**, используйте:

```bash
make sync
```

Эта команда:
- пересобирает контейнеры,
- поднимает `mysql` и `api`,
- выполняет `alembic upgrade heads`,
- затем перезапускает весь стек (`api`, `worker`, `admin_service`, `admin_web`, `ngrok`).

Это важно, потому что в репозитории могут одновременно существовать несколько Alembic head-веток; `upgrade heads` корректно применяет все актуальные вершины, а `upgrade head` в таком состоянии падает с ошибкой `Multiple head revisions are present`.

Для live-режима (чтобы изменения применялись сразу без ручного rebuild):

```bash
make compose-dev-up
make compose-dev-logs
make compose-dev-down
```

## 2) Поменял `.env`

Перезапусти **API** и **worker** (см. выше). Они читают конфиг при старте.

> Если перед этим был `git pull` или обновление ветки — **сначала** выполни миграции:
>
> ```bash
> cd backend
> ../.venv/bin/alembic upgrade heads
> ```
>
> Иначе возможна ситуация, когда `/telegram/webhook` отвечает `200`, но worker валится на обработке update из-за отсутствующих колонок/индексов в БД.

## 3) Добавил миграцию (Alembic)

Применить миграции и перезапустить сервисы (включая admin_service):

Перед рискованными миграциями полезно сначала сделать backup:

```bash
make backup-db
```

Потом уже выполнять `alembic upgrade heads` и рестарт сервисов.

```bash
cd backend
set -a && source ../.env && set +a
../.venv/bin/alembic upgrade heads
cd ..

pkill -f "uvicorn app.main" || true
pkill -f "python -m app.worker" || true

set -a && source .env && set +a
PYTHONPATH="$(pwd)/backend" ./.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-level info \
  > .api.log 2>&1 &

set -a && source .env && set +a
PYTHONPATH="$(pwd)/backend" ./.venv/bin/python -m app.worker > .worker.log 2>&1 &

set -a && source .env && set +a
./scripts/admin_service.sh \
  > .admin_api.log 2>&1 &
```

> Для миграции v2.0 (таблицы `bot_settings`, `pickup_places`) нужен перезапуск admin_service.

## 4) Сменился ngrok URL

Если ngrok перезапустился и URL поменялся — обнови webhook:

```bash
set -a && source .env && set +a
PUBLIC_URL="https://xxxx.ngrok-free.dev"  # подставь новый URL
WEBHOOK_URL="${PUBLIC_URL%/}/telegram/webhook"

curl -s "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/setWebhook" \
  --data-urlencode "url=$WEBHOOK_URL" \
  --data-urlencode "secret_token=$TELEGRAM_WEBHOOK_SECRET"
```

Проверка:

```bash
curl -s "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getWebhookInfo"
```

## 5) Проверка «живости»

```bash
curl http://127.0.0.1:8000/health
```

В Telegram:

```
/menu
/status
```

## 6) Admin panel (если используется)

Если запущен `admin_service`:

- перезапусти сервис (uvicorn) при изменениях
- убедись, что `/health` отвечает

> Надёжный вариант запуска: `./scripts/admin_service.sh`. Он использует `python -m uvicorn`, поэтому не ломается из-за устаревшего shebang внутри `.venv-admin/bin/uvicorn` после пересоздания окружения.

Для `admin-web` — достаточно перезапустить `npm run dev` (или пересобрать production bundle).

> Примечание: Telegram не позволяет менять цвета кнопок. Мы используем эмодзи и понятные подписи,
> чтобы визуально "раскрасить" меню и сделать его удобнее.

### [[11-security]]