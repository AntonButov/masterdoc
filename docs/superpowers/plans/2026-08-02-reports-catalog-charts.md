# Reports Catalog + Vico Charts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn «Отчёты» into a catalog of six selectable reports with professional number formatting, Vico charts on five KPI views, and seeded Smoke-org work orders so charts are not empty.

**Architecture:** Local master→detail stack inside `ReportsScreen` (same pattern as `MyWorkOrdersScreen`). Detail loads `manager-kpis` and/or `equipment-downtime` for the selected period. Dashboard keeps existing KPI formulas; adds internal seed that clears org WOs and inserts a deterministic set. Ops workflow seeds Fixaverse Smoke.

**Tech Stack:** KMP Compose (`client-app`), Vico `3.2.3` (`compose` + `compose-m3`), Ktor dashboard-service, GitHub Actions workflow_dispatch.

**Spec:** `masterdoc/docs/superpowers/specs/2026-08-02-reports-catalog-charts-design.md`

## Global Constraints

- UI never shows raw UUIDs / user ids — asset names / «Оборудование» (`ui-names-not-ids`).
- Hours & %: **1** decimal, Russian comma; counts: integers; `sampleSize == 0` → `н/д`.
- Do not change `GET /reports/manager-kpis` or `equipment-downtime` response shape / formulas.
- No client-side mock KPI data when API empty — seed Smoke via API.
- Focused unit tests only; no local Wasm/full desktop distribution builds.
- Commit/push per affected repo; watch Actions; finish with `/smoke-test` on Reports.
- Branches: prefer `feat/reports-catalog-charts` in `client-app` and `dashboard-service` (or continue current feature branch if already on reports work — stay consistent).

## File structure

| File | Responsibility |
|------|----------------|
| `client-app/gradle/libs.versions.toml` | Pin `vico = "3.2.3"` |
| `client-app/composeApp/build.gradle.kts` | `vico-compose` + `vico-compose-m3` in commonMain |
| `client-app/.../ManagerKpiFormatters.kt` | `formatHours`, `formatPercent`, metric helpers |
| `client-app/.../ReportCatalog.kt` | enum/ids + 6 catalog items |
| `client-app/.../ReportChartModels.kt` | pure DTO → chart series (labels + floats) |
| `client-app/.../ReportCharts.kt` | thin Vico wrappers (column / horizontal bar) |
| `client-app/.../ReportsScreen.kt` | catalog + detail navigation + period |
| `client-app/.../ManagerKpiFormattersTest.kt` | formatter tests |
| `client-app/.../ReportCatalogTest.kt` | catalog + series mapper tests |
| `client-app/.../ReportsScreenTest.kt` | keep downtime row tests |
| `dashboard-service/.../ManagerReportsSeed.kt` | pure seed plan + apply via store |
| `dashboard-service/.../Application.kt` | `POST /internal/orgs/{orgId}/seed-manager-reports` |
| `dashboard-service/.../ManagerReportsSeedTest.kt` | seed → non-zero KPI samples |
| `dashboard-service/.github/workflows/seed-manager-reports.yml` | ops seed for Smoke |

---

### Task 1: Number formatters (client-app)

**Repo:** `client-app`  
**Branch:** `feat/reports-catalog-charts`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ManagerKpiFormatters.kt`
- Modify: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ManagerKpiFormattersTest.kt`

**Interfaces:**
- Produces:
  - `fun formatHours(value: Double): String` → `"4,5 ч"` (1 decimal)
  - `fun formatPercent(value: Double): String` → `"92,1%"`
  - `fun formatManagerKpiMetric(value: Double, sampleSize: Int, suffix: String = ""): String` — uses rounded hours-style when suffix is hours; still `н/д` if sampleSize=0

- [ ] **Step 1: Extend failing tests**

Add to `ManagerKpiFormattersTest.kt`:

