# Work orders + weekly board (dashboard-service)

**Date:** 2026-07-25  
**Status:** implementation contract  
**Related:** [TOIR_AI_SYSTEM_DESIGN.md](../../../TOIR_AI_SYSTEM_DESIGN.md), [equipment-technologist-design](2026-07-22-equipment-technologist-design.md), [B2B_MVP_SCOPE.md](../../../B2B_MVP_SCOPE.md)

## Scope (MVP)

- `dashboard-service` owns **WorkOrder** + weekly **board** + **PPR scheduler** (auto-create from active maintenance maps).
- Public paths via **api-gateway** under feature `board` (`/work-orders`).
- UI (7-day board + duration hours) — see [2026-07-26-board-seven-day-duration-design.md](2026-07-26-board-seven-day-duration-design.md).

## Types and lifecycle

| Field | Values |
|-------|--------|
| `type` | `ppr` \| `emergency` |
| `status` | `new` → `in_progress` → `closed` |

- Create always: `status=new`, `assigneeId=null` (other services PATCH assignee later).
- Transitions: `new→in_progress`, `in_progress→closed`. No reopen / no cancel in MVP.
- Assignee change only while not `closed`.

## Required equipment binding

Every work order **must** have:

- `assetId` — unit of equipment
- `siteId` — site of that asset

For `type=ppr`: also `maintenanceMapId` + `maintenanceMapItemId`; map must be same org and `map.assetId == assetId`.

## API (service :8092 → gateway 1:1)

| Method | Path | Notes |
|--------|------|--------|
| POST | `/work-orders` | Body requires `type`, `title`, `assetId`, `siteId`, `dueAt`; ppr requires map+item ids |
| GET | `/work-orders/{id}` | |
| GET | `/work-orders/board?weekStart=&weeks=` | ISO weeks Mon–Sun; default `weeks=4`, `weekStart` = Monday of current UTC week |
| PATCH | `/work-orders/{id}` | `status?`, `title?`, `dueAt?`, `assigneeId?` (JSON null clears) |
| POST | `/internal/scheduler/tick` | Idempotent PPR generation; **not** exposed on gateway |

Board shape:

```json
{ "weeks": [ { "weekStart": "2026-07-20", "items": [ /* WorkOrder */ ] } ] }
```

Empty weeks are included in the range.

## Scheduler

- Source: maintenance maps with `status=active` and `activatedAt` set on confirm.
- Items with `interval.unit == days` only; `hours` / `cycles` skipped.
- Due dates: `activatedAt.date + n * every` for `n ≥ 1` while `dueAt` ∈ `[horizonStart, horizonEnd)`.
- Horizon: Monday of current week … + `BOARD_HORIZON_WEEKS` (default 4).
- WO fields: `type=ppr`, `source=scheduler`, `assetId` from map, `siteId` from catalog asset lookup.
- Idempotent key: `(orgId, maintenanceMapId, maintenanceMapItemId, dueAt)`.
- After map `confirm`, run tick for that org (map).

## Auth

Gateway: Bearer JWT + feature `board`. Services trust `X-Org-Id` / `X-User-Id`.
