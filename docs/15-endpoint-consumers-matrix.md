# 15. Endpoint → Consumer matrix

Практичная матрица: кто вызывает какие endpoint-ы, где смотреть реализацию и как проверять.

> Актуально на 2026-03-11.

## Легенда

- **Admin Web** — frontend `admin-web`
- **Bot/Worker** — telegram bot + worker backend
- **External** — внешние системы по service-key/M2M

## Backend (`backend`, порт 8000)

| Endpoint | Admin Web | Bot/Worker | External | Где реализовано |
|---|---:|---:|---:|---|
| `GET /health` | - | - | ✅ | `backend/app/main.py` |
| `GET /capabilities` | - | - | ✅ | `backend/app/main.py` |
| `POST /telegram/webhook` | - | ✅ (Telegram platform) | - | `backend/app/main.py` |

## Admin service (`admin_service`, порт 8010)

### Auth / Users / Roles

| Endpoint group | Admin Web | Bot/Worker | External | Где реализовано |
|---|---:|---:|---:|---|
| `/auth/*` | ✅ | ✅ (`/open_app` flow via service key) | ⚠️ частично | `admin_service/app/api/routers/auth.py` |
| `/me`, `/users*` | ✅ | - | - | `admin_service/app/api/routers/users.py` |
| `/roles`, `/permissions` | ✅ | - | - | `admin_service/app/api/routers/roles.py` |

### Catalog / Orders / Analytics / Settings

| Endpoint group | Admin Web | Bot/Worker | External | Где реализовано |
|---|---:|---:|---:|---|
| `/catalogs*`, `/chats*`, `/admin-user-chats*` | ✅ | - | - | `admin_service/app/api/routers/catalogs.py` |
| `/orders*` + exports | ✅ | - | - | `admin_service/app/api/routers/orders.py` |
| `/analytics/*` | ✅ | - | - | `admin_service/app/api/routers/analytics.py` |
| `/settings`, `/pickup-places*` | ✅ | - | - | `admin_service/app/api/routers/settings.py` |
| `/templates/excel` | ✅ | - | - | `admin_service/app/api/routers/templates.py` |
| `/debug/*` | ✅ (owner/debug mode) | - | - | `admin_service/app/api/routers/debug.py` |
| `/settings/dev-mode` | ✅ (owner) | - | - | `admin_service/app/api/routers/admin_settings.py` |

### Integration M2M

| Endpoint | Admin Web | Bot/Worker | External | Где реализовано |
|---|---:|---:|---:|---|
| `GET /integrations/capabilities` | - | - | ✅ | `admin_service/app/api/routers/integrations.py` |
| `GET /integrations/chats` | - | - | ✅ | `admin_service/app/api/routers/integrations.py` |
| `GET /integrations/catalogs/open` | - | - | ✅ | `admin_service/app/api/routers/integrations.py` |
| `GET /integrations/pickup-places` | - | - | ✅ | `admin_service/app/api/routers/integrations.py` |
| `GET /integrations/orders` | - | - | ✅ | `admin_service/app/api/routers/integrations.py` |

---

## Где смотреть endpoint-ы быстрее всего

1. **Swagger UI**:
   - `http://localhost:8000/docs`
   - `http://localhost:8010/docs`

2. **OpenAPI JSON**:
   - `http://localhost:8000/openapi.json`
   - `http://localhost:8010/openapi.json`

3. **Исходники router-ов**:
   - `admin_service/app/api/routers/*.py`
   - `backend/app/main.py`

4. **Клиентские вызовы фронта**:
   - `admin-web/src/api/client.ts`

5. **M2M-discovery**:
   - `GET /integrations/capabilities`
   - `GET /capabilities`

---

## Мини-проверка (визуально)

- Если endpoint есть в Swagger, но нет в `client.ts` → вероятно не подключён в UI.
- Если endpoint есть в `client.ts`, но не открывается в Swagger → рассинхрон backend/frontend.
- Для внешних интеграций всегда проверяй `X-Service-Key` и `integrations/capabilities`.
### [[16-endpoints-handbook-for-share]]