```kotlin
@Test
fun roundsHoursToOneDecimal() {
    assertEquals("4,6 ч", formatHours(4.567))
    assertEquals("0,0 ч", formatHours(0.0))
}

@Test
fun roundsPercentToOneDecimal() {
    assertEquals("92,1%", formatPercent(92.14))
    assertEquals("100,0%", formatPercent(100.0))
}

@Test
fun formatsMetricWithLongDoubleAsOneDecimal() {
    assertEquals("4,6 ч", formatManagerKpiMetric(4.567, 2, suffix = " ч"))
}
```

Update existing `formatsEmptyKpiSampleAsNotAvailable` if needed so non-empty path still expects one decimal.

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd client-app
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.ManagerKpiFormattersTest'
```

Expected: FAIL — `formatHours` / `formatPercent` unresolved or wrong rounding.

- [ ] **Step 3: Implement formatters**

In `ManagerKpiFormatters.kt`:

```kotlin
internal fun formatHours(value: Double): String =
    "${formatOneDecimal(value)} ч"

internal fun formatPercent(value: Double): String =
    "${formatOneDecimal(value)}%"

internal fun formatManagerKpiMetric(
    value: Double,
    sampleSize: Int,
    suffix: String = "",
): String =
    if (sampleSize == 0) {
        "н/д"
    } else {
        "${formatOneDecimal(value)}$suffix"
    }

private fun formatOneDecimal(value: Double): String {
    val scaled = kotlin.math.round(value * 10.0) / 10.0
    val asLong = scaled.toLong()
    return if (scaled == asLong.toDouble()) {
        "$asLong,0"
    } else {
        scaled.toString().replace('.', ',')
    }
}
```

Remove any remaining raw `value.toString().replace` usage from report UI in later tasks; for this task only formatters + tests.

Also fix `hours()` helper later in ReportsScreen to call `formatHours` (Task 3+).

- [ ] **Step 4: Run tests — expect PASS**

Same Gradle command as Step 2. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ManagerKpiFormatters.kt \
  composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ManagerKpiFormattersTest.kt
git commit -m "$(cat <<'EOF'
fix(reports): round KPI hours and percents to one decimal

EOF
)"
```

---

### Task 2: Report catalog model + list navigation (client-app)

**Repo:** `client-app`  
**Branch:** `feat/reports-catalog-charts`

**Files:**
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCatalog.kt`
- Create: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ReportCatalogTest.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportsScreen.kt`

**Interfaces:**
- Produces:
  - `enum class ReportId { KpiSummary, PlannedVsEmergency, PprCompliance, Backlog, DowntimeRanking, EquipmentDowntime }`
  - `data class ReportCatalogItem(val id: ReportId, val title: String, val subtitle: String)`
  - `fun reportCatalogItems(): List<ReportCatalogItem>` — exactly 6, fixed Russian titles from spec

- [ ] **Step 1: Failing catalog test**

```kotlin
package pro.masterdoc.client.ui.screens

import kotlin.test.Test
import kotlin.test.assertEquals

class ReportCatalogTest {
    @Test
    fun catalogHasSixReportsInStableOrder() {
        val items = reportCatalogItems()
        assertEquals(6, items.size)
        assertEquals(
            listOf(
                ReportId.KpiSummary,
                ReportId.PlannedVsEmergency,
                ReportId.PprCompliance,
                ReportId.Backlog,
                ReportId.DowntimeRanking,
                ReportId.EquipmentDowntime,
            ),
            items.map { it.id },
        )
        assertEquals("Сводка KPI", items.first().title)
        assertEquals("Простои оборудования", items.last().title)
    }
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.ReportCatalogTest'
```

- [ ] **Step 3: Implement `ReportCatalog.kt`**

```kotlin
package pro.masterdoc.client.ui.screens

enum class ReportId {
    KpiSummary,
    PlannedVsEmergency,
    PprCompliance,
    Backlog,
    DowntimeRanking,
    EquipmentDowntime,
}

data class ReportCatalogItem(
    val id: ReportId,
    val title: String,
    val subtitle: String,
)

fun reportCatalogItems(): List<ReportCatalogItem> =
    listOf(
        ReportCatalogItem(ReportId.KpiSummary, "Сводка KPI", "MTTR, MTBF и готовность"),
        ReportCatalogItem(ReportId.PlannedVsEmergency, "Плановые vs аварийные", "Объём и часы работ"),
        ReportCatalogItem(ReportId.PprCompliance, "Выполнение ППР", "Вовремя, с опозданием, открытые"),
        ReportCatalogItem(ReportId.Backlog, "Очередь заявок", "Возраст и просрочки"),
        ReportCatalogItem(ReportId.DowntimeRanking, "Рейтинг простоев", "Оборудование по часам простоя"),
        ReportCatalogItem(ReportId.EquipmentDowntime, "Простои оборудования", "Шкала простоев по дням"),
    )
```

