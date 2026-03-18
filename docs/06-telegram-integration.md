
# Telegram integration

Эта страница описывает интеграцию бота с Telegram Bot API.

## Архитектура

1) Telegram шлёт update на webhook endpoint.
2) API сохраняет raw update в БД (`tg_updates`) и быстро отвечает 200 OK.
3) Worker забирает необработанные записи из `tg_updates` и:
	- сохраняет `MessageSnapshot`
	- распознаёт команды/заказы
	- обновляет агрегированные `Order`/`OrderLine`
	- отправляет ответы назад в Telegram.

## Endpoint

- Webhook: `POST /telegram/webhook`
- Health: `GET /health`

См. `backend/app/main.py`.

## Webhook secret token (Bot API)

Если задан `TELEGRAM_WEBHOOK_SECRET`, то все запросы к `/telegram/webhook` должны содержать header:

`X-Telegram-Bot-Api-Secret-Token: <secret>`

Иначе сервер отвечает `401`.

## Настройки окружения

Минимально:
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_WEBHOOK_SECRET` (рекомендуется)

Дополнительно:
- `REPLY_MODE` = `group|dm` — если `dm`, бот старается отвечать пользователям в личку.

Также используется `ADMIN_USER_IDS` для админских команд.

## Полезные ссылки

- Запуск и установка webhook: `README.md`
- Быстрый перезапуск: [[11-restart]]

### [[07-excel-export]]