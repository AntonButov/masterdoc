# Overdue open work orders report — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add catalog report «Просроченные»: list of open work orders with `dueAt` before today, tap opens read-only card.

**Architecture:** `dashboard-service` adds pure filter + `GET /reports/overdue-open-work-orders` (WorkOrder JSON array). Gateway already proxies `/reports` and allows `reports` → `GET /work-orders/{id}`. `client-app` adds catalog item, repository method, list UI + read-only card (same pattern as EquipmentWorkOrders).

**Tech Stack:** Kotlin/Ktor (`dashboard-service`), Compose Multiplatform (`client-app`), existing `WorkOrderDto` / `formatAssigneeLabel` / `WorkOrderDetailScreen`.

**Spec:** `docs/superpowers/specs/2026-08-16-overdue-open-work-orders-report-design.md` (masterdoc repo)

## Global Constraints

- UI: never show raw UUIDs / user ids; titles via `formatWorkOrderDisplayTitle`; assignee via `formatAssigneeLabel` or «не назначен».
- No period selector (snapshot «сейчас»).
- Open = `new` | `in_progress`; overdue = parseable `dueAt` LocalDate **strictly before** UTC today (same as `ManagerKpis` backlog overdue).
- Drop unparseable `dueAt`. Sort: `dueAt` asc, then `id` asc.
- Empty list → `200 []`. Error copy: fixed Russian strings only.
- Card: `WorkOrderDetailScreen(readOnly = true, allowMediaMutations = false)` with attachment/comment repos for view.
- Focused Gradle tests only; no local Wasm / full distribution.
- Each task: commit → push default branch → watch CI/deploy → smoke for that task’s surface.
- Smoke tenant: **Fixaverse Smoke** (`383177088934346755`) only.

---

## File structure

| File | Responsibility |
|---|---|
| `dashboard-service/.../OverdueOpenWorkOrdersReport.kt` | `selectOverdueOpenWorkOrders(orders, today)` |
| `dashboard-service/.../OverdueOpenWorkOrdersReportTest.kt` | Unit filter/sort |
| `dashboard-service/.../WorkOrderStore.kt` | `overdueOpenWorkOrders(orgId, today)` |
| `dashboard-service/.../Application.kt` | `GET /reports/overdue-open-work-orders` |
| `dashboard-service/.../WorkOrderRoutesTest.kt` | HTTP route smoke |
| `client-app/.../ReportCatalog.kt` | `ReportId.OverdueOpenWorkOrders` + row |
| `client-app/.../ReportCatalogTest.kt` | 12 items, last = Просроченные |
| `client-app/auth/.../WorkOrderModels.kt` | `overdueOpenWorkOrders()` |
| `client-app/auth/.../WorkOrdersRepositoryTest.kt` | URL + decode |
| `client-app/.../OverdueOpenWorkOrdersReportUi.kt` | List + read-only card |
| `client-app/.../ReportsScreen.kt` | Branch to new screen |

---

### Task 1: dashboard overdue-open report API

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/dashboard-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/dashboard/OverdueOpenWorkOrdersReport.kt`
- Create: `src/test/kotlin/pro/masterdoc/dashboard/OverdueOpenWorkOrdersReportTest.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt` (after equipment-work-orders route)
- Modify: `src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt`

**Interfaces:**
- Consumes: `WorkOrder`, `WorkOrderStatus`, `WeekDates.parseDate` / dueAt date helper used by ManagerKpis
- Produces: `fun selectOverdueOpenWorkOrders(orders: List<WorkOrder>, today: LocalDate): List<WorkOrder>`
- Produces: `WorkOrderStore.overdueOpenWorkOrders(orgId: String, today: LocalDate): List<WorkOrder>`
- Produces: `GET /reports/overdue-open-work-orders` → JSON array of `WorkOrder`

- [ ] **Step 1: Write failing unit test**

Create `OverdueOpenWorkOrdersReportTest.kt`:

```kotlin
package pro.masterdoc.dashboard

import java.time.LocalDate
import kotlin.test.Test
import kotlin.test.assertEquals

class OverdueOpenWorkOrdersReportTest {
    private val today = LocalDate.parse("2026-08-16")

    @Test
    fun includesOpenPastDueExcludesClosedOnTimeAndBadDue() {
        val overdueNew =
            workOrder("o1", status = WorkOrderStatus.new, dueAt = "2026-08-01")
        val overdueIp =
            workOrder("o2", status = WorkOrderStatus.in_progress, dueAt = "2026-08-10")
        val dueToday =
            workOrder("today", status = WorkOrderStatus.new, dueAt = "2026-08-16")
        val future =
            workOrder("fut", status = WorkOrderStatus.new, dueAt = "2026-08-20")
        val closedOverdue =
            workOrder("cl", status = WorkOrderStatus.closed, dueAt = "2026-08-01", closedAt = "2026-08-05T00:00:00Z")
        val badDue =
            workOrder("bad", status = WorkOrderStatus.new, dueAt = "not-a-date")

        val result =
            selectOverdueOpenWorkOrders(
                listOf(overdueNew, overdueIp, dueToday, future, closedOverdue, badDue),
                today,
            )
        assertEquals(listOf("o1", "o2"), result.map { it.id })
    }

