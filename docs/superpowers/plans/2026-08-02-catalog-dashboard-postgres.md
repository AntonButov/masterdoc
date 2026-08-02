# Catalog + Dashboard Postgres Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Persist catalog (sites, assets, user-scopes) and dashboard work orders in separate Postgres instances so redeploys no longer wipe operational data.

**Architecture:** Clone `maintenance-service` pattern per repo: HikariCP + Flyway + JDBC repositories; compose services named `catalog-postgres` / `dashboard-postgres` (never bare `postgres`); Testcontainers for route tests; CI runtime `.env` gains `POSTGRES_*` + `DATABASE_URL`.

**Tech Stack:** Kotlin 2.1 / Ktor 3.0.3, PostgreSQL 16, HikariCP 5.1, Flyway 10.21, Testcontainers 1.20.4, GitHub Actions deploy.

**Spec:** `masterdoc/docs/superpowers/specs/2026-08-02-catalog-dashboard-postgres-design.md`

## Global Constraints

- Compose DB service names: `catalog-postgres`, `dashboard-postgres` only (VPS collision rule).
- HTTP APIs and DTO shapes unchanged.
- QR / `qrToken` out of scope.
- No automatic RAM→SQL dump; after first catalog deploy run Seed Demo Assets once.
- Prefer Testcontainers (`disabledWithoutDocker = true`) like maintenance.
- Push → CI → deploy per repo; do not run heavy local Gradle full suites unless a single targeted test.
- Multi-repo: commit/push in the repo each task touches; watch Actions.
- UI names rule N/A (backend-only); never leak secrets into commits.

## File map

| File | Role |
| --- | --- |
| `catalog-service/build.gradle.kts` | PG + Hikari + Flyway + Testcontainers deps |
| `catalog-service/.../Db.kt` | connect + migrate |
| `catalog-service/.../db/migration/V1__catalog.sql` | sites, assets, user_scopes |
| `catalog-service/.../JdbcSiteStore.kt` etc. | JDBC stores |
| `catalog-service/.../Application.kt` | wire DataSource into module |
| `catalog-service/deploy/docker-compose.yml` | catalog-postgres + DATABASE_URL |
| `catalog-service/.github/workflows/ci.yml` | POSTGRES_* in runtime.env |
| `catalog-service/src/test/...` | Testcontainers + existing route tests |
| `dashboard-service/*` | same pattern for work_orders |

**Ship order:** finish catalog (Tasks 1–5) and green deploy + seed Demo, then dashboard (Tasks 6–10).

---

### Task 1: catalog — deps, Db, Flyway schema

**Repo:** `catalog-service`

**Files:**
- Modify: `build.gradle.kts`
- Create: `src/main/kotlin/pro/masterdoc/catalog/Db.kt`
- Create: `src/main/resources/db/migration/V1__catalog.sql`
- Test: `src/test/kotlin/pro/masterdoc/catalog/DbMigrateTest.kt`

**Interfaces:**
- Produces: `Db.connect(jdbcUrl, user, pass): HikariDataSource` (Flyway migrate); tables below

- [ ] **Step 1: Add failing migrate smoke test**

```kotlin
@Testcontainers(disabledWithoutDocker = true)
class DbMigrateTest {
    companion object {
        @Container @JvmStatic
        val postgres = PostgreSQLContainer("postgres:16-alpine")
            .withDatabaseName("catalog")
            .withUsername("catalog")
            .withPassword("catalog")
    }

    @Test
    fun migratesCleanSchema() {
        Db.connect(postgres.jdbcUrl, postgres.username, postgres.password).use { ds ->
            ds.connection.use { c ->
                c.createStatement().use { st ->
                    st.executeQuery(
                        "SELECT tablename FROM pg_tables WHERE schemaname='public' ORDER BY 1",
                    ).use { rs ->
                        val names = buildList { while (rs.next()) add(rs.getString(1)) }
                        assertTrue("assets" in names)
                        assertTrue("sites" in names)
                        assertTrue("user_scopes" in names)
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 2: Run test — expect FAIL** (Db missing)

Run: `./gradlew test --tests pro.masterdoc.catalog.DbMigrateTest`  
Expected: compile/fail missing `Db`

- [ ] **Step 3: Add deps + Db + V1 SQL**

`build.gradle.kts` — copy postgres/Hikari/Flyway/Testcontainers lines from `maintenance-service/build.gradle.kts`.

`Db.kt` — copy from `maintenance-service/.../Db.kt`, change default DB name/user to `catalog`.

`V1__catalog.sql`:

```sql
CREATE TABLE sites (
    id TEXT NOT NULL,
    org_id TEXT NOT NULL,
    name TEXT NOT NULL,
    address TEXT,
    lat DOUBLE PRECISION,
    lon DOUBLE PRECISION,
    geofence_radius_m INTEGER,
    PRIMARY KEY (org_id, id)
);

