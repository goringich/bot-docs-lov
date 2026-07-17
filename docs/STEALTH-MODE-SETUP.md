# Stealth Mode Setup

Статус: `current`

Назначение: инструкция по подключению Telegram-бота к чату в скрытом или ограниченном режиме видимости.

## Режимы работы

Бот «Отличный Улов» может работать в трёх режимах видимости (`bot_visible`):

| Режим | Описание | Когда использовать |
|-------|----------|-------------------|
| **`hidden`** | Бот молчит, никак не реагирует. Полностью невидим для участников | Первая неделя, когда проверяешь парсинг на реальных сообщениях |
| **`reactions_only`** | Бот ставит 👍 на распознанные заказы + пишет в лог ошибки | Режим «мягкого» старта: участники видят реакцию, но бот не пишет в чат |
| **`full`** | Бот отвечает текстом: подтверждения заказов, ошибки, подсказки | Полноценная работа после обкатки |

---

## Шаг 1. Создать бота в Telegram

1. Откройте [@BotFather](https://t.me/BotFather) в Telegram
2. `/newbot` → введите имя (например, «Отличный Улов») и username (например, `otlichniy_ulov_bot`)
3. Скопируйте **TELEGRAM_BOT_TOKEN** (значение хранится только вне репозитория)
4. `/setprivacy` → выберите вашего бота → **Disable** (чтобы бот видел ВСЕ сообщения в группе, а не только команды)

Важно: без отключения Privacy Mode бот не увидит обычные сообщения в группе.

---

## Шаг 2. Добавить бота в чат

1. Откройте нужный Telegram-чат (группу/супергруппу)
2. Настройки чата → «Добавить участника» → найдите вашего бота по username
3. Если чат — **супергруппа**: сделайте бота **администратором** (без прав, чтобы он мог читать сообщения)
   - Минимальные права: ☑️ Чтение сообщений
   - Для `reactions_only` режима: ☑️ Отправка сообщений (для реакций)
   - Все остальные права можно отключить

---

## Шаг 3. Настроить `.env`

```env
# Telegram Bot
TELEGRAM_BOT_TOKEN=[REDACTED_SECRET]
ADMIN_USER_IDS=123456789,987654321    # Telegram user_id администраторов
TELEGRAM_WEBHOOK_SECRET=[REDACTED_SECRET]

# Database
DB_HOST=mysql
DB_PORT=3306
DB_USER=root
DB_PASSWORD=[REDACTED_SECRET]
DB_NAME=tg_orders

# Admin API
ADMIN_API_KEY=[REDACTED_SECRET]
```

---

## Шаг 4. Запустить инфраструктуру

```bash
# Запуск всех сервисов
make compose-up

# Короткий алиас тоже поддерживается
make up

# Или по отдельности:
docker compose -f infra/docker-compose.yml up -d mysql
docker compose -f infra/docker-compose.yml up -d api worker
docker compose -f infra/docker-compose.yml up -d admin_service admin_web
```

---

## Шаг 5. Узнать `chat_id` чата

Бот автоматически зарегистрирует чат при первом сообщении. Но для ручной настройки:

**Вариант 1** — отправьте любое сообщение в чат и посмотрите в логах:
```bash
docker compose -f infra/docker-compose.yml logs worker | grep "chat_id"
```

**Вариант 2** — используйте API:
```bash
curl https://api.telegram.org/bot<BOT_TOKEN>/getUpdates | python -m json.tool | grep '"id"'
```

**Вариант 3** — Admin API:
```bash
curl -H "Authorization: Bearer <ADMIN_API_KEY>" http://localhost:8010/api/chats
```

---

## Шаг 6. Установить скрытый режим

### Через Admin Panel (рекомендуется)

1. Откройте `http://localhost:4173` → авторизация
2. Раздел **Настройки** → вкладка **Бот** → блок **Видимость бота в чатах**
3. В нужном чате выберите режим `hidden`
4. Готово (применяется сразу)

### Через Admin API

```bash
# Установить hidden режим
curl -X PATCH \
  -H "Authorization: Bearer <ADMIN_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"bot_visible": "hidden"}' \
  http://localhost:8010/api/chats/<chat_id>
```

### Через SQL (для отладки)

```sql
UPDATE chats SET bot_visible = 'hidden' WHERE chat_id = -1001234567890;
```

---

## Шаг 7. Заполнить каталог

```bash
# Загрузить реальный каталог (45 товаров + 8 пунктов выдачи)
cd backend
PYTHONPATH=. python scripts/seed_catalog_real.py --chat-id <TG_CHAT_ID>

# Или только пункты выдачи:
PYTHONPATH=. python scripts/seed_catalog_real.py --chat-id <TG_CHAT_ID> --pickup-only
```

Или через Admin Panel → Каталог → Создать каталог.

> После обновления кода или `git pull` сначала обязательно применяйте миграции:
>
> ```bash
> cd backend
> ../.venv/bin/alembic upgrade heads
> ```
>
> Если этого не сделать, webhook может принимать update, но worker будет падать при обработке из-за несовпадения схемы БД и кода.

---

## Шаг 8. Проверить работу

### Мониторинг в реальном времени

```bash
# Логи worker (обработка сообщений)
docker compose -f infra/docker-compose.yml logs -f worker

# Логи API
docker compose -f infra/docker-compose.yml logs -f api
```

### Проверка через Admin Panel

1. **Dashboard** — новые заказы должны появляться
2. **Аналитика** — статистика по обработанным сообщениям
3. **Каталог** — совпадения товаров в заказах

### Проверка через API

```bash
# Последние заказы
curl -H "Authorization: Bearer <ADMIN_API_KEY>" \
  http://localhost:8010/api/orders?limit=10

# Статистика
curl -H "Authorization: Bearer <ADMIN_API_KEY>" \
  http://localhost:8010/api/dashboard/stats

# Экспорт истории заказов (audit)
curl -L -H "Authorization: Bearer <ADMIN_API_KEY>" \
  "http://localhost:8010/api/orders/export-history?status=active" \
  -o orders_history.xlsx
```

---

## Переключение режимов

### hidden → reactions_only (мягкий старт)

Когда убедились, что парсер корректно разбирает >95% заказов:

```bash
curl -X PATCH \
  -H "Authorization: Bearer <ADMIN_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"bot_visible": "reactions_only"}' \
  http://localhost:8010/api/chats/<chat_id>
```

Бот начнёт ставить 👍 на распознанные заказы. Участники увидят реакцию, но бот не будет писать текст.

### reactions_only → full (полная работа)

Когда готовы к полноценной работе:

```bash
curl -X PATCH \
  -H "Authorization: Bearer <ADMIN_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"bot_visible": "full"}' \
  http://localhost:8010/api/chats/<chat_id>
```

---

## Чеклист перед запуском

- [ ] `.env` заполнен, `TELEGRAM_BOT_TOKEN` корректный
- [ ] Privacy Mode **отключён** в BotFather
- [ ] Бот добавлен в чат как участник (или админ для супергрупп)
- [ ] `bot_visible = hidden` для нового чата
- [ ] Каталог загружен (`seed_catalog_real.py` или через панель)
- [ ] Пункты выдачи настроены
- [ ] Логи worker показывают обработку сообщений
- [ ] Тестовый заказ появляется в Admin Panel

---

## Решение проблем

| Проблема | Решение |
|----------|---------|
| Бот не видит сообщения | Проверьте Privacy Mode в BotFather (должен быть Disabled) |
| Заказы не создаются | Проверьте, что каталог open (`status = 'open'`) |
| Пункт выдачи не определяется | Добавьте алиасы в PickupPlace или через панель |
| Товар не совпадает | Добавьте алиасы к CatalogItem (через панель или SQL) |
| Бот отвечает, хотя mode=hidden | Перезапустите worker: `docker compose restart worker` |
| 403 ошибка в API | Проверьте ADMIN_API_KEY в `.env` и заголовке Authorization |

---

## Архитектура скрытого режима

```
Telegram Chat
    │
    ├── Сообщение пользователя
    │
    ▼
  Worker (polling/webhook)
    │
    ├── Сохраняет MessageSnapshot
    ├── Определяет looks_like_order()
    │
    ├── IF order:
    │   ├── parse_order_text_with_db()
    │   ├── evaluate_order_lines()
    │   ├── Сохраняет Order + OrderLines
    │   │
    │   └── IF bot_visible == 'hidden':
    │   │     └── ничего не делает ✓
    │   │
    │   └── IF bot_visible == 'reactions_only':
    │   │     └── ставит 👍 реакцию
    │   │
    │   └── IF bot_visible == 'full':
    │         └── отправляет текстовое подтверждение
    │
    └── IF not order:
        └── пропускает (не реагирует ни в каком режиме)
```
