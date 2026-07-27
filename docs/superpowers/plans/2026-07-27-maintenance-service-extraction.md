# Maintenance-service extraction (ППР + MCP) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract maintenance maps (ППР) into a new `maintenance-service` repo with Postgres persistence and a sibling `maintenance-mcp`, then rewire gateway/dashboard/technologist to use it while keeping public `/maintenance-maps` unchanged.

**Architecture:** New Ktor service owns maps CRUD + confirm/reject on Postgres. Separate MCP process in the same repo proxies draft create/update to that API. `dashboard-service` keeps WO/board/PprScheduler but loads active maps and validates PPR WOs over HTTP. `technologist-mcp` loses map tools.

**Tech Stack:** Kotlin 2.1 / JVM 21, Ktor 3.0.3, kotlinx.serialization, PostgreSQL 16, Flyway, HikariCP, JDBC (no Exposed), Docker Compose, GitHub Actions deploy.

**Spec:** [docs/superpowers/specs/2026-07-27-maintenance-service-extraction-design.md](../specs/2026-07-27-maintenance-service-extraction-design.md)

## Global Constraints

- Repo/service name exactly `maintenance-service`; MCP process name `maintenance-mcp`.
- Ports: API **8098**, MCP **8099**, Postgres internal only.
- Public paths and JSON shapes stay identical to current dashboard maps API.
- MCP writes always `status=draft`, `source=ai_generated`.
- Scheduler stays in `dashboard-service`; maps never stay in dashboard memory.
- No migration of old in-memory maps.
- Do not run heavy local full multi-module builds; prefer targeted `./gradlew test`. CI builds/deploys on push.
- Multi-repo: commit + push each affected repo when its tasks finish; watch Actions for product repos.
- Deploy pattern matches `dashboard-service` / `technologist-mcp` (rsync + compose `--wait`).

## File map

| Area | Files |
|------|--------|
| New `maintenance-service/` | Gradle app, `Application.kt`, `Models.kt`, `MapRepository.kt`, `Db.kt`, Flyway SQL, Dockerfile, `deploy/docker-compose.yml` (api+mcp+postgres), MCP `mcp/Application.kt`, tests, `.github/workflows/ci.yml` |
| `api-gateway-service` | `GatewayConfig.kt`, `EquipmentRoutes.kt`, deploy env / CI runtime.env, tests if any |
| `dashboard-service` | Strip maps from `Application.kt`; `MaintenanceMapClient.kt`; adapt `PprScheduler` / `WorkOrderStore`; tests |
| `technologist-mcp` | Remove map tools from `Application.kt` + tests; drop unused `DASHBOARD_BASE_URL` if unused |
| `technologist-service` | Route map MCP tools to `MAINTENANCE_MCP_BASE_URL`; confirm → maintenance base; deploy/CI env |
| `masterdoc` | Update equipment-technologist design pointers |

---

### Task 1: Scaffold `maintenance-service` repo (API + Postgres compose)

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/maintenance-service` (create)

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradlew*`, `gradle/`, `Dockerfile`, `README.md`
- Create: `deploy/docker-compose.yml`, `src/main/resources/db/migration/V1__maps.sql`
- Create: `src/main/kotlin/pro/masterdoc/maintenance/Db.kt`
- Create: `src/main/kotlin/pro/masterdoc/maintenance/Application.kt` (health only)
- Create: `src/test/kotlin/pro/masterdoc/maintenance/HealthTest.kt`
- Create: `.github/workflows/ci.yml` (test + deploy stub matching dashboard pattern; ports 8098/8099)
- Create GitHub repo `Fixaverse/maintenance-service` (or org used by siblings — check `gh repo view` on `dashboard-service` for exact org) and push `main`

**Interfaces:**
- Produces: `DATABASE_URL` / `POSTGRES_*` wiring; Flyway migrate on startup; `GET /health` → `{"status":"ok"}`
- Produces: compose services `postgres`, `maintenance-service` (MCP added in Task 4)

- [ ] **Step 1: Copy Gradle wrapper from `dashboard-service` and create build files**

`settings.gradle.kts`:
```kotlin
rootProject.name = "maintenance-service"
```

`build.gradle.kts` dependencies must include Ktor server stack (same versions as dashboard), plus:
```kotlin
implementation("org.postgresql:postgresql:42.7.4")
implementation("com.zaxxer:HikariCP:5.1.0")
implementation("org.flywaydb:flyway-core:10.21.0")
implementation("org.flywaydb:flyway-database-postgresql:10.21.0")
implementation("io.ktor:ktor-client-cio-jvm:$ktor")
implementation("io.ktor:ktor-client-content-negotiation-jvm:$ktor")
testImplementation("org.testcontainers:postgresql:1.20.4")
testImplementation("org.testcontainers:junit-jupiter:1.20.4")
```
`mainClass`: `pro.masterdoc.maintenance.ApplicationKt`

