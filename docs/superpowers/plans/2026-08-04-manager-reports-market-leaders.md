# Manager reports market leaders pack — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 4 market-leader style manager reports (KPI trends, reactive/completion, engineer workload, failure frequency) with separate `/reports/*` APIs, richer Demo seed, and client-app catalog UI.

**Architecture:** `dashboard-service` exposes four new GET endpoints under `/reports` (gateway already proxies). Compute from existing work orders. `client-app` adds catalog entries + detail screens using existing chart primitives. Seed assigns engineers and creates wavy emergency/planned patterns.

**Tech Stack:** Kotlin/Ktor (dashboard), Compose Multiplatform (client), kotlinx.serialization, existing Canvas charts.

**Spec:** `masterdoc/docs/superpowers/specs/2026-08-04-manager-reports-market-leaders-design.md`

## Global Constraints

- Separate endpoint per new report (do **not** bloat `manager-kpis`).
- UI: never show raw user/asset UUIDs — display names; fallback «Инженер» / «Оборудование».
- Do not break existing 6 reports or their API shapes.
- Failure frequency counts **emergency** WOs **created** in window.
- Engineer workload: closed WOs with non-null `assigneeId`; hours from `startedAt…closedAt` when both set else 0.
- KPI trends: period ≤30d → day buckets; longer → week buckets.
- Commit/push each repo after its tasks; CI builds on GitHub (no heavy local gradle).
- Demo org `382715225649971203` must get quality seed after merge.

---

### Task 1: dashboard DTOs + reactive-completion + kpi-trends

**Files:**
- Create: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/ManagerReportExtras.kt`
- Modify: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt`
- Modify: `dashboard-service/src/main/kotlin/pro/masterdoc/dashboard/Application.kt`
- Create/Modify tests: `dashboard-service/src/test/kotlin/pro/masterdoc/dashboard/ManagerReportExtrasTest.kt`, extend `WorkOrderRoutesTest.kt`

**Interfaces:**
- Produces: `computeReactiveCompletion(orders, from, to): ReactiveCompletionReport`
- Produces: `computeKpiTrends(orders, from, to, now): KpiTrendsReport`
- Store methods: `reactiveCompletion`, `kpiTrends`
- Routes: `GET /reports/reactive-completion`, `GET /reports/kpi-trends`

- [ ] Add serializable DTOs matching spec JSON.
- [ ] Implement compute with same Instant/parse helpers patterns as `ManagerKpis.kt` (reuse private helpers by moving shared parse helpers to internal visibility or duplicating carefully).
- [ ] For trends: split `[from,to]` into day or week buckets; per bucket compute MTTR/MTBF/availability using same formulas as `computeManagerKpis` but scoped to bucket window.
- [ ] Wire store + routes like `manager-kpis`.
- [ ] Unit tests: zero created → completion 0 / reactive handling; bucket day vs week selection; empty org returns empty points / zeros.
- [ ] Route smoke in `WorkOrderRoutesTest`: GET returns 200 with expected keys after seed or fixtures.
- [ ] Commit + push `dashboard-service` branch/main.

### Task 2: dashboard engineer-workload + failure-frequency

**Files:**
- Modify: `ManagerReportExtras.kt`, `WorkOrderStore.kt`, `Application.kt`, tests

**Interfaces:**
- Produces: `computeEngineerWorkload`, `computeFailureFrequency` (top 15)
- Routes: `GET /reports/engineer-workload`, `GET /reports/failure-frequency`

- [ ] Implement + tests + routes.
- [ ] Commit + push.

### Task 3: enrich manager-reports seed

**Files:**
- Modify: `ManagerReportsSeed.kt`, `SeedManagerReportsRequest` (+ optional `assigneeIds: List<String>?`)
- Modify: internal seed route handler
- Modify: `.github/workflows/seed-manager-reports.yml` to pass assignee ids if available from catalog users / fixed demo engineer ids
- Modify: `ManagerReportsSeedTest.kt`

**Behavior:**
- Accept optional `assigneeIds` (default synthetic `seed-engineer-1..3` if missing).
- After closing emergencies/PPR, set assignee round-robin via `store.update(..., assigneePresent=true, assigneeId=...)`.
- Bias more emergencies onto first 1–2 assets (failure leader).
- Vary repair hours sinusoidally by day index so trends are not flat.
- Commit + push.

### Task 4: client auth models + repository

**Files:**
- Modify: `client-app/auth/.../ManagerKpiModels.kt` (or new `ManagerReportExtraModels.kt`)
- Modify: `WorkOrdersRepository` / reports repo methods in `WorkOrderModels.kt`
- Tests: `WorkOrdersRepositoryTest.kt` decode fixtures

- [ ] DTOs mirror server.
- [ ] Four suspend getters.
- [ ] Commit + push (can batch with Task 5 if same PR wave — prefer one client commit after Task 5).

### Task 5: client catalog + detail UI

**Files:**
- Modify: `ReportCatalog.kt`, `ReportCatalogTest.kt`
- Modify: `ReportsScreen.kt`, `ReportChartModels.kt` (+ tests)
- Possibly `ManagerKpiFormatters.kt`

**UI:**
- Append 4 catalog items with Russian titles/descriptions from spec.
- Detail: load correct endpoint by `ReportId`; period 7/30/90; help footer.
- Trends: column chart of one primary series or three small charts — prefer three metric rows + column for availability OR multi-point chart for MTTR over time (use `ReportColumnChart` with bucket labels truncated).
- Reactive: metric rows + column for planned vs emergency counts + completion %.
- Workload: horizontal bars with resolved engineer labels (need org users list if available — else show email from a users API; if none, «Инженер» + do not show id).
- Failure frequency: horizontal bars with asset `displayName()`.

**Engineer name resolution:** reuse whatever admin/users or board assignee label helper exists (`formatAssigneeLabel` pattern). If only id available without lookup, use «Инженер» (never truncated UUID).

- [ ] Tests for catalog size ≥ 10 and non-blank descriptions.
- [ ] Commit + push client-app.

### Task 6: ship gate + Demo seed + smoke

- [ ] Ensure dashboard + client CI green on main.
- [ ] `gh workflow run seed-manager-reports.yml -f org_id=382715225649971203`
- [ ] Also seed Smoke `383177088934346755` for agent smoke.
- [ ] `/smoke-test`: open all 4 new reports + spot-check old KPI; screenshots + critical visual check.

---

## Spec coverage

| Spec item | Task |
|-----------|------|
| kpi-trends API | 1 |
| reactive-completion API | 1 |
| engineer-workload API | 2 |
| failure-frequency API | 2 |
| Demo seed quality | 3 |
| Client catalog + UI | 4–5 |
| Smoke | 6 |
