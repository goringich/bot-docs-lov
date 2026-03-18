# 24. Карта интеграционного комплекта документации

Дата: 2026-03-18

Этот файл фиксирует состав интеграционного комплекта документации по трём направлениям:

1. архитектура бота и provider-адаптация;
2. API и бизнес-процессы;
3. темизация и UI-рефакторинг FluffyChat.

## Назначение

Документ нужен как навигационная карта раздела:

- показывает, какие материалы описывают архитектуру бота;
- показывает, какие материалы описывают API и бизнес-процессы;
- показывает, где зафиксированы рамки темизации и UI-рефакторинга.

Это не отдельная спецификация, а нейтральный index по связанным документам.

## 1) Состав документов

### A. Архитектура бота и provider-адаптация

1. [[19-bot-architecture]]
2. [[22-messenger-agnostic-integration-spec]]
3. [[23-provider-bridge-runbook]]

Содержательное покрытие:

- технологии и библиотеки;
- внутренняя архитектура, модули, команды/сценарии;
- почему Telegram transport не переносится напрямую;
- как подключать Matrix/VK/MAX через provider bridge.

### B. API и бизнес-процессы

4. [[20-api-and-business-flows-for-mobile-integration]]
5. [[14-api-reference]]
6. [[17-admin-service-endpoints-manual]]
7. [[13-integration-api]]

Содержательное покрытие:

- карта endpoint-ов;
- последовательности вызовов по бизнес-процессам;
- роли, scope и ограничения;
- текущее состояние read/write контуров.

### C. Темизация и UI-рефакторинг FluffyChat

8. [[21-fluffychat-theming-and-ui-refactor-scope]]

Содержательное покрытие:

- обоснование рефакторинга как инженерного scope;
- target architecture для theme/token layer;
- acceptance criteria и границы этапов.

### D. Обзорный входной документ

9. [[18-integration-overview]]

Это обзорная входная точка: объясняет, что читать и в каком порядке.

---

## 2) Рекомендуемый порядок чтения

1. [[18-integration-overview]]
2. [[19-bot-architecture]]
3. [[20-api-and-business-flows-for-mobile-integration]]
4. [[21-fluffychat-theming-and-ui-refactor-scope]]
5. [[22-messenger-agnostic-integration-spec]]
6. [[23-provider-bridge-runbook]]
7. [[14-api-reference]]
8. [[17-admin-service-endpoints-manual]]
9. [[13-integration-api]]

## 3) Какой комплект закрывает какие вопросы

### Если нужно понять архитектуру текущего бота и границы адаптации под Matrix

Читать:

1. [[19-bot-architecture]]
2. [[22-messenger-agnostic-integration-spec]]
3. [[23-provider-bridge-runbook]]

### Если нужно понять API, роли и последовательности вызовов

Читать:

1. [[20-api-and-business-flows-for-mobile-integration]]
2. [[14-api-reference]]
3. [[17-admin-service-endpoints-manual]]
4. [[13-integration-api]]

### Если нужно оценить scope темизации и UI-рефакторинга FluffyChat

Читать:

1. [[21-fluffychat-theming-and-ui-refactor-scope]]

## 4) Практически достаточный набор для технической оценки

Если нужен один самостоятельный комплект, который закрывает архитектуру, API, бизнес-flow и scope UI-рефакторинга, достаточно следующих документов:

1. [[18-integration-overview]]
2. [[19-bot-architecture]]
3. [[20-api-and-business-flows-for-mobile-integration]]
4. [[21-fluffychat-theming-and-ui-refactor-scope]]
5. [[22-messenger-agnostic-integration-spec]]
6. [[23-provider-bridge-runbook]]
7. [[14-api-reference]]
8. [[17-admin-service-endpoints-manual]]
9. [[13-integration-api]]
