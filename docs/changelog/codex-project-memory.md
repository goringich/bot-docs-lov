# Codex project memory

Статус: `current`

Назначение: короткая живая память для Codex по проекту `otlichniy-ulov`, чтобы новые сессии сразу подхватывали актуальные инварианты, последние проблемные зоны и не требовали повторных пользовательских объяснений.

## Читать в первую очередь, если задача касается

- парсинга заказов;
- каталога как source of truth;
- `partial` / `rejected` / `unknown_item`;
- `export-template*`, `distribution export`, Excel-выгрузок;
- ручного импорта каталога текстом;
- живых расхождений между каталогом, заказами и export.

## Канонические инварианты

- Источник истины для товарного состава: operator-owned source text — ручной
  `import-text` либо, при включённом `feature_catalog_admin_post_sync`,
  подтверждённая admin/sender-chat публикация со строгой формой списка.
- Заказы должны перепарсиваться по текущему каталогу; каталог никогда не
  достраивается из пользовательских заказов, unresolved queue или
  непроверенного `tg_updates`.
- Export строится по уже распарсенным заказам и каталожной семантике, а не по эвристикам Excel-шаблона.
- Excel-шаблон задаёт только сетку, стили и внешнее представление. Он не должен переопределять бизнес-смысл единицы товара.
- Если строка заказа привязана к `catalog_item` / SKU, количество и единица должны интерпретироваться только через `catalog.unit_hint` / `pack_hint`.
- Если товар отсутствует в каталоге, правильное поведение по умолчанию: оставить заказ `partial` / `unknown_item`, а не "умно" допридумать новую позицию.

## Что пользователь хотел в последних сессиях

- Сделать итоговый export "идеальным" именно по цепочке `catalog text -> parsed orders -> export`.
- Избегать хардкода конкретных названий товаров; предпочитать общие правила нормализации и сопоставления.
- Считать проблему в первую очередь source-of-truth проблемой, если export формально строится, но половина строк пустая из-за `unknown_item`.
- При разборе подобных кейсов сначала проверять живые catalog/order/export данные для конкретного чата, а не только unit-тесты.

## Актуальная живая проблема, подтверждённая 2026-04-30

### Балашиха, каталог `id=9`

- Live `export-template-distribution` собирается, `raw_text` попадает в колонку состава заказа, каталожные позиции в принципе раскладываются по колонкам.
- Но проблема остаётся на уровне source-of-truth: из `91` заказа только `44` дали хотя бы одну товарную ячейку в live export, ещё `47` строк ушли без товарного заполнения.
- По `290` `order_lines` только `72` были `ok`, а `218` находились в `unknown_item`.
- Это означает, что главный дефект был не в самой XLSX-механике, а в неполном покрытии каталога относительно реальных заказов.

### Частые live unknown_item для Балашихи

- `Риет с крабом`
- `Ряпушка`
- `Филе Хека`
- `Омуль`
- `Филе минтая`
- `Форель` / `форель свежая` / `Форель охлаждённая`
- `Печень и икра минтая`
- `Дорадо`
- `Чука`
- `Синие мидии`

### Практический вывод

- Если похожая жалоба повторяется, сначала проверить, не неполон ли сам catalog text для конкретного чата.
- Не тратить время на Excel/UI-слой, пока не подтверждено, что нужные товары вообще присутствуют в каталоге или покрыты alias-ами.
- Следующий repair-flow для такого кейса: расширить source catalog или aliases, перепарсить заказы, затем повторить live export и сравнить новую долю заполненных строк.

## MAX-заказы: ingress перед парсером

Проверка 2026-06-22 показала: в live DB было только `5` MAX `tg_updates` (`3` bot_added, `2` message_created) и только `1` заказ `source_provider=max`. Поэтому жалобу вида «не появились заказы из MAX» сначала проверять как ingress/webhook/backfill проблему, а не как catalog coverage.

