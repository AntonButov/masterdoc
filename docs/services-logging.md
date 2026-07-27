# Логирование runtime-сервисов Fixaverse

> Проверено: 2026-07-27 · источники: аудит кода + CI deploy сессии

## Вердикт

Логи сервисов идут в **stdout/stderr** (Docker `compose logs` или `journalctl` для backend). Критичные ops-события пишутся в формате `event=<name> key=value`; токены, JWT, тела запросов и клинический текст в логи не кладутся.

**С чего начать:** на VPS `cd /opt/<repo> && docker compose logs -f | grep event=` · backend: `journalctl -u masterdoc-backend -f`.

## Куда смотреть

| Сервис | Путь на VPS | Как читать |
|--------|-------------|------------|
| api-gateway, feature, catalog, document, dashboard, black-box, technologist, technologist-mcp, maintenance-service | `/opt/<repo>/` | `docker compose logs -f` |
| backend (`masterdoc-backend`) | systemd | `journalctl -u masterdoc-backend -f` |

Отдельных файлов `/var/log/...` в деплое нет.

## Что залогировано (2026-07-27)

| Сервис | Commit / ветка | Критичные `event=` |
|--------|----------------|-------------------|
| api-gateway-service | `462c596`+ main | `jwt_invalid`, `auth_denied`, `upstream_*`, `proxy_error`, `startup`/`shutdown` |
| backend | `cd6edb9` main | CallLogging + `upstream_error`/`handler_error`/`chat_timing`/`detect_*`/`case_report_saved`; без PHI в message |
| feature-service | `46f9667` master | `startup` (JWT flags), `auth_failed` |
| catalog-service | logging + later | CallLogging, StatusPages WARN, `startup` |
| document-service | logging + later | `documents_recovered`, `document_saved`, `upload_rejected` |
| dashboard-service | `eb37988` main | `catalog_lookup_failed`, `scheduler_tick`/`scheduler_failed` (maps via HTTP) |
| black-box-service | `9ae54c0` main | `auth_denied`, `audit_appended` |
| technologist-service | `8555af7` main | `job_*`, `document_text_fetch`, `mcp_call`, `package_confirmed` |
| technologist-mcp | `d59ea8a` main | `tool_call`/`tool_denied`/`tool_error`/`upstream_call` (asset tools only) |
| maintenance-service | `d057b44` main | `startup`, `dashboard_tick_failed`, CallLogging; MCP role on :8099 |

План работ: [docs/superpowers/plans/2026-07-26-services-critical-logging.md](superpowers/plans/2026-07-26-services-critical-logging.md).

## Не брать / оговорки

- Не логировать `Authorization`, JWT, `client_secret`, `BLACK_BOX_INTERNAL_TOKEN`, текст документов, answer/result case-report, body Onyx.
- `OnyxException` содержит только HTTP status в `message` (без upstream body).
- CallLogging на `/health` может шуметь — при необходимости фильтровать на уровне логгера.

## История правок

| Дата | Что изменили |
|------|----------------|
| 2026-07-26 | первичная запись после аудита и дологирования критичных участков |
| 2026-07-27 | добавлен `maintenance-service` (+ MCP); dashboard maps вынесены |
