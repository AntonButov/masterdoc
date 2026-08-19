# Warehouse ЗИП Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship maintenance-attached inventory: `warehouse-service`, feature `warehouse`, role `storekeeper`, client «Склад» (остатки + рекомендации), receipt overlay, WO issue, weekly text advice.

**Architecture:** New Ktor+Postgres service owns parts/stock/movements/links/advice. Gateway proxies `/warehouse/*` with feature gates. feature-service adds catalog + role. client-app adds nav/screen + WO materials block. Weekly tick computes Russian text advice only.

**Tech Stack:** Kotlin 21, Ktor 3, Flyway, Postgres, Spring (feature-service), Compose Multiplatform client, GitHub Actions deploy.

**Spec:** [docs/superpowers/specs/2026-08-19-warehouse-zip-design.md](../specs/2026-08-19-warehouse-zip-design.md)

## Global Constraints

- Wire id exactly `warehouse`; RU title exactly `Склад`.
- Role id exactly `storekeeper`; RU title exactly `Кладовщик`.
- UI: names only — never show raw UUIDs / user ids ([ui-names-not-ids](../../../../.cursor/rules/ui-names-not-ids.mdc)).
- Port **8104**; Compose service name **not** bare `postgres` (use `warehouse-postgres`).
- Negative onHand on issue → **400**; no override in MVP.
- Technologist must not create SKUs.
- Multi-repo: commit/push each affected repo when its tasks finish; watch Actions.
- Do not run heavy local client wasm builds; targeted JVM tests OK. CI builds on push.
- Smoke tenant: Fixaverse Smoke after green deploy.

## File map

| Area | Files |
|------|--------|
| warehouse-service (new) | Application, Db, stores, Flyway V1, Dockerfile, deploy/compose, CI, README, tests |
| feature-service | FeatureCatalog, ProductRoleService, tests |
| api-gateway-service | ProductFeatures, GatewayConfig, WarehouseRoutes, CI env |
| masterdoc-zitadel | terraform roles + expected.yaml |
| client-app | FeatureId, NavCatalog, FeatureLabels, WarehouseScreen, WO materials, API client |
| masterdoc | design (done), this plan |

---

