# Testing strategy (MVP)

Тесты в проекте ориентированы на быстрые проверки критичной логики (парсинг, обработка апдейтов, команды админа, экспорт).

## Принцип

- Для любого code/behavior change обязательна пропорциональная целевая проверка; «рутинность» не отменяет доказательство результата.
- Для рискованных изменений (БД, миграции, парсинг, экспорт, роутинг апдейтов) — запускать целевые тесты.
- Любые изменения, затрагивающие `login/auth/cookie/CSRF/middleware` или mutating API контракты, считаются рискованными по умолчанию.
- Для таких изменений перед завершением обязательно запускать `make check-post-preflight`.
- Проверки — только один слой: перед заявлением о завершении применяется [[33-task-completion-gate]]. Для runtime/data-задачи нужны также доказательства deploy и фактического live-результата.

## Где лежат тесты

- `backend/tests/` — pytest.
- `admin_service/tests/` — pytest + FastAPI TestClient.
- `admin-web/src/**/*.test.ts(x)` — Vitest + React Testing Library.

## Рекомендуемые быстрые проверки

1) order processing / pipeline smoke
	- `backend/tests/test_order_processing_mvp.py`
	- `backend/tests/test_replay_pipeline_smoke.py`
	- `backend/tests/test_verify_catalog_lifecycle_script.py`

2) catalog lifecycle / admin commands + экспорт
	- `admin_service/tests/test_catalog_lifecycle.py`
	- `admin_service/tests/test_catalogs_legacy_schema_compat.py`
	- `backend/tests/test_admin_commands_catalog_and_pickup.py`

3) frontend критичные UX/ACL сценарии
	- `admin-web/src/components/MainLayout.test.tsx`
	- `admin-web/src/pages/SettingsPage.test.tsx`
	- `admin-web/src/theme/index.test.ts`

4) frontend e2e (Playwright)
	- `admin-web/e2e/acl-navigation.spec.ts`

Запуск frontend-тестов (через portable Node wrapper):

	- `bash scripts/admin_web.sh npm run test`
	- `bash scripts/admin_web.sh npm run e2e`

## Каноничные локальные команды

### Предпочтительный единый запуск

	- `make py-envs`
	- `make check-lint-python`
	- `make check-lint-frontend`
	- `make check-fast`
	- `make check-post-preflight`
	- `make check-catalogs-preflight`
	- `make check`
	- `make check-docker`
	- `make check-agent-contract`
	- `bash scripts/run_checks.sh help`

### Каноничный bootstrap Python env

	- `make py-envs`
	- `make py-env-backend`
	- `make py-env-admin-service`
	- `bash scripts/setup_python_envs.sh help`

### Минимальный Python lint baseline

	- `python -m pip install -r requirements-dev.txt`
	- `make check-lint-python`
	- `bash scripts/run_checks.sh lint-python`

### Минимальный frontend lint baseline

	- `bash scripts/admin_web.sh npm ci`
	- `make check-lint-frontend`
	- `bash scripts/run_checks.sh admin-web-lint`

### Backend (targeted smoke/regression)

	- `cd backend && PYTHONPATH=. ../.venv/bin/python -m pytest -q tests/test_rate_limiter.py tests/test_order_processing_mvp.py tests/test_replay_pipeline_smoke.py tests/test_parser_upgrade.py tests/test_verify_catalog_lifecycle_script.py`

### Admin Service

	- `./.venv-admin/bin/python -m pytest -q admin_service/tests`
	- `PYTHONPATH=admin_service ./.venv-admin/bin/python -m pytest -q admin_service/tests/test_catalog_lifecycle.py admin_service/tests/test_catalogs_legacy_schema_compat.py`
	- `scripts/run_checks.sh` запускает 10-second FastAPI `TestClient` probe
	  перед каждой зависимой от него suite. Чистая lifecycle-регрессия в
	  `catalogs-preflight` проходит до probe; если probe зависает, runner честно
	  завершает оставшуюся TestClient-проверку с ошибкой вместо бесконечного
	  ожидания; см. [[changelog/known-issues]].

### Admin Web

	- `cd admin-web && npm ci && npm run build && npm run test`
	- `cd admin-web && npx playwright install chromium && npm run e2e`
	- `cd admin-web && npm run api:smoke` *(явный live-check контракта с `admin_service`, если настроены `API_SMOKE_*`)*

## Что сейчас запускает CI

Файл: `.github/workflows/ci.yml`

Pipeline сейчас включает:
- проверку обязательного completion gate для agent entry points;
- Python lint baseline (`ruff`);
- frontend lint baseline (`eslint src`);
- targeted backend pytest;
- полный `admin_service/tests` на Python 3.12, совпадающем с production image;
- `admin-web` build + Vitest;
- Playwright smoke (`acl-navigation.spec.ts`).
- Docker build sanity для трёх основных сервисов.

Это не «максимальный» regression suite, а базовый защитный контур, который должен оставаться быстрым и стабильным.

## Что проверяем

- Идемпотентность обработки апдейтов (повторный update не ломает состояние)
- Статусы строк заказа (`ok/unknown_item/bad_qty/stopped`)
- Формирование ответов/команд
- Экспорт создаёт xlsx и содержит ожидаемые листы/минимум данных
- Owner-only и role-based видимость разделов в админке не ломается
- Settings tabs и антиспам-действия (load/delete/dismiss) работают стабильно (RTL tests)

## Примечания

- Для локального запуска тестов обычно нужен `PYTHONPATH=...` (см. примеры в `README.md` и в истории команд).

### [[10-ops-runbook]]
