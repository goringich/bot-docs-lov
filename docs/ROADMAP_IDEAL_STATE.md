# Ideal State Roadmap

Статус: `target`

Назначение: перечень задач до целевого состояния системы. Файл не описывает текущее состояние проекта.

> **Важно:** этот файл описывает _оставшиеся_ задачи до ideal state.  
> Исторически часть пунктов в нём уже была закрыта. Актуальный снимок текущего состояния см. в [[changelog/implementation-status]].

## Already Done (v2.0)
- ✅ Bot Settings UI (режим ответов, тексты, точки выдачи)
- ✅ Dark Theme (переключатель + localStorage)
- ✅ Enhanced Bot UX (minimal buttons, emoji)
- ✅ Improved Order Recognition (filter recipes)
- ✅ CSV Export (с фильтрацией)
- ✅ Analytics Charts (bar charts с анимацией)
- ✅ Database-driven Configuration
- ✅ Users page in admin UI
- ✅ Role / region management for admin users
- ✅ Delivery workspace for pickup admins
- ✅ Section-level visibility settings in admin UI
- ✅ Automated tests at repository level (`backend`, `admin_service`, `admin-web/e2e`)

## Phase 2: Advanced Features (Optional)

### 1. AI Integration (Medium Priority)
**Current State**: AI module exists, but main production path intentionally remains regex-first/offline-safe

**Tasks**:
- [ ] Keep regex-first default and add a real local/offline provider behind `AI_MODE=local`
- [ ] Only after that, optionally add fallback: regex → local AI if confidence < 0.7
- [ ] Create UI settings for AI:
  - Provider selection (disabled / local / external)
  - Temperature slider
  - Enable/disable AI fallback
- [ ] Add API key management in SettingsPage

**Files to edit**:
- `backend/app/worker.py` — integrate local provider safely behind feature flag
- `admin-web/src/pages/SettingsPage.tsx` — add AI config section
- `admin_service/app/main.py` — add AI settings endpoints

### 2. Order/Catalog Editing in UI (High Priority)
**Current State**: UI is read-only for orders and catalog

**Tasks**:
- [ ] Add "Edit Order" button on OrdersPage details modal
- [ ] Allow changing order line quantities
- [ ] Allow adding/removing order lines
- [ ] Add "Edit Item" on catalog management page
- [ ] Allow changing item price/title/SKU
- [ ] Add audit log for all edits (who changed what when)

**New Components**:
- `admin-web/src/pages/CatalogManagementPage.tsx`
- `admin-web/src/components/OrderEditDialog.tsx`
- `admin-web/src/components/CatalogItemEditDialog.tsx`

**New API Endpoints**:
- `PATCH /orders/{id}` — update order fields
- `POST /orders/{id}/lines` — add order line
- `PATCH /orders/{id}/lines/{line_id}` — edit line
- `DELETE /orders/{id}/lines/{line_id}` — remove line
- `PATCH /catalog-items/{id}` — edit catalog item

### 3. Excel Preview in UI (Medium Priority)
**Current State**: Excel files generated but not visible in UI

**Tasks**:
- [ ] Add "Last Reports" section on TemplatesPage
- [ ] Store generated Excel files in database (blob or file path)
- [ ] Add endpoint `GET /reports` — list recent exports
- [ ] Add endpoint `GET /reports/{id}/download` — download specific report
- [ ] Integrate SheetJS or ExcelJS for in-browser preview
- [ ] Add thumbnail/preview of first sheet

**New Table**:
```sql
CREATE TABLE export_reports (
  id INT PRIMARY KEY AUTO_INCREMENT,
  report_type VARCHAR(50),  -- full/assembly/pickup
  file_path VARCHAR(255),
  generated_by_user_id INT,
  created_at DATETIME,
  order_count INT,
  file_size_bytes BIGINT
);
```

### 4. Real-time Notifications (Low Priority)
**Current State**: Admins must refresh to see new orders

**Tasks**:
- [ ] Add WebSocket support to admin_service
- [ ] Create notification service (Redis Pub/Sub or SSE)
- [ ] Add "New Order" toast in UI when order arrives
- [ ] Add "Order Status Changed" notification
- [ ] Add notification preferences in SettingsPage

**Tech Stack**:
- Backend: FastAPI WebSocket or Server-Sent Events (SSE)
- Frontend: WebSocket client or EventSource API
- Optional: Redis for pub/sub across multiple workers