Текущие MAX-инварианты:
- `MAX GET /messages?chat_id=...` с текущим bot token может вернуть пустую историю даже для известного чата; без payload в `tg_updates` нечего перепарсить.
- На 2026-06-22 dry-run backfill для известных `chat_id=-72153821712958` и `chat_id=888777` вернул `fetched=0`; live DB по-прежнему имела только `1` MAX-заказ, значит «все заказы из MAX» не восстановить без правильного chat_id, доступа бота к истории или внешнего экспорта сообщений.
- На 2026-06-23 `GET /chats` вернул один реальный групповой чат `-72153821712958` («Отличный Улов Балашиха, Железнодорожный»), но `GET /chats/{chatId}/members/me` показал `is_admin=false`, без `read_all_messages`. Поэтому повторное добавление бота и пользовательский переключатель «доступ к истории» сами по себе не дали API-доступ: `GET /messages` по-прежнему вернул `0`.
- Перед backfill проверять фактические права командой `backend/scripts/import_max_messages.py --list-chats`. Для старой истории бот должен отображаться администратором с правом `read_all_messages`.
- Для полного backfill всех доступных групповых чатов использовать `backend/scripts/import_max_messages.py --all-chats --list-chats --dry-run`, затем без `--dry-run`. Скрипт обходит `GET /chats` и историю каждого чата постранично, кладёт сообщения в `tg_updates`, дальше их обрабатывает обычный worker.
- Для точечного backfill остаётся `backend/scripts/import_max_messages.py --chat-id <id> --count 100`; `--chat-id` можно повторять.
- MAX parser нормализует literal `\n` в настоящие переносы до order parser.
- MAX `update_id` должен строиться из распарсенного `message.body.mid` / `event_id`, а не из времени приёма.
- Если worker падает на `Duplicate entry ... user_profiles.tg_user_id`, заказ из update откатывается; user profile upsert должен быть атомарным.
- Если order-like сообщение пришло, когда в чате нет `open` catalog/сбора, worker должен оставить diagnostic snapshot `telegram_order_without_catalog` / `max_order_without_catalog`; sysadmin observability поднимает это как warning-alert, чтобы было видно, что сбор фактически начался в чате раньше каталога.

## Предпочтительный порядок действий для Codex

1. Найти конкретный чат/каталог, по которому есть жалоба.
2. Проверить живое соотношение `orders`, `order_lines`, `ok`, `unknown_item`, `partial`, `rejected`.
3. Если export строится, но строки пустые, сначала считать это симптомом catalog coverage gap.
4. Чинить матчинг общими правилами, а не одноразовыми SKU-исключениями, если проблема действительно в нормализации.
5. После любого исправления проверять не только тесты, но и хотя бы один живой export на реальном каталоге, если среда это позволяет.

## Parser health modernization 2026-07-13

- Production entrypoint: `run_order_pipeline_with_db`, parser `typed-v3`.
- Matcher auto-select требует score `0.86` и margin `0.08`; conflicts дают abstention.
- Structured diagnostics хранятся в `parser_runs/parser_line_decisions`, а не только в `error_text`.
- AI фактически работает только как local shadow reranker; external LLM и auto-match выключены.
- Operator queue/API и targeted reparse сохраняют before/after и manual locks.
- Final export/catalog close блокируются parser-health preflight; owner override аудируется.
- Текущий live baseline/post-reparse snapshot находится в `runtime-reports/parser-health/current.{json,md}`. На 2026-07-13 глобально остаётся `616 unknown_item` и `39 bad_qty`; это не исправляется принудительными SKU. Для активного catalog 17: `35 unknown_item`, `0 bad_qty`, reparse дал `0 SKU/status changes`; требуются operator/catalog decisions.

## Catalog conflict incident и preservation audit 2026-07-15

- Все `290 catalog_conflict` старого snapshot были сгруппированы по source
  catalog: `9=150`, `11=70`, `14=40`, `15=30`. У каждого каталога top candidate
  повторял первую по id позицию со score `0`: conflict logic создавала кандидата
  без лексического product evidence.
- Общий fix запрещает zero-evidence conflicts, generic shell aliases и переходы
  между несовместимыми product classes (fish/roe/fillet/canned/pate). Это не
  SKU-specific whitelist.
- Историческое падение `2478 → 2296` (`-182`) разложено по каталогам:
  `9 +2`, `11 -186`, `14 +2`, `15 +1`, `17 -1`. Для catalog 11 удалена
  дублированная последовательность строк: before/after distribution totals
  совпали. Два старых `ВАНАМЕЙ` qty не имели явного количества; восемь единиц
  `риета из тунца` были ошибочно связаны с единственным catalog SKU
  `РИЕТ С КРАБОМ`. Это снятые ложные количества, не доказательство удаления
  заказанного товара: исходные строки остаются в raw text/unknown queue.
- Targeted dry-run/apply выполнен только для `reason_codes=[catalog_conflict]`
  каталогов `9, 11, 14, 15`. После apply: общий line count неизменен (`2297`),
  `line_delta=0`, `source_line_removed_count=0`, raw order hashes совпали.
  Все уменьшения старых SKU вручную проверены как false links; товары остались
  в исходном заказе и unresolved queue.
