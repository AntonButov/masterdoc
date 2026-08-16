# Equipment work-orders detailed report — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a catalog report «Детальный отчёт» where a manager picks one asset and sees every work order whose lifetime overlaps 7/30/90 days, then opens the existing work-order card read-only.

**Architecture:** `dashboard-service` adds `GET /reports/equipment-work-orders?assetId&from&to` (same `WorkOrder` JSON as `GET /work-orders`). Gateway already proxies `/reports`. A dedicated `GET /work-orders/{id}` gate adds `reports` only for that path so the card can load; list/board/PATCH stay closed. `client-app` adds catalog item, repository call, picker + list, and `WorkOrderDetailScreen(readOnly = true)`.

**Tech Stack:** Kotlin/Ktor (`dashboard-service`, `api-gateway-service`), Compose Multiplatform (`client-app`), kotlinx.serialization, existing `WorkOrderDto` / `AppStatusChip`.

**Spec:** `docs/superpowers/specs/2026-08-16-equipment-work-orders-report-design.md` (this repo)

## Global Constraints

- UI copy: never show raw UUIDs / user ids; asset `displayName()` / title via `formatWorkOrderDisplayTitle`; fallback «Оборудование» / «Заявка».
- Do not change existing report formulas or JSON shapes.
- Period overlap: `start = createdAt`; `end = closedAt ?: to`; include if `start <= to && end >= from`; drop unparseable `createdAt`.
- `assetId` required → HTTP 400; empty list → 200 `[]`.
- Sort: `createdAt` desc, then `id` desc.
- Gateway: `reports` may `GET /work-orders/{id}` only; not list, not `/board`, not PATCH/POST.
- Card: existing `WorkOrderDetailScreen` + `readOnly = true`; no new detail screen.
- Focused Gradle tests only; no local Wasm / full desktop distribution builds.
- Commit in the repo you change; push that repo after the task (CI builds on GitHub).
- Russian UI labels. Period selector 7/30/90 shared with other reports.

---

## File structure

| File | Responsibility |
|---|---|
| `dashboard-service/.../EquipmentWorkOrdersReport.kt` | Pure `selectEquipmentWorkOrders(orders, assetId, from, to)` |
| `dashboard-service/.../EquipmentWorkOrdersReportTest.kt` | Overlap / sort / other-asset unit tests |
| `dashboard-service/.../WorkOrderStore.kt` | `equipmentWorkOrders(orgId, assetId, from, to)` → list + select |
| `dashboard-service/.../Application.kt` | `GET /reports/equipment-work-orders` |
| `dashboard-service/.../WorkOrderRoutesTest.kt` | HTTP 400 without assetId; overlap via store fixtures |
| `api-gateway-service/.../EquipmentRoutes.kt` | `GET /work-orders/{id}` before prefix; `reports` unless `id == "board"` |
| `api-gateway-service/.../WorkOrderProxyRoutesTest.kt` | reports: GET by id allowed; list/board/PATCH forbidden |
| `client-app/.../ReportCatalog.kt` | `ReportId.EquipmentWorkOrders` + catalog row |
| `client-app/.../ReportCatalogTest.kt` | 11 items, last is detailed report |
| `client-app/auth/.../WorkOrderModels.kt` | `WorkOrdersRepository.equipmentWorkOrders` |
| `client-app/auth/.../WorkOrdersRepositoryTest.kt` | URL + decode list |
| `client-app/.../EquipmentWorkOrdersReportUi.kt` | Picker + list + read-only card host |
| `client-app/.../ReportsScreen.kt` | Branch to new UI; `PeriodSelector` → `internal` |
| `client-app/.../MainShellContent.kt` | Pass optional attachments/comments into `ReportsScreen` |

---

### Task 1: dashboard overlap + report route

**Repo:** `dashboard-service` (work in that git root)

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/dashboard/EquipmentWorkOrdersReport.kt`
- Create: `src/test/kotlin/pro/masterdoc/dashboard/EquipmentWorkOrdersReportTest.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt` (after `failureFrequency`)
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt` (after `GET /reports/failure-frequency`)
- Modify: `src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt`

**Interfaces:**
- Consumes: `WorkOrder`, `WorkOrderStore.list(orgId)`, `parseReportBoundary` (route only)
- Produces: `fun selectEquipmentWorkOrders(orders: List<WorkOrder>, assetId: String, from: Instant, to: Instant): List<WorkOrder>`
- Produces: `WorkOrderStore.equipmentWorkOrders(orgId: String, assetId: String, from: Instant, to: Instant): List<WorkOrder>`
- Produces: `GET /reports/equipment-work-orders?assetId=&from=&to=` → JSON array of `WorkOrder`

