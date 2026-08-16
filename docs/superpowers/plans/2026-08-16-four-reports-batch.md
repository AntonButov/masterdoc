# Four reports batch — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development. Checkbox steps.

**Goal:** Ship four catalog reports: site WOs, time-to-first-action, PPR plan/fact, closures without photos.

**Architecture:** Additive `GET /reports/...` in dashboard; client catalog + list UIs mirroring Equipment/Overdue patterns; gateway unchanged.

**Tech Stack:** Kotlin/Ktor, Compose Multiplatform, existing `WorkOrderDto` / `WorkOrderDetailScreen`.

**Spec:** `docs/superpowers/specs/2026-08-16-four-reports-batch-design.md` (masterdoc)

## Global Constraints

- No raw ids in UI; fixed Russian error/empty strings; read-only card with `allowMediaMutations = false`.
- Focused Gradle tests only; no local Wasm distribution.
- Each task: commit → PR → squash merge main → watch CI/deploy → smoke for that surface.
- Smoke org: Fixaverse Smoke only.
- Catalog grows 12 → 16; append order: SiteWorkOrders, TimeToFirstAction, PprPlanFact, ClosuresWithoutPhotos (after OverdueOpen).
- Reuse `selectEquipmentWorkOrders` overlap helpers where possible for site report (same overlap, filter `siteId`).

---

### Task 1: dashboard site-work-orders

**Repo:** `dashboard-service`

**Files:**
- Create `SiteWorkOrdersReport.kt` + test (copy equipment overlap, filter `siteId`)
- Modify `WorkOrderStore.kt`, `Application.kt`, `WorkOrderRoutesTest.kt`

**Produces:** `selectSiteWorkOrders(...); GET /reports/site-work-orders?siteId&from&to` (400 without siteId)

- [ ] TDD unit + route; commit; PR merge; watch deploy
- [ ] API smoke: unauth 401 on new path; health 200
- [ ] Report → `masterdoc/.superpowers/sdd/task-1-site-report.md`

---

### Task 2: client site-work-orders UI

**Repo:** `client-app`

**Files:** `ReportCatalog.kt` (+1 item, test expect 13), `WorkOrderModels.kt` `siteWorkOrders`, `SiteWorkOrdersReportUi.kt` (mirror Equipment with site picker via `listSites()`), `ReportsScreen.kt` branch

- [ ] TDD catalog+repo; UI; PR merge; deploy; UI smoke (picker by site name → list → card → back)
- [ ] Report → `task-2-site-report.md`

---

### Task 3: dashboard time-to-first-action

**Repo:** `dashboard-service`

**Produces:** `selectTimeToFirstAction(orders, from, to)`; `GET /reports/time-to-first-action?from&to`

Filter: parseable `createdAt` in `[from,to]`. Sort: null `startedAt` first, then duration desc, id desc.

- [ ] TDD; merge; deploy; API smoke 401
- [ ] Report → `task-3-tta-report.md`

---

### Task 4: client time-to-first-action UI

**Repo:** `client-app` — catalog 14; `TimeToFirstActionReportUi.kt` with period + hours line

- [ ] Merge; deploy; UI smoke
- [ ] Report → `task-4-tta-report.md`

---

### Task 5: dashboard ppr-plan-fact

**Repo:** `dashboard-service`

**Produces:** `selectPprPlanFact(orders, fromDate, toDate)`; `GET /reports/ppr-plan-fact?from&to`  
(type=ppr, dueAt date in range)

- [ ] Merge; deploy; API smoke
- [ ] Report → `task-5-ppr-report.md`

---

### Task 6: client ppr-plan-fact UI

**Repo:** `client-app` — catalog 15; outcome labels per spec

- [ ] Merge; deploy; UI smoke
- [ ] Report → `task-6-ppr-report.md`

---

### Task 7: dashboard closures-without-photos

**Repo:** `dashboard-service`

**Produces:** `selectClosuresWithoutPhotos(orders, from, to)`; `GET /reports/closures-without-photos?from&to`

- [ ] Merge; deploy; API smoke
- [ ] Report → `task-7-closures-report.md`

---

### Task 8: client closures-without-photos UI

**Repo:** `client-app` — catalog **16**; final UI smoke full catalog regression (Просроченные + Детальный still present)

- [ ] Merge; deploy; UI smoke
- [ ] Report → `task-8-closures-report.md`

---

## Patterns to copy

| Feature | Copy from |
|---|---|
| Site overlap | `EquipmentWorkOrdersReport.kt` |
| Snapshot-style list UI | `OverdueOpenWorkOrdersReportUi.kt` |
| Period + picker | `EquipmentWorkOrdersReportUi.kt` |
| Period only | KPI report detail / overdue without picker |
| Catalog test | bump size + last id each client task |
| Store/route | after existing report routes in `Application.kt` |