- Strict catalog-driven Excel больше не назначает unresolved line в товарную
  колонку по похожему header. В before/after workbook совпадают data-row count и
  multiset колонки 4; SKU totals теперь отражают только catalog-linked lines.
- Sanitized parser-health report содержит top-3 candidate SKU/score/strategy,
  conflict flags и source catalog, а также фактические runtime flags. На момент
  проверки runtime: embeddings enabled, reranker `production`, LLM `disabled`,
  auto-match requested/effective включён при holdout false-confident rate `0.0`.
- Audit-only workbook сравнение выполнялось без catalog date-window preflight.
  Production export catalog 9 всё ещё требует отдельной правки данных окна
  (`opened_at > closed_at`); parser repair не выдаётся за исправление этого 204.

## Общий parser/ML hardening 2026-07-15

- Исправлены общие real-shape правила без SKU whitelist: quantity-first
  (`2 шт форели`), comma-separated descriptor continuation, canonical pickup
  alias fallback и fuzzy pickup, где generic `самовывоз` не является адресом.
- Широкий species alias больше не стирает явный preparation form: например,
  `Корюшка с/м` не может выбрать `Корюшка вяленая`.
- Weight↔discrete conversion разрешён только через явный `pack_hint`. В seed
  реального каталога у 1-кг упаковок мёда зафиксирован `pack_hint=1 кг`;
  отсутствие pack metadata остаётся `unit_conflict`.
- Live Ivanovo regression `Камбала 1 шт / Креветки 250+ 1уп` намеренно остаётся
  unresolved: первой позиции нет в source catalog, у второй catalog unit `кг`
  без pack conversion. Требование случайного `ok` удалено и заменено проверкой
  `no_candidates + unit_conflict`.
- Parser/pickup/live-message набор: `397 passed`. Gold v2: top-1,
  segmentation, metadata, quantity/unit, line status и end-to-end = `1.0`;
  deterministic и AI-shadow false-confident rate = `0.0`.
- Post-hardening `catalog_conflict` dry-run: catalog `9` scanned `0`, catalogs
  `14/15` дали `0 changes`; catalog `11` дал только два representation/metadata
  changes (`no_pickup` catalog и нормализация unresolved title), без SKU/status
  улучшения, с одним preservation review. Apply намеренно не выполнялся: он не
  исправлял товар и не проходил нулевой review gate.

## Финальный targeted repair и startup safety 2026-07-15

- Найден и закрыт опасный legacy path: startup/online catalog-heal при новом
  `intent_not_order` делал `order.lines.clear()`. Теперь broad heal пропускает
  non-order/admin-origin и блокирует любое уменьшение SKU quantity, line count
  или source fingerprint. Live restart catalog 17 подтвердил:
  `scanned=37`, `updated=0`, `non_order_skipped=1`, `preservation_blocked=0`.
- До этого guard один availability-вопрос без числа уже потерял ложную
  `bad_qty` строку. Это не засчитано как repair и строка не реконструировалась
  догадкой. Raw source и parser diagnostics доказывают вопрос без заказанного
  количества; явный `archive_non_orders=true` только soft-archive source и не
  удалил дополнительных строк.
- Дополнительный exact-scope dry-run/apply catalog 11 исправил три real-shape
  записи: `Марина/Мария` больше не превращаются в SKU `Маринад`; `тунец ×5`
  сохранён, строка с сёмгой сохранена unresolved, `клубника ×1 ящик` восстановлена
  как отдельная unresolved позиция. `line_delta=-2` относится только к двум
  metadata false lines; matched SKU quantity decrease = `0`; raw hashes равны.
- В catalog 14 вручную подтверждён `риет с крабом ×2`; apply дал
  `line_delta=0`, matched SKU quantity decrease = `0`. Соседняя склеенная строка
  `риет/паштет/минтай` оставлена operator review и не форсирована.
- Full distribution Excel v12→v13: data-row count и multiset `Текст заказа`
  совпали для catalogs `9=167`, `11=137`, `14=127`, `15=27`. Единственное
  изменение product totals: catalog 14 `РИЕТ С КРАБ +2`; catalogs 9/11/15 —
  без product delta.
- Sanitized live snapshot после repair: `810 orders / 2294 lines`,
  `ok=1709`, `unknown_item=545`, `bad_qty=40`; reasons:
  `matched=342`, `qty_missing=18`, `unit_conflict=3`, `low_score=243`,
  `catalog_conflict=67`, `ambiguous_margin=4`. Health остаётся `blocked`:
  оставшиеся abstentions требуют source-catalog/operator решений, а не
  принудительного auto-match.
