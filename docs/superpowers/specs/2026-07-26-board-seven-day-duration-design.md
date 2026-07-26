# Board: 7-day week + work duration (hours)

**Date:** 2026-07-26  
**Status:** approved for planning  
**Related:** [2026-07-25-work-orders-board-design.md](2026-07-25-work-orders-board-design.md), `client-app` BoardScreen / WorkOrderDetailScreen, `dashboard-service` WorkOrder, `fixaverse-design` / client design-system

## Problem

Current board is four **week** columns with dense cards (type, status, due, assignee). It should be a **single calendar week (Mon–Sun)** where a work order is a short horizontal card spanning one or more **working days**. Full fields live on the detail screen.

## Decisions

| Topic | Choice |
|-------|--------|
| Duration storage | `durationHours: Int` on WorkOrder |
| Day span on board | `spanDays = max(1, ceil(durationHours / 8))` (8 h = 1 workday) |
| Anchor date | `dueAt` = **start** date (YYYY-MM-DD); card extends forward |
| Weekend | Span uses **working days only** (Mon–Fri); Sat/Sun skipped |
| Board API shape | Unchanged (`weeks[]` + items); client lays out Mon–Sun |
| Card content | Short **title** + light type accent (ППР / авария) only |
| Detail | Existing detail screen + duration hours (view / PATCH) |
| Visual system | client-app design-system + `fixaverse-design` theme tokens |

## Data / API (`dashboard-service`)

- Add **`durationHours`** to WorkOrder create body, entity, GET, PATCH.
- Validation: `durationHours >= 1`; reject otherwise.
- Default when missing (scheduler, legacy rows): **8**.
- `dueAt` remains required YYYY-MM-DD and means **start**.
- `GET /work-orders/board` response shape unchanged; items include `durationHours`.

## Client layout rules

Pure functions (unit-tested), no server layout:

1. `spanDays(hours) = max(1, ceil(hours / 8.0))`
2. From `dueAt`, walk forward counting only Mon–Fri until `spanDays` working days are placed → set of calendar dates occupied.
3. Visible week = Mon–Sun of selected week (default: week containing today). Prev/next week controls.
4. A WO is shown on a week if its occupied dates intersect that week’s Mon–Fri (or any day of the week grid if a date falls on weekend only via calendar placement — in practice occupied dates are only weekdays).
5. If the span crosses a week boundary, each week shows the **clip** of the bar for that week (continuation on adjacent week when navigated).

## UI

### Board

- Header: title «Доска», week range, ← / →.
- Seven equal day columns (weekday short name + date).
- Cards as horizontal bars across day columns; stack vertically when overlapping.
- Click → `WorkOrderDetailScreen`.
- Empty week: short empty state (no fake cards).

### Board card (compact)

- Title (truncate).
- Type accent only (color / thin strip or chip) — no status, assignee, equipment, hours on the board.

### Detail

- Show and PATCH `durationHours` («Длительность, ч»).
- Keep existing fields and status actions.

## Components / modules

| Unit | Responsibility |
|------|----------------|
| `WorkOrderDuration` (common) | `spanDays`, working-day date set, week clip |
| `BoardScreen` | Week nav, 7-col grid, load board, open detail |
| `WorkOrderBoardCard` | Compact bar (title + type accent) |
| `WorkOrderDetailScreen` | Duration field + existing detail |
| `WorkOrdersRepository` / DTOs | `durationHours` wire |
| `dashboard-service` store/routes | Persist + validate duration |

## Out of scope

- Drag-resize duration on the board.
- Server-side board day layout payload.
- Changing PPR scheduler horizon logic beyond defaulting `durationHours = 8`.

## Test plan

- Unit: `spanDays` (1→1, 8→1, 9→2, 16→2, 17→3); Friday + 3 workdays → Fri, Mon, Tue; week clip across Sunday.
- API: create/patch with `durationHours`; reject `< 1`; default 8 when omitted.
- UI/smoke: week has 7 day headers; card shows title only; open detail shows hours; week arrows change range.
