# Docs Registry

Строгий реестр файлов в `docs/`.

## Правила чтения

| Поле | Значение |
|---|---|
| `Тип` | `overview`, `spec`, `reference`, `runbook`, `guide`, `adr`, `roadmap`, `changelog`, `report`, `archive` |
| `Статус = current` | использовать как рабочее описание текущей системы |
| `Статус = historical` | использовать только как исторический контекст |
| `Статус = target` | описывает целевое состояние или план, а не текущую реализацию |
| `Статус = decision` | фиксирует принятое архитектурное решение |

## Приоритет документов

1. Текущие `handbook` / `reference` / `runbook` документы.
2. `changelog/implementation-status.md` как снимок фактического состояния.
3. `adr/*` как источник архитектурных решений.
4. `roadmap`, `upgrade/*`, `PRODUCTION_*` только как план или исторический контекст.

## Корень `docs/`

| Файл | Тип | Статус | Описание |
|---|---|---|---|
| `00-project-brief.md` | overview | current | Краткое описание продукта, основного сценария работы и состава входных данных заказа. |
| `01-requirements.md` | spec | current | Базовые требования к системе: цель, сущности, масштаб, ограничения и обязательное поведение. |
| `02-glossary.md` | reference | current | Словарь терминов проекта и базовых сущностей домена. |
| `03-architecture.md` | spec | current | Высокоуровневая архитектура: компоненты, поток обработки сообщений, роли gateway и worker. |
| `04-data-model.md` | reference | current | Описание текущей схемы данных и основных моделей backend. |
| `05-parsing-rules.md` | spec | current | Правила распознавания заказов из текста, ограничения парсера и связанные тесты. |
| `06-telegram-integration.md` | reference | current | Контур интеграции с Telegram Bot API: webhook, worker и обратные ответы. |
| `07-excel-export.md` | reference | current | Все режимы Excel/XLSX-экспорта, их реализации и назначение. |
| `08-admin-flows.md` | guide | current | Операционные сценарии администраторов и правила работы с каталогами и доступами. |
| `09-testing-strategy.md` | guide | current | Правила запуска тестов и перечень критичных зон для проверки. |
| `10-ops-runbook.md` | runbook | current | Операционные процедуры запуска, обновления и проверки системы. |
| `11-restart.md` | runbook | current | Короткая памятка по перезапуску сервисов после изменений. |
| `11-security.md` | guide | current | Базовые требования по секретам, окружению и безопасной эксплуатации проекта. |
| `12-admin-panel.md` | overview | current | Функциональная карта `admin-web`, роли, ограничения разделов и поведение интерфейса. |
| `12-bot-improvements-analysis.md` | archive | historical | Анализ возможных улучшений бота и UX-наблюдений на момент составления заметки. |
| `13-integration-api.md` | archive | historical | Исторический интеграционный API-обзор; актуальные данные вынесены в `14` и OpenAPI. |
| `14-api-reference.md` | reference | current | Сводный перечень фактически реализованных API по сервисам. |
| `15-endpoint-consumers-matrix.md` | reference | current | Матрица соответствия endpoint-ов и их потребителей. |
| `16-endpoints-handbook-for-share.md` | reference | current | Полный прикладной справочник endpoint-ов для передачи внешней команде. |
| `17-admin-service-endpoints-manual.md` | reference | current | Практический manual по endpoint-ам `admin_service` и способам их вызова. |
| `18-integration-overview.md` | overview | current | Входной обзор интеграционного контура и связанных документов. |
| `19-bot-architecture.md` | spec | current | Текущая архитектура бота и подход к адаптации под внешние мессенджеры. |
| `20-api-and-business-flows-for-mobile-integration.md` | spec | current | Интеграционные API и последовательности вызовов для mobile и внешних клиентов. |
| `22-messenger-agnostic-integration-spec.md` | spec | current | Универсальный provider-agnostic контракт интеграции мессенджеров. |
| `23-provider-bridge-runbook.md` | runbook | current | Практический runtime-контракт для ingress/outbound bridge внешних провайдеров. |
| `24-admin-auth-and-integration-security-runbook.md` | runbook | current | Актуальная схема аутентификации `admin_service`, one-time login и M2M security. |
| `24-integration-docs-map.md` | overview | current | Карта интеграционного комплекта документов и рекомендуемый порядок чтения. |
| `25-project-handbook.md` | overview | current | Сквозной handbook по системе: домен, сервисы, потоки данных, роли и границы ответственности. |
| `_index.md` | moc | current | **Map of Content (Obsidian).** Точка входа в базу знаний; навигация по всем доменам проекта. |
| `26-export-contract.md` | spec | current | Строгий контракт Excel-экспорта (template / strict / distribution): фиксированные колонки, привязка к датам каталога, безопасность. |
| `26-catalog-source-of-truth.md` | spec | current | Каталог как единственный источник истины: какие правки и доучивания запрещены. |
| `27-security-runbook.md` | runbook | current | Сборный security-чеклист: auth, CSRF, экспорт, sysadmin-тогглы, рейт-лимит. |
| `28-stability-playbook.md` | runbook | current | Чек-лист стабильности orders/parser/catalog/export — обязателен перед закрытием задачи. |
| `29-ai-knowledge-base.md` | guide | current | Как Claude / Copilot / Codex должны пользоваться этой папкой и Obsidian-vault. |
| `30-user-flows.md` | guide | current | Пользовательские потоки по ролям: customer, pickup admin, regional admin, owner, sysadmin. |
| `31-rate-limit-buckets.md` | reference | current | Карта рейт-лимит-бакетов middleware admin_service. |
| `PRODUCTION_ROADMAP.md` | roadmap | target | План перевода системы в production-ready состояние; отражает целевую, а не фактическую реализацию. |
| `PRODUCTION_UPGRADE_v2.0.md` | archive | historical | Исторический checklist апгрейда `v2.0`; не является текущим source of truth. |
| `ROADMAP_IDEAL_STATE.md` | roadmap | target | Перечень оставшихся задач до целевого состояния системы. |
| `STEALTH-MODE-SETUP.md` | guide | current | Инструкция по подключению Telegram-бота в скрытом режиме и режимам его видимости. |
| `roadmap.md` | roadmap | target | Общий план улучшений проекта с разбивкой по этапам и направлениям. |