- [ ] **Step 1: Write the failing unit test**

Create `src/test/kotlin/pro/masterdoc/dashboard/EquipmentWorkOrdersReportTest.kt`:

```kotlin
package pro.masterdoc.dashboard

import java.time.Instant
import kotlin.test.Test
import kotlin.test.assertEquals

class EquipmentWorkOrdersReportTest {
    private val from = Instant.parse("2026-07-01T00:00:00Z")
    private val to = Instant.parse("2026-07-31T23:59:59Z")

    @Test
    fun includesOpenCreatedBeforePeriodAndClosedInsideAndExcludesOutside() {
        val openOld = workOrder("open-old", assetId = "pump", createdAt = "2026-06-01T00:00:00Z", closedAt = null)
        val closedInside =
            workOrder(
                "closed-inside",
                assetId = "pump",
                createdAt = "2026-07-10T00:00:00Z",
                closedAt = "2026-07-15T00:00:00Z",
            )
        val closedBefore =
            workOrder(
                "closed-before",
                assetId = "pump",
                createdAt = "2026-06-01T00:00:00Z",
                closedAt = "2026-06-15T00:00:00Z",
            )
        val createdAfter =
            workOrder("created-after", assetId = "pump", createdAt = "2026-08-01T00:00:00Z", closedAt = null)
        val otherAsset =
            workOrder("other", assetId = "fan", createdAt = "2026-07-10T00:00:00Z", closedAt = null)

        val result =
            selectEquipmentWorkOrders(
                orders = listOf(openOld, closedInside, closedBefore, createdAfter, otherAsset),
                assetId = "pump",
                from = from,
                to = to,
            )

        assertEquals(listOf("closed-inside", "open-old"), result.map { it.id })
    }

    @Test
    fun dropsUnparseableCreatedAt() {
        val bad = workOrder("bad", assetId = "pump", createdAt = "not-a-date", closedAt = null)
        assertEquals(
            emptyList(),
            selectEquipmentWorkOrders(listOf(bad), "pump", from, to).map { it.id },
        )
    }

    @Test
    fun sortsByCreatedAtDescThenIdDesc() {
        val a = workOrder("a", assetId = "pump", createdAt = "2026-07-10T00:00:00Z")
        val b = workOrder("b", assetId = "pump", createdAt = "2026-07-20T00:00:00Z")
        val c = workOrder("c", assetId = "pump", createdAt = "2026-07-20T00:00:00Z")
        assertEquals(
            listOf("c", "b", "a"),
            selectEquipmentWorkOrders(listOf(a, b, c), "pump", from, to).map { it.id },
        )
    }

    private fun workOrder(
        id: String,
        assetId: String,
        createdAt: String,
        closedAt: String? = null,
    ) = WorkOrder(
        id = id,
        orgId = "org-1",
        type = WorkOrderType.emergency,
        status = if (closedAt == null) WorkOrderStatus.new else WorkOrderStatus.closed,
        title = id,
        assetId = assetId,
        siteId = "site",
        dueAt = "2026-07-15",
        durationHours = 8,
        source = WorkOrderSource.api,
        createdAt = createdAt,
        updatedAt = createdAt,
        closedAt = closedAt,
    )
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
./gradlew test --tests pro.masterdoc.dashboard.EquipmentWorkOrdersReportTest --no-daemon
```

Expected: FAIL compiling (`selectEquipmentWorkOrders` unresolved).

- [ ] **Step 3: Write minimal selection implementation**

Create `src/main/kotlin/pro/masterdoc/dashboard/EquipmentWorkOrdersReport.kt`:

```kotlin
package pro.masterdoc.dashboard

import java.time.Instant

fun selectEquipmentWorkOrders(
    orders: List<WorkOrder>,
    assetId: String,
    from: Instant,
    to: Instant,
): List<WorkOrder> {
    require(assetId.isNotBlank()) { "assetId required" }
    require(!to.isBefore(from)) { "to must be on or after from" }
    return orders
        .filter { it.assetId == assetId }
        .filter { overlapsPeriod(it, from, to) }
        .sortedWith(
            compareByDescending<WorkOrder> { it.createdAt.toInstantOrNull() ?: Instant.EPOCH }
                .thenByDescending { it.id },
        )
}

private fun overlapsPeriod(order: WorkOrder, from: Instant, to: Instant): Boolean {
    val start = order.createdAt.toInstantOrNull() ?: return false
    val end = order.closedAt?.toInstantOrNull() ?: to
    return !start.isAfter(to) && !end.isBefore(from)
}

private fun String.toInstantOrNull(): Instant? = runCatching { Instant.parse(this) }.getOrNull()
```

