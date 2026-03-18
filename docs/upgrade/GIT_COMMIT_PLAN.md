# Git Commit Plan — v2.0 Production Upgrade

## Files to commit

### Backend (Python)
1. `backend/app/models.py` — Added BotSetting and PickupPlace tables
2. `backend/app/presenter/order_presenter.py` — Enhanced UX with minimal buttons
3. `backend/app/domain/order_domain.py` — Improved looks_like_order() heuristics
4. `backend/app/worker.py` — Updated messages with emoji
5. `backend/migrations/versions/c9f8a1234567_add_bot_settings_and_pickup_places.py` — New migration

### Admin Service (Python)
6. `admin_service/app/main.py` — Added settings & pickup places API endpoints

### Admin Web (TypeScript/React)
7. `admin-web/src/App.tsx` — Added dark theme context and toggle
8. `admin-web/src/api/client.ts` — Added settings & pickup places methods
9. `admin-web/src/pages/SettingsPage.tsx` — Connected to real API
10. `admin-web/src/pages/OrdersPage.tsx` — Added CSV export
11. `admin-web/src/components/MainLayout.tsx` — Added theme toggle button

### Documentation
12. `README.md` — Updated "What's New" section
13. [[changelog/decisions-log]] — Added v2.0 section
14. [[PRODUCTION_UPGRADE_v2.0]] — Deployment checklist
15. [[upgrade/UPGRADE_SUMMARY]] — User-friendly summary

## Commit message

```
feat(v2.0): production-ready upgrade with UI settings management

Backend:
- Add bot_settings and pickup_places tables
- Improve bot UX with minimal inline keyboards
- Enhance order recognition (filter recipes, media links)
- Add emoji and Markdown to bot messages

Admin API:
- Add GET/POST /settings endpoints
- Add CRUD endpoints for /pickup-places
- Enable bot configuration without code changes

Admin UI:
- Add dark theme with persistent toggle
- Implement real SettingsPage (reply mode, messages, pickup places)
- Add CSV export on OrdersPage
- Enhance AnalyticsPage with bar charts

Migration: c9f8a1234567_add_bot_settings_and_pickup_places

BREAKING: Requires database migration before deployment
```

## Commands to execute

```bash
cd /home/goringich/Desktop/father-bot

# Stage all changes
git add backend/app/models.py
git add backend/app/presenter/order_presenter.py
git add backend/app/domain/order_domain.py
git add backend/app/worker.py
git add backend/migrations/versions/c9f8a1234567_add_bot_settings_and_pickup_places.py
git add admin_service/app/main.py
git add admin-web/src/App.tsx
git add admin-web/src/api/client.ts
git add admin-web/src/pages/SettingsPage.tsx
git add admin-web/src/pages/OrdersPage.tsx
git add admin-web/src/components/MainLayout.tsx
git add README.md
git add docs/changelog/decisions-log.md
git add docs/PRODUCTION_UPGRADE_v2.0.md
git add docs/upgrade/UPGRADE_SUMMARY.md

# Commit
git commit -m "feat(v2.0): production-ready upgrade with UI settings management

Backend:
- Add bot_settings and pickup_places tables
- Improve bot UX with minimal inline keyboards  
- Enhance order recognition (filter recipes, media links)
- Add emoji and Markdown to bot messages

Admin API:
- Add GET/POST /settings endpoints
- Add CRUD endpoints for /pickup-places
- Enable bot configuration without code changes

Admin UI:
- Add dark theme with persistent toggle
- Implement real SettingsPage (reply mode, messages, pickup places)
- Add CSV export on OrdersPage
- Enhance AnalyticsPage with bar charts

Migration: c9f8a1234567_add_bot_settings_and_pickup_places

BREAKING: Requires database migration before deployment"

# Push (if needed)
git push origin master
```
