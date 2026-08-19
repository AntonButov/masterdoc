# Warehouse ЗИП (maintenance-attached inventory)

**Date:** 2026-08-19  
**Status:** approved (ready for plan)  
**Related:** [FSM_PROCESS.md](../../../masterdoc-toir/FSM_PROCESS.md), [B2B_MVP_SCOPE.md](../../../B2B_MVP_SCOPE.md), [black-box feature design](./2026-07-26-black-box-feature-design.md)

## Problem

Spare parts (ЗИП) live outside Fixaverse: no on-hand qty, no asset↔part links, no issue against work orders, no storekeeper role. Replenishment advice is tribal knowledge.

## Goal

Ship **maintenance-attached inventory** (not WMS / 1C):

1. Nomenclature + on-hand stock per org (site-scoped balances).
2. Many-to-many asset ↔ part links (storekeeper read-only).
3. Storekeeper receipt overlay (part + qty → stock ↑).
4. Engineer issues parts from WO detail (stock ↓, tied to work order + asset).
5. Weekly text replenishment advice on Warehouse **«Рекомендации»** tab (not ИЭ manuals, not separate nav feature).
6. Product feature `warehouse` + product role `storekeeper` (admin assigns).

## Locked decisions

| Topic | Decision |
|-------|----------|
| Approach | New `warehouse-service`; WO spine for issues |
| «IE» in original brief | Tab **Рекомендации** inside **Склад** |
| MVP scope | Phase A + light C-lite (weekly text advice) |
| Edit asset↔part | `equipment` / `admin` only |
| Issue from WO | `engineer` (or `board`/`tickets` holders with WO access) via gateway; not storekeeper-only |
| Receipt | `warehouse` feature |
| Negative stock | Blocked in MVP |
| Technologist / ИЭ | Must **not** auto-create SKUs |

## Non-goals (defer)

- PartRequest queue, WO status «Ждёт запчасть», soft/hard reserve
- Multi-bin WMS, lots/serials, 1C sync, auto PO
- Auto ABC/XYZ / auto-write min/max
- Creating parts from Technologist drafts

## Product model

| Wire id | Title RU | Nav |
|---------|----------|-----|
| `warehouse` | Склад | Primary nav item |

| Role id | Title RU | Default features |
|---------|----------|------------------|
| `storekeeper` | Кладовщик | `warehouse` |

Admin keeps invite / set-features; can grant `warehouse` to any role.

## Domain

| Entity | Fields (min) | Writer |
|--------|--------------|--------|
| **Part** | id, orgId, name, uom, sku?, vendorCode?, unitCost? | warehouse create; equipment may create when linking |
| **StockBalance** | orgId, siteId, partId, onHand | via movements only |
| **AssetPart** | orgId, assetId, partId, qtyHint?, critical | equipment/admin |
| **StockMovement** | id, orgId, siteId, partId, type `receipt`\|`issue`\|`adjust`, qty (>0), workOrderId?, assetId?, actorUserId, createdAt | receipt: warehouse; issue: WO path |
| **ReplenishAdvice** | id, orgId, windowStart, windowEnd, textRu, partIds[], computedAt | weekly internal job |

UI shows **names** (part name, asset name, site name) — never raw ids ([ui-names-not-ids](../../../../.cursor/rules/ui-names-not-ids.mdc)).

## API (warehouse-service, port **8104**)

Org scope via gateway headers (`X-Org-Id`, `X-User-Id`) as other services.

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | ok |
| GET | `/parts` | list parts (+ optional onHand aggregate) |
| POST | `/parts` | create part |
| GET | `/parts/{id}` | detail + balances + asset links |
| GET | `/stock?siteId=` | balances |
| POST | `/stock/receipt` | `{ partId, siteId, qty }` → movement + onHand↑ |
| POST | `/stock/issue` | `{ partId, siteId, qty, workOrderId, assetId }` → onHand↓ or 400 if insufficient |
| GET/PUT/DELETE | `/assets/{assetId}/parts` | links; PUT replaces set or POST one link |
| GET | `/advice/latest` | latest ReplenishAdvice for org |
| POST | `/internal/advice/tick` | compute weekly advice (internal token) |

Gateway: `proxyPrefix("/warehouse", …, features = …)` with split read/write where needed:

- Read stock/parts/advice: `warehouse` **or** `equipment` **or** `engineer` **or** `board` **or** `tickets` (WO materials suggest)
- Write receipt / part create (storekeeper): `warehouse`
- Write asset↔part: `equipment` **or** `admin`
- Write issue: `engineer` **or** `board` **or** `tickets` **or** `warehouse`

## UX

### Screen «Склад» (`warehouse`)

Tabs: **Остатки** | **Рекомендации**

**Остатки:** table part name, onHand, uom, linked equipment names (read-only for storekeeper). Button **Приход** → overlay: pick part, qty, site → OK.

**Рекомендации:** latest advice text (empty: «Пока нет рекомендаций»). Read-only.

Equipment screen (or asset card): manage compatible parts when `equipment` granted.

### Work order detail

Section **Запчасти** (after Оборудование): compatible parts for WO asset + onHand; **Взять** with qty → issue. Hide section if caller lacks warehouse read grants.

## Weekly advice algorithm (MVP)

Inputs: `issue` movements in last 7 days, current onHand, optional `unitCost`, `critical` links.

For each part with usage:

- `suggestedQty = max(0, usageQty − onHand)`
- Rank by `suggestedQty * (unitCost ?: 1)` descending; critical parts first within ties
- Render Russian paragraph listing part **names** and suggested qty / bulk hint — store as `textRu` only (no auto receipt)

## Services / repos

| Repo | Change |
|------|--------|
| `warehouse-service` | **New** (Ktor + Postgres + Flyway; clone black-box/maintenance pattern) |
| `feature-service` | catalog + `storekeeper` default |
| `api-gateway-service` | ProductFeatures, GatewayConfig URL, proxy routes, deploy env |
| `client-app` | FeatureId, nav, WarehouseScreen, WO materials, DI |
| `masterdoc-zitadel` | role key `warehouse` |
| `masterdoc` | this spec; B2B/STRATEGY backlog → phased track |

## Success criteria

- Admin can assign role Кладовщик / feature Склад
- Storekeeper sees stock + equipment names, receives stock via overlay, sees advice tab
- Engineer issues part on WO → onHand decreases by qty
- Weekly tick produces non-empty advice when there was usage
- Smoke on Fixaverse Smoke after deploy

## Out of scope follow-ups

FSM PartRequest queue, «Ждёт запчасть», min/max alerts, returns, multi-storeroom transfer.