- [ ] **Step 4: Re-run unit tests**

```bash
./gradlew test --tests pro.masterdoc.dashboard.EquipmentWorkOrdersReportTest --no-daemon
```

Expected: PASS.

- [ ] **Step 5: Write failing route test**

In `WorkOrderRoutesTest.kt` add (same `withApplication` / `FakeMaintenanceMapGateway` / `clock` as `managerKpisRouteUsesOrgHeaderAndDateBoundaries`):

```kotlin
@Test
fun equipmentWorkOrdersRequiresAssetIdAndFiltersOverlap() = withApplication {
    val maps = FakeMaintenanceMapGateway()
    val orders = WorkOrderStore(dataSource)
    application {
        module(maps, orders, AllowAllAssetLookup, PprScheduler(maps, orders, AllowAllAssetLookup, clock), clock)
    }
    val included =
        orders.create(
            "org-1",
            CreateWorkOrderRequest(WorkOrderType.emergency, "Утечка", "pump-1", "site-1", "2026-07-22"),
            now = Instant.parse("2026-07-10T00:00:00Z"),
        )
    orders.create(
        "org-1",
        CreateWorkOrderRequest(WorkOrderType.emergency, "Чужой станок", "other", "site-1", "2026-07-22"),
        now = Instant.parse("2026-07-10T00:00:00Z"),
    )

    val missing =
        client.get("/reports/equipment-work-orders?from=2026-07-01&to=2026-07-31") {
            header("X-Org-Id", "org-1")
        }
    assertEquals(HttpStatusCode.BadRequest, missing.status)

    val response =
        client.get("/reports/equipment-work-orders?assetId=pump-1&from=2026-07-01&to=2026-07-31") {
            header("X-Org-Id", "org-1")
        }
    assertEquals(HttpStatusCode.OK, response.status)
    val items = json.parseToJsonElement(response.bodyAsText()).jsonArray
    assertEquals(1, items.size)
    assertEquals(included.id, items.single().jsonObject["id"]!!.jsonPrimitive.content)
    assertEquals("Утечка", items.single().jsonObject["title"]!!.jsonPrimitive.content)
}
```

- [ ] **Step 6: Run route test to verify it fails**

```bash
./gradlew test --tests pro.masterdoc.dashboard.WorkOrderRoutesTest.equipmentWorkOrdersRequiresAssetIdAndFiltersOverlap --no-daemon
```

Expected: FAIL (404 or unhandled path).

- [ ] **Step 7: Wire store + route**

In `WorkOrderStore.kt` after `failureFrequency`:

```kotlin
fun equipmentWorkOrders(
    orgId: String,
    assetId: String,
    from: Instant,
    to: Instant,
): List<WorkOrder> = selectEquipmentWorkOrders(list(orgId), assetId, from, to)
```

In `Application.kt` after `GET /reports/failure-frequency`:

```kotlin
get("/reports/equipment-work-orders") {
    val assetId =
        call.request.queryParameters["assetId"]?.takeIf { it.isNotBlank() }
            ?: throw IllegalArgumentException("assetId required")
    val from = parseReportBoundary(call.request.queryParameters["from"], isEnd = false)
    val to = parseReportBoundary(call.request.queryParameters["to"], isEnd = true)
    call.respond(
        workOrderStore.equipmentWorkOrders(
            orgId = call.orgId(),
            assetId = assetId,
            from = from,
            to = to,
        ),
    )
}
```

- [ ] **Step 8: Run dashboard tests for this feature**

```bash
./gradlew test --tests pro.masterdoc.dashboard.EquipmentWorkOrdersReportTest --tests pro.masterdoc.dashboard.WorkOrderRoutesTest.equipmentWorkOrdersRequiresAssetIdAndFiltersOverlap --no-daemon
```

Expected: PASS.

- [ ] **Step 9: Commit and push `dashboard-service`**

