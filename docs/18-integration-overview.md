# 18. Интеграционный контур проекта
## Состав интеграционного раздела документации

Ключевые связанные документы:

- [[18-integration-overview]] — этот обзор и карта раздела;
- [[19-bot-architecture]] — архитектура текущего бота и модель адаптации под Matrix;
- [[20-api-and-business-flows-for-mobile-integration]] — API, ограничения и бизнес-процессы интеграции;
- [[22-messenger-agnostic-integration-spec]] — универсальный контракт интеграции мессенджеров;
- [[14-api-reference]] — полный reference по реализованным API;
- [[17-admin-service-endpoints-manual]] — прикладной manual по endpoint-ам `admin_service`.

Дополнительный технический слой:

- [[15-endpoint-consumers-matrix]].

## Тематическая карта раздела

### Архитектура бота и адаптация под провайдеры

- [[19-bot-architecture]]
- [[22-messenger-agnostic-integration-spec]]
- [[23-provider-bridge-runbook]]

Покрывает:

- стек и библиотеки;
- модульную архитектуру runtime;
- границы переиспользования domain-core;
- transport-адаптацию для Matrix/VK/MAX.

### API и бизнес-процессы интеграции

- [[20-api-and-business-flows-for-mobile-integration]]
- [[14-api-reference]]
- [[17-admin-service-endpoints-manual]]
- [[13-integration-api]]

Покрывает:

- полный API-контур по сервисам;
- последовательности вызовов по ключевым процессам;
- ролевые ограничения и scope;
- текущие границы read/write контрактов.

## Архитектурное резюме

### 1. Архитектура текущего бота

Текущая логика бота пригодна для переиспользования, но **не Telegram transport-слой**.

Переиспользуемые части:

- доменная логика заказов;
- парсер сообщений и нормализация позиций;
- модели данных и хранение заказов/строк/каталогов/точек выдачи;
- бизнес-правила модерации, статусов и выдачи;
- административный API и web-админка.

Части, которые не переносятся напрямую в Matrix:

- Telegram webhook/update format;
- Telegram Bot API вызовы (`sendMessage`, `sendDocument`, reactions, callback buttons);
- Telegram-специфичные inline-кнопки, callback query, deep links и DM-flow.

Итог: для Matrix нужен **новый adapter / integration layer**, который будет работать с комнатами, событиями и отправкой сообщений Matrix / Maubot, но поверх уже существующего order-domain.

Подробности: [[19-bot-architecture]].

### 2. API и интеграция мобильного клиента

В проекте уже есть:

- `backend` — контур приёма сообщений/ивентов и обработки заказов;
- `admin_service` — основной REST API для управления пользователями, каталогами, заказами, выдачей, аналитикой и экспортами;
- read-oriented M2M endpoints `/integrations/*` для внешних систем.

Важно: **текущий внешний M2M-контур в основном read-only**. Он покрывает discovery, чаты, открытые каталоги, точки выдачи и выгрузку заказов. Если в FluffyChat планируется полностью нативное создание/изменение заказов через формы, это потребует либо:

1. нового Matrix-бота, который будет передавать структурированный текст/события в существующий order flow;
2. либо отдельного публичного write API для customer/mobile-сценария.

Последовательности вызовов и рекомендуемые integration flows описаны в [[20-api-and-business-flows-for-mobile-integration]].

## Рекомендуемая последовательность изучения

1. [[18-integration-overview]] (обзор)
2. [[19-bot-architecture]] (runtime и transport)
3. [[20-api-and-business-flows-for-mobile-integration]] (API choreography)
4. [[22-messenger-agnostic-integration-spec]] и [[23-provider-bridge-runbook]] (provider contract)
5. [[14-api-reference]], [[17-admin-service-endpoints-manual]], [[13-integration-api]] (детали endpoint-ов)

## Текущее состояние интеграционного контура

### Готово и стабильно

- Python backend с webhook → queue → worker обработкой;
- provider-agnostic ingress `POST /integrations/{provider}/webhook` с envelope-normalization в общий worker pipeline;
- provider-specific ingress parsers для Matrix / VK / MAX / webhook payload;
- capability-driven adapter layer и provider-aware outbound dispatch;
- MySQL / SQLAlchemy модель заказов;
- парсер заказов и нормализация строк;
- admin API для каталогов, заказов, ролей, выдачи, аналитики, экспортов;
- web-админка для операционной работы;
- базовый M2M слой `/integrations/*`;
- adapter classes + bootstrap validation (`telegram`, `webhook`, `matrix`, `vk`, `max`).

### Потребует отдельной адаптации или расширения

- vendor-native connector lifecycle для Matrix / VK / MAX, если требуется обходиться без bridge-слоя;
- сопоставление room / event / user semantics между Matrix и текущей моделью;
- UI-рефакторинг FluffyChat под theme tokens;
- при необходимости — customer/mobile write API для нативных форм заказа.

## Ключевое архитектурное ограничение

На сегодня проект **не публикует отдельный customer-facing write API** уровня «создать заказ из мобильной формы» как внешний стабильный контракт. Основной production-сценарий создания заказа сейчас идёт через бот и внутренний order pipeline.

Это не блокирует интеграцию, но определяет дальнейшее архитектурное решение:

- либо Matrix-бот остаётся основным способом ввода заказа;
- либо в отдельной итерации проектируетcя и стабилизируется новый public write contract.

Именно поэтому `docs/19...` и `docs/20...` следует читать вместе: один документ описывает reusable domain и границы transport-слоя, второй — API и реальные интеграционные сценарии.

## Смежные документы

- [[03-architecture]]
- [[04-data-model]]
- [[06-telegram-integration]]
- [[12-admin-panel]]
- [[14-api-reference]]
- [[22-messenger-agnostic-integration-spec]]

### [[19-bot-architecture]]