### 5. Advanced Analytics (Medium Priority)
**Current State**: Basic charts, no deep insights

**Tasks**:
- [ ] Add time-series chart (orders per hour/day/week)
- [ ] Add pie chart for status distribution
- [ ] Add "Top Customers" leaderboard
- [ ] Add "Revenue Tracking" (if prices available)
- [ ] Add cohort analysis (repeat customers)
- [ ] Add export statistics as PDF report

**Libraries**:
- Frontend: recharts (already in package.json)
- Backend: pandas for complex calculations

### 6. Backup & Recovery (High Priority for Production)
**Current State**: No automatic backups

**Tasks**:
- [ ] Add scheduled database backup (daily)
- [ ] Store backups on external storage (S3/MinIO/local)
- [ ] Add "Restore from Backup" button in UI
- [ ] Add backup history viewer
- [ ] Add export/import all settings as JSON

**New Scripts**:
- `scripts/backup_db.sh` — mysqldump wrapper
- `scripts/restore_db.sh` — restore from backup
- Cron job or Docker service for scheduling

### 7. Advanced User Management (Medium Priority)
**Current State**: базовый и средний уровень user-management уже реализован in `UsersPage`

**Already available**:
- ✅ список пользователей;
- ✅ создание / удаление;
- ✅ назначение ролей;
- ✅ активация / деактивация;
- ✅ управление регионами;
- ✅ reset password;
- ✅ owner-promotion с подтверждением.

**Tasks that still make sense**:
- [ ] Add activity log (login history, actions)
- [ ] Add audit trail UI for role/region/password changes
- [ ] Add bulk actions for large admin teams
- [ ] Add optional invite/onboarding flow instead of manual password handoff

### 8. Mobile Optimization (Low Priority)
**Current State**: UI works on mobile but not optimized

**Tasks**:
- [ ] Test all pages on mobile devices
- [ ] Adjust table layouts for narrow screens
- [ ] Add mobile-friendly navigation (bottom nav?)
- [ ] Optimize touch interactions
- [ ] Add PWA manifest for "Add to Home Screen"

### 9. Localization (Very Low Priority)
**Current State**: All Russian, no i18n

**Tasks**:
- [ ] Extract all strings to i18n files
- [ ] Add language selector in UI
- [ ] Support English + Russian
- [ ] Add locale-aware date/number formatting

### 10. Testing & CI/CD (High Priority for Production)
**Current State**: automated tests exist and baseline CI is assembled, but CI/CD is not yet full-spectrum

**Already available in repo**:
- ✅ backend pytest suite;
- ✅ admin_service pytest suite;
- ✅ admin-web Playwright scenarios.
- ✅ GitHub Actions baseline (`.github/workflows/ci.yml`)

**Tasks**:
- [ ] Stabilize local Python env so tests run predictably on one documented interpreter
- [ ] Add GitHub Actions for:
  - Linting (ruff, eslint)
  - Type checking (mypy, tsc)
  - Test running
  - Docker build verification
- [ ] Separate smoke checks from slower regression suites
- [ ] Document canonical local test commands in README / testing docs

## 📊 Priority Matrix

### Must Have (Before Real Production)
1. Order/Catalog Editing (High)
2. Backup & Recovery (High)
3. Testing & CI/CD (High)

### Should Have (Within 1-2 Months)
4. AI Integration (Medium)
5. Excel Preview (Medium)
6. Advanced Analytics (Medium)
7. Advanced User Management (Medium)

### Nice to Have (Future)
8. Real-time Notifications (Low)
9. Mobile Optimization (Low)
10. Localization (Very Low)

## 🚀 Recommended Next Steps

### Week 1-2: Core Stability
1. Add automated backups
2. Stabilize and document critical tests
3. Set up CI/CD pipeline

### Week 3-4: Admin Productivity
4. Implement order editing
5. Add catalog management
6. Enable AI fallback

### Month 2: User Experience
7. Add Excel preview
8. Enhance analytics
9. Improve mobile UI

### Month 3+: Advanced Features
10. Real-time notifications
11. Advanced user-management and audit UI
12. Localization (if needed)

---

**Current State**: ✅ Production-Ready для базового use case (сбор заказов + экспорт + настройки)

**Target State**: 🎯 Enterprise-Ready с полным UI управлением, автобэкапами, тестами и CI/CD