CREATE TABLE assets (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL,
    site_id TEXT NOT NULL,
    name TEXT NOT NULL,
    inventory_no TEXT,
    category TEXT,
    description TEXT,
    status TEXT NOT NULL,
    source TEXT NOT NULL,
    document_ids JSONB NOT NULL DEFAULT '[]'::jsonb
);

CREATE INDEX assets_org_id_idx ON assets (org_id);
CREATE INDEX assets_org_site_idx ON assets (org_id, site_id);

CREATE TABLE user_scopes (
    org_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    site_ids JSONB NOT NULL DEFAULT '[]'::jsonb,
    asset_ids JSONB NOT NULL DEFAULT '[]'::jsonb,
    PRIMARY KEY (org_id, user_id)
);
```

- [ ] **Step 4: Run test — expect PASS**

- [ ] **Step 5: Commit and push; watch CI** (deploy may still be in-memory app — OK)

```bash
git add build.gradle.kts src/main/kotlin/pro/masterdoc/catalog/Db.kt \
  src/main/resources/db/migration/V1__catalog.sql \
  src/test/kotlin/pro/masterdoc/catalog/DbMigrateTest.kt
git commit -m "feat(catalog): add Postgres Flyway schema scaffold"
git push
gh run watch
```

---

### Task 2: catalog — JDBC stores + wire module

**Repo:** `catalog-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/catalog/JdbcSiteStore.kt`
- Create: `src/main/kotlin/pro/masterdoc/catalog/JdbcAssetStore.kt`
- Create: `src/main/kotlin/pro/masterdoc/catalog/JdbcScopeStore.kt`
- Modify: `src/main/kotlin/pro/masterdoc/catalog/Application.kt` — `main`/`module` take `DataSource`; construct JDBC stores; keep empty-sites seed
- Modify: existing route tests to use Testcontainers + `Db.connect` like maintenance `withApplication`

**Interfaces:**
- Consumes: `DataSource` from Task 1
- Produces: store classes with **same public methods** as current `SiteStore` / `AssetStore` / `ScopeStore` (move logic from in-memory classes into JDBC; delete ConcurrentHashMap bodies or replace classes in place)

- [ ] **Step 1: Convert one existing test (e.g. `AssetRoutesTest.createDraftAndConfirm`) to Testcontainers helper; run — FAIL or still pass on memory**

Add companion postgres + `withApplication` that calls `module(ds)` after `Db.connect`.

- [ ] **Step 2: Implement JDBC stores**

Use `kotlinx.serialization.json.Json` for JSONB list encode/decode of `documentIds` / `siteIds` / `assetIds`.  
Preserve all validation and status transitions from current store code.

- [ ] **Step 3: Wire `main()`**

```kotlin
fun main() {
    val ds = Db.connect()
    embeddedServer(Netty, port = ...) {
        module(ds)
    }.start(wait = true)
}

