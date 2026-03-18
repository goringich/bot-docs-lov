# Security Guidelines

Документация по безопасности проекта.

## 🔐 Секреты и учётные данные

### Обязательно в `.env`:
```env
# Telegram Bot
BOT_TOKEN=<от @BotFather>
TELEGRAM_WEBHOOK_SECRET=<random-string>

# Database
MYSQL_HOST=mysql
MYSQL_USER=orders
MYSQL_PASSWORD=<strong-password>
MYSQL_DATABASE=orders

# JWT для админ-панели
ADMIN_API_JWT_SECRET=<random-32-char-string>
ADMIN_API_JWT_EXPIRE_MIN=60
ADMIN_API_BOOTSTRAP_TOKEN=<random-token-for-first-admin>
ADMIN_API_CORS_ORIGINS=http://localhost:4173

# ngrok (для dev)
NGROK_AUTHTOKEN=<from-ngrok-dashboard>
```

### Никогда не коммитить:
- `.env` файлы с реальными данными
- Дампы базы данных
- Лог-файлы с персональными данными
- SSH ключи

## 🔑 Аутентификация

### Telegram Bot
- Admin IDs проверяются через `admin_user_ids` в конфиге
- Команды разделены на public (все) и admin (только из списка)

### Admin Panel
- **One-time links**: JWT токен с TTL 5 минут
- **Auto-provision**: новый admin_user создаётся при первом входе по tg_user_id
- **Password hashing**: bcrypt с cost=12
- **Trusted-device login**: опция `Запомнить это устройство` использует secure credential storage / password manager браузера; пароль не хранится в открытом `localStorage` самого приложения
- **Manual logout protection**: после явного `Выйти` / `Сменить аккаунт` отключается silent auto-login, чтобы браузер не входил обратно мгновенно без намерения пользователя
- **Step-up auth for exports**: чувствительные XLSX-экспорты требуют повторного ввода пароля и краткоживущего confirmation token, дополнительно привязанного к device fingerprint при наличии binding

### Роли
- `owner` — полный доступ
- `admin` — управление каталогами, заказами
- `pickup_admin` — выдача заказов на назначенных точках (level=15)
- `manager` — просмотр заказов
- `viewer` — только чтение

### Дополнительные ограничения чувствительных данных
- `analytics/*` — backend owner-only
- `/exports/*` и `/orders/export*` — backend owner-only + password confirmation
- Telegram admin-бот больше не отдаёт статистику, списки заказов, диагностику и выгрузки; чувствительные данные доступны только в web-админке

## 🛡️ Защита данных

### Маскирование в логах
```python
# ✅ Правильно
logger.info("User ***%s created order", user_id % 10000)

# ❌ Неправильно
logger.info("User %s phone %s", user_id, phone)
```

### Персональные данные (PII)
- `phone_last4` — только последние 4 цифры
- `customer_name` — хранится в БД, не логируется
- `tg_user_id` — маскируется в логах

### SQL Injection
Все запросы через SQLAlchemy ORM:
```python
# ✅ Безопасно
db.query(Order).filter(Order.id == order_id).first()

# ❌ Уязвимо
db.execute(f"SELECT * FROM orders WHERE id = {order_id}")
```

## 🐳 Docker Security

### Production hardening (реализовано)
- **Non-root users**: Все Dockerfiles используют `appuser:appgroup` (UID 1001)
- **Multi-stage builds**: `admin-web` собирается в node, раздаётся nginx
- **Parameterized secrets**: MySQL credentials через env vars в docker-compose
- **No hardcoded tokens**: `ADMIN_ONE_TIME_KEY` читается только из `.env`

### Webhook security (реализовано)
- **Constant-time comparison**: `hmac.compare_digest()` для проверки `X-Telegram-Bot-Api-Secret-Token`
- **Message truncation**: Ограничение 4096 символов перед отправкой в Telegram API

