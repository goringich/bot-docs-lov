# Known issues

Статус: `current`

Назначение: перечень известных проблем и закрытых дефектов с кратким описанием исправлений.

## Resolved

- ~~(2026-06-18) Dark-mode: белый текст на белом фоне в OrdersPage (raw text), ExcelPreviewPage (header), TemplatesPage (column cards), DashboardPage (QuickStat) — hardcoded hex-цвета~~ → **Закрыто:** все заменены на theme-aware palette tokens (`text.primary`, `grey.900`/`grey.50` по mode, `primary.main`/`primary.contrastText`, `info`/`success`/`warning`).
- ~~(2026-06-18) CatalogsPage: layout overflow при resize — `calc(100vw - 320px)` не учитывал динамическую ширину drawer~~ → **Закрыто:** заменено на `maxWidth: "100%"`.
- ~~(2026-06-18) MainLayout: active menu item с `primary.light` — низкий контраст с белым текстом~~ → **Закрыто:** заменено на `primary.main` / hover `primary.dark`.
- ~~(2026-06-18) Excel-экспорт: отсутствовало профессиональное оформление (нет заголовков, стилей, авто-ширины)~~ → **Закрыто:** полная перезапись с 4 листами, русскими заголовками, blue header, zebra-striping, auto-width, freeze panes.
- ~~(2026-06-18) Settings: хаотичный бесконечный скролл~~ → **Закрыто:** реорганизация в 7 вкладок (Общие, Сообщения, Точки выдачи, Уведомления, Тема, Функционал, Система).
- ~~(2026-06-16) `handlers/` модуль использовал `Catalog.status == "active"` вместо `"open"` — все handler-команды каталогов не находили активные каталоги~~ → **Закрыто:** заменены все 10 вхождений в admin_actions.py и admin_commands.py.
- ~~(2026-06-16) `handlers/admin_commands.py` создавал `CatalogItem(status="active")` — поле не существует в модели~~ → **Закрыто:** заменено на `sku` + `is_active`.
- ~~(2026-06-16) Сломанный emoji U+FFFD (replacement character) в `order_presenter.py` вместо 👤~~ → **Закрыто:** заменён на корректный символ.
- ~~(2026-06-16) Ошибка отступов в `admin_commands.py` строка 317 (`aliases=aliases,`)~~ → **Закрыто:** выравнивание исправлено.
- ~~(2026-06-16) Мёртвый код `_maybe_handle_admin_command()` в `worker.py`~~ → **Закрыто:** удалён.
- ~~(2026-03-09) OrdersPage: `handleExportHistoryXlsx` вложена внутрь `handleExportXlsx` → кнопка «📜 История» не работала~~ → **Закрыто:** функция извлечена как sibling, orphan import `HistoryIcon` исправлен.
- ~~(2026-03-09) CatalogsPage: при смене режима видимости бота Select вызывал `loadChats()` → полный re-render → аккордеон схлопывался~~ → **Закрыто:** optimistic local state update с rollback.
- ~~(2026-01-22) Для работы `/open_app` требуется, чтобы admin_user был создан в БД вручную~~ → **Закрыто:** добавлен `/auth/auto-provision` endpoint, который автоматически создаёт пользователя при первом входе из Telegram.
- ~~(2026-01) Временный стоп `CatalogItem.stop_until` учитывается при обработке заказов, но авто-снятие стопа не выполняется~~ → **Закрыто:** добавлен `services/auto_unstop.py`, worker каждые ~60 сек проверяет и снимает просроченные стопы.
- ~~(2026-01) Admin web пока включает только базовые экраны (login, список админов)~~ → **Закрыто:** добавлены CatalogsPage, UsersPage, расширена аналитика.
- ~~(2026-01-23) Сломанные emoji в клавиатуре бота~~ → **Закрыто:** исправлены в `order_presenter.py`.
- ~~(2026-01) `datetime.utcnow` deprecated~~ → **Закрыто:** заменён на `datetime.now(UTC)` во всех файлах (~25 мест).
- ~~(2026-01) `_format_number` crash в `order_domain.py`~~ → **Закрыто:** исправлено на `format_number`.
- ~~(2026-01) PATCH/DELETE эндпоинты возвращали 200 для несуществующих ресурсов~~ → **Закрыто:** добавлены проверки существования и 404 ответы.
- ~~(2026-01) `load_config()` вызывался ~20 раз за один update~~ → **Закрыто:** заменён на `get_config()` кэшированный singleton.
- ~~(2026-01) Admin-web Dockerfile запускал dev-сервер~~ → **Закрыто:** multi-stage build с nginx.