```bash
git add src/main/kotlin/pro/masterdoc/dashboard/EquipmentWorkOrdersReport.kt \
  src/test/kotlin/pro/masterdoc/dashboard/EquipmentWorkOrdersReportTest.kt \
  src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt \
  src/main/kotlin/pro/masterdoc/dashboard/Application.kt \
  src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt
git commit -m "$(cat <<'EOF'
feat: report work orders by equipment and period

EOF
)"
git push
```

Watch the triggered GitHub Action and only then treat the task as done.

---

### Task 2: gateway GET work-order by id for reports

**Repo:** `api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt`
- Modify: `src/test/kotlin/pro/masterdoc/gateway/WorkOrderProxyRoutesTest.kt`

**Interfaces:**
- Consumes: existing `proxyPrefix("/work-orders")` (unchanged write/read lists), `forward`, `requireAnyFeature`
- Produces: `GET /work-orders/{id}` allowed for `board|engineer|tickets|reports` when `id != "board"`; `GET /work-orders/board` still `board|engineer|tickets` only

- [ ] **Step 1: Write failing tests**

Append to `WorkOrderProxyRoutesTest.kt`:

```kotlin
@Test
fun `GET work-order by id allowed with reports feature`() = testApplication {
    application {
        module(
            GatewayConfig.testDefaults().copy(dashboardServiceBaseUrl = "http://127.0.0.1:1"),
            GatewayDeps(
                featureClient = featureClientWith("reports"),
                backendClient = BackendProxyClient { _, _, _, _ -> error("unused") },
                tokenValidator = TokenValidator.accepting(),
            ),
        )
    }
    val response =
        client.get("/work-orders/wo-1") {
            header(HttpHeaders.Authorization, "Bearer good")
        }
    assertTrue(response.status == HttpStatusCode.BadGateway || response.status == HttpStatusCode.OK)
    assertTrue(response.status != HttpStatusCode.Forbidden)
}

@Test
fun `GET work-orders list forbidden with only reports feature`() = testApplication {
    application {
        module(
            GatewayConfig.testDefaults(),
            GatewayDeps(
                featureClient = featureClientWith("reports"),
                backendClient = BackendProxyClient { _, _, _, _ -> error("unused") },
                tokenValidator = TokenValidator.accepting(),
            ),
        )
    }
    val list =
        client.get("/work-orders") {
            header(HttpHeaders.Authorization, "Bearer good")
        }
    assertEquals(HttpStatusCode.Forbidden, list.status)
    val board =
        client.get("/work-orders/board") {
            header(HttpHeaders.Authorization, "Bearer good")
        }
    assertEquals(HttpStatusCode.Forbidden, board.status)
}

@Test
fun `PATCH work-orders forbidden with only reports feature`() = testApplication {
    application {
        module(
            GatewayConfig.testDefaults(),
            GatewayDeps(
                featureClient = featureClientWith("reports"),
                backendClient = BackendProxyClient { _, _, _, _ -> error("unused") },
                tokenValidator = TokenValidator.accepting(),
            ),
        )
    }
    val response =
        client.patch("/work-orders/wo-1") {
            header(HttpHeaders.Authorization, "Bearer good")
            contentType(ContentType.Application.Json)
            setBody("""{"status":"in_progress"}""")
        }
    assertEquals(HttpStatusCode.Forbidden, response.status)
}
```

- [ ] **Step 2: Run tests to verify GET-by-id fails**

```bash
./gradlew test --tests pro.masterdoc.gateway.WorkOrderProxyRoutesTest --no-daemon
```

Expected: `GET work-order by id allowed with reports feature` FAIL (Forbidden). List/PATCH tests may already pass.

- [ ] **Step 3: Register dedicated GET before the work-orders prefix**

In `EquipmentRoutes.kt`:
1. Add `import io.ktor.server.routing.get`
2. Inside `installEquipmentRoutes` `routing { }`, **immediately before** `proxyPrefix("/work-orders", ...)`:

```kotlin
get("/work-orders/{id}") {
    val id = call.parameters["id"].orEmpty()
    val readGate =
        if (id == "board") {
            listOf("board", "engineer", "tickets")
        } else {
            listOf("board", "engineer", "tickets", "reports")
        }
    if (!call.requireAnyFeature(deps, readGate)) return@get
    forward(
        client,
        config.dashboardServiceBaseUrl,
        call.request.uri,
        call,
        deps,
        scopeFilterHint = true,
    )
}
```

