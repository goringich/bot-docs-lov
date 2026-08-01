\# Ops runbook (локально/сервер)

Эта страница — короткий набор операционных процедур: как запускать, обновлять и проверять работоспособность.

Если нужна самая короткая памятка — см. [[11-restart]].

## Предпосылки

- В корне репозитория есть `.env` (см. `README.md`).
- БД поднята (обычно MySQL через `infra/docker-compose.yml`).
- Виртуальное окружение создано в `.venv`.

## Сервисы

### API (webhook)

API принимает Telegram updates и кладёт их в БД в очередь.

Ожидаемая команда запуска (из корня репо):

- `PYTHONPATH=$(pwd)/backend ./.venv/bin/uvicorn app.main:app ...`

### Worker

Worker обрабатывает очередь `tg_updates`.

Ожидаемая команда запуска (из корня репо):

- `PYTHONPATH=$(pwd)/backend ./.venv/bin/python -m app.worker`

### Admin API

Админ‑сервис запускается отдельно:

- `./scripts/admin_service.sh`

### Admin Web

Веб‑панель (dev):

- `npm install && npm run dev` в `admin-web/`

### CachyOS: локальный и Tailscale-доступ

На CachyOS админка публикуется **только** на двух интерфейсах одного контейнера:

- локально: `http://127.0.0.1:${ADMIN_WEB_PORT:-4173}/login`;
- в tailnet: `http://${ADMIN_WEB_NETWORK_BIND_IP}:${ADMIN_WEB_NETWORK_PORT}/login`.

Для текущего узла значения `ADMIN_WEB_NETWORK_BIND_IP=100.76.165.75` и
`ADMIN_WEB_NETWORK_PORT=4173`. `ADMIN_WEB_URL` и
`ADMIN_API_CORS_ORIGINS` должен включать этот сетевой URL, а также
`http://localhost:4173` и `http://127.0.0.1:4173` для loopback-разработки;
`ADMIN_WEB_URL` остаётся сетевым URL для magic links и Host allowlist. Не публикуйте наружу
`8010` (admin API), `8000` (webhook API), `3306` (MySQL) или `4040` (ngrok).

После изменения этих переменных или `infra/docker-compose.yml` примените:

```bash
cd infra
docker compose --env-file ../.env up -d --force-recreate admin_service admin_web
curl -I http://127.0.0.1:4173/login
curl -I http://100.76.165.75:4173/login
```


## Обновление кода

1) подтянуть изменения (git pull)
2) если менялись зависимости — переустановить requirements
3) если есть миграции — выполнить `alembic upgrade heads`
4) перезапустить API + worker

После рестарта worker выполняет startup reparse не только для `needs_admin/partial/rejected`, но и для сохранённых `active` заказов. Это нужно, чтобы фиксы парсера, ручные правки каталога и alias-обновления реально переписывали старые `order_lines` и отражались в экспорте.

Важно: worker **не** дописывает каталог из runtime/admin-feed сообщений. Каталог обновляется отдельно (через админские операции или ручной синк), а parser/worker только читает текущий каталог как источник истины.

**Последняя миграция:** `b2c3d4e5f6a7` — bot_visible 3-mode (hidden/reactions_only/full), regional_admin role seed, high-load indexes, admin_user_regions table.

Подробные команды (копипаст) есть в [[11-restart]].

## Docker Compose (рекомендуемый быстрый старт)

Быстрый запуск всех сервисов (DB + API + worker + admin):

```bash
cd infra
docker compose --env-file ../.env up -d --build
```

Сокращённые команды через Makefile:

```bash
make compose-up
make compose-restart
make compose-down
```

Перезапуск только сервисов приложения:

```bash
cd infra
docker compose --env-file ../.env up -d --build api worker admin_service admin_web
```

> Если `admin_service` недавно менялся, обычного `restart` может быть недостаточно: сервис собирается из Docker image, поэтому для подхвата кода нужен именно `docker compose up -d --build admin_service`.

Остановить всё:

```bash
cd infra
docker compose --env-file ../.env down
```

## Проверка работоспособности

- healthcheck API: `GET /health`
- посмотреть последние логи: `.api.log` и `.worker.log`

Рекомендуемая минимальная проверка:
1) `curl http://127.0.0.1:8000/health`
2) отправить тестовое сообщение в чат и убедиться, что worker не падает

Если нужно вручную массово перепарсить уже сохранённые заказы после ручного обновления каталога, alias-правок или parser-fix, используйте `backend/scripts/sync_catalog_from_tg_updates.py` с флагом `--reprocess-all` для нужного чата.

## Миграции

Миграции лежат в `backend/migrations/versions/`.

Применение:
- зайти в `backend/`
- загрузить env из `../.env`
- выполнить `alembic upgrade heads`

Актуальная миграция v2.0:
- `c9f8a1234567_add_bot_settings_and_pickup_places.py`
- `a1b2c3d4e5f6_add_bot_visible_and_admin_chats.py` — bot_visible на чатах + таблица admin_user_chats
- `e4g5h6i7j8k9_add_delivery_tracking.py` — таблицы order_deliveries, admin_user_pickup_points + поле delivery_status на orders

После миграции рекомендуется перезапустить:
- `api`
- `worker`
- `admin_service`

### Legacy admin schema drift

Начиная с security hardening, `admin_service` использует дополнительные колонки для audit/session/login tracking. Если окружение было обновлено со старой версии без отдельной миграции для admin-таблиц, новый сервис при старте сам пытается добавить недостающие колонки в `admin_audit_logs`, `admin_login_attempts` и `admin_active_sessions`.

Практически это значит:

- после обновления `admin_service` делайте именно пересборку: `docker compose up -d --build admin_service`;
- если раньше `/auth/login` падал на `Unknown column ...`, после пересборки сервис должен сам выровнять схему на старте;
- если auto-backfill не помог, проверьте логи `admin_service` — проблема уже не в старом коде образа, а в правах/состоянии БД.

> Для `admin_service` используйте `./scripts/admin_service.sh`, а не shebang-скрипт `./.venv-admin/bin/uvicorn`: после пересоздания venv он может указывать на старый Python и падать ещё до старта API.

Если сервисы запущены в Docker Compose:

```bash
cd infra
docker compose --env-file ../.env exec api alembic upgrade heads
```

## Бэкап и восстановление БД

Теперь backup/recovery входят в штатный operational flow проекта.

### Создать backup

Из корня репозитория:

```bash
make backup-db
```

Или напрямую:

```bash
bash ./scripts/backup_db.sh
```

Поведение:
- если доступен `docker compose` и запущен сервис `mysql`, скрипт использует контейнер;
- иначе пытается использовать локальный `mysqldump`;
- результат сохраняется в `backups/db/*.sql.gz`.

Дополнительно можно управлять режимом:

```bash
bash ./scripts/backup_db.sh --mode docker
bash ./scripts/backup_db.sh --mode native
```

### Восстановить backup

```bash
make restore-db FILE=backups/db/your_dump.sql.gz
```

Или напрямую:

```bash
bash ./scripts/restore_db.sh --file backups/db/your_dump.sql.gz
```

Скрипт попросит явное подтверждение перед restore.

Для неинтерактивного сценария:

```bash
bash ./scripts/restore_db.sh --file backups/db/your_dump.sql.gz --yes
```

### Когда делать backup обязательно

- перед ручным restore;
- перед рискованными миграциями/операциями на production;
- перед массовыми ручными правками данных;
- перед upgrade, если не уверен в консистентности окружения.

### Примечание

Excel export — это operational выгрузка для работы, а не замена техническому backup БД.

### [[11-restart]]