## Open Issues

- (2026-03-24) **Endpoint `/feedback` not implemented**: MainLayout отправляет POST `/feedback` при сабмите формы «Сообщить об ошибке», но backend endpoint пока не реализован. Frontend graceful-degrade: ошибка молча игнорируется. Следующий шаг — создать endpoint и forward в Telegram DM администратору.
- (2026-03-24) **Sysadmin role must be manually assigned**: Роль `sysadmin` пока создаётся вручную в БД (`INSERT INTO admin_roles`), автоматического provisioning через UI нет.
- (2026-03-18) Универсальный provider-agnostic ingress и bridge-based non-Telegram egress уже реализованы (`POST /integrations/{provider}/webhook` + outbound dispatch через `*_OUTBOUND_URL`), но production-ready vendor-native connectors по-прежнему остаются только для Telegram. Для Matrix/VK/MAX требуется довести transport client, точные provider-specific signature semantics и интеграционные тесты capability profile до production уровня.

- (2026-03-18) Для внешней mobile / Matrix интеграции пока нет отдельного публичного customer-facing write API уровня «создать/изменить заказ из нативной формы». Текущий внешний контур `/integrations/*` ориентирован на чтение, а production-создание заказа идёт через bot/message pipeline. Для FluffyChat/Matrix это означает архитектурный выбор: либо новый adapter reuse поверх существующего message-driven flow, либо отдельная итерация на проектирование стабильного write contract.

- (2026-03-14) `admin-web` vitest: часть сценариев `SettingsPage.test.tsx` всё ещё ожидает старые вкладки/лейблы (`Безопасность` и прежнюю структуру секций), поэтому targeted run на `SettingsPage` падает даже при успешной сборке и при исправных текущих layout/style изменениях. Нужна синхронизация тестовых ожиданий с актуальной IA страницы настроек.

- (2026-07-17) Backend integration tests (admin_service) падают на Python 3.14 из-за несовместимости `bcrypt 5.x` + `passlib`: `ValueError: password cannot be longer than 72 bytes`. Причина — `passlib.hash.bcrypt` проверяет `bcrypt.__about__.__version__`, которого нет в bcrypt 5.x, и fallback-ветка некорректно кодирует пароль. **Workaround:** `pip install bcrypt==4.2.1` или заменить passlib на чистый bcrypt. Не блокирует unit-тесты и runtime — только тесты, вызывающие `hash_password` через conftest `seed_owner`.

- (2026-07-16) Парсер: `extract_customer_name()` не принимает `known_places` — использует hardcoded set. Если точка выдачи не в hardcoded списке, её название может быть принято за имя клиента. Рекомендуется передавать known_places в функцию.
- (2026-07-16) Парсер: составные адреса вроде "Иваново 3- Межевая" определяются как meta только если "Иваново" или "Межевая" есть в known_places. Без этого линия будет разобрана как продукт с qty=3.
- (2026-07-16) Парсер: "НН" (Нижний Новгород) как префикс города перед точкой ("ННВанеева") не обрабатывается как отдельный паттерн.
- (2026-07-15) Delivery-tracking миграция (`e4g5h6i7j8k9_add_delivery_tracking.py`) добавлена, но ещё не применена на production. Нужно выполнить `alembic upgrade heads` и перезапустить admin_service для подхвата новых таблиц.
- (2026-07-15) `delivery_status` на заказах не влияет на логику бота (worker.py). Статус выдачи — чисто административная пометка для pickup-админов. Если потребуется автоматика (например, уведомление покупателя о выдаче), нужно расширить worker.
- (2026-03-13) Поля `что выдано / что осталось` в модалке выдачи пока сохраняются как текстовые snapshots по строкам заказа. Это удобно для операторов, но не даёт строгой числовой валидации «остаток + выдано = исходное количество» для всех форматов qty (`1 тушка`, `0.5 кг`, `2 уп`). Для полной консистентности нужен отдельный qty-normalization слой именно для delivery UI.