Do **not** add `reports` to `proxyPrefix("/work-orders")` `readFeatures`. Leave writeFeatures unchanged.

- [ ] **Step 4: Re-run gateway work-order proxy tests**

```bash
./gradlew test --tests pro.masterdoc.gateway.WorkOrderProxyRoutesTest --no-daemon
```

Expected: PASS (including existing board/tickets/charts cases).

- [ ] **Step 5: Commit and push `api-gateway-service`**

```bash
git add src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt \
  src/test/kotlin/pro/masterdoc/gateway/WorkOrderProxyRoutesTest.kt
git commit -m "$(cat <<'EOF'
feat: allow reports feature to read a work order by id

EOF
)"
git push
```

Watch CI/deploy.

---

### Task 3: client catalog + repository

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCatalog.kt`
- Modify: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ReportCatalogTest.kt`
- Modify: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt`
- Modify: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt`

**Interfaces:**
- Produces: `ReportId.EquipmentWorkOrders`
- Produces: catalog title `"Детальный отчёт"`, subtitle `"Заявки по единице оборудования"`
- Produces: `suspend fun WorkOrdersRepository.equipmentWorkOrders(assetId: String, from: String, to: String): List<WorkOrderDto>`
- URL: `{gateway}/reports/equipment-work-orders?assetId={assetId}&from={from}&to={to}`

- [ ] **Step 1: Write failing catalog test**

Replace `catalogHasTenReportsInStableOrder` with eleven items. Last id `EquipmentWorkOrders`, last title `"Детальный отчёт"`:

```kotlin
@Test
fun catalogHasElevenReportsInStableOrder() {
    val items = reportCatalogItems()
    assertEquals(11, items.size)
    assertEquals(
        listOf(
            ReportId.KpiSummary,
            ReportId.PlannedVsEmergency,
            ReportId.PprCompliance,
            ReportId.Backlog,
            ReportId.DowntimeRanking,
            ReportId.EquipmentDowntime,
            ReportId.KpiTrends,
            ReportId.ReactiveCompletion,
            ReportId.EngineerWorkload,
            ReportId.FailureFrequency,
            ReportId.EquipmentWorkOrders,
        ),
        items.map { it.id },
    )
    assertEquals("Сводка KPI", items.first().title)
    assertEquals("Детальный отчёт", items.last().title)
    items.forEach { item ->
        assertTrue(item.description.isNotBlank(), "description missing for ${item.id}")
    }
}
```

Delete or stop calling the old ten-item test.

- [ ] **Step 2: Run catalog test to verify it fails**

```bash
./gradlew :composeApp:desktopTest --tests pro.masterdoc.client.ui.screens.ReportCatalogTest --no-daemon
```

Expected: FAIL (`EquipmentWorkOrders` unresolved and/or size 10).

- [ ] **Step 3: Add catalog item**

In `ReportCatalog.kt` add enum value `EquipmentWorkOrders` after `FailureFrequency`. Append:

```kotlin
ReportCatalogItem(
    id = ReportId.EquipmentWorkOrders,
    title = "Детальный отчёт",
    subtitle = "Заявки по единице оборудования",
    description =
        "По выбранному оборудованию показывает все заявки, которые пересекают выбранный период: " +
            "открытые и закрытые. Нажмите строку, чтобы открыть карточку.",
),
```

- [ ] **Step 4: Re-run catalog test**

Same command as Step 2. Expected: PASS.

- [ ] **Step 5: Write failing repository test**

In `WorkOrdersRepositoryTest.kt`:

```kotlin
@Test
fun equipmentWorkOrdersUsesAssetAndPeriod() =
    runBlocking {
        val tokens = InMemoryTokenStore()
        tokens.write(AuthTokens(accessToken = "at"))
        val http =
            RecordingGatewayHttpClient { method, url, _, _ ->
                assertEquals("GET", method)
                assertEquals(
                    "https://api.test/reports/equipment-work-orders?assetId=pump-1&from=2026-07-01&to=2026-07-31",
                    url,
                )
                GatewayHttpResponse(
                    200,
                    """[{"id":"wo-1","orgId":"o","type":"emergency","status":"new","title":"Утечка","assetId":"pump-1","siteId":"s","dueAt":"2026-07-22","source":"api","createdAt":"2026-07-10T00:00:00Z","updatedAt":"t"}]""",
                )
            }
        val items =
            WorkOrdersRepository(config = config, http = http, tokenStore = tokens)
                .equipmentWorkOrders(assetId = "pump-1", from = "2026-07-01", to = "2026-07-31")
        assertEquals("wo-1", items.single().id)
        assertEquals("Утечка", items.single().title)
    }
```

