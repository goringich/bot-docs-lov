# Implementation Status

_Актуально на 2026-03-13._

Статус: `current`

Назначение: снимок фактического состояния проекта на указанную дату. Используется как рабочий reference по реализованным частям системы.

Этот файл — **текущий снимок состояния проекта**, а не исторический отчёт о январском апгрейде. Если какой-то старый markdown спорит с этим файлом, верим этому файлу, [[14-api-reference]] и фактическому коду.

## Что реально уже сделано

### Admin Web: рабочее место админов уже не «базовое»

**Location**: `admin-web/src/`

Фактически присутствуют страницы:
- `DashboardPage.tsx`
- `OrdersPage.tsx`
- `DeliveryPage.tsx`
- `CatalogsPage.tsx`
- `AnalyticsPage.tsx`
- `UsersPage.tsx`
- `SettingsPage.tsx`
- `RegisterPage.tsx`
- `PickupAdminPage.tsx`
- `DebugPage.tsx`

Что это означает на практике:
- есть RBAC-маршрутизация по ролям;
- есть section-level visibility через настройки `section_vis_*`;
- есть отдельный delivery-flow для `pickup_admin`;
- есть UI для пользователей, ролей, регионов, сброса пароля и owner-promotion;
- есть настройки, шаблоны Excel, аналитика, каталоги и рабочий экран заказов.

### Admin Service: API уже модульный, а не «несколько ручек в main.py»

**Location**: `admin_service/app/api/routers/`

Фактически выделены router-модули:
- `auth.py`
- `users.py`
- `roles.py`
- `catalogs.py`
- `orders.py`
- `deliveries.py`
- `analytics.py`
- `settings.py`
- `admin_settings.py`
- `templates.py`
- `integrations.py`
- `debug.py`
- `spam.py`

Дополнительно в `admin_service/app/main.py` уже есть:
- rate limiting middleware;
- security headers middleware;
- audit logging middleware;
- startup refresh core tables;
- CORS и lifespan-инициализация.

### Backend / bot-flow: продвинутые улучшения уже в коде

**Location**: `backend/app/`, `backend/tests/`

Подтверждено по структуре репозитория и known issues:
- улучшен парсинг qty / телефонов / pickup-place;
- есть авто-снятие временного stop;
- есть rate limiter на отправку в Telegram;
- Excel export существенно усилен;
- есть delivery tracking и pickup-flow;
- есть AI-модуль, но он пока не полностью встроен в основной runtime-flow.

### Тесты уже есть, это больше не «0 автотестов»

Найдено в репозитории:
- `backend/tests/*.py` — парсер, excel, spam, worker, replay/smoke и др.;
- `admin_service/tests/*.py` — API, startup regressions, settings keys, spam router;
- `admin-web/e2e/*.ts` — Playwright smoke / ACL / settings-spam сценарии.

Итого: автотесты **есть**, и теперь есть базовый CI-пайплайн в `.github/workflows/ci.yml` с Python lint baseline, frontend lint baseline, тестами и Docker build sanity.

## Что сделано частично

### AI fallback и AI-настройки

Сейчас:
- AI-модуль и сервисный слой существуют;
- но AI не является обязательной частью production-flow в `worker.py`;
- архитектура сохранена так, чтобы позже можно было подключить **локальный/offline provider** без зависимости от внешних сервисов;
- нет подтверждённого production-ready UI/endpoint-слоя для безопасного управления AI-параметрами и секретами.

### Редактирование заказов и каталога

Сейчас:
- UI уже умеет операционные действия, фильтры, выдачу, быстрые действия;
- но полноценный line-by-line editor заказа и полноценный CRUD-редактор каталога как отдельный advanced workflow ещё не зафиксированы как завершённые.

### Тестирование и DevOps

Сейчас:
- локальные тесты есть;
- базовый CI уже добавлен (`.github/workflows/ci.yml`), включая Python lint baseline, frontend lint baseline и Docker build sanity;
- Docker/compose и ops-docs есть;
- но полноценного scheduled backup-пайплайна ещё нет; пока добавлен ручной reproducible backup/restore baseline через project scripts.

## Что ещё не закрыто ❌

### Открытые технические и операционные пункты

1. **Применить delivery-tracking миграцию на production**  
    См. [[changelog/known-issues]]: требуется `alembic upgrade heads` + рестарт `admin_service`.

2. **Починить локальное Python test-окружение**  
    Сейчас `.venv` и системный Python рассинхронизированы, из-за чего локальные targeted тесты запускаются нестабильно.

3. **Расширить CI после базового baseline**  
    Базовый pipeline уже есть; следующим шагом полезно расширить lint coverage и более широкие regression suites.

4. **Автоматизировать scheduled backups поверх нового baseline**  
    Ручные backup/restore scripts уже есть, но cron/systemd/облачное хранилище ещё не собраны в production flow.

5. **Доделать строгую qty-normalization для delivery UI**  
    Сейчас поля `что выдано / что осталось` работают как текстовые snapshots, без полной числовой инвариантности.

6. **Не форсировать AI в production-path до появления локального провайдера**  
    Сейчас решение принято: AI остаётся optional feature, а основной flow — regex-first/offline-safe.

7. **Решить судьбу in-memory rate limiting**  
    Для single-instance ок, для горизонтального масштабирования нужен Redis/shared backend.

## Что делать следующим — короткий и приземлённый список

### Ближайший practical backlog

1. Применить pending migration на production и отметить это в ops-docs.
2. Починить reproducible local test environment.
3. Расширить `.github/workflows/ci.yml` до более широкого lint coverage + более полного regression coverage.
4. Выбрать один из двух путей:
    - либо full order/catalog editing как следующая бизнес-фича,
    - либо production-hardening: backup + CI + env cleanup.
5. Если возвращаться к AI, то только через local/offline provider или другой self-hosted вариант, а не как обязательную SaaS-зависимость.

## Анти-путаница

Следующие старые формулировки больше не считаем актуальными:
- «Admin web пока включает только базовые экраны»
- «Users page ещё нет»
- «Автотестов нет»
- «Admin API — это несколько новых ручек в `main.py`»

Они были верны исторически, но уже не соответствуют текущему состоянию репозитория.