- [ ] **Step 4: Refactor `ReportsScreen` to catalog → detail stack**

Mirror `MyWorkOrdersScreen`:

```kotlin
var selectedReport by remember { mutableStateOf<ReportId?>(null) }

selectedReport?.let { reportId ->
    ReportDetailScreen(
        reportId = reportId,
        reportsRepository = reportsRepository,
        equipmentRepository = equipmentRepository,
        onBack = { selectedReport = null },
        modifier = modifier,
    )
    return
}

AppScaffold(title = "Отчёты", modifier = modifier) { padding ->
    LazyColumn(...) {
        items(reportCatalogItems(), key = { it.id.name }) { item ->
            Column(
                Modifier.fillMaxWidth().clickable { selectedReport = item.id }.padding(...)
            ) {
                AppText(item.title, style = AppTextStyle.Title)
                AppText(item.subtitle, style = AppTextStyle.Label)
            }
        }
    }
}
```

Move existing KPI dump + Gantt into `ReportDetailScreen` temporarily (still may show one report’s content; for this task wire:
- `KpiSummary` → old summary metrics with new formatters
- `EquipmentDowntime` → existing Gantt
- other ids → placeholder text «Скоро» OR thin metric-only sections without charts yet

Prefer: for non-Gantt ids show the existing text sections that belong to that id only (split `ManagerKpiSections`), still no Vico — charts come in Task 4.

Detail must have:
- `AppScaffold(title = item.title, onNavigateBack = onBack)`
- `PeriodSelector` 7/30/90
- Load APIs only for needed data:
  - EquipmentDowntime → downtime + assets
  - others → managerKpis + assets (assets needed for ranking)

- [ ] **Step 5: Run catalog + existing reports tests**

```bash
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.ReportCatalogTest' --tests 'pro.masterdoc.client.ui.screens.ReportsScreenTest' --tests 'pro.masterdoc.client.ui.screens.ManagerKpiFormattersTest'
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCatalog.kt \
  composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportsScreen.kt \
  composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ReportCatalogTest.kt
git commit -m "$(cat <<'EOF'
feat(reports): catalog of six reports with detail navigation

EOF
)"
```

---

### Task 3: Add Vico dependency (client-app)

**Repo:** `client-app`  
**Branch:** `feat/reports-catalog-charts`

**Files:**
- Modify: `gradle/libs.versions.toml`
- Modify: `composeApp/build.gradle.kts`
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCharts.kt` (compile smoke composable)

**Interfaces:**
- Produces: dependency resolves; `ReportColumnChart` / `ReportHorizontalBarChart` stubs that compile

- [ ] **Step 1: Pin Vico in version catalog**

```toml
# [versions]
vico = "3.2.3"

# [libraries]
vico-compose = { group = "com.patrykandpatrick.vico", name = "compose", version.ref = "vico" }
vico-compose-m3 = { group = "com.patrykandpatrick.vico", name = "compose-m3", version.ref = "vico" }
```

```kotlin
// composeApp commonMain.dependencies
implementation(libs.vico.compose)
implementation(libs.vico.compose.m3)
```

- [ ] **Step 2: Resolve / compile check**

```bash
./gradlew :composeApp:compileKotlinDesktop --quiet
```

Expected: SUCCESS.  
If Compose BOM / CMP `1.7.3` conflicts with Vico’s Compose requirement: bump `composeMultiplatform` in `libs.versions.toml` to the minimum version that resolves (document bump in commit message). Do **not** introduce a second chart library.

- [ ] **Step 3: Minimal chart wrappers**

Create `ReportCharts.kt` using current Vico 3.x API from https://guide.vico.patrykandpatrick.com/getting-started (CartesianChartHost + ColumnCartesianLayer / HorizontalAxis). Keep wrappers thin:

```kotlin
data class ReportChartPoint(val label: String, val value: Float)

@Composable
fun ReportColumnChart(points: List<ReportChartPoint>, modifier: Modifier = Modifier) { /* Vico column */ }

@Composable
fun ReportHorizontalBarChart(points: List<ReportChartPoint>, modifier: Modifier = Modifier) { /* Vico horizontal */ }
```

If horizontal layer API differs, use column chart with category labels for ranking as interim — but prefer horizontal bars for ranking per spec.

- [ ] **Step 4: Commit**

```bash
git add gradle/libs.versions.toml composeApp/build.gradle.kts \
  composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCharts.kt
git commit -m "$(cat <<'EOF'
feat(reports): add Vico chart dependency and wrappers

EOF
)"
```

---

### Task 4: Chart series mappers + wire charts into KPI details (client-app)

**Repo:** `client-app`  
**Branch:** `feat/reports-catalog-charts`

**Files:**
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportChartModels.kt`
- Modify: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ReportCatalogTest.kt` (or new `ReportChartModelsTest.kt`)
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportsScreen.kt`

