# Maintenance service extraction (ППР + MCP)

**Date:** 2026-07-27  
**Status:** approved (ready for plan)  
**Related:** [2026-07-22-equipment-technologist-design.md](2026-07-22-equipment-technologist-design.md), [2026-07-25-work-orders-board-design.md](2026-07-25-work-orders-board-design.md), [2026-07-22-maintenance-map-practices.md](2026-07-22-maintenance-map-practices.md)

## Problem

ППР (maintenance maps) сегодня живут **в памяти** `dashboard-service` (`MaintenanceMapStore` / `ConcurrentHashMap`). MCP-тулы ППР сидят в `technologist-mcp` и пишут туда же. Данные теряются при рестарте; граница «доска/WO» vs «карты обслуживания» размыта.

## Goal

Вынести ППР в отдельную репу и runtime **`maintenance-service`**: собственный HTTP API, Postgres на диске, отдельный MCP-процесс в той же репе. Публичный контракт клиента не менять.

## Decisions (locked)

| Тема | Выбор |
|------|--------|
| Имя репы / сервиса | `maintenance-service` (не `mantainanc-service`) |
| Упаковка | **Два процесса** в одной репе: HTTP API + MCP |
| Персист | **Postgres** в compose (volume на диск) |
| Планировщик ППР→WO | **Остаётся в `dashboard-service`**; active maps читает по HTTP |
| Подход | Lift-and-shift текущего JSON-контракта maps + Postgres |
| Миграция старых карт | **Нет** — in-memory данные эфемерны |

## Boundaries

### New repo: `Fixaverse/maintenance-service`

Deploy root on API VPS: `/opt/maintenance-service/`.

| Process | Default port | Role |
|---------|--------------|------|
| `maintenance-service` | 8098 | REST maps CRUD, confirm/reject, list (incl. active) |
| `maintenance-mcp` | 8099 | MCP tools for draft maps only |
| `postgres` | internal | Persistent store for maps + items |

### Leaves `dashboard-service`

- `MaintenanceMap*` models, `MaintenanceMapStore`, `/maintenance-maps*` routes
- Local map dependency of `PprScheduler` and WO `type=ppr` validation

### Stays in `dashboard-service`

- Work orders, weekly board
- `PprScheduler` + `/internal/scheduler/tick` (maps via HTTP client)
- Catalog asset lookup for WO site resolution

### Leaves `technologist-mcp`

- `create_draft_maintenance_map`
- `update_draft_maintenance_map`

### Stays in `technologist-mcp`

- `create_draft_asset`, `update_draft_asset`, `get_document_meta`

## Public API (unchanged paths)

Gateway continues to expose:

- `POST/GET/PATCH /maintenance-maps`
- `GET /maintenance-maps/{id}`
- `POST /maintenance-maps/{id}/confirm`
- `POST /maintenance-maps/{id}/reject`

Proxy target changes from `DASHBOARD_SERVICE_BASE_URL` → **`MAINTENANCE_SERVICE_BASE_URL`**.

Feature gates unchanged: `equipment` + `charts`.

JSON shapes stay aligned with current `MaintenanceMap` / items / enums (`draft`/`active`, `manual`/`ai_generated`, kinds, intervals, criticality).

### Create rules

- Require non-blank `title`, non-empty `items`
- Asset must exist in catalog (`CATALOG_BASE_URL`) for the `X-Org-Id`
- MCP creates always `status=draft`, `source=ai_generated`

### Confirm / reject

- Confirm: `draft` → `active`, set `activatedAt`, then trigger dashboard scheduler tick for that `mapId` (HTTP `POST` dashboard `/internal/scheduler/tick?orgId=&mapId=`)
- Reject: delete draft only; 204

### Scheduler feed

Dashboard needs active maps without org-local store. Maintenance exposes listing that can filter active maps (extend existing `GET /maintenance-maps` with `status=active`, and allow org filter via `X-Org-Id`; for global hourly tick, maintenance provides an internal list endpoint such as `GET /internal/active-maps` trusted on the private network — same trust model as today's internal tick).

Preferred:

- Hourly / explicit tick: dashboard calls `GET {MAINTENANCE}/internal/active-maps?orgId=&mapId=`
- WO create `type=ppr`: dashboard `GET {MAINTENANCE}/maintenance-maps/{id}` to validate map + item

## Postgres schema

```text
maintenance_maps
  id            text PK
  org_id        text not null
  asset_id      text not null
  title         text not null
  status        text not null  -- draft | active
  source        text not null  -- manual | ai_generated
  activated_at  timestamptz null
  created_at    timestamptz not null
  updated_at    timestamptz not null

maintenance_map_items
  id              text PK
  map_id          text not null references maintenance_maps(id) on delete cascade
  title           text not null
  kind            text not null
  interval_every  int not null
  interval_unit   text not null
  criticality     text not null
  source_ref      text null
```

Indexes: `(org_id)`, `(org_id, asset_id)`, `(status)` / `(org_id, status)`.

Migrations at startup (Flyway or equivalent SQL runner). Postgres data dir on a Docker volume.

## MCP · `maintenance-mcp`

Tools (moved from `technologist-mcp`):

| Tool | Upstream |
|------|----------|
| `create_draft_maintenance_map` | `POST {MAINTENANCE}/maintenance-maps` with `source=ai_generated` |
| `update_draft_maintenance_map` | `PATCH {MAINTENANCE}/maintenance-maps/{id}` |

Same HTTP MCP shape as today: `GET /mcp/tools`, `POST /mcp/tools/call`, header `X-Org-Id`.

## Flows

```text
technologist-service
  → MAINTENANCE_MCP_BASE_URL  (draft map tools)
  → MAINTENANCE_SERVICE_BASE_URL  (confirm map, if used)

client → gateway /maintenance-maps → maintenance-service → Postgres

POST .../confirm
  → maintenance-service: persist active + activated_at
  → dashboard /internal/scheduler/tick?mapId=
      → GET maintenance internal active map(s)
      → create PPR work orders in dashboard store
```

## Affected repos / cutover

1. Create & deploy `maintenance-service` (API + MCP + Postgres), smoke CRUD/confirm.
2. Switch `api-gateway-service` proxy + env.
3. Switch `technologist-service` env (`MAINTENANCE_MCP_BASE_URL`, maintenance base for confirm).
4. Refactor `dashboard-service` (remove maps; HTTP client for scheduler + WO validation).
5. Strip map tools from `technologist-mcp`.
6. Update `masterdoc` docs (this spec + equipment-technologist contract pointers).

Order matters: **maintenance live before** gateway cutover. No dual-write.

## Out of scope

- Persisting work orders / board
- Changing client deep links or UI
- Renaming public paths
- Migrating historical in-memory maps
- Moving PPR→WO scheduler into maintenance-service

## Success criteria

- `GET/POST https://api.masterdoc.pro/maintenance-maps` served by maintenance-service from Postgres
- Maps survive container restart
- Technologist draft PPR still works via `maintenance-mcp`
- Confirm still activates map and produces scheduler WOs on the board
- dashboard-service no longer owns `/maintenance-maps`
- CI: test + deploy jobs for the new repo on `main`/`master`