fun Application.module(dataSource: DataSource) {
    val sites = JdbcSiteStore(dataSource)
    val assets = JdbcAssetStore(dataSource)
    val scopes = JdbcScopeStore(dataSource)
    // existing routing unchanged
}
```

- [ ] **Step 4: Run full catalog tests**

Run: `./gradlew test`  
Expected: PASS (Docker required for Testcontainers)

- [ ] **Step 5: Commit push watch**

```bash
git commit -m "feat(catalog): persist sites, assets, scopes in Postgres"
git push && gh run watch
```

---

### Task 3: catalog — compose + CI DATABASE_URL

**Repo:** `catalog-service`

**Files:**
- Modify: `deploy/docker-compose.yml` — add `catalog-postgres` + volume `catalog_pg`; `depends_on`; `DATABASE_URL` env on app
- Modify: `.github/workflows/ci.yml` — runtime.env includes:

```
POSTGRES_DB=catalog
POSTGRES_USER=catalog
POSTGRES_PASSWORD=<from secret or default catalog for smoke; prefer secret CATALOG_POSTGRES_PASSWORD if present else catalog>
CATALOG_SERVICE_PORT=8091
```

Mirror maintenance CI pattern (defaults OK for private VPS if documented).

- [ ] **Step 1: Update compose** (copy structure from `maintenance-service/deploy/docker-compose.yml`, rename services/ports/DB)
- [ ] **Step 2: Update CI runtime.env**
- [ ] **Step 3: Commit push; watch deploy success**
- [ ] **Step 4: After deploy green — run Seed Demo Assets** for org `382715225649971203`
- [ ] **Step 5: Count Org Assets — expect TOTAL=4; redeploy note: next catalog push must keep TOTAL=4**

Verify persistence:

```bash
gh workflow run seed-demo-assets.yml -f org_id=382715225649971203
# after success:
gh workflow run count-org-assets.yml -f org_id=382715225649971203 -f org_label='Fixaverse Demo'
```

Optional: trigger empty workflow-only commit is NOT needed — instead run count, then `docker compose restart catalog-service` via a tiny ops workflow OR wait until next real push and re-count.

---

### Task 4: catalog — persistence regression test (process-level)

**Repo:** `catalog-service`

**Files:**
- Create: `src/test/kotlin/pro/masterdoc/catalog/AssetPersistenceTest.kt`

- [ ] **Step 1: Write test** — create asset on ds1, close ds1, reconnect Db.connect same container, get asset by id → found
- [ ] **Step 2: Run — PASS**
- [ ] **Step 3: Commit push**

```kotlin
@Test
fun survivesNewConnectionPool() {
    Db.connect(postgres.jdbcUrl, postgres.username, postgres.password).use { ds ->
        // insert via JdbcAssetStore + ensure site
    }
    Db.connect(postgres.jdbcUrl, postgres.username, postgres.password).use { ds ->
        val loaded = JdbcAssetStore(ds).get(orgId, assetId)
        assertEquals("Компрессор", loaded.name)
    }
}
```

---

### Task 5: catalog — README note

**Repo:** `catalog-service`

- [ ] **Step 1:** Document `DATABASE_URL`, compose service name, seed-after-first-deploy in `README.md`
- [ ] **Step 2:** Commit push (docs-only OK)

---

### Task 6: dashboard — deps, Db, Flyway schema

**Repo:** `dashboard-service`

**Files:**
- Modify: `build.gradle.kts`
- Create: `src/main/kotlin/pro/masterdoc/dashboard/Db.kt`
- Create: `src/main/resources/db/migration/V1__work_orders.sql`
- Test: `src/test/kotlin/pro/masterdoc/dashboard/DbMigrateTest.kt`

**SQL columns** (all `WorkOrder` fields as TEXT/INT/TIMESTAMPTZ as appropriate):

```sql
CREATE TABLE work_orders (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL,
    type TEXT NOT NULL,
    status TEXT NOT NULL,
    title TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    site_id TEXT NOT NULL,
    due_at TEXT NOT NULL,
    duration_hours INTEGER NOT NULL,
    assignee_id TEXT,
    maintenance_map_id TEXT,
    maintenance_map_item_id TEXT,
    created_by TEXT,
    description TEXT,
    source TEXT NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    started_at TEXT,
    closed_at TEXT
);
CREATE INDEX work_orders_org_id_idx ON work_orders (org_id);
CREATE INDEX work_orders_org_assignee_idx ON work_orders (org_id, assignee_id);
CREATE INDEX work_orders_org_created_by_idx ON work_orders (org_id, created_by);
CREATE INDEX work_orders_org_status_idx ON work_orders (org_id, status);
```

- [ ] Steps: failing migrate test → implement → PASS → commit push watch

---

### Task 7: dashboard — JdbcWorkOrderStore + wire module

**Repo:** `dashboard-service`

**Files:**
- Replace/implement JDBC behind `WorkOrderStore` API (same methods: create, get, list, update status/assignee, deleteByOrg, board helpers as today)
- Modify: `Application.kt` to pass `DataSource`
- Modify: `WorkOrderRoutesTest.kt` (+ others) → Testcontainers `withApplication`

- [ ] **Step 1:** Adapt tests to Testcontainers  
- [ ] **Step 2:** Implement JDBC store (port logic from `WorkOrderStore.kt`)  
- [ ] **Step 3:** `./gradlew test` PASS  
- [ ] **Step 4:** Commit `feat(dashboard): persist work orders in Postgres` + push + watch

---

### Task 8: dashboard — compose + CI

**Repo:** `dashboard-service`

- [ ] Add `dashboard-postgres` + volume `dashboard_pg` to `deploy/docker-compose.yml`
- [ ] CI runtime.env: `POSTGRES_DB=dashboard`, `POSTGRES_USER=dashboard`, `POSTGRES_PASSWORD=dashboard` (or secret), keep existing AI/catalog URLs
- [ ] Commit push watch deploy

---

### Task 9: dashboard — persistence regression test

- [ ] Create WO → new pool → get by id  
- [ ] Commit push

---

### Task 10: ship gate + smoke

- [ ] Confirm catalog Demo still has 4 assets after any catalog deploys from this plan  
- [ ] Create one emergency WO via API/UI in Smoke or Demo; restart dashboard (or note next deploy); WO still listed  
- [ ] Run `/smoke-test` skill on client-app against Demo/Smoke: Оборудование list non-empty; Заявки/board still load  
- [ ] Report SHA + CI URLs for both repos

---

## Self-review (plan vs spec)

| Spec requirement | Task |
| --- | --- |
| catalog-postgres + sites/assets/scopes | 1–5 |
| dashboard-postgres + work_orders | 6–9 |
| maintenance-style stack | 1, 6 |
| API unchanged | 2, 7 |
| CI/compose secrets | 3, 8 |
| Seed Demo after first catalog deploy | 3, 10 |
| Survive redeploy | 4, 9, 10 |
| No QR | — |

No TBD placeholders. Types: stores keep existing Kotlin method signatures from current classes.
