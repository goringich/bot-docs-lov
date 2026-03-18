# ADR 0003 - Idempotency model

## Status
accepted

## Context
Telegram может ретраить update. edited_message меняет текст существующего сообщения. Нельзя допускать дублей и удвоения количества.

## Decision
- Идемпотентность апдейтов: unique update_id в tg_updates.
- Идемпотентность по сообщению: unique (chat_id, message_id) в message_snapshots.
- Для edited_message:
  - считаем hash текста
  - если hash не изменился, обработку пропускаем
  - если изменился, пересчитываем вклад сообщения в заказ
- Вклад сообщения хранится отдельно как order_message_contrib (round_id + chat_id + message_id).

## Consequences
- Изменение сообщения заменяет вклад, а не добавляет новый.
- Можно корректно пересчитывать агрегированные заказы.
- Модель хорошо масштабируется и упрощает отладку.