**Interfaces:**
- Produces:
  - `fun kpiSummaryChartPoints(kpis: ManagerKpis): List<ReportChartPoint>`
  - `fun plannedVsEmergencyChartPoints(kpis: ManagerKpis): List<ReportChartPoint>`
  - `fun pprComplianceChartPoints(kpis: ManagerKpis): List<ReportChartPoint>`
  - `fun backlogChartPoints(kpis: ManagerKpis): List<ReportChartPoint>`
  - `fun downtimeRankingChartPoints(kpis: ManagerKpis, assets: List<AssetDto>): List<ReportChartPoint>` — labels via `displayName()`, never raw id

- [ ] **Step 1: Failing mapper tests**

```kotlin
@Test
fun plannedVsEmergencySeriesUsesCounts() {
    val points = plannedVsEmergencyChartPoints(sampleKpis(plannedCount = 10, emergencyCount = 4))
    assertEquals(listOf("Плановые", "Аварийные"), points.map { it.label })
    assertEquals(listOf(10f, 4f), points.map { it.value })
}

@Test
fun downtimeRankingUsesAssetNames() {
    val points =
        downtimeRankingChartPoints(
            kpis = sampleKpis(ranking = listOf(ManagerKpiDowntimeRow("a1", 18.5, 1))),
            assets = listOf(AssetDto(id = "a1", orgId = "o", siteId = "s", name = "Насос №1", status = "active", source = "manual")),
        )
    assertEquals("Насос №1", points.single().label)
    assertEquals(false, points.single().label.contains("a1"))
}
```

- [ ] **Step 2: Run — FAIL, then implement mappers**

Russian labels:

| Series | Labels |
|--------|--------|
| summary | `MTTR`, `MTBF`, `Готовность` — values: mttrHours, mtbfHours, availabilityPercent (document units in subtitle under chart) |
| planned | `Плановые`, `Аварийные` — use **counts** for bars; show hours as text rows via `formatHours` |
| ppr | `Вовремя`, `С опозданием`, `Просрочено`, `В срок` |
| backlog | `<7 дн`, `7–30 дн`, `>30 дн` (+ overdue as separate `KpiValue`, not necessarily a 4th bar) |
| ranking | top rows already sorted by API — `value = downtimeHours.toFloat()` |

- [ ] **Step 3: Wire each `ReportId` detail body**

For each KPI report:
1. Period label: `«Период: с {from} по {to}»` or «Последние N дней»
2. Metric rows with `formatHours` / `formatPercent` / integers
3. Chart above or below metrics via `ReportColumnChart` / `ReportHorizontalBarChart`
4. Empty: if all relevant counts/hours are zero and ranking empty → empty copy, **do not** render empty axes

`KpiSummary`: 3 cards + column chart of the three values.  
`DowntimeRanking`: horizontal bars + text rows with `formatHours` and open interval counts.

