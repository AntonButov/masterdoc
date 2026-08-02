# Remaining Domain Postgres Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Persist the remaining process-RAM domain stores (AI messages, audit events, technologist jobs) in per-service Postgres, and drop dashboard’s in-memory fallback so tests and prod share one JDBC path.

**Architecture:** Clone `maintenance-service` / `catalog-service` pattern per repo: HikariCP + Flyway + JDBC repository; compose DB service names `*-postgres` (never bare `postgres`); Testcontainers for migrate + persistence tests; CI runtime `.env` gains `POSTGRES_*` + `DATABASE_URL`. Ship one service at a time: green CI/deploy before the next.

**Tech Stack:** Kotlin / Ktor, PostgreSQL 16, HikariCP, Flyway, Testcontainers, GitHub Actions deploy.

**Prior work (done — do not redo):**
- `catalog-service` — `JdbcSiteStore` / `JdbcAssetStore` / `JdbcScopeStore` + `catalog-postgres` (see `2026-08-02-catalog-dashboard-postgres.md`).
- `dashboard-service` — prod `main()` already `WorkOrderStore(dataSource)` + `dashboard-postgres`; RAM path remains only when `dataSource == null` (tests).
- `maintenance-service` — already Postgres.

**Out of scope:**
- `map-service` LocationStore (live presence; OK in RAM).
- `document-service` meta `ConcurrentHashMap` (index over durable files).
- Gateway JWKS cache.
- Client `InMemoryTokenStore` / PKCE.
- Shared Postgres cluster / PgBouncer.
- Automatic RAM→SQL dump on cutover.

## Global Constraints

- Compose DB names: `ai-message-postgres`, `black-box-postgres`, `technologist-postgres` only.
- HTTP APIs and DTO shapes unchanged.
- No automatic export of current RAM; after first deploy data starts empty (acceptable for messages/audit/jobs).
- Prefer Testcontainers (`disabledWithoutDocker = true`) like maintenance/catalog.
- Push → CI → deploy per repo; no heavy local full Gradle suites unless a single targeted test.
- Multi-repo: commit/push only in the repo the task touches; watch Actions.
- Retention: unbounded deques today — add pragmatic caps in SQL (below) so disks do not grow forever.

## File map (remaining)

| Repo | Files |
| --- | --- |
| `dashboard-service` | `WorkOrderStore.kt` (require DataSource), route tests wire Testcontainers/`Db.connect` |
| `ai-message-service` | `build.gradle.kts`, `Db.kt`, `V1__ai_messages.sql`, `JdbcAiMessageStore` (or evolve `AiMessageStore`), `Application.kt`, `deploy/docker-compose.yml`, CI, tests, README |
| `black-box-service` | same pattern for `audit_events` |
| `technologist-service` | same pattern for `technologist_jobs` (+ unique index org+document for in-flight coalesce) |

**Ship order:** Task 0 (dashboard cleanup) → Tasks 1–4 (ai-message) → Tasks 5–8 (black-box) → Tasks 9–12 (technologist).

---

### Task 0: dashboard — remove in-memory fallback

**Repo:** `dashboard-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt` — `module(... workOrderStore: WorkOrderStore)` no default `WorkOrderStore()`
- Modify: `src/test/kotlin/pro/masterdoc/dashboard/*RoutesTest.kt` (and any other `WorkOrderStore()` call sites) — construct via `Db.connect` + Testcontainers (shared companion container OK)

**Interfaces:**
- Consumes: existing `Db.connect`, `V1__work_orders.sql`
- Produces: `WorkOrderStore(dataSource: DataSource)` — non-null only; delete `ConcurrentHashMap` branch

- [ ] **Step 1:** Write/adjust one route test to require DataSource (fail if still calling no-arg ctor after removal)
- [ ] **Step 2:** Make `dataSource` required; delete RAM `byId` path and dead code
- [ ] **Step 3:** Fix all tests to use JDBC store
- [ ] **Step 4:** Targeted tests green → commit push → `gh run watch`

```bash
git commit -m "$(cat <<'EOF'
refactor(dashboard): require JDBC WorkOrderStore, drop RAM fallback

EOF
)"
```

---

### Task 1: ai-message — deps, Db, Flyway

**Repo:** `ai-message-service`

**Files:**
- Modify: `build.gradle.kts` — copy postgres/Hikari/Flyway/Testcontainers from `maintenance-service/build.gradle.kts`
- Create: `src/main/kotlin/pro/masterdoc/aimessage/Db.kt` (defaults: db/user/pass `aimessage`)
- Create: `src/main/resources/db/migration/V1__ai_messages.sql`
- Test: `src/test/kotlin/pro/masterdoc/aimessage/DbMigrateTest.kt`

**Schema:**

```sql
CREATE TABLE ai_messages (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL,
    kind TEXT NOT NULL,
    work_order_id TEXT NOT NULL,
    site_id TEXT NOT NULL,
    engineer_id TEXT NOT NULL,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    distance_m DOUBLE PRECISION,
    radius_m INTEGER,
    engineer_lat DOUBLE PRECISION,
    engineer_lon DOUBLE PRECISION,
    site_lat DOUBLE PRECISION,
    site_lon DOUBLE PRECISION,
    accuracy_m DOUBLE PRECISION
);
CREATE INDEX ai_messages_org_created_idx ON ai_messages (org_id, created_at DESC);
```