- [ ] **Step 2: Add Flyway V1 and Db bootstrap**

`V1__maps.sql` — exact tables from spec (`maintenance_maps`, `maintenance_map_items` + indexes).

`Db.kt` — HikariDataSource from `DATABASE_URL` (default `jdbc:postgresql://localhost:5432/maintenance?user=maintenance&password=maintenance`), run Flyway, expose `DataSource`.

- [ ] **Step 3: Minimal `Application.kt` + health test with Testcontainers**

Health test starts Postgres container, migrates, `testApplication { module(ds) }`, `GET /health` → 200.

- [ ] **Step 4: Dockerfile + compose (postgres + api)**

Compose: postgres 16 alpine with volume `maintenance_pg`, healthcheck; API depends_on healthy postgres; env `DATABASE_URL=jdbc:postgresql://postgres:5432/maintenance?user=maintenance&password=maintenance`, `PORT=8098`, `CATALOG_BASE_URL`, `DASHBOARD_BASE_URL` (for confirm tick later).

- [ ] **Step 5: CI workflow + create remote + push**

Mirror `dashboard-service/.github/workflows/ci.yml` with `REMOTE_PATH=/opt/maintenance-service`, image `masterdoc-maintenance-service`, health `8098`. Deploy compose must start postgres+api. Document required secrets (same API VPS trio).

```bash
cd /Users/antonbutov/Documents/MYPROJECTS/Fixaverse/maintenance-service
./gradlew test --no-daemon
# create repo under same org as dashboard-service, push main, gh run watch
```

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "chore: scaffold maintenance-service with Postgres and health"
git push -u origin main
```

---

### Task 2: Maps repository + public `/maintenance-maps` API

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/maintenance-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/maintenance/Models.kt`
- Create: `src/main/kotlin/pro/masterdoc/maintenance/MapRepository.kt`
- Create: `src/main/kotlin/pro/masterdoc/maintenance/AssetChecker.kt`
- Modify: `Application.kt` — routes
- Create: `src/test/kotlin/pro/masterdoc/maintenance/MaintenanceMapRoutesTest.kt` (port tests from `dashboard-service/.../MaintenanceMapRoutesTest.kt`)

**Interfaces:**
- Consumes: `DataSource` from Task 1
- Produces: same DTOs as dashboard `MaintenanceMap*` (package `pro.masterdoc.maintenance`)
- Produces: `MapRepository.create/list/get/update/confirm/reject/listActive`
- Produces: routes matching dashboard (Created on POST, NoContent on reject, BadRequest validation)

- [ ] **Step 1: Copy failing route tests from dashboard `MaintenanceMapRoutesTest`**

Adapt package + `application { module(dataSource = ..., assetChecker = AllowAll) }`. Keep cases: createUpdateConfirm, emptyItemsRejected, intervalEveryLessThanOneRejected, listFiltersByOrgAndAssetId, rejectDraftSucceeds, unknown asset rejected (add with denying checker).

- [ ] **Step 2: Run tests — expect FAIL**

```bash
./gradlew test --tests 'pro.masterdoc.maintenance.MaintenanceMapRoutesTest' --no-daemon
```

- [ ] **Step 3: Implement Models + JDBC MapRepository + routes**

Validation rules identical to current `MaintenanceMapStore`. Persist items in same transaction as map. `confirm` sets `status=active`, `activated_at=now()`. `reject` deletes draft map (cascade items). `list(orgId, assetId?, status?)`.

Asset check: HTTP GET catalog `/assets/{id}` with `X-Org-Id` (copy pattern from dashboard `CatalogAssetLookup` / checker) — injectable for tests.

- [ ] **Step 4: Run tests — expect PASS; commit + push; watch CI**

```bash
git commit -m "feat: persist maintenance maps in Postgres"
git push && gh run watch
```

---

