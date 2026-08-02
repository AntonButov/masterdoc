# Catalog + Dashboard Postgres persistence — Design

date: 2026-08-02  
repos: `catalog-service`, `dashboard-service`  
reference: `maintenance-service` (Hikari + Flyway + JDBC + named `*-postgres` compose)

## Goal

Redeploy / restart of catalog or dashboard must **not** wipe sites, assets, user-scopes, or work orders. Today all four live in process `ConcurrentHashMap` RAM.

## Product decisions (session-settled)

| Decision | Choice |
| --- | --- |
| Scope v1 | **Catalog + dashboard work orders** (not black-box / documents) |
| Topology | **Approach A** — separate Postgres per service (`catalog-postgres`, `dashboard-postgres`) |
| Stack | HikariCP + Flyway + raw JDBC (same as maintenance-service) |
| Shared Postgres cluster | Out of scope v1 |
| QR / `qrToken` | Out of scope (separate equipment-qr design) |
| Live RAM migration | No automatic export; post-deploy **seed Demo assets** via existing ops workflow |

## Problem

- `catalog-service`: `SiteStore`, `AssetStore`, `ScopeStore` → `ConcurrentHashMap` in `Application.kt`.
- `dashboard-service`: `WorkOrderStore` → `ConcurrentHashMap`.
- Any compose recreate / image deploy empties Demo/Smoke operational data; sites partially recover via «Цех 1» empty-list seed; assets and WOs do not.

## Architecture

```
catalog-service ──DATABASE_URL──► catalog-postgres (volume catalog_pg)
dashboard-service ──DATABASE_URL──► dashboard-postgres (volume dashboard_pg)
```

Compose service names must **not** be bare `postgres` (shared API VPS: directory `deploy` → colliding `deploy-postgres-1`).

### catalog-service

1. **Deps** — postgresql driver, HikariCP, Flyway (mirror maintenance `build.gradle.kts`).
2. **`Db.connect()`** — migrate on boot from `DATABASE_URL` (default local JDBC for tests/dev).
3. **Flyway `V1__catalog.sql`**
   - `sites` — id, org_id, name, address, lat, lon, geofence_radius_m; PK `(org_id, id)` or unique `(org_id, id)`.
   - `assets` — id, org_id, site_id, name, inventory_no, category, description, status, source, document_ids JSONB; PK id (globally unique UUID today) + index `(org_id)`.
   - `user_scopes` — org_id, user_id, site_ids JSONB, asset_ids JSONB; PK `(org_id, user_id)`.
4. **Repos** — JDBC implementations keeping current store method surface (`list`/`get`/`create`/`update`/`move`/`confirm`/`reject`/`delete`, scope `get`/`put`/`covers`/`candidates`/`filterAllowed`).
5. **HTTP** — unchanged routes and DTOs.
6. **Empty sites seed** — keep «Цех 1» / `ceh-1` behavior when org has zero sites.
7. **Compose** — add `catalog-postgres` + volume `catalog_pg`; wire `DATABASE_URL`; `depends_on` healthy; CI secrets `POSTGRES_DB` / `USER` / `PASSWORD` (or single URL) like maintenance.
8. **Tests** — route tests against real Postgres (Testcontainers) or CI service container; do not rely on wiped in-memory maps.

### dashboard-service

1. Same Db/Flyway/Hikari pattern.
2. **Flyway `V1__work_orders.sql`** — columns for full `WorkOrder` model: id, org_id, type, status, title, asset_id, site_id, due_at, duration_hours, assignee_id, maintenance_map_id, maintenance_map_item_id, created_by, description, source, created_at, updated_at, started_at, closed_at. Indexes: `(org_id)`, `(org_id, assignee_id)`, `(org_id, created_by)`, `(org_id, status)`.
3. **`WorkOrderStore`** — persist create/update/list/get/delete-by-org; scheduler and board queries read from DB.
4. **HTTP** — unchanged public contract.
5. **Compose** — `dashboard-postgres` + `dashboard_pg`; `DATABASE_URL`; CI secrets.
6. **Tests** — WorkOrder route tests on Postgres.

### Deploy / ops

- First production deploy after this change: empty DBs; run **Seed Demo Assets** (and Smoke if needed) once; thereafter deploys keep data.
- Document in catalog/dashboard README: persistence + secret names.
- Healthcheck still HTTP; postgres health via compose `pg_isready`.

## Out of scope v1

- Persisting black-box audit log, document-service meta maps
- Cross-service shared Postgres / PgBouncer
- Automatic one-shot dump of current RAM into SQL before cutover
- Equipment QR token column (follow-up when QR ships)

## Success criteria

1. Create site + asset + WO → redeploy both services → entities still present for same `X-Org-Id`.
2. User-scope bindings survive catalog restart.
3. Existing API behavior (status transitions, tickets-only filters, scope filter) unchanged.
4. CI green with Postgres in compose/test path.
5. Demo four-asset seed survives a subsequent catalog image deploy.

## Test plan (summary)

- Catalog: CRUD site/asset/scope; confirm/reject; scope filter list; restart process / new connection still sees rows.
- Dashboard: create/list/patch WO; clear-org; board week query; restart retains WOs.
- Deploy smoke (Fixaverse Smoke or Demo): seed → deploy → count assets / list WOs > 0.