- [ ] **Step 6: Run repository test to verify it fails**

```bash
./gradlew :auth:jvmTest --tests pro.masterdoc.client.auth.WorkOrdersRepositoryTest.equipmentWorkOrdersUsesAssetAndPeriod --no-daemon
```

Expected: FAIL (`equipmentWorkOrders` unresolved).

- [ ] **Step 7: Implement repository method**

In `WorkOrdersRepository` (next to other report getters):

```kotlin
suspend fun equipmentWorkOrders(assetId: String, from: String, to: String): List<WorkOrderDto> {
    val response =
        http.get(
            url = "${base()}/reports/equipment-work-orders?assetId=$assetId&from=$from&to=$to",
            headers = mapOf("Authorization" to "Bearer ${bearer()}"),
        )
    if (!response.isSuccessful) {
        throw GatewayHttpException(
            response.status,
            response.body.ifBlank { "equipment work orders report failed" },
        )
    }
    return json.decodeFromString(response.body)
}
```

- [ ] **Step 8: Re-run repository test**

Same command as Step 6. Expected: PASS.

- [ ] **Step 9: Commit and push `client-app`**

```bash
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportCatalog.kt \
  composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/ReportCatalogTest.kt \
  auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt \
  auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt
git commit -m "$(cat <<'EOF'
feat: catalog and API client for equipment work-order report

EOF
)"
git push
```

Watch CI. Catalog-only is shippable (opens a stub/empty detail until Task 4); if `ReportDetailScreen` falls through to `MarketLeaderReportSection` for the new id, Task 4 must land before calling the screen done — implement Task 4 immediately after this push, do not leave the new catalog row showing «Нет данных».

---

### Task 4: client detail UI + read-only card

**Repo:** `client-app`

**Files:**
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/EquipmentWorkOrdersReportUi.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportsScreen.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/shell/MainShellContent.kt`

**Interfaces:**
- Consumes: `WorkOrdersRepository.equipmentWorkOrders`, `EquipmentRepository.listAssets`, `ReportId.EquipmentWorkOrders`, `PeriodSelector`, `formatWorkOrderDisplayTitle`, `AssetDto.displayName()`, `workOrderTypeLabelRu`, `workOrderStatusLabelRu`
- Produces: `EquipmentWorkOrdersReportScreen(...)` used from `ReportsScreen` when `selectedReport == EquipmentWorkOrders`

- [ ] **Step 1: Expose period selector**

In `ReportsScreen.kt` change `private fun PeriodSelector` and `private fun ReportHelpFooter` to `internal fun` so the new file in the same package can call them.

- [ ] **Step 2: Add dedicated screen file**

Create `EquipmentWorkOrdersReportUi.kt`:

```kotlin
package pro.masterdoc.client.ui.screens

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.DropdownMenuItem
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.ExposedDropdownMenuBox
import androidx.compose.material3.ExposedDropdownMenuDefaults
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import kotlinx.coroutines.CancellationException
import pro.masterdoc.client.auth.AssetDto
import pro.masterdoc.client.auth.AttachmentsRepository
import pro.masterdoc.client.auth.CommentsRepository
import pro.masterdoc.client.auth.EquipmentRepository
import pro.masterdoc.client.auth.GatewayHttpException
import pro.masterdoc.client.auth.IsoDates
import pro.masterdoc.client.auth.WorkOrderDto
import pro.masterdoc.client.auth.WorkOrdersRepository
import pro.masterdoc.client.auth.workOrderStatusLabelRu
import pro.masterdoc.client.auth.workOrderTypeLabelRu
import pro.masterdoc.client.designsystem.components.AppScaffold
import pro.masterdoc.client.designsystem.components.AppStatusChip
import pro.masterdoc.client.designsystem.components.AppStatusChipTone
import pro.masterdoc.client.designsystem.components.AppText
import pro.masterdoc.client.designsystem.components.AppTextStyle
import pro.masterdoc.client.designsystem.theme.ClientSpacing
import pro.masterdoc.client.platform.localEpochDay

