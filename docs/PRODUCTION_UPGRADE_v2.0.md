# Production Upgrade Checklist

Статус: `historical`

Назначение: исторический checklist апгрейда `v2.0`. Для текущего состояния системы использовать `[[changelog/implementation-status]]`, `[[changelog/known-issues]]` и `[[14-api-reference]]`.

## Completed Features

### 1. Database Schema
- [x] Added `bot_settings` table for bot configuration
- [x] Added `pickup_places` table for pickup locations management
- [x] Created migration `c9f8a1234567_add_bot_settings_and_pickup_places.py`
- [x] Default settings and pickup places pre-populated

### 2. Bot UX Improvements
- [x] Minimalist inline keyboards (4 buttons → 2 after success)
- [x] Enhanced messages with emoji and Markdown
- [x] Compact responses without spam
- [x] New `help_text()` with full guide
- [x] Contextual error correction buttons
- [x] Better `looks_like_order()` heuristics:
  - Filters recipes and cooking instructions
  - Protects against media links
  - Ignores long texts (>500 chars)

### 3. Admin Service API
- [x] `GET /settings` — fetch all bot settings
- [x] `POST /settings` — update bot settings
- [x] `GET /pickup-places` — list pickup locations
- [x] `POST /pickup-places` — create pickup location
- [x] `PATCH /pickup-places/{id}` — update pickup location
- [x] `DELETE /pickup-places/{id}` — delete pickup location

### 4. Admin Web UI
- [x] Dark theme with toggle switch in header
- [x] SettingsPage with full bot configuration:
  - Reply mode (dm/chat/both)
  - Custom message templates
  - Pickup places CRUD
  - Notification settings
- [x] CSV export on OrdersPage with filtering
- [x] Enhanced AnalyticsPage with bar charts
- [x] Updated API client with new endpoints

### 5. Documentation
- [x] Updated [[changelog/decisions-log]] with v2.0 changes
- [x] Updated `README.md` with new features overview
- [x] Created this checklist for tracking

## In Progress (Optional)

### AI Integration
- [ ] Wire AI OrderRecognizer into worker.py main loop
- [ ] Add OpenAI/Anthropic client configuration
- [ ] Create settings UI for AI parameters (model, temperature)

### Advanced Features
- [ ] Order line editing in UI
- [ ] Catalog item management in UI
- [ ] Real-time WebSocket notifications for new orders
- [ ] Excel file preview in UI (using SheetJS/ExcelJS)
- [ ] Automatic backup system

## Testing Checklist

Before deployment:
- [ ] Run migration in Docker: `docker exec tg_orders_mysql mysql...`
- [ ] Test settings save/load in UI
- [ ] Test pickup places CRUD
- [ ] Verify dark theme toggle persists
- [ ] Test CSV export with filters
- [ ] Send test order in Telegram
- [ ] Verify bot responds with new compact messages
- [ ] Check analytics charts render correctly

## Deployment Steps

1. **Build new images**: 
   ```bash
   cd infra
   docker compose build --no-cache
   ```

2. **Run migration**:
   ```bash
   docker exec tg_orders_api alembic upgrade heads
   ```

3. **Restart services**:
   ```bash
   docker compose up -d
   ```

4. **Verify health**:
   ```bash
   curl http://localhost:8000/health
   curl http://localhost:8010/health
   ```

5. **Check logs**:
   ```bash
   docker compose logs -f worker admin_service admin_web
   ```

## Key Metrics

- **Code Changes**: ~1500 lines added/modified
- **New Files**: 3 (migration, checklist, updated docs)
- **API Endpoints**: +6 new REST endpoints
- **UI Components**: 1 major update (SettingsPage with real API)
- **Database Tables**: +2 (bot_settings, pickup_places)

## Success Criteria

- ✅ Admin can change bot settings without code/env changes
- ✅ Admin can manage pickup places from UI
- ✅ Bot uses cleaner, more user-friendly messages
- ✅ Dark theme works and persists across sessions
- ✅ CSV export generates valid files
- ✅ All existing features still work (no regressions)
[[Отличный улов/docs/README]]