- [ ] **Step 1:** Failing `DbMigrateTest` (expect tables present)
- [ ] **Step 2:** Implement deps + `Db` + SQL → test PASS
- [ ] **Step 3:** Commit push watch

---

### Task 2: ai-message — JDBC store + wire main

**Repo:** `ai-message-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/aimessage/Application.kt` — `AiMessageStore(dataSource)`; `main()` calls `Db.connect()`
- Keep method surface: `append(CreateAiMessageRequest): AiMessage`, `list(orgId, limit, offset): List<AiMessage>`
- Optional retention: after append, delete rows for org beyond newest N (e.g. 5000) — document constant

**Interfaces:**
- Produces: JDBC-backed store; HTTP routes unchanged

- [ ] **Step 1:** Persistence test — append → new connection pool → list contains message
- [ ] **Step 2:** Implement JDBC; remove `ConcurrentLinkedDeque`
- [ ] **Step 3:** Route tests on Testcontainers
- [ ] **Step 4:** Commit push watch

---

### Task 3: ai-message — compose + CI

**Repo:** `ai-message-service`

**Files:**
- Modify: `deploy/docker-compose.yml` — `ai-message-postgres` + volume `ai_message_pg`; `DATABASE_URL`; `depends_on` healthy
- Modify: `.github/workflows/ci.yml` — `POSTGRES_DB/USER/PASSWORD` + `DATABASE_URL` in runtime.env (mirror maintenance)

- [ ] **Step 1:** Compose + CI
- [ ] **Step 2:** Commit push; confirm deploy success
- [ ] **Step 3:** README — `DATABASE_URL`, service name, empty-on-first-deploy note

---

### Task 4: ai-message — smoke persistence (ops)

- [ ] After green deploy: POST one AI message (or trigger geofence path), restart service via next deploy or compose restart, GET list — message still present
- [ ] Record result in PR/commit notes or chat report

---

### Task 5: black-box — deps, Db, Flyway

**Repo:** `black-box-service`

Same as Task 1; defaults db/user `blackbox`.

**Schema:**

```sql
CREATE TABLE audit_events (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    at TIMESTAMPTZ NOT NULL,
    method TEXT NOT NULL,
    path TEXT NOT NULL,
    status INTEGER NOT NULL,
    action TEXT,
    request_summary TEXT,
    response_summary TEXT
);
CREATE INDEX audit_events_org_at_idx ON audit_events (org_id, at DESC);
CREATE INDEX audit_events_user_at_idx ON audit_events (user_id, at DESC);
```

Keep existing redact/truncate in append path before INSERT.

- [ ] Failing migrate test → implement → PASS → commit push

---

### Task 6: black-box — JDBC `AuditStore` + wire main

**Files:**
- Modify: `Application.kt` — `AuditStore(dataSource)`; remove deque
- Methods: `append`, `list(orgId?, userId?, from?, to?, limit, offset)` — same filters in SQL WHERE

- [ ] Persistence + route tests on Testcontainers
- [ ] Commit push watch

---

### Task 7: black-box — compose + CI + README

- Compose: `black-box-postgres` / volume `black_box_pg`
- CI runtime.env + deploy watch
- README persistence note
- Optional retention: keep last 50k events globally or per org (constant in store)

---

### Task 8: black-box — smoke

- [ ] Append audit via normal client traffic or direct POST → redeploy → list still shows event

---

### Task 9: technologist — deps, Db, Flyway

**Repo:** `technologist-service`

Defaults db/user `technologist`.

**Schema:**

```sql
CREATE TABLE technologist_jobs (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL,
    document_id TEXT NOT NULL,
    site_id TEXT NOT NULL,
    asset_id TEXT NOT NULL,
    status TEXT NOT NULL,
    draft_asset_id TEXT,
    draft_map_id TEXT,
    error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX technologist_jobs_org_doc_idx ON technologist_jobs (org_id, document_id);
-- Coalesce only in-flight: partial unique optional; otherwise SELECT FOR UPDATE / app logic like today
```

- [ ] Migrate test → commit push

---

### Task 10: technologist — JDBC `JobStore` + wire main

**Files:**
- Modify: `Application.kt` `JobStore` — replace dual `ConcurrentHashMap` with JDBC
- Preserve coalesce: if existing job for `orgId+documentId` has status `queued|running`, return it; else insert new
- `get` / `getById` / `update` persist status, draft ids, error

- [ ] Persistence test: create job → new pool → get by id; coalesce in-flight
- [ ] Commit push watch

---

### Task 11: technologist — compose + CI + README

- Compose: `technologist-postgres` / volume `technologist_pg`
- Wire `DATABASE_URL` on API container (MCP stays proxy if separate)
- CI + deploy watch + README

---

### Task 12: technologist — smoke

- [ ] Start a job (or seed via API) → redeploy → job status still readable by id

---

## Success criteria

1. Redeploy ai-message / black-box / technologist does not wipe their domain rows.
2. Dashboard tests and prod both use JDBC only (no silent RAM store).
3. Public HTTP contracts unchanged.
4. CI green with Postgres in compose/test path for each touched repo.
5. map/document left as-is (documented exceptions).

## Execution note

Run via **subagent-driven-development**: one fresh implementer subagent per task, review between tasks, final branch review per repo after its task block.
