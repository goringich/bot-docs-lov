---
title: Production order reconciliation — ingress, exactly-once and backfill
type: runbook
status: current
tags: [orders, ingress, reconciliation, backfill, production, observability]
updated: 2026-08-02
related:
  - "[[23-provider-bridge-runbook]]"
  - "[[28-stability-playbook]]"
  - "[[32-parser-health-pipeline]]"
  - "[[33-task-completion-gate]]"
---

# Production order reconciliation

> [!important] Цель
> Каждое order-like provider message должно иметь ровно один canonical order
> либо machine-readable diagnostic с точной причиной. Отсутствие сообщения в
> доступной истории провайдера не считается доказательством отсутствия заказа.

## Stable identity

- Ingress identity: `tg_updates.provider_event_key`. Native event/update ID
  имеет приоритет; exact retry одного payload дедуплицируется.
- Order identity: `orders.source_event_key`, построенный из
  `provider + provider chat + provider message`.
- Diagnostic identity: `message_snapshots.source_event_key` использует тот же
  provider/chat/message key.
- Старый 31-bit `tg_updates.update_id` сохраняется для совместимости и поиска,
  но больше не является unique exactly-once boundary.
- Исторические дубли не удаляются. Миграция назначает stable key самому раннему
  canonical order, а остальные строки оставляет без key для reconciliation и
  operator review.

Миграция: `p2q3r4s5t6u7`.

## Worker queue

Допустимые состояния:

`new → processing → done`

При transient failure:

`processing → retry → processing`, с bounded exponential backoff. После
`WORKER_MAX_UPDATE_ATTEMPTS` update становится `failed`, остаётся в БД и
показывается sysadmin alert. Lease `processing`, зависший дольше
`WORKER_PROCESSING_STALE_SECONDS`, автоматически возвращается в `retry`.

Ни `retry`, ни `failed`, ни diagnostic нельзя удалять для улучшения статистики.

## Read-only production report

```sh
docker-compose -f infra/docker-compose.yml --profile ops run --rm \
  order_reconciliation_reporter
```

Stable sanitized artifact:

- `runtime-reports/reconciliation/current.json`.

Он содержит breakdown по provider / hashed chat / catalog / day:

- raw updates, message-bearing raw payloads и raw hashes;
- order-like messages;
- `tg_updates` по status;
- canonical orders, duplicates и missing orders;
- повторные raw deliveries отдельно от duplicate canonical orders;
- unresolved lines;
- отдельно unclassified/broken payloads, которые нельзя молча исключить из знаменателя;
- unrecoverable messages и причины;
- current / operational / historical scope;
- migration head, environment и deployed commit.

Одного поля `environment=production` недостаточно: parser-health также обязан
нести `deployed_commit` и Alembic current/head. Generic host `APP_ENV` не может
переклассифицировать локальную БД в production evidence.

Customer text, имена, телефоны, raw provider/chat/user IDs и secrets в artifact
запрещены. Полный raw остаётся только в production audit tables и
operator-only UI.

`healthy` запрещён, пока не доказаны одновременно:

- production environment;
- deployed commit;
- migration at head;
- явный `collection_chat_ids` scope рабочих чатов;
- проверенный history access либо внешний экспорт каждого провайдера;
- `unaccounted_order_like_messages = 0`;
- каждый recoverable message имеет ровно один order;
- очередь не содержит `new/retry/processing`; `failed` отдельно остаётся
  operational debt до операторского решения;
- generic snapshot kinds (`message`, `edited_message`, `unknown`) не считаются
  конкретной диагностикой пропущенного заказа;
- каждый raw update классифицирован либо явно отмечен `payload_decode_error`.

## Backfill

Любой запуск сначала dry-run. Каталог выбирается только по provider chat и
точному lifecycle window; приблизительная дата без chat/window match запрещена.

```sh
python backend/scripts/collect_orders_from_tg_updates.py \
  --provider max \
  --all-provider-catalogs \
  --catalog-status closed \
  --datetime-from 2026-01-01T00:00:00 \
  --datetime-to 2026-08-02T00:00:00 \
  --json
```

После operator review тот же scope допускает `--apply-changes`. Повторный
dry-run обязан дать `would_import=0`; `already_present` не является дублем, а
`identity_catalog_conflict` блокирует apply до ручной сверки lifecycle.
Admin UI для raw `tg_updates` и Telegram JSON не показывает apply до успешного
dry-run и передаёт scope-bound receipt; изменение каталога, файла или окна
делает receipt недействительным.

Если MAX/API не даёт историю, сначала используется реальный provider export
или `import_max_messages.py --all-chats --list-chats --dry-run`. Источник с
`history_access=unknown` остаётся unrecoverable/blocked, пока права или экспорт
не подтверждены.

Telegram Desktop export возвращается только через raw ingress и обычный worker:

```sh
python backend/scripts/replay_telegram_export.py \
  --file /operator/export/result.json \
  --chat-id <exact-source-chat-id> --json
```

Команда по умолчанию dry-run. После сверки тот же scope получает
`--apply-changes`; повтор обязан показать `enqueued=0` и
`already_present=<candidate_messages>`. Исходные timestamp/user/message ID
сохраняются, `--only-orders` запрещён. Legacy direct catalog import также
default-dry-run, требует точный `catalog_id` и lifecycle window; API apply
принимает только receipt от byte-for-byte того же export/scope и никогда не
архивирует существующие orders перед backfill.

## Rollout и rollback

Перед live-write:

1. backup DB;
2. сохранить current reconciliation и queue counts;
3. применить `alembic upgrade heads`;
4. пересобрать/restart API, worker, admin service и web;
5. выполнить read-only reconciliation;
6. проверить controlled test order end-to-end;
7. выполнить backfill dry-run;
8. только после отдельного подтверждения — apply backfill.

Rollback к старому worker требует сначала остановить writers и перевести
оставшиеся `retry` обратно в `new`; данные не удаляются. Additive identity
columns лучше оставить. Schema downgrade допустим только после backup и
экспорта reconciliation evidence.
