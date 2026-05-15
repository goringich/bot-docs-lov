---
title: Security runbook — auth / cookies / CSRF / export
type: runbook
status: current
tags: [security, auth, csrf, rate-limit, owner, runbook]
updated: 2026-05-01
related:
  - "[[11-security]]"
  - "[[24-admin-auth-and-integration-security-runbook]]"
  - "[[26-export-contract]]"
  - "[[31-rate-limit-buckets]]"
---

# Security runbook

> [!important] Когда задача «risky»
> Любое касание `auth/login/cookie/CSRF/middleware/POST-эндпоинтов`, `routers/catalogs.py`, `core_tables`, или схема-совместимости `chats / catalogs / catalog_items` — **risky**. Запускай `make check-post-preflight` (см. ниже) перед закрытием задачи.

## Слои контроля доступа

```
HTTP request
  │
  ├─ RateLimitMiddleware                  (per-IP buckets, см. [[31-rate-limit-buckets]])
  ├─ CSRF / cookie middleware             (см. [[24-admin-auth-and-integration-security-runbook]])
  ├─ get_current_user (JWT bearer)        (Authorization header)
  ├─ Role gate                            (is_owner / is_owner_or_sysadmin / is_pickup_admin)
  ├─ Pickup-scope guard                   (_apply_pickup_scope_filter / _ensure_order_pickup_scope)
  └─ Password re-auth для чувствительных  (require_password_confirmation("export"))
```

## Чек-лист PR в auth/middleware

- [ ] Любой новый POST/PATCH/DELETE — есть `is_owner` или явное role-ограничение.
- [ ] Любая чувствительная выгрузка — `require_password_confirmation("export")` + `_require_owner_export_access`.
- [ ] Pickup-admin не получает чужие точки (`_get_pickup_scope_titles`).
- [ ] CSRF-токен обновлён, если поменялся `auth/login` или ротация куков.
- [ ] Bucket в `RateLimitMiddleware._bucket_key` и `_get_limit` синхронны.
- [ ] `make check-post-preflight` пройден (см. `Makefile`).

## Excel-формула инъекции

Все ячейки выгрузки прогоняются через `_sanitize_excel_value`. Запрещены ведущие `=`, `+`, `-`, `@`. Любая новая запись в ячейку — через эту функцию (см. `admin_service/tests/test_export_template_sanitization.py`).

## Секреты и окружение

См. [[11-security]] и [[10-ops-runbook]]. В CI токены подменены на `test-*`. Никаких реальных секретов в `.github/workflows/ci.yml`.

## Sysadmin-настройки и скрытие фич

| Ключ | Назначение |
|---|---|
| `export_mode_template / _strict / _distribution / _flexible` | Скрытие режимов экспорта целиком. |
| `export_filter_show_date_range / _pickup` | Скрытие опциональных полей фильтра экспорта. |
| `feature_catalog_reparse / _reparse_all / _import_text / _clone / _delete` | Гейтинг feature-флагов каталога. |
| `delivery_show_*` | Скрытие элементов модуля выдачи. |

Все ключи объявлены в `admin_service/app/api/routers/settings.py::SYSADMIN_UI_DEFAULTS` и могут быть выставлены только sysadmin. Owner-доступ ограничен `NON_OWNER_SETTINGS_KEYS` ∪ `SYSADMIN_UI_DEFAULTS`.

## Owner debug mode

`owner_debug_mode` — глобальный «view as» для owner. Любой новый «опасный» инструмент в админке должен быть гейтом `if owner_debug_mode: ...`.

## Журналирование

- `audit` — мутирующие действия в `admin_service/app/audit.py::write_audit`.
- `login_attempts` / `security_summary` — read-only эндпоинты, по `security` bucket рейт-лимита.

## Минимальный smoke

```sh
make check-post-preflight        # auth/cookie/CSRF/middleware
make check-catalogs-preflight    # caталог + core_tables compat
make check-prebuild              # перед compose-build / restart / sync
```
