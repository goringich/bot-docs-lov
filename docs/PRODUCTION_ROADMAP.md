# Production Roadmap — Father Bot v2.0

## Executive Summary

Цель: превратить MVP в production-ready систему с:
- AI-распознаванием заказов
- Полнофункциональной админ-панелью
- Ролевой моделью доступа
- Аналитикой и отчётами

## Phase 1: Backend Architecture (AI + Clean Code)

### 1.1 AI Order Recognition Service
- [ ] Создать `backend/app/ai/` модуль
- [ ] Интегрировать LLM (OpenAI/local) для парсинга заказов
- [ ] Hybrid approach: regex + AI fallback
- [ ] Confidence scoring для каждого поля

### 1.2 Clean Architecture
```
backend/app/
├── ai/                    # AI services
│   ├── order_recognizer.py
│   ├── prompts.py
│   └── models.py
├── core/                  # Business logic
│   ├── orders.py
│   ├── catalogs.py
│   └── analytics.py
├── handlers/              # Telegram handlers
├── services/              # External services
├── repo/                  # Data access
└── api/                   # REST API endpoints
```

### 1.3 New Database Schema
- [ ] `order_recognitions` — AI parsing results
- [ ] `analytics_snapshots` — daily aggregates
- [ ] `export_templates` — custom Excel templates

## Phase 2: Admin Service RBAC

### 2.1 Role Hierarchy
```
owner (level 0)
  └── superadmin (level 10)
       └── admin (level 50)
            └── manager (level 80)
                 └── viewer (level 100)
```

### 2.2 Permissions
- `users.view`, `users.edit`, `users.delete`
- `orders.view`, `orders.edit`, `orders.export`
- `catalogs.view`, `catalogs.edit`, `catalogs.manage`
- `analytics.view`, `analytics.export`
- `settings.view`, `settings.edit`
- `templates.view`, `templates.edit`

### 2.3 API Endpoints
- [ ] `/api/v1/orders` — CRUD + filters + pagination
- [ ] `/api/v1/catalogs` — full management
- [ ] `/api/v1/analytics` — dashboards data
- [ ] `/api/v1/templates` — Excel template management
- [ ] `/api/v1/exports` — generate reports

## Phase 3: Admin Web UI

### 3.1 Pages
- **Dashboard** — overview with charts
- **Orders** — list, filter, edit, export
- **Catalogs** — manage items, stops
- **Users** — RBAC management
- **Analytics** — charts and reports
- **Templates** — Excel template editor
- **Settings** — system configuration

### 3.2 Components
- Data tables with sorting/filtering
- Charts (recharts/chart.js)
- Excel template preview
- Real-time notifications
- Dark mode support

### 3.3 Tech Stack
- React 18 + TypeScript
- MUI v5 + custom theme
- React Query for data fetching
- React Router v6
- Recharts for visualizations

## Phase 4: UX Improvements

### 4.1 Telegram Bot UX
- [ ] Inline order editing
- [ ] Order confirmation flow
- [ ] Smart suggestions
- [ ] Multi-language support
- [ ] Order status notifications

### 4.2 Admin Panel UX
- [ ] Keyboard shortcuts
- [ ] Bulk operations
- [ ] Export presets
- [ ] Search everywhere
- [ ] Activity feed

## Implementation Priority

1. **Week 1**: AI recognition + Backend API
2. **Week 2**: Admin UI pages + RBAC
3. **Week 3**: Analytics + Templates
4. **Week 4**: Polish + Testing

## Files to Create/Modify

### Backend
- `backend/app/ai/order_recognizer.py`
- `backend/app/ai/prompts.py`
- `backend/app/api/v1/routes.py`
- `backend/app/core/analytics.py`

### Admin Service
- `admin_service/app/api/v1/` (new module)
- `admin_service/app/permissions.py`

### Admin Web
- `admin-web/src/pages/*.tsx` (7+ pages)
- `admin-web/src/components/*.tsx` (20+ components)
- `admin-web/src/hooks/*.ts`
- `admin-web/src/theme/`

### [[PRODUCTION_UPGRADE_v2.0]]