- Фактические runtime flags snapshot: embeddings `true`, reranker
  `production/true`, LLM `disabled/false`, auto-match requested/effective
  `true/true`, holdout false-confident rate `0.0`, external AI calls `0`.
- Проверка: parser/reparse/replay suite `456 passed`; обязательный export
  contract `43 passed`; POST preflight в production Python 3.12 `13 passed`;
  API/admin health green. Локальный Python 3.14 TestClient по-прежнему зависает,
  поэтому production preflight проверен в собранном контейнере.

## Catalog ingress и deadline stabilization 2026-07-25

- Корень незакрывшегося catalog `17`: `/catalog_import` корректно разбирал
  `date: 12.07.2026..13.07.2026`, но при INSERT отбрасывал `end_dt` и сохранял
  `closed_at=NULL`; все active-catalog запросы доверяли только `status=open`.
- Новый импорт сохраняет конец диапазона как end-of-day. Worker и admin API
  восстанавливают старые пропущенные deadlines только из строгого диапазона в
  operator-owned title (`Городец 12.07-13.07.2026`) и закрывают истёкшие
  каталоги. Проверка идёт на startup, периодически в worker, перед order intake
  и при чтении списка каталогов.
- `closed_at` теперь intake cutoff, а не награда за чистый parser-health.
  Истёкший каталог прекращает принимать новые заказы; unresolved по-прежнему
  блокируют final export/manual finalization через preflight.
- Опциональный ключ `feature_catalog_admin_post_sync` (default `true`,
  выключается в `DebugPage → Каталог`) разрешает только подтверждённым
  admin/sender-chat product posts создавать/обновлять позиции. Поддержаны
  price posts и списки с `КГ/ШТ/УП/БАНКА/ПАЧ/НАБ`; повтор idempotent, память
  добавляет aliases/unit, STOP деактивирует только названную позицию.
- Клиентский текст никогда не входит в catalog sync. Если order-like сообщение
  пришло до первого каталога, auto-create восстанавливает только уже
  диагностированные `*_order_without_catalog` сообщения текущего окна
  (максимум 14 дней) через общий typed parser и помечает snapshot
  `order_backfilled`; неоднозначные строки остаются unresolved.
- UI открытия больше не отправляет старые `opened_at/closed_at` закрытого
  lifecycle, а date-only закрытие означает конец выбранного UTC-дня. Полное
  resulting window валидируется даже при PATCH только одной границы.
- Для `autoflush=False` worker snapshot upsert теперь обновляет pending ORM-row,
  а не создаёт второй `message_snapshots` перед диагностической
  переклассификацией; ORM снова отражает существующий unique `(chat_id,
  message_id)`. Config loaders допускают environment-only запуск при
  недоступном локальном dotenv, не ослабляя обязательные `must(...)` значения.

## Semantic ML error-taxonomy hardening 2026-07-25

- Найден ложный ML-симптом: при `PARSER_AI_AUTO_MATCH=0` semantic fallback
  всё равно ходил в embeddings и заменял честный `no_candidates` на
  `low_score` с ближайшим, но нерелевантным SKU (например, отсутствующая
  «ряпушка» выглядела как слабый матч к другой рыбе).
- Production semantic path теперь включается только при
  `PARSER_RERANKER_MODE=production` **и** свежем version-matched evaluation
  artifact, открывающем auto-gate. В `shadow`/`disabled` intake, replay и
  reparse не вызывают Ollama и сохраняют deterministic decision.
- Даже при открытом gate semantic score может усилить только уже
  лексически подтверждённого conflict-free кандидата. Векторная близость без
  product evidence не может выбрать SKU. `no_candidates`,
  `ambiguous_margin`, `catalog_conflict` и `unit_conflict` не переписываются
  ML-слоем.
- Добавлены regression tests на ложный `low_score`, disabled-reranker и
  overconfident vector neighbour; gold v2 сохранил `top-1=1.0` и
  `false_confident_match_rate=0.0`.

## Evidence-based task completion 2026-07-25

- `docs/33-task-completion-gate.md` — обязательный terminal gate для всех
  agent surfaces. `COMPLETE` требует evidence по всем применимым слоям;
  runtime/data request дополнительно требует deploy и live business proof.
- При недоступном Docker/API/DB задача остаётся `BLOCKED`, а не маскируется
  словами «готово после рестарта». `make check-agent-contract` проверяет, что
  этот контракт не выпал из AGENTS/Claude/Copilot/run checks.
- Local Python 3.14.5 (and the available local 3.13 package set) hangs in a
  minimal FastAPI `TestClient` probe outside project code.
  `scripts/run_checks.sh` now probes it with a 10-second timeout before every
  admin-service TestClient suite and fails clearly instead of hanging; Python 3.12,
  matching the production image and admin-service CI, remains the full-admin
  verification route.
- `backend/scripts/verify_catalog_lifecycle.py` evaluates sanitized live
  `catalog_id/status/closed_at` evidence without opening config or DB. It is
  covered by `test_verify_catalog_lifecycle_script.py` and is not a substitute
  for querying the deployed runtime first.
- Просроченный `closed_at` теперь reconciled не только lifecycle worker и
  catalog list, но и прямым catalog detail и bootstrap-options read paths:
  оператор не увидит такой каталог как `open` до фонового прохода.

## Автосбор, reminder-каталог и ежедневный отчёт 2026-07-26

- Пользователь зафиксировал постоянный бизнес-ритм: сборы идут с понедельника
  по воскресенье, обычно раз в две недели, но иногда каждую неделю. Настройка
  cadence обязательна; default — `2` недели.
- Default-окно: понедельник `09:15` → воскресенье `18:00`,
  `Europe/Moscow`. Даты можно вручную поправить у созданного каталога.
- В `Настройки → Каталог` нужен ответственный Telegram-получатель, рабочие
  чаты, день/несколько времён напоминаний и время ежедневного отчёта.
- Reminder отправляется только в неделю перед следующим циклом. После того как
  ответственный прислал полный operator-owned текст и будущий каталог создан,
  оставшиеся напоминания для этого цикла прекращаются.
- Публикация после reminder не дописывает будущие товары в текущий сбор:
  создаётся/обновляется отдельный `scheduled`-каталог. В `opened_at` он
  автоматически заменяет текущий `open`; в `closed_at` intake прекращается.
- Повторные ежедневные каталожные публикации администратора идемпотентно
  обновляют тот же каталог. SKU повторяющейся позиции берётся из последней
  версии каталога этого чата; новая позиция получает детерминированный SKU.
  Aliases/unit продолжают приходить только из `catalog_item_memory`.
- Клиентские сообщения никогда не создают товары/SKU. Действует существующий
  strict admin/sender-chat ingress `feature_catalog_admin_post_sync`.
- Ежедневный Telegram-отчёт содержит новые заказы за день, всего заказов в
  активном сборе и число `partial/needs_admin/rejected`.
- Для `@username` нужен связанный активный `admin_users.tg_user_id`; бот не
  может начать личный диалог с произвольным ником, пока пользователь не открыл
  бота. Числовой Telegram ID поддерживается напрямую.
- Полный контракт: [[34-collection-automation]].

## Provider reconciliation и exactly-once hardening 2026-08-02

- Найдены два ingress-дефекта: non-Telegram `update_id` был глобальным 31-bit
  hash, а order dedupe оставался `(catalog_id, source_message_id)`. Это могло
  дать false duplicate между provider/chat и повтор одного сообщения после
  catalog switch.
- Миграция `p2q3r4s5t6u7` добавляет provider event/message identity в
  `tg_updates`, `orders` и `message_snapshots`. Historical duplicates не
  удаляются; canonical key получает самая ранняя строка, остальные видны в
  reconciliation.
- Worker теперь использует bounded `retry` с exponential backoff и возвращает
  stale `processing` lease в очередь. Terminal `failed` остаётся operator alert.
- Ручное/автоматическое закрытие intake больше не блокируется parser-health;
  unresolved по-прежнему блокирует final export без owner override.
- Hourly ops job создаёт sanitized production reconciliation по provider/chat/
  catalog/day. `healthy` fail-closed без deployed commit, explicit working
  chats, provider history proof, migration head и zero-unaccounted.
- Parser-health больше не наследует generic host `APP_ENV`: production scope
  задаётся явно, а отсутствующий deployed commit или Alembic head блокирует
  report. Это устраняет ложную маркировку локальной БД как production.
- Telegram Desktop export теперь default-dry-run проходит через `tg_updates` и
  обычный worker с исходными timestamp/user/message IDs; повтор enqueue
  идемпотентен. Direct catalog import требует exact lifecycle window и stable
  identity, не архивирует существующие orders и применяет только receipt того
  же dry-run scope.
- Полный контракт: [[36-production-order-reconciliation]].