- ~~(2026-07-15) Дублирование маршрута `GET /pickup-places`: и `settings_router`, и `integrations_router` определяют этот эндпоинт (первый с JWT, второй с service key). Функционально корректно, но может вызвать путаницу — рекомендуется вынести в общий helper.~~ → **Закрыто (2026-03-13):** обе ручки теперь используют общий helper `list_pickup_places_data()`, логика chat/title/aliases не дублируется.

- (2026-03-13) Backend endpoint `GET /orders/export-history` сохранён для audit/integration сценариев, хотя отдельная кнопка в UI убрана. Это осознанное разделение operator UX и audit API, но старые внутренние инструкции могут ещё ссылаться на «кнопку истории».

- (2026-03-13) Legacy локальные Python-окружения могут оставаться несогласованными, если запускать проект в обход каноничного bootstrap flow. Базовый путь теперь стандартизирован через `bash scripts/setup_python_envs.sh all` / `make py-envs`, но старые вручную собранные `.venv`/системные интерпретаторы всё ещё могут вести себя по-разному, пока пользователь не пересоберёт env по новому сценарию.

- (2026-03-11) Часть исторических API-описаний в старых документах может отставать от фактических route-ов. В качестве источника истины используем [[14-api-reference]] + OpenAPI (`/openapi.json`) каждого сервиса.