    @Test
    fun sortsByDueAtAscThenIdAsc() {
        val a = workOrder("b", status = WorkOrderStatus.new, dueAt = "2026-08-01")
        val b = workOrder("a", status = WorkOrderStatus.new, dueAt = "2026-08-01")
        val c = workOrder("c", status = WorkOrderStatus.new, dueAt = "2026-07-01")
        assertEquals(
            listOf("c", "a", "b"),
            selectOverdueOpenWorkOrders(listOf(a, b, c), today).map { it.id },
        )
    }

    private fun workOrder(
        id: String,
        status: WorkOrderStatus,
        dueAt: String,
        closedAt: String? = null,
    ) = WorkOrder(
        id = id,
        orgId = "org-1",
        type = WorkOrderType.emergency,
        status = status,
        title = id,
        assetId = "a1",
        siteId = "s1",
        dueAt = dueAt,
        durationHours = 8,
        source = WorkOrderSource.api,
        createdAt = "2026-08-01T00:00:00Z",
        updatedAt = "2026-08-01T00:00:00Z",
        closedAt = closedAt,
    )
}
```

- [ ] **Step 2: Run test — expect FAIL** (symbol missing)

```bash
cd /Users/antonbutov/Documents/MYPROJECTS/fixaverse/dashboard-service
./gradlew test --tests 'pro.masterdoc.dashboard.OverdueOpenWorkOrdersReportTest'
```

- [ ] **Step 3: Implement pure function + store + route**

`OverdueOpenWorkOrdersReport.kt`:

```kotlin
package pro.masterdoc.dashboard

import java.time.LocalDate

fun selectOverdueOpenWorkOrders(
    orders: List<WorkOrder>,
    today: LocalDate,
): List<WorkOrder> =
    orders
        .filter { it.status == WorkOrderStatus.new || it.status == WorkOrderStatus.in_progress }
        .filter { due ->
            val d = WeekDates.parseDate(due.dueAt) ?: return@filter false
            d.isBefore(today)
        }
        .sortedWith(compareBy({ it.dueAt }, { it.id }))
```

Add on `WorkOrderStore`:

```kotlin
fun overdueOpenWorkOrders(orgId: String, today: LocalDate): List<WorkOrder> =
    selectOverdueOpenWorkOrders(list(orgId), today)
```

In `Application.kt` after equipment-work-orders:

```kotlin
get("/reports/overdue-open-work-orders") {
    val today = clock.instant().atZone(java.time.ZoneOffset.UTC).toLocalDate()
    call.respond(workOrderStore.overdueOpenWorkOrders(call.orgId(), today))
}
```

(Use the same `clock` already injected into `module` — match existing pattern; if Application uses `Clock.systemUTC()` elsewhere for reports, follow that file’s existing clock access.)

- [ ] **Step 4: Route test**

In `WorkOrderRoutesTest.kt` add test that creates open overdue + closed + due-today with fixed `clock`, asserts only overdue open ids returned, and empty org returns `[]`.

- [ ] **Step 5: Run focused tests — PASS**

```bash
./gradlew test --tests 'pro.masterdoc.dashboard.OverdueOpenWorkOrdersReportTest' --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.overdueOpenWorkOrders*'
```

- [ ] **Step 6: Commit on feature branch, open PR, merge to main**

```bash
git checkout -b feature/overdue-open-report
git add src/main/kotlin/pro/masterdoc/dashboard/OverdueOpenWorkOrdersReport.kt \
  src/test/kotlin/pro/masterdoc/dashboard/OverdueOpenWorkOrdersReportTest.kt \
  src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt \
  src/main/kotlin/pro/masterdoc/dashboard/Application.kt \
  src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt
git commit -m "$(cat <<'EOF'
feat: report open work orders past due date