### Task 3: Confirm → dashboard scheduler tick + internal active maps

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/maintenance-service`

**Files:**
- Modify: `Application.kt` — after confirm, POST dashboard tick; add `GET /internal/active-maps`
- Create: `src/main/kotlin/pro/masterdoc/maintenance/DashboardTickClient.kt`
- Test: confirm triggers tick client (mock/fake); active-maps filters

**Interfaces:**
- Produces: `GET /internal/active-maps?orgId=&mapId=` → `{ "items": [MaintenanceMap...] }` (only `status=active`)
- Produces: on confirm success, `POST {DASHBOARD_BASE_URL}/internal/scheduler/tick?orgId=&mapId=` (fire-and-log errors; still return confirmed map)

- [ ] **Step 1: Failing tests for active-maps filter and tick invocation on confirm**

- [ ] **Step 2: Implement client + routes**

- [ ] **Step 3: Tests PASS; commit + push**

```bash
git commit -m "feat: active-maps feed and confirm triggers dashboard tick"
```

---

### Task 4: `maintenance-mcp` process in same repo

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/maintenance-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/maintenance/mcp/McpApplication.kt` (or separate source set — prefer **same module**, second main via `application { mainClass }` override OR second Gradle application plugin entry: simplest = second `main` in `mcp/McpMain.kt` and Dockerfile `MCP` target / second image build)
- Prefer **two Docker images from one repo**: `Dockerfile` (API) + `Dockerfile.mcp` (MCP), both `./gradlew installDist` with different `MAIN_CLASS` build-arg
- Create: tests for map tools (port from `technologist-mcp/.../McpToolsTest.kt` map cases)
- Modify: `deploy/docker-compose.yml` — add `maintenance-mcp` service port 8099, `MAINTENANCE_BASE_URL=http://maintenance-service:8098`
- Modify: CI to build/push both images or one image with two commands — **recommended:** one installDist, two Dockerfiles setting `CMD` to `maintenance-service` vs `maintenance-mcp` scripts if dual application names; simplest workable approach: single fat install with two main classes selected by env `APP=api|mcp` in one entrypoint script

**Recommended packaging (lock this):**
- One Gradle project, two mains: `pro.masterdoc.maintenance.ApplicationKt` and `pro.masterdoc.maintenance.mcp.McpApplicationKt`
- `deploy/docker-compose.yml` builds **two** images (`Dockerfile` ARG `MAIN_CLASS`) or uses one image with `command:` override if using a shell entrypoint — use **two Dockerfiles** copying the same installDist and different `CMD` paths: generate two start scripts via `application` plugin `applicationDefaultJvmArgs` — easiest path used in plan:

Create `scripts/docker-entrypoint.sh`:
```bash
case "${APP_ROLE:-api}" in
  mcp) exec /app/bin/maintenance-service-mcp ;;
  *) exec /app/bin/maintenance-service ;;
esac
```

Use Gradle application + second distribution OR manually add second start script in Dockerfile. Concrete approach for implementer: add subproject `mcp` only if single-module dual-main is painful; **default: single module, `tasks.register` second JavaExec / installDist named `maintenance-mcp`**.

**Interfaces:**
- Produces: tools `create_draft_maintenance_map`, `update_draft_maintenance_map` only
- Produces: `GET /mcp/tools`, `POST /mcp/tools/call`, `GET /health`

- [ ] **Step 1: Port map-tool tests from technologist-mcp; expect FAIL**

- [ ] **Step 2: Implement MCP (copy structure from technologist-mcp, strip non-map tools; upstream `MAINTENANCE_BASE_URL`)**

- [ ] **Step 3: Wire compose + CI deploy both containers; commit + push; watch deploy**

```bash
git commit -m "feat: add maintenance-mcp for draft PPR tools"
```

---

### Task 5: Gateway → `MAINTENANCE_SERVICE_BASE_URL`

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/GatewayConfig.kt` — add `maintenanceServiceBaseUrl` (env `MAINTENANCE_SERVICE_BASE_URL`, default `http://127.0.0.1:8098`)
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt` — `/maintenance-maps` uses maintenance URL
- Modify: `deploy/.env.example`, `.github/workflows/ci.yml` runtime.env
- Test: update any proxy test that assumed dashboard for maps (add/adjust)

- [ ] **Step 1: Failing/adjust test that maps proxy target is maintenance config field**

- [ ] **Step 2: Implement config + route switch**

- [ ] **Step 3: `./gradlew test`; commit; push; `gh run watch`**

```bash
git commit -m "feat: proxy maintenance-maps to maintenance-service"
```

**Deploy note:** Do this **after** Task 4 is live on VPS (maintenance healthy). If maintenance not deployed yet, hold the push or keep default until Task 4 deploy succeeded.

---

### Task 6: Dashboard — remove local maps; HTTP client for scheduler + WO validation

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/dashboard-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/dashboard/MaintenanceMapClient.kt`
- Modify: `PprScheduler.kt` — take `MaintenanceMapGateway` instead of `MaintenanceMapStore`
- Modify: `WorkOrderStore.kt` — replace `MaintenanceMapStore?` with suspend/blocking gateway interface for PPR validation
- Modify: `Application.kt` — delete map routes/store/models (or move shared enums only if still needed for WO — keep WO types; delete map DTOs)
- Modify: tests — `MaintenanceMapRoutesTest.kt` **delete**; update `WorkOrderRoutesTest.kt` to fake gateway
- Modify: `deploy` / CI env — add `MAINTENANCE_SERVICE_BASE_URL=http://host.docker.internal:8098`
- Modify: `README.md`