### API security (реализовано)
- **Settings key allowlist**: POST /settings принимает только `ALLOWED_SETTINGS_KEYS`
- **Resource existence checks**: 404 для PATCH/DELETE несуществующих ресурсов
- **Restricted CORS**: Явный список методов и заголовков вместо `"*"`
- **Password validation**: Минимум 8 символов, uppercase + lowercase + цифра
- **Input sanitization**: Очистка HTML/script тегов из пользовательского ввода
- **CSRF tokens**: Генерация уникальных CSRF токенов
- **JWT jti**: Уникальный идентификатор каждого токена + server-side session tracking
- **Session revocation**: Каждый JWT привязан к `ActiveSession` записи; отзыв сессии блокирует токен через JTI проверку в `deps.py`
- **Tiered rate limiting**: login=5/min, auth=10/min, export=3/min, global=100/min; Retry-After header
- **Audit logging**: Логирование всех mutation-запросов с IP, User-Agent, device fingerprint и severity
- **Security headers**: CSP (no inline scripts), COOP, COEP, CORP, HSTS, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, Cache-Control: no-store
- **Suspicious request blocking**: Middleware regex блокирует path traversal, XSS, SQL injection, code injection, prototype pollution, доступ к sensitive файлам (.env, .git)
- **Brute-force protection**: Account lockout после 5 неудачных попыток за 15 мин (блок 30 мин); IP блокировка после 20 попыток за 60 мин
- **Constant-time comparisons**: `hmac.compare_digest()` для всех secret/token сравнений
- **Disabled docs endpoints**: Swagger UI, ReDoc и OpenAPI schema отключены в production
- **Oversized header protection**: Запросы с заголовками > 8192 байт блокируются

### Frontend security (реализовано)
- **Anti-DevTools**: Блокировка F12, Ctrl+Shift+I/J/C, Ctrl+U/S/P, right-click; debugger timing detection; console override
- **Anti-copy**: Запрет выделения и копирования в таблицах и элементах `.secure-data`; input/textarea поля разрешены
- **Anti-print**: При печати тело страницы скрывается, показывается предупреждение
- **Session guard**: Автоматический logout при неактивности > 30 мин; cross-tab синхронизация через localStorage events
- **Tab visibility**: При переходе на другую вкладку чувствительные элементы размываются
- **Security meta tags**: `noindex`, `nofollow`, `noarchive`, `no-referrer`

### Security dashboard (owner-only)
- **Threat level indicator**: Автоматическая оценка уровня угрозы (low/medium/high/critical)
- **Login attempts viewer**: Таблица всех попыток входа с IP, User-Agent, fingerprint
- **Active sessions management**: Просмотр и отзыв отдельных сессий
- **Panic button**: Экстренный отзыв всех сессий с разлогиниванием всех администраторов
- **Stealth mode**: Маскировка IP-адресов и email в интерфейсе безопасности
- **Top attacking IPs**: Список наиболее подозрительных IP-адресов

## 🔍 Checklist перед деплоем

- [ ] `.env` отсутствует в git
- [ ] Все пароли сильные (16+ символов)
- [ ] `ADMIN_API_JWT_SECRET` уникальный
- [ ] Admin IDs актуальны
- [ ] Логи не содержат PII
- [ ] Rate limiting настроен
- [ ] HTTPS для admin panel
- [ ] Backup процедуры работают

## 🚨 Incident Response

### При утечке BOT_TOKEN:
1. Revoke токен через @BotFather
2. Создать новый токен
3. Обновить `.env`
4. Перезапустить сервисы

### При утечке DB credentials:
1. Сменить пароль в MySQL
2. Обновить `.env`
3. Перезапустить сервисы
4. Проверить логи на подозрительную активность

### При утечке ADMIN_API_JWT_SECRET:
1. Сгенерировать новый секрет
2. Все сессии станут невалидными
3. Админам нужно будет войти заново

## 📝 Ссылки

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [SQLAlchemy Security](https://docs.sqlalchemy.org/en/20/core/security.html)

### [[12-admin-panel]]