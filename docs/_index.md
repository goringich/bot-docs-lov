---
title: Otlichniy Ulov — Map of Content
type: moc
status: current
tags: [moc, hub, project/otlichniy-ulov]
updated: 2026-07-25
---

# Otlichniy Ulov — карта документации

> [!info] Где смотреть
> Эти документы живут в репозитории `otlichniy-ulov/docs/` и одновременно отображаются в Obsidian под `/Отличный улов/docs/` через [[obsidian-repo-mounts]] (bind-mounts, не копии и не симлинки). Любая правка в одном месте — это правка в другом.

> [!important] Правила работы с базой знаний
> 1. Перед началом работы с парсером, каталогом, заказами или экспортом — открой [[28-stability-playbook]] и [[changelog/codex-project-memory]].
> 2. Не строй догадки о бизнес-логике из шаблона Excel или из текста сообщения клиента — открывай [[26-export-contract]] и каталог.
> 3. Любая правка контракта (рейт-лимит, шаблон экспорта, формат API) обязана сопровождаться обновлением соответствующего документа в этой папке.

## 🧭 Точка входа

- [[00-project-brief]] — зачем продукт существует, основной сценарий, входы.
- [[01-requirements]] — обязательные инварианты.
- [[02-glossary]] — словарь терминов (catalog, order, line, distribution, pickup, raw_text, …).
- [[25-project-handbook]] — справочник по компонентам.
- [[30-user-flows]] — пользовательские потоки по ролям *(новое)*.

## 🏗 Архитектура и данные

- [[03-architecture]] — gateway / worker / admin / web разделение.
- [[04-data-model]] — таблицы `chats / catalogs / catalog_items / orders / order_lines / order_deliveries / bot_settings`.
- [[19-bot-architecture]] — структура Telegram-бота.
- [[06-telegram-integration]] — webhook, worker, ответный канал.

## 📦 Каталог и парсер (source of truth)

- [[05-parsing-rules]] — что распознаём, что нет, лимиты количества.
- [[26-catalog-source-of-truth]] — почему каталог нельзя «доучивать» из заказов *(новое)*.
- [[32-parser-health-pipeline]] — typed parser, diagnostics, shadow AI, repair queue, reparse и final preflight.
- [[changelog/codex-project-memory]] — короткая «живая» память по парсеру и каталогу.

## 📝 Заказы

- [[28-stability-playbook]] — правила стабильности заказов *(новое)*.
- [[32-parser-health-pipeline]] — health monitoring и zero-unresolved lifecycle.
- [[08-admin-flows]] — операционные сценарии (открытие/закрытие раздачи, ручной импорт каталога).
- [[12-admin-panel]] — карта `admin-web`, разделы и роли.

## 📤 Excel-экспорт

- [[07-excel-export]] — обзор всех режимов и реализаций экспорта.
- [[26-export-contract]] — **строгий контракт** strict/distribution экспорта *(новое)*.
- [[changelog/known-issues]] — открытые проблемы экспорта.

## 🔐 Безопасность

- [[11-security]] — базовые требования по секретам и окружению.
- [[24-admin-auth-and-integration-security-runbook]] — auth/CSRF/cookie runbook.
- [[27-security-runbook]] — собранный чек-лист для PR в auth/middleware/export *(новое)*.
- [[31-rate-limit-buckets]] — карта рейт-лимитов middleware *(новое)*.

## 🛠 Эксплуатация

- [[10-ops-runbook]] — запуск, обновления, проверки.
- [[11-restart]] — короткий рестарт-чеклист.
- [[23-provider-bridge-runbook]] — мост к провайдерам.

## 🌐 Интеграции и API

- [[13-integration-api]] · [[14-api-reference]] · [[15-endpoint-consumers-matrix]]
- [[17-admin-service-endpoints-manual]] · [[18-integration-overview]]
- [[20-api-and-business-flows-for-mobile-integration]]
- [[22-messenger-agnostic-integration-spec]]

## 🤖 AI агенты и эта база знаний

- [[29-ai-knowledge-base]] — как Claude / Copilot должны читать эту папку *(новое)*.
- [[33-task-completion-gate]] — доказуемое завершение задачи: проверки, live-результат и честный `BLOCKED`.
- `../AGENTS.md` — глобальный prompt-runbook для Codex/Claude.
- `../.github/copilot-instructions.md` — инструкции для GitHub Copilot.
- `../CLAUDE.md` — точка входа Claude Code в репозиторий.

## 🧪 Качество и тесты

- [[09-testing-strategy]] — какие тесты обязательны для каждой подсистемы.
- [[28-stability-playbook#тестовый-минимум]] — pre-merge чек-лист.
- [[33-task-completion-gate]] — terminal gate перед заявлением «готово».

## 📜 История и решения

- [[changelog/codex-project-memory]] · [[changelog/decisions-log]] · [[changelog/implementation-status]] · [[changelog/known-issues]]
- [[PRODUCTION_ROADMAP]] · [[PRODUCTION_UPGRADE_v2.0]]

---

> [!tip] Соглашения этой базы
> - Заголовки секций — `## 🏷 Название` (с emoji-категорией) для быстрого визуального скана.
> - Связки — Obsidian-style: `[[имя-файла]]` (без `.md`), якоря — `[[имя-файла#заголовок]]`.
> - Callouts — `> [!note]`, `> [!important]`, `> [!warning]`, `> [!tip]`, `> [!danger]`.
> - Frontmatter — `title / type / status / tags / updated`.
> - Имя файла — `NN-kebab-case.md`, где `NN` — стабильный префикс. Не переименовывай существующие — добавляй новый префикс.