- (2026-01) Wizard для добавления товара реализован как текстовый шаблон команды, а не полноценный пошаговый callback-flow.
- ~~(2026-01) Качество распознавания единиц/фасовок ограничено: сложные случаи вроде `2×250г => 0.5 кг` пока не интерпретируются автоматически.~~ → **Закрыто (2026-06-17):** `parse_qty()` вычисляет итоговый вес (`2×250г → 500 г`), авто-конвертация г→кг при ≥1000г. Расширены паттерны unit на ~60 вариантов (коробка, баночка, стакан, ведро и т.д.).
- ~~(2026-01) Точки выдачи: нет подсказок/выбора при опечатках~~ → **Закрыто:** добавлен fuzzy matching с Levenshtein distance (25% допуск).
- ~~(2026-01-21) Telegram API имеет rate limit (~30 сообщений/сек). При массовой обработке старых обновлений может возникать HTTP 429.~~ → **Закрыто (2026-06-18):** добавлен `rate_limiter.py` с token-bucket (global 25 tok/sec + per-chat 1 tok/sec), интегрирован в `_send_user_reply()` worker.
- (2026-06) Rate limiting intentionally lives in-memory for single-instance outbound Telegram protection. Это нормально для одного bot worker/process, даже при больших чатах, потому что лимит считается по скорости исходящих сообщений, а не по числу участников. Но при горизонтальном масштабировании (несколько worker/process/replica) нужен Redis/shared rate limiter, иначе инстансы не будут видеть лимиты друг друга.
- (2026-06) `worker.py` монолитный (2000+ строк) — рефакторинг на handler-классы не завершён.
- (2026-01-21) ngrok требует валидный authtoken из `.env`. При его отсутствии контейнер будет бесконечно перезапускаться с ошибкой ERR_NGROK_4018.
- ~~(2026-01) `excel_export.py`: в режиме `pickup` всё равно загружаются и обрабатываются все order_lines (лишняя работа, не критично).~~ → **Закрыто:** добавлен `continue` для pickup mode, пропуск загрузки order_lines и catalog_items.
- ~~(2026-06-15) Парсер: сложные комбо-заказы со смешанным форматом (часть строк — товары, часть — комментарии) могут дать false positive на строки-комментарии.~~ → **Закрыто (2026-06-17):** умная предобработка (`_smart_preprocess_text`): strip emoji bullets, разделение слитых полей (имя+телефон, телефон+место), нормализация `=` как разделителя, расширены `_PRODUCT_KEYWORDS` на ~30 новых записей с учётом опечаток.
- ~~(2026-06-15) Парсер: телефон, слитый с точкой выдачи без пробела (`8086Ванеева`), определяется, но pickup place может не распознаться без DB fuzzy lookup.~~ → **Закрыто (2026-06-17):** `_smart_preprocess_text` теперь разделяет `8086Ванеева` → `8086 Ванеева`, `_PHONE_GLUED_RE` + `_PHONE_5D_RE` корректно извлекают 4-значные и 5-значные телефоны.
- ~~(2026-06-15) Analytics fallback: при недоступности API фронтенд показывает mock-данные без индикации (нет предупреждения пользователю).~~ → **Закрыто (2026-06-17):** добавлен `isMockData` state + `<Alert severity="warning">` баннер "⚠️ API аналитики недоступен — отображаются демо-данные" в AnalyticsPage.
- ~~(2026-03-11) Исторически испорченные заказы попадают в шаблонный Excel как есть: например, у заказа `#40` quantity `6210 кг` у `Филе трески проложное` отражает старую parser/data ошибку, а не баг текущего экспорта.~~ → **Закрыто (2026-06-17):** добавлены sanity-пороги (`_MAX_SANE_QTY`) в `excel_export.py` — строки с абсурдными qty (>100 кг, >200 шт и т.д.) исключаются из итоговых сумм.
- ~~(2026-07-15) Парсер: телефонные номера / длинные числа (4424, 89123456789) интерпретировались как qty, давая абсурдные результаты (напр. 4424000 кг).~~ → **Закрыто:** добавлены sanity-пороги `_MAX_SANE_QTY` в `text_parser.py` + guard для implicit single-unit regex. Числа сверх порога возвращают `(None, None, None)`.
- ~~(2026-06-18) Аналитика ограничена до owner, но полноценная система «uowner сам назначает какие разделы видны каким ролям» пока не реализована — требуется UI для per-role section permissions.~~ → **Закрыто (2026-06-18):** реализована полная система section visibility с `SectionPermissionsContext`, 10 `section_vis_*` настроек, UI вкладка "Доступ" в Settings.
- ~~(2026-06-17) Парсер: сообщения-отзывы с конкретным весом ("Рыбина на 3.850 кг. с бонусом") детектируются как заказ — нет семантического анализа intent (feedback vs order).~~ → **Закрыто (2026-03-13):** в `looks_like_order()` добавлен guard для qty-only сообщений без product keyword и без клиентской идентификации.
- ~~(2026-07-15) Быстрые кнопки выдачи в OrdersPage работали только для заказов со статусом `active`.~~ → **Закрыто (2026-03-13):** quick-actions теперь доступны и для `needs_admin` / `partial`; статус `transferred` убран из one-click сценария и ведёт в полноценный delivery flow с обязательными данными нового получателя.
- ~~(2026-06) `admin_user_regions` таблица создана, но UI для назначения регионов regional_admin пока не реализован.~~ → **Закрыто (2026-03-13):** в `UsersPage` добавлена колонка `Города / регионы`, отдельный editor и backend-scope по пересечению регионов.
- (2026-06-17) Парсер всё ещё использует эвристику, а не полноценный intent-анализ: сложные отзывы/жалобы с явными названиями товаров могут потребовать дополнительного NLP-слоя или whitelist/blacklist паттернов. Конкретный кейс без product keyword (`"Рыбина на 3.850 кг. с бонусом"`) уже закрыт, но класс задачи шире.

## Workarounds

### Rate limiting от Telegram API
При ошибках 429 Too Many Requests:
```sql
-- Очистить очередь необработанных updates
UPDATE tg_updates SET status = 'done', error_text = 'rate_limit_skip' 
WHERE status IN ('new', 'processing');
```

### ngrok authtoken
1. Зарегистрируйтесь на https://ngrok.com
2. Скопируйте authtoken из dashboard
3. Добавьте в `.env`: `NGROK_AUTHTOKEN=your_token_here`
4. Перезапустите: `make compose-restart`

