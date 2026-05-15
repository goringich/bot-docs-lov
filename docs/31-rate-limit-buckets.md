---
title: Rate-limit buckets — admin_service middleware
type: reference
status: current
tags: [rate-limit, middleware, security, ops]
updated: 2026-05-01
related:
  - "[[27-security-runbook]]"
  - "[[26-export-contract]]"
  - "[[../admin_service/app/main.py]]"
---

# Rate-limit buckets

`admin_service/app/main.py::RateLimitMiddleware` распределяет HTTP-запросы по бакетам по `(client_ip, path-pattern)` и применяет per-bucket лимит.

## Бакеты

| Bucket | Лимит (req/min) | Path-патерны |
|---|---:|---|
| `login` | 5 | `/auth/login` |
| `bootstrap` | 5 | `/auth/bootstrap` |
| `auth` | 10 | прочие `/auth/*` |
| `export` | 20 | `/orders/export*`, `/exports/*` (кроме `/exports/presets`) |
| `security` | 60 | `/exports/presets`, `/debug/*`, `/audit-log`, `/login-attempts`, `/security-summary`, `/security-activity` |
| `global` | 100 | всё остальное |

## Правила

> [!warning] При изменении бакетов
> 1. Любая правка `_bucket_key` обязана иметь синхронную правку в `_get_limit` (одинаковый набор условий).
> 2. Любая правка обоих обязана иметь обновление `admin_service/tests/test_export_rate_limit.py`.
> 3. Read-only «горячие» эндпоинты (то, что UI дёргает на каждый mount страницы) должны жить в `security`/`global`, **не** в `export`. Иначе UI начнёт ловить 429 на ровном месте.

## Защита от DoS / IP-spoofing

- `_max_buckets = 50_000` — при переполнении dictionary дроп старшей половины (LRU по последнему хиту).
- `_cleanup_interval = 60s` — периодическая очистка пустых bucket'ов.
- `Retry-After` — возвращается клиенту с временем до первого истёкшего хита.

## Как менять лимит

В `__init__`:

```python
self._auth_limit = 10        # /auth/*
self._login_limit = 5        # /auth/login
self._export_limit = 20      # heavy export endpoints
self._security_limit = 60    # read-only security endpoints
```

«Глобальный» лимит — в `app.add_middleware(RateLimitMiddleware, max_requests=..., window_seconds=...)` в самом `main.py` (по умолчанию 100/60s).

## Канонический FAQ

**Почему `/exports/presets` не в `export`?**
Потому что UI грузит его на каждый mount ExportPage; со старым `export_limit=3` это упирало в 429 за пару секунд работы. См. `test_export_presets_use_security_bucket_not_export_bucket`.

**Почему именно 20 для `export`?**
Тяжёлые эндпоинты (build XLSX из 50k строк) — оператор не должен дёргать их 60 раз в минуту, но 3 — слишком жёстко (UI пингует, повторные попытки, e2e-проверки). 20 — компромисс, который даёт человеку дышать и не валит fileserver.

## Связанные тесты

- `admin_service/tests/test_export_rate_limit.py`
- `backend/tests/test_rate_limiter.py`