## `docs/adr/`

| Файл | Тип | Статус | Описание |
|---|---|---|---|
| `adr/0001-telegram-api-choice.md` | adr | decision | Выбор Telegram Bot API, webhook-модели и выноса тяжёлой обработки в worker. |
| `adr/0002-db-choice.md` | adr | decision | Выбор MySQL и базовых технических решений для слоя хранения. |
| `adr/0003-idempotency-model.md` | adr | decision | Модель идемпотентности update-ов, сообщений и `edited_message`. |

## `docs/architecture/`

| Файл | Тип | Статус | Описание |
|---|---|---|---|
| `architecture/microservices.md` | spec | target | Целевая микросервисная архитектура и разделение зон ответственности между сервисами. |

## `docs/changelog/`

| Файл | Тип | Статус | Описание |
|---|---|---|---|
| `changelog/decisions-log.md` | changelog | current | Журнал зафиксированных продуктовых и архитектурных решений по мере развития проекта. |
| `changelog/codex-project-memory.md` | changelog | current | Короткая живая память для Codex: текущие инварианты, последние live-проблемы и ожидания пользователя по parser/catalog/export. |
| `changelog/implementation-status.md` | changelog | current | Снимок текущего фактического состояния проекта и реализованных частей системы. |
| `changelog/known-issues.md` | changelog | current | Список известных проблем и уже закрытых дефектов. |
| `changelog/refactoring-jan2026.md` | changelog | historical | Исторический журнал январского рефакторинга и сделанных структурных изменений. |

## `docs/upgrade/`

| Файл | Тип | Статус | Описание |
|---|---|---|---|
| `upgrade/FINAL_REPORT.md` | report | historical | Итоговый отчёт по historical upgrade `v2.0`. |
| `upgrade/GIT_COMMIT_PLAN.md` | archive | historical | Исторический план коммитов и перечень файлов для upgrade `v2.0`. |
| `upgrade/PROGRESS_VISUALIZATION.md` | archive | historical | Визуальное представление прогресса upgrade `v2.0`. |
| `upgrade/START_HERE.md` | archive | historical | Навигационный индекс по historical upgrade-пакету. |
| `upgrade/UPGRADE_SUMMARY.md` | report | historical | Краткая сводка выполненных изменений в рамках upgrade `v2.0`. |
| `upgrade/WHAT_I_DID.md` | report | historical | Перечень файлов и изменений, внесённых в historical upgrade `v2.0`. |

## Использование реестра

| Задача | Основные файлы |
|---|---|
| Понять систему целиком | `25-project-handbook.md`, `03-architecture.md`, `04-data-model.md` |
| Разобраться с API | `14-api-reference.md`, `16-endpoints-handbook-for-share.md`, `17-admin-service-endpoints-manual.md` |
| Работать с интеграциями | `18-integration-overview.md`, `20-api-and-business-flows-for-mobile-integration.md`, `22-messenger-agnostic-integration-spec.md`, `23-provider-bridge-runbook.md`, `24-admin-auth-and-integration-security-runbook.md` |
| Разобраться с эксплуатацией | `10-ops-runbook.md`, `11-restart.md`, `11-security.md` |
| Проверить текущее состояние | `changelog/implementation-status.md`, `changelog/known-issues.md` |
| Посмотреть историю решений | `adr/*`, `changelog/decisions-log.md`, `upgrade/*` |