**Interfaces:**
- Produces:
```kotlin
interface MaintenanceMapGateway {
    fun get(orgId: String, id: String): MaintenanceMapSnapshot
    fun listActive(orgId: String?, mapId: String?): List<MaintenanceMapSnapshot>
}
data class MaintenanceMapSnapshot(
    val id: String,
    val orgId: String,
    val assetId: String,
    val activatedAt: String?,
    val items: List<MaintenanceMapItemSnapshot>,
)
```
- `HttpMaintenanceMapGateway` calls `GET /maintenance-maps/{id}` and `GET /internal/active-maps`
- `PprScheduler.tick` uses `listActive`
- Confirm path no longer in dashboard (clients hit gateway → maintenance); remove confirm→tick from dashboard routes

- [ ] **Step 1: Update WorkOrderRoutesTest with FakeMaintenanceMapGateway; delete map route tests**

- [ ] **Step 2: Implement gateway + scheduler/WO changes; remove map HTTP routes**

- [ ] **Step 3: `./gradlew test`; commit; push; watch**

```bash
git commit -m "refactor: load PPR maps from maintenance-service over HTTP"
```

---

### Task 7: Technologist stack — maintenance MCP + confirm URL

**Repos:**  
- `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/technologist-service`  
- `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/technologist-mcp`

**Files (technologist-service):**
- Modify: `Application.kt` / deps — `MAINTENANCE_MCP_BASE_URL` (default `http://127.0.0.1:8099`), `MAINTENANCE_SERVICE_BASE_URL` (confirm)
- Modify: `Agents.kt` `RoutingMcpToolClient` — route `create_draft_maintenance_map` / `update_draft_maintenance_map` to maintenance MCP client
- Modify: confirm call currently `dashboardBaseUrl/maintenance-maps/.../confirm` → maintenance base
- Modify: deploy + CI env; tests

**Files (technologist-mcp):**
- Modify: `Application.kt` — remove map tools from `allowed` / `when`; remove `dashboardBaseUrl` if unused
- Modify: tests — drop map tool cases; assert tools list no longer includes map tools
- Modify: compose/CI — remove `DASHBOARD_BASE_URL` if unused

- [ ] **Step 1: technologist-mcp — failing test that map tools absent; implement; push**

- [ ] **Step 2: technologist-service — route map tools + confirm; tests; push; watch both**

```bash
# technologist-mcp
git commit -m "refactor: move PPR MCP tools out to maintenance-mcp"

# technologist-service
git commit -m "feat: call maintenance-mcp and maintenance-service for PPR"
```

---

### Task 8: Docs sync + smoke checklist

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/masterdoc`

**Files:**
- Modify: `docs/superpowers/specs/2026-07-22-equipment-technologist-design.md` — maps owner = maintenance-service :8098; MCP maps = maintenance-mcp :8099; update tables/ports
- Modify: `docs/services-logging.md` if it lists services
- Optionally mark extraction spec status `implemented` when smoke done

**Smoke (manual / agent with auth):**
1. Create draft map via MCP or POST `/maintenance-maps`
2. Restart maintenance container — map still listed
3. Confirm → WO appears after tick
4. Technologist flow still produces draft PPR

- [ ] **Step 1: Update docs; commit; push**

```bash
git commit -m "docs: point PPR maps and MCP at maintenance-service"
```

- [ ] **Step 2: Record smoke results in the PR/commit message or short note in spec Status line**

---

## Cutover order (enforce)

1. Tasks 1–4 deployed green on VPS  
2. Task 5 gateway  
3. Task 6 dashboard  
4. Task 7 technologist  
5. Task 8 docs  

## Self-review (plan vs spec)

| Spec item | Task |
|-----------|------|
| New repo + two processes + Postgres | 1, 4 |
| Public API unchanged | 2, 5 |
| Schema maps/items | 1–2 |
| Confirm → dashboard tick | 3 |
| Internal active maps | 3, 6 |
| MCP map tools moved | 4, 7 |
| Scheduler stays in dashboard | 6 |
| Gateway/env cutover | 5 |
| Strip technologist-mcp maps | 7 |
| No historical migration | (explicit non-task) |
| CI deploy new repo | 1, 4 |

No TBD placeholders. Types `MaintenanceMapSnapshot` introduced in Task 6 for dashboard decoupling from full maintenance DTO package.
