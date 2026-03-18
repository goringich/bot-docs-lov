# Docs index

Быстрый вход в документацию проекта.

## Obsidian navigation

- [[README]]
- [[18-integration-overview]]
- [[24-integration-docs-map]]
- [[changelog/implementation-status]]
- [[changelog/decisions-log]]

## Обязательные стартовые документы

- [[00-project-brief]]
- [[03-architecture]]
- [[04-data-model]]
- [[10-ops-runbook]]
- [[11-security]]

## API и интеграции

- [[12-admin-panel]] — человеческое описание админки, ролей и операционного контура
- [[18-integration-overview]] — обзор интеграционного контура проекта
- [[19-bot-architecture]] — архитектура бота и адаптация под Matrix
- [[20-api-and-business-flows-for-mobile-integration]] — интеграционные API и бизнес-процессы
- [[21-fluffychat-theming-and-ui-refactor-scope]] — архитектурные рамки темизации и UI-рефакторинга FluffyChat
- [[22-messenger-agnostic-integration-spec]] — универсальный контракт интеграции для Telegram/Matrix/VK/MAX
- [[23-provider-bridge-runbook]] — практический ingress/egress runbook для Matrix/VK/MAX/Webhook bridge
- [[14-api-reference]] — полная карта API по микросервисам
- [[13-integration-api]] — M2M API (`/integrations/*`)
- [[15-endpoint-consumers-matrix]] — кто использует какие endpoint-ы
- [[16-endpoints-handbook-for-share]] — подробное описание каждого запроса

## Готовые PDF для отправки

- [[export/16-endpoints-handbook-for-share.pretty.pdf]] — полный «красивый» справочник
- [[export/16-endpoints-handbook-for-share.external.pretty.pdf]] — версия для внешних партнёров (без debug/dev-mode)

Генератор: `scripts/maintenance/generate_api_pdf.py`.

## Changelog / решения

- [[changelog/decisions-log]]
- [[changelog/known-issues]]
- [[changelog/implementation-status]] — актуальный статус: что уже реализовано, что частично, что осталось

## Статус и roadmap

- [[changelog/implementation-status]] — текущий снимок проекта на сегодня
- [[ROADMAP_IDEAL_STATE]] — оставшиеся задачи до ideal state
- [[upgrade/FINAL_REPORT]] — исторический отчёт по январскому upgrade, полезен как архив, но не как source of truth
