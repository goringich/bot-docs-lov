---
title: AI agents knowledge base — how Claude / Copilot / Codex use this docs vault
type: guide
status: current
tags: [ai, claude, copilot, codex, prompts, obsidian, knowledge-base]
updated: 2026-05-01
related:
  - "[[_index]]"
  - "[[../AGENTS.md]]"
  - "[[../CLAUDE.md]]"
  - "[[../.github/copilot-instructions.md]]"
---

# AI агенты и эта база знаний

> [!important] Контракт «один источник, две поверхности»
> Папка `docs/` — единственный источник истины для AI-агентов в этом репозитории. Папка bind-mount'нута в Obsidian-vault (`/Отличный улов/docs/`) через [[obsidian-repo-mounts]]. Любой агент должен читать **отсюда**, а не из training data, не из Obsidian-копий, не из кэша.

## Поверхности

| Агент | Точка входа в репозитории | Что должно быть прочитано первым |
|---|---|---|
| **Claude Code** (CLI / IDE) | `CLAUDE.md` (root) | `_index.md`, `AGENTS.md`, `changelog/codex-project-memory.md` |
| **GitHub Copilot Chat** | `.github/copilot-instructions.md` | `_index.md`, `AGENTS.md`, `changelog/codex-project-memory.md` |
| **OpenAI Codex / агентные раннеры** | `AGENTS.md` (root) | те же три |
| **Obsidian** (как читалка для человека) | `_index.md` | связанные `[[wikilinks]]` |

## Что обязан делать любой AI-агент

1. **Читать `docs/_index.md` перед началом любой нетривиальной задачи** — это MOC всей базы.
2. **Если задача про парсер / каталог / заказы / экспорт** — открыть [[26-catalog-source-of-truth]], [[26-export-contract]], [[28-stability-playbook]].
3. **Если задача про auth / middleware / роли / экспорт** — открыть [[27-security-runbook]] и [[31-rate-limit-buckets]].
4. **Любой контрактный документ — это барьер для регрессии**. Если код противоречит документу, либо документ устарел (PR в обе стороны), либо код некорректен.
5. **Не выдумывать имена эндпоинтов / ключей настроек**: всегда сверять с фактом (`grep` в `admin_service/app/api/routers/`).

## Что AI-агенту делать **нельзя**

> [!danger] Антипаттерны (видели в реальных регрессиях)
> - Доучивать каталог из заказов (см. [[26-catalog-source-of-truth#антипаттерны]]).
> - Добавлять metadata-колонку в экспорт без wired в `_fill_strict_template_row` (см. [[28-stability-playbook#тестовый-минимум]]).
> - Менять рейт-лимит middleware без обновления `test_export_rate_limit.py` (см. [[31-rate-limit-buckets#правила]]).
> - Класть sysadmin-тогглы на SettingsPage (см. `AGENTS.md → Placement rules`).
> - «Чинить» падающий тест переименованием expectations или skip-маркером.

## Как агенту цитировать документы

Внутри ответов и комментариев в коде — относительные пути от корня репозитория:

```
docs/26-export-contract.md#жесткий-формат-таблицы
docs/28-stability-playbook.md#тестовый-минимум
```

В Obsidian — wikilinks:

```
[[26-export-contract#жёсткий-формат-таблицы]]
```

## Связь с Obsidian (operator-side)

Vault: `/home/goringich/Desktop/Obsidian/Отличный улов/`. Bind-mount держится через [[obsidian-repo-mounts]]. Конфигурация (manifest) — отдельный файл (см. `~/obsidian-repo-mounts/examples/`). Чтобы убедиться:

```sh
mountpoint /home/goringich/Desktop/Obsidian/Отличный улов/docs
# expected: is a mountpoint
```

Если bind-mount не активен — Obsidian увидит пустую папку или старый снимок. Перезагрузи mount через `obsidian-repo-mounts` (или соответствующий `systemd` unit, см. roadmap проекта `obsidian-repo-mounts`).

## Свежесть документов

- Колонка `updated:` в frontmatter ставится вручную при правке.
- Колонка `status:` — `current / historical / target / decision / archive`. AI-агент не доверяет `historical / archive` как описанию текущего поведения.
- При расхождении документа с кодом — приоритет: фактическое поведение в `git log` за последние 30 дней (там видны реальные fixes).

## Минимальный prompt для нового агента

```
You are working in the otlichniy-ulov repo. Before any non-trivial task:
1. Read docs/_index.md.
2. If task touches parser/catalog/orders/export: also read
   docs/26-catalog-source-of-truth.md, docs/26-export-contract.md,
   docs/28-stability-playbook.md.
3. If task touches auth/middleware/roles/security: also read
   docs/27-security-runbook.md and docs/31-rate-limit-buckets.md.
4. Follow AGENTS.md "Mandatory workflow" and "Export contract checklist".
5. Never invent endpoint names or settings keys — grep first.
6. Test before claiming done — at minimum the targeted suite from
   docs/28-stability-playbook.md#тестовый-минимум.
```