@OptIn(ExperimentalMaterial3Api::class)
@Composable
internal fun EquipmentWorkOrdersReportScreen(
    reportsRepository: WorkOrdersRepository,
    equipmentRepository: EquipmentRepository,
    attachmentsRepository: AttachmentsRepository?,
    commentsRepository: CommentsRepository?,
    onBack: () -> Unit,
    modifier: Modifier = Modifier,
) {
    val catalogItem = reportCatalogItems().first { it.id == ReportId.EquipmentWorkOrders }
    var days by remember { mutableStateOf(30) }
    var assets by remember { mutableStateOf<List<AssetDto>>(emptyList()) }
    var selectedAssetId by remember { mutableStateOf<String?>(null) }
    var menuExpanded by remember { mutableStateOf(false) }
    var orders by remember { mutableStateOf<List<WorkOrderDto>>(emptyList()) }
    var assetsLoading by remember { mutableStateOf(true) }
    var assetsError by remember { mutableStateOf<String?>(null) }
    var ordersLoading by remember { mutableStateOf(false) }
    var ordersError by remember { mutableStateOf<String?>(null) }
    var selectedOrderId by remember { mutableStateOf<String?>(null) }
    val today = localEpochDay()
    val fromDay = today - days + 1
    val selectedAsset = assets.firstOrNull { it.id == selectedAssetId }

    LaunchedEffect(equipmentRepository) {
        assetsLoading = true
        assetsError = null
        try {
            assets = equipmentRepository.listAssets().items.sortedBy { it.displayName().lowercase() }
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            assetsError = e.message ?: "Не удалось загрузить оборудование"
            assets = emptyList()
        } finally {
            assetsLoading = false
        }
    }

    LaunchedEffect(reportsRepository, selectedAssetId, days) {
        val assetId = selectedAssetId
        if (assetId.isNullOrBlank()) {
            orders = emptyList()
            ordersError = null
            ordersLoading = false
            return@LaunchedEffect
        }
        ordersLoading = true
        ordersError = null
        try {
            orders =
                reportsRepository.equipmentWorkOrders(
                    assetId = assetId,
                    from = IsoDates.formatEpochDay(fromDay),
                    to = IsoDates.formatEpochDay(today),
                )
        } catch (e: CancellationException) {
            throw e
        } catch (e: GatewayHttpException) {
            ordersError = e.message ?: "Не удалось загрузить отчёт"
            orders = emptyList()
        } catch (e: Exception) {
            ordersError = e.message ?: "Не удалось загрузить отчёт"
            orders = emptyList()
        } finally {
            ordersLoading = false
        }
    }

    selectedOrderId?.let { orderId ->
        WorkOrderDetailScreen(
            repository = reportsRepository,
            orderId = orderId,
            onBack = { selectedOrderId = null },
            equipmentRepository = equipmentRepository,
            attachmentsRepository = attachmentsRepository,
            commentsRepository = commentsRepository,
            readOnly = true,
            modifier = modifier,
        )
        return
    }

    AppScaffold(title = catalogItem.title, onNavigateBack = onBack, modifier = modifier) { padding ->
        LazyColumn(
            modifier = Modifier.fillMaxSize().padding(padding),
            contentPadding = androidx.compose.foundation.layout.PaddingValues(ClientSpacing.md),
            verticalArrangement = Arrangement.spacedBy(ClientSpacing.md),
        ) {
            item {
                PeriodSelector(selected = days, onSelected = { days = it })
            }
            item {
                ExposedDropdownMenuBox(
                    expanded = menuExpanded,
                    onExpandedChange = { menuExpanded = it },
                ) {
                    OutlinedTextField(
                        value = selectedAsset?.displayName() ?: "",
                        onValueChange = {},
                        readOnly = true,
                        label = { Text("Оборудование") },
                        placeholder = { Text("Выберите оборудование") },
                        trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = menuExpanded) },
                        modifier = Modifier.menuAnchor().fillMaxWidth(),
                    )
                    ExposedDropdownMenu(
                        expanded = menuExpanded,
                        onDismissRequest = { menuExpanded = false },
                    ) {
                        assets.forEach { asset ->
                            DropdownMenuItem(
                                text = { Text(asset.displayName()) },
                                onClick = {
                                    selectedAssetId = asset.id
                                    menuExpanded = false
                                },
                            )
                        }
                    }
                }
            }
            when {
                assetsLoading -> item { CircularProgressIndicator() }
                assetsError != null -> item { AppText(text = assetsError!!) }
                selectedAssetId == null ->
                    item { AppText(text = "Выберите оборудование", style = AppTextStyle.Label) }
                ordersLoading -> item { CircularProgressIndicator() }
                ordersError != null -> item { AppText(text = ordersError!!) }
                orders.isEmpty() ->
                    item {
                        AppText(
                            text = "Нет заявок по этому оборудованию за выбранный период",
                            style = AppTextStyle.Label,
                        )
                    }
                else ->
                    items(orders, key = { it.id }) { order ->
                        Column(
                            modifier =
                                Modifier
                                    .fillMaxWidth()
                                    .clickable { selectedOrderId = order.id }
                                    .padding(vertical = ClientSpacing.sm),
                            verticalArrangement = Arrangement.spacedBy(ClientSpacing.xs),
                        ) {
                            AppText(
                                text = formatWorkOrderDisplayTitle(order.title),
                                style = AppTextStyle.Title,
                            )
                            AppText(text = "${workOrderTypeLabelRu(order.type)} · ${order.dueAt}")
                            AppStatusChip(
                                text = workOrderStatusLabelRu(order.status),
                                tone =
                                    if (order.status == "new") {
                                        AppStatusChipTone.Accent
                                    } else {
                                        AppStatusChipTone.Muted
                                    },
                            )
                        }
                    }
            }
            item { ReportHelpFooter(text = catalogItem.description) }
        }
    }
}
```

Do not put raw `asset.id` in the dropdown or list.

- [ ] **Step 3: Branch from ReportsScreen**

In `ReportsScreen`, add optional `attachmentsRepository` / `commentsRepository` (default `null`). When `selectedReport == ReportId.EquipmentWorkOrders`, render `EquipmentWorkOrdersReportScreen` and return (do not use `ReportDetailScreen`).

```kotlin
fun ReportsScreen(
    reportsRepository: WorkOrdersRepository,
    equipmentRepository: EquipmentRepository,
    adminUsersRepository: AdminUsersRepository? = null,
    attachmentsRepository: AttachmentsRepository? = null,
    commentsRepository: CommentsRepository? = null,
    modifier: Modifier = Modifier,
) {
    var selectedReport by remember { mutableStateOf<ReportId?>(null) }

    if (selectedReport == ReportId.EquipmentWorkOrders) {
        EquipmentWorkOrdersReportScreen(
            reportsRepository = reportsRepository,
            equipmentRepository = equipmentRepository,
            attachmentsRepository = attachmentsRepository,
            commentsRepository = commentsRepository,
            onBack = { selectedReport = null },
            modifier = modifier,
        )
        return
    }
    // existing catalog / ReportDetailScreen
}
```

Add the matching imports.

- [ ] **Step 4: Wire shell**

In `MainShellContent.kt` `NavDestinationId.Reports` branch pass `attachmentsRepository` and `commentsRepository` into `ReportsScreen`.

- [ ] **Step 5: Compile/test focused client pieces**

```bash
./gradlew :composeApp:desktopTest --tests pro.masterdoc.client.ui.screens.ReportCatalogTest --no-daemon
./gradlew :auth:jvmTest --tests pro.masterdoc.client.auth.WorkOrdersRepositoryTest.equipmentWorkOrdersUsesAssetAndPeriod --no-daemon
```

Expected: PASS. Do not run Wasm/`assemble` locally.

- [ ] **Step 6: Commit and push `client-app`**

```bash
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/EquipmentWorkOrdersReportUi.kt \
  composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/ReportsScreen.kt \
  composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/shell/MainShellContent.kt
git commit -m "$(cat <<'EOF'
feat: detailed report list of work orders for selected equipment

EOF
)"
git push
```

Watch CI/deploy. After green pipeline, run `/smoke-test` (Fixaverse Smoke): Отчёты → Детальный отчёт → выбрать оборудование по имени → список заявок за 30 дней → открыть карточку → Назад в список. UI must not show ids.

---

## Self-review

| Spec requirement | Task |
|---|---|
| Catalog «Детальный отчёт» at end | 3 |
| Period 7/30/90 | 4 (`PeriodSelector`) |
| Equipment picker by name | 4 |
| List title / type · dueAt / status | 4 |
| Overlap rule + sort + 400 without assetId | 1 |
| `GET /reports/equipment-work-orders` | 1 |
| Read-only `WorkOrderDetailScreen` | 4 |
| Gateway GET by id for reports only | 2 |
| Names not ids | 4 + Global Constraints |
| Comments/photos optional / 403-safe | 4 (`readOnly`, optional repos) |