### Task 1: Scaffold warehouse-service + schema + health

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/warehouse-service` (create; `git init` + remote later or mirror black-box-service layout)

**Files:**
- Create: `build.gradle.kts`, `settings.gradle.kts`, `gradlew*`, `Dockerfile`, `deploy/docker-compose.yml`, `.github/workflows/ci.yml`, `README.md`
- Create: `src/main/kotlin/pro/masterdoc/warehouse/{Application.kt,Db.kt}`
- Create: `src/main/resources/db/migration/V1__warehouse.sql`
- Test: `src/test/kotlin/.../HealthAndMigrateTest.kt`

**Schema (V1):**

```sql
CREATE TABLE parts (
  id TEXT PRIMARY KEY,
  org_id TEXT NOT NULL,
  name TEXT NOT NULL,
  uom TEXT NOT NULL DEFAULT 'шт',
  sku TEXT,
  vendor_code TEXT,
  unit_cost DOUBLE PRECISION,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX parts_org_idx ON parts(org_id);

CREATE TABLE stock_balances (
  org_id TEXT NOT NULL,
  site_id TEXT NOT NULL,
  part_id TEXT NOT NULL REFERENCES parts(id),
  on_hand DOUBLE PRECISION NOT NULL DEFAULT 0,
  PRIMARY KEY (org_id, site_id, part_id)
);

CREATE TABLE asset_parts (
  org_id TEXT NOT NULL,
  asset_id TEXT NOT NULL,
  part_id TEXT NOT NULL REFERENCES parts(id),
  qty_hint DOUBLE PRECISION,
  critical BOOLEAN NOT NULL DEFAULT FALSE,
  PRIMARY KEY (org_id, asset_id, part_id)
);

CREATE TABLE stock_movements (
  id TEXT PRIMARY KEY,
  org_id TEXT NOT NULL,
  site_id TEXT NOT NULL,
  part_id TEXT NOT NULL REFERENCES parts(id),
  type TEXT NOT NULL,
  qty DOUBLE PRECISION NOT NULL,
  work_order_id TEXT,
  asset_id TEXT,
  actor_user_id TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX stock_movements_org_created_idx ON stock_movements(org_id, created_at DESC);

CREATE TABLE replenish_advice (
  id TEXT PRIMARY KEY,
  org_id TEXT NOT NULL,
  window_start TIMESTAMPTZ NOT NULL,
  window_end TIMESTAMPTZ NOT NULL,
  text_ru TEXT NOT NULL,
  part_ids TEXT[] NOT NULL DEFAULT '{}',
  computed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX replenish_advice_org_computed_idx ON replenish_advice(org_id, computed_at DESC);
```

- [ ] **Step 1:** Copy gradle wrapper + build pattern from `black-box-service` (port 8104, mainClass `pro.masterdoc.warehouse.ApplicationKt`).
- [ ] **Step 2:** Implement `Db.connect()` + Flyway migrate; `GET /health`.
- [ ] **Step 3:** Testcontainers migrate + health test; `./gradlew test`.
- [ ] **Step 4:** Dockerfile + compose (`warehouse-postgres`, port 8104) + CI deploy workflow mirroring black-box-service.
- [ ] **Step 5:** Commit in warehouse-service (message: `feat: scaffold warehouse-service with Flyway schema`).

---

### Task 2: warehouse-service domain API + advice tick

**Repos:** warehouse-service

**Endpoints** (headers `X-Org-Id` required; `X-User-Id` for writes):

- `GET/POST /parts`, `GET /parts/{id}`
- `GET /stock?siteId=`
- `POST /stock/receipt` body `{partId,siteId,qty}`
- `POST /stock/issue` body `{partId,siteId,qty,workOrderId,assetId}` — 400 if onHand < qty
- `GET /assets/{assetId}/parts`, `PUT /assets/{assetId}/parts` body `{items:[{partId,qtyHint?,critical?}]}`
- `GET /advice/latest`
- `POST /internal/advice/tick` (optional `WAREHOUSE_INTERNAL_TOKEN` / `X-Internal-Token`) — compute last 7d issues → `textRu`

Advice text example shape (RU, names only):  
`За неделю списано: Подшипник 6308 ×3 (остаток 1). Рекомендуется докупить оптом: Подшипник 6308 — 2 шт.`

- [ ] **Step 1:** Failing route tests (receipt then issue; issue insufficient; asset links; advice tick).
- [ ] **Step 2:** Implement JDBC stores + routes.
- [ ] **Step 3:** `./gradlew test` green.
- [ ] **Step 4:** Commit `feat: warehouse parts stock issue receipt and weekly advice`.

---

### Task 3: feature-service + zitadel

**Repos:** feature-service, masterdoc-zitadel

- [ ] Add `FeatureDefinition("warehouse", "Склад")` to FeatureCatalog (sorted ENTRIES).
- [ ] Add role `storekeeper` / `Кладовщик` / `listOf("warehouse")` to ProductRoleService DEFAULTS + ROLE_IDS.
- [ ] Update FeatureCatalogTest / RolesControllerTest expectations (catalog size + ids).
- [ ] zitadel: add `warehouse` to `locals.roles` and expected.yaml.
- [ ] Commit/push each repo.

---

### Task 4: api-gateway warehouse proxy

**Repos:** api-gateway-service

- [ ] `ProductFeatures` + `GatewayConfig.warehouseServiceBaseUrl` (`WAREHOUSE_SERVICE_BASE_URL`, default `http://127.0.0.1:8104`).
- [ ] Install routes: proxy `/warehouse` with feature rules from spec (read vs write vs issue vs asset-parts). Prefer dedicated route handlers if proxyPrefix cannot split write features cleanly — match existing patterns.
- [ ] CI bootstrap env `WAREHOUSE_SERVICE_BASE_URL=http://host.docker.internal:8104`.
- [ ] Unit/route tests if gateway has them for similar proxies.
- [ ] Commit/push.

---

### Task 5: client-app Warehouse screen + WO materials

**Repos:** client-app

- [ ] `FeatureId.Warehouse("warehouse")`, `NavDestinationId.Warehouse`, NavCatalog, FeatureLabels, MainShellContent.
- [ ] `WarehouseRepository` / API client for gateway `/warehouse/*`.
- [ ] `WarehouseScreen`: tabs Остатки | Рекомендации; Приход overlay (part, qty, site).
- [ ] `WorkOrderDetailScreen`: section Запчасти after Оборудование — list compatible + onHand + Взять.
- [ ] Resolve asset/part/site **names** for display (catalog lookups already used on WO detail).
- [ ] Tests for FeatureLabels / nav inclusion.
- [ ] Commit/push (CI builds client).

---

### Task 6: Wire deploy + smoke readiness

- [ ] Ensure warehouse-service pushed with deploy secrets pattern; gateway env upsert for VPS.
- [ ] Grant `warehouse` on Smoke org storekeeper or admin for smoke (document in README if manual).
- [ ] After green Actions: run smoke-test skill against Склад + WO issue path.
- [ ] Final branch review.

## Progress

| Task | Status |
|------|--------|
| 1 Scaffold | pending |
| 2 Domain API | pending |
| 3 feature + zitadel | pending |
| 4 gateway | pending |
| 5 client | pending |
| 6 deploy + smoke | pending |