Replace private `hours()` in ReportsScreen with `formatHours`.

- [ ] **Step 4: Tests PASS**

```bash
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.ReportCatalogTest' --tests 'pro.masterdoc.client.ui.screens.ReportChartModelsTest' --tests 'pro.masterdoc.client.ui.screens.ManagerKpiFormattersTest' --tests 'pro.masterdoc.client.ui.screens.ReportsScreenTest'
```

- [ ] **Step 5: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(reports): Vico charts on manager KPI report details

EOF
)"
```

---

### Task 5: Seed manager reports data (dashboard-service)

**Repo:** `dashboard-service`  
**Branch:** `feat/reports-catalog-charts`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/dashboard/ManagerReportsSeed.kt`
- Create: `src/test/kotlin/pro/masterdoc/dashboard/ManagerReportsSeedTest.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt`
- Modify: `src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt` (HTTP smoke for seed route)

**Interfaces:**
- Produces:
  - `@Serializable data class SeedManagerReportsRequest(val siteId: String, val assetIds: List<String>)`
  - `@Serializable data class SeedManagerReportsResponse(val deleted: Int, val created: Int, val orgId: String)`
  - `fun seedManagerReports(store: WorkOrderStore, orgId: String, siteId: String, assetIds: List<String>, now: Instant): SeedManagerReportsResponse`
  - Route: `POST /internal/orgs/{orgId}/seed-manager-reports` with JSON body `{ "siteId", "assetIds": [...] }` — requires ≥1 assetId

Seed algorithm (relative to `now`, use `store.clearOrg` then `create` + `update` with controlled timestamps):

1. Require `assetIds.isNotEmpty()`; cycle assets for variety (min 1).
2. `deleted = store.clearOrg(orgId)`.
3. Create closed emergencies with start/close pairs spanning last 14 days (varied durations) on ≥2 assets; ensure ≥2 closed emergencies on `assetIds[0]` for MTBF.
4. Create PPR WOs with `dueAt` in last 30 days: one closed on-time, one closed late, one open overdue (`dueAt` < today), one open pending (`dueAt` ≥ today).
5. Create open backlog: `createdAt` ages via `now` offsets: −3d, −15d, −45d; one with `dueAt` in the past for overdue.
6. Create one open emergency `in_progress` with `startedAt` recent for Gantt.
7. Return `created` count.

Use Russian titles without ids (`«Авария компрессора»`, `«ППР насос»`, …).

- [ ] **Step 1: Failing unit test**

```kotlin
@Test
fun seedProducesNonZeroManagerKpiSamples() {
    val store = WorkOrderStore()
    val now = Instant.parse("2026-08-02T12:00:00Z")
    val result =
        seedManagerReports(
            store = store,
            orgId = "smoke",
            siteId = "ceh-1",
            assetIds = listOf("asset-a", "asset-b"),
            now = now,
        )
    assertTrue(result.created >= 8)
    val kpis =
        store.managerKpis(
            orgId = "smoke",
            from = Instant.parse("2026-07-03T00:00:00Z"),
            to = Instant.parse("2026-08-02T23:59:59.999999999Z"),
            now = now,
        )
    assertTrue(kpis.mttrSampleSize > 0)
    assertTrue(kpis.emergencyCount > 0)
    assertTrue(kpis.plannedCount > 0)
    assertTrue(kpis.pprOnTime + kpis.pprLate + kpis.pprOpenOverdue + kpis.pprOpenPending > 0)
    assertTrue(kpis.backlogUnder7d + kpis.backlog7to30d + kpis.backlogOver30d > 0)
    assertTrue(kpis.downtimeRanking.isNotEmpty())
}
```

Adapt `store.managerKpis` call to the actual signature in `WorkOrderStore` / `ManagerKpis.kt`.

- [ ] **Step 2: Implement seed + route**

Wire in `Application.kt` next to existing `delete("/internal/orgs/{orgId}/work-orders")`:

```kotlin
post("/internal/orgs/{orgId}/seed-manager-reports") {
    val orgId = call.parameters["orgId"]?.takeIf { it.isNotBlank() }
        ?: throw IllegalArgumentException("orgId required")
    val body = call.receive<SeedManagerReportsRequest>()
    call.respond(
        seedManagerReports(
            store = workOrderStore,
            orgId = orgId,
            siteId = body.siteId,
            assetIds = body.assetIds,
            now = clock.instant(),
        ),
    )
}
```

- [ ] **Step 3: Route test** — POST seed → GET manager-kpis → assert emergencyCount > 0

- [ ] **Step 4: Run**

```bash
./gradlew test --tests '*ManagerReportsSeed*' --tests '*WorkOrderRoutes*'
```

Expected: PASS (or narrow to new tests if suite is huge — still run seed + one routes test).

- [ ] **Step 5: Commit**

```bash
git commit -m "$(cat <<'EOF'
feat(reports): internal seed for manager KPI demo work orders

EOF
)"
```

---

### Task 6: Ops workflow to seed Smoke (dashboard-service)

**Repo:** `dashboard-service`  
**Branch:** `feat/reports-catalog-charts`

**Files:**
- Create: `.github/workflows/seed-manager-reports.yml`

Pattern after `catalog-service/.github/workflows/seed-demo-assets.yml`:

- `workflow_dispatch` input `org_id` default `383177088934346755` (Fixaverse Smoke)
- SSH to API VPS
- `GET http://127.0.0.1:<catalog-port>/sites` + `/assets` with `X-Org-Id`
- Pick first site id; collect asset ids (fail if none — message to run seed-demo-assets first)
- `POST http://127.0.0.1:<dashboard-port>/internal/orgs/${ORG_ID}/seed-manager-reports` with JSON body
- Print created/deleted and a quick `GET /reports/manager-kpis?from&to` summary

Use the same localhost ports as other dashboard/catalog ops scripts in this monorepo (catalog `8091` from seed-demo-assets; find dashboard port from `deploy/docker-compose` or existing workflows — copy exactly).

- [ ] **Step 1: Add workflow file**
- [ ] **Step 2: Commit**

```bash
git commit -m "$(cat <<'EOF'
ci: workflow to seed manager reports for an org

EOF
)"
```

---

### Task 7: Ship — push, seed, CI, smoke (controller)

**Repos:** `client-app`, `dashboard-service`  
**Skill:** `/smoke-test` after green deploy

- [ ] **Step 1: Push both repos** (`feat/reports-catalog-charts` or merge to default per team practice). If product rule requires default-branch deploy, open/merge PRs or push to `master`/`main` as usual for these services.

- [ ] **Step 2: `gh run watch`** until test+deploy success on both.

- [ ] **Step 3: Run seed workflow**

```bash
cd dashboard-service
gh workflow run seed-manager-reports.yml -f org_id=383177088934346755
gh run watch
```

- [ ] **Step 4: `/smoke-test`** — org Fixaverse Smoke, user with `reports`:
  1. Nav «Отчёты» → list of 6 titles
  2. Open «Сводка KPI» → rounded numbers + chart; Back
  3. Open «Плановые vs аварийные» → chart
  4. Open «Рейтинг простоев» → names not ids + bars
  5. Open «Простои оборудования» → Gantt with intervals
  6. Screenshot each key screen; write `.superpowers/sdd/smoke-reports-catalog-charts.md`

- [ ] **Step 5: Report PASS/FAIL to user**

---

## Self-review (plan vs spec)

| Spec requirement | Task |
|------------------|------|
| Catalog of 6 reports, master→detail + Back | Task 2 |
| Period on detail only | Task 2 |
| 1 decimal hours/%, н/д | Task 1 |
| Vico charts on 5 KPI reports | Tasks 3–4 |
| Gantt unchanged as separate report | Task 2 (+ Task 4 no Vico) |
| Seed Smoke via API | Tasks 5–6 |
| No KPI formula/API shape change | Task 5 (seed only) |
| Names not ids | Task 4 mappers + seed titles |
| CI + smoke | Task 7 |

No TBD placeholders. Types aligned: `ReportId`, `ReportChartPoint`, `SeedManagerReportsRequest`.