EOF
)"
git push -u origin HEAD
gh pr create --title "feat: overdue open work orders report" --body "## Summary
- GET /reports/overdue-open-work-orders
- Pure filter + route tests
## Test plan
- [x] unit + route tests
"
gh pr merge --squash --delete-branch
```

- [ ] **Step 7: Release — watch CI + deploy on main**

```bash
gh run list --branch main --limit 3
gh run watch <run-id>
```

Expected: test + Deploy success.

- [ ] **Step 8: API smoke (after green deploy)**

```bash
# unauth
curl -sS -o /tmp/ov -w '%{http_code}\n' 'https://api.masterdoc.pro/reports/overdue-open-work-orders'
# expect 401
curl -sS -o /tmp/h -w '%{http_code}\n' 'https://api.masterdoc.pro/health'
# expect 200
```

Write short smoke note in task report: unauth 401 + health 200. Full UI smoke is Task 2.

---

### Task 2: client catalog + overdue list UI

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCatalog.kt`
- Modify: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ReportCatalogTest.kt`
- Modify: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt`
- Modify: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt`
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/OverdueOpenWorkOrdersReportUi.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportsScreen.kt`

**Interfaces:**
- Consumes: Task 1 endpoint; `WorkOrderDetailScreen`; `formatAssigneeLabel`; `AdminUsersRepository` optional for names
- Produces: catalog 12 items; UI list without period chips

- [ ] **Step 1: Failing catalog + repository tests**

Update `ReportCatalogTest` to expect **12** items, last `OverdueOpenWorkOrders` / title «Просроченные»; keep `EquipmentWorkOrders` as second-to-last.

Add repository test:

```kotlin
fun overdueOpenWorkOrdersHitsExactPath() = runBlocking {
    // mock GET .../reports/overdue-open-work-orders → decode list
}
```

- [ ] **Step 2: Run — FAIL**

```bash
./gradlew :composeApp:testDebugUnitTest --tests 'pro.masterdoc.client.ui.screens.ReportCatalogTest' \
  :auth:test --tests 'pro.masterdoc.client.auth.WorkOrdersRepositoryTest.overdueOpen*'
```

(Use the project’s actual test task names if different — match existing `ReportCatalogTest` / `WorkOrdersRepositoryTest` invocation from CI or README.)

- [ ] **Step 3: Catalog + repository**

Add enum + catalog item:

```kotlin
ReportCatalogItem(
    id = ReportId.OverdueOpenWorkOrders,
    title = "Просроченные",
    subtitle = "Открытые с истёкшим сроком",
    description =
        "Показывает открытые заявки, у которых срок уже прошёл. " +
            "Нажмите строку, чтобы открыть карточку.",
)
```

Repository:

```kotlin
suspend fun overdueOpenWorkOrders(): List<WorkOrderDto> {
    val response =
        http.get(
            url = "${base()}/reports/overdue-open-work-orders",
            headers = mapOf("Authorization" to "Bearer ${bearer()}"),
        )
    if (!response.isSuccessful) {
        throw GatewayHttpException(
            response.status,
            response.body.ifBlank { "overdue open work orders report failed" },
        )
    }
    return json.decodeFromString(response.body)
}
```

- [ ] **Step 4: UI screen**

Create `OverdueOpenWorkOrdersReportUi.kt` mirroring `EquipmentWorkOrdersReportUi` **without** PeriodSelector / asset picker:

- Load `adminUsersRepository?.listUsers()` (or existing list API used by engineer workload) for assignee labels; if null/fail, still show list with «не назначен» / generic «Инженер».
- `LaunchedEffect` → `reportsRepository.overdueOpenWorkOrders()`
- Rows: title; `type · dueAt`; status chip; assignee line
- Empty / loading / error states per Global Constraints
- Tap → `WorkOrderDetailScreen(..., readOnly = true, allowMediaMutations = false)`
- Pass attachments/comments from `ReportsScreen` like equipment report

Branch in `ReportsScreen` before other details:

```kotlin
if (selectedReport == ReportId.OverdueOpenWorkOrders) {
    OverdueOpenWorkOrdersReportScreen(...)
    return
}
```

- [ ] **Step 5: Run tests — PASS**, then commit + PR + merge to main (same flow as Task 1)

- [ ] **Step 6: Release — watch Deploy to app.fixaverse.ru success**

- [ ] **Step 7: UI smoke** (`/smoke-test` skill)

App: https://app.fixaverse.ru/  
Org: Fixaverse Smoke  
Checklist:

1. Profile org = Fixaverse Smoke  
2. Отчёты → last item «Просроченные» (12 items; «Детальный отчёт» still present)  
3. List shows overdue opens OR empty Russian state; names not ids  
4. If rows exist: tap → read-only card; Back → list  
5. Console: no red errors  

Screenshots + Read required. Report PASS/FAIL/PARTIAL with SHA + CI URL.

---

## Self-review (plan)

- Spec coverage: inclusion, API, UX, card, no gateway — Tasks 1–2 cover all.
- No placeholders left in steps.
- Types: `selectOverdueOpenWorkOrders` / `overdueOpenWorkOrders()` consistent across tasks.
