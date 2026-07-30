# Customer Tickets (Заявки) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fill the client `Tickets` («Заявки») stub so a user with feature `tickets` can create emergency work orders for scoped assets, list own active/closed WOs, and open a read-only detail — end-to-end across catalog grants, gateway ACL, dashboard WO fields, and UI.

**Architecture:** Wire feature `tickets` in feature-service + Zitadel. Extend WorkOrder with `createdBy` + `description`. Gateway allows tickets to GET/POST work-orders and GET assets/sites (scoped). Dashboard sets `createdBy` from `X-User-Id`, forces own-only lists for tickets-only callers, rejects PATCH for them. Client `TicketsScreen` replaces stub; admin binds customers via reused scope screen. Photos deferred.

**Tech Stack:** Kotlin/Ktor (gateway, dashboard, catalog), Spring feature-service, Compose Multiplatform client-app, Zitadel expected.yaml; CI via GitHub Actions (no heavy local full builds).

**Spec:** `masterdoc/docs/superpowers/specs/2026-07-29-customer-tickets-design.md`

## Global Constraints

- Wire id is `tickets` (not `userRequests` / not IdP role `requester`).
- Photos are out of scope (no UI control).
- Reuse `user-scopes` for site binding; no new Ticket entity.
- Statuses: active = `new`|`in_progress`; completed = `closed`.
- Detail: `WorkOrderDetailScreen(readOnly = true)`.
- Multi-repo: commit/push in the repo each task touches; watch Actions.
- Prefer targeted unit/route tests; push → CI for full verify.

## File map

| File | Role |
| --- | --- |
| `feature-service/.../FeatureCatalog.kt` | Add `tickets` / «Заявки» |
| `masterdoc-zitadel/terraform/expected.yaml` | Add `tickets` role key |
| `dashboard-service/.../WorkOrderStore.kt` | `createdBy`, `description`, list filter |
| `dashboard-service/.../Application.kt` | Create ACL, force createdBy, get/patch gates |
| `api-gateway-service/.../EquipmentRoutes.kt` | tickets on WO/assets/sites ACL |
| `client-app/.../WorkOrderModels.kt` | DTO + `list(createdBy=)` |
| `client-app/.../TicketsScreen.kt` | New screen |
| `client-app/.../MainShellContent.kt` | Wire Tickets |
| `client-app/.../EngineerScopeScreen.kt` | Filter by feature param |
| `client-app/.../UsersScreen.kt` | «Привязка заказчиков» |
| `client-app/.../WorkOrderDetailScreen.kt` | Show description when present |

---

### Task 1: feature-service — catalog `tickets`

**Repo:** `feature-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/feature/features/FeatureCatalog.kt`
- Modify: `src/test/kotlin/pro/masterdoc/feature/features/FeatureCatalogTest.kt`

**Interfaces:**
- Produces: catalog id `tickets`, titleRu `Заявки`; `filter` accepts grant `tickets`

- [ ] **Step 1: Update failing expectations in FeatureCatalogTest**

In `catalog returns all features with russian titles`, set size to `7` and include `tickets` / `Заявки` in sorted lists (after `equipment` alphabetically… actually sort by id: `admin, black_box, board, charts, engineer, equipment, tickets`).

```kotlin
assertEquals(7, items.size)
assertEquals(
    listOf("admin", "black_box", "board", "charts", "engineer", "equipment", "tickets"),
    items.map { it.id },
)
assertEquals(
    listOf("Админ", "Чёрный ящик", "Доска", "ППР", "Инженер", "Оборудование", "Заявки"),
    items.map { it.titleRu },
)
```

Add:

```kotlin
@Test
fun `includes tickets definition`() {
    val item = catalog.catalog().single { it.id == "tickets" }
    assertEquals("Заявки", item.titleRu)
}
```

- [ ] **Step 2: Run test — expect FAIL**

Run: `./gradlew test --tests pro.masterdoc.feature.features.FeatureCatalogTest`  
Expected: FAIL (size 6 vs 7 / missing tickets)

- [ ] **Step 3: Add catalog entry**

```kotlin
FeatureDefinition("tickets", "Заявки"),
```

in `FeatureCatalog.ENTRIES` (keep alphabetical by id if list is unsorted — `catalog()` sorts by id).

- [ ] **Step 4: Run test — expect PASS**

- [ ] **Step 5: Commit and push; watch Action**

```bash
git add src/main/kotlin/pro/masterdoc/feature/features/FeatureCatalog.kt \
  src/test/kotlin/pro/masterdoc/feature/features/FeatureCatalogTest.kt
git commit -m "feat: add tickets feature to catalog"
git push
gh run watch
```

---

### Task 2: masterdoc-zitadel — expected `tickets` key

**Repo:** `masterdoc-zitadel`

**Files:**
- Modify: `terraform/expected.yaml`
- Modify smoke/invite workflows if they hardcode `INVITE_ROLE_KEYS` without tickets (optional for smoke org; do **not** add tickets to demo smoke invites unless needed)

**Interfaces:**
- Produces: `roles` contains `tickets` for verify suite

- [ ] **Step 1: Add to expected.yaml**

```yaml
roles:
  - board
  - charts
  - engineer
  - equipment
  - admin
  - black_box
  - tickets
```

- [ ] **Step 2: Commit and push**

```bash
git add terraform/expected.yaml
git commit -m "feat: expect tickets feature key in Zitadel project"
git push
```

Note: live Zitadel apply / terraform may be a separate ops step; verify suite will fail against live until role exists — document in commit body if apply is blocked. If terraform module defines roles from this file, include terraform change in same commit.

- [ ] **Step 3: Watch CI / report**

---

### Task 3: dashboard — `createdBy` + `description` + tickets ACL

**Repo:** `dashboard-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt`
- Modify: `src/test/kotlin/pro/masterdoc/dashboard/WorkOrderRoutesTest.kt`

**Interfaces:**
- Consumes: `X-User-Id`, `X-Caller-Features` (existing)
- Produces: `WorkOrder.createdBy: String?`, `WorkOrder.description: String?`; `CreateWorkOrderRequest.description: String?`; `list(orgId, assigneeId?, createdBy?)`; create allowed for `board` **or** `tickets`

- [ ] **Step 1: Write failing route tests**

Add tests (patterns from existing `WorkOrderRoutesTest`):

1. `create sets createdBy from X-User-Id and stores description`
2. `tickets caller can create emergency without board`
3. `tickets-only list forces createdBy=self (ignores other createdBy query)`
4. `tickets-only get foreign WO returns 404`
5. `tickets-only PATCH returns 400`

Sketch for (1)+(2):

```kotlin
@Test
fun ticketsCallerCreatesWithCreatedByAndDescription() = testApplication {
    // install app with assets site for a1
    val res = client.post("/work-orders") {
        header("X-Org-Id", "org-1")
        header("X-User-Id", "customer-1")
        header("X-Caller-Features", "tickets")
        contentType(ContentType.Application.Json)
        setBody("""{"type":"emergency","title":"Шум","assetId":"a1","siteId":"s1","dueAt":"2026-07-29","description":"Сильный шум подшипника"}""")
    }
    assertEquals(HttpStatusCode.Created, res.status)
    val body = json.parseToJsonElement(res.bodyAsText()).jsonObject
    assertEquals("customer-1", body["createdBy"]!!.jsonPrimitive.content)
    assertEquals("Сильный шум подшипника", body["description"]!!.jsonPrimitive.content)
}
```

- [ ] **Step 2: Run tests — expect FAIL**

Run: `./gradlew test --tests pro.masterdoc.dashboard.WorkOrderRoutesTest`  
Expected: FAIL (board-only create / missing fields)

- [ ] **Step 3: Extend model + store**

```kotlin
// WorkOrder + CreateWorkOrderRequest
val createdBy: String? = null,
val description: String? = null,

// create(...):
val description = req.description?.trim()?.takeIf { it.isNotBlank() }
require(description == null || description.length <= 4000) { "description too long" }
// ...
createdBy = createdByParam, // new arg from Application
description = description,
```

```kotlin
fun list(orgId: String, assigneeId: String? = null, createdBy: String? = null): List<WorkOrder> =
    byId.values
        .filter { it.orgId == orgId }
        .filter { assigneeId == null || it.assigneeId == assigneeId }
        .filter { createdBy == null || it.createdBy == createdBy }
        .sortedWith(...)
```

- [ ] **Step 4: Update Application routes**

```kotlin
private fun ApplicationCall.isTicketsOnly(): Boolean {
    val f = callerFeatures() ?: return false
    return "tickets" in f && "board" !in f && "engineer" !in f && "admin" !in f
}

post("/work-orders") {
    val features = call.callerFeatures()
    if (features != null && "board" !in features && "tickets" !in features) {
        throw IllegalArgumentException("Caller requires board or tickets feature to create work orders")
    }
    // ...
    val created = workOrderStore.create(
        orgId = orgId,
        req = req.copy(source = source),
        createdBy = call.userId(),
        ...
    )
}

get("/work-orders") {
    var createdBy = call.request.queryParameters["createdBy"]?.takeIf { it.isNotBlank() }
    if (call.isTicketsOnly()) createdBy = call.userId()
    var items = workOrderStore.list(orgId, assigneeId, createdBy)
    // existing scope filter...
}

get("/work-orders/{id}") {
    val wo = workOrderStore.get(...)
    if (call.isTicketsOnly() && wo.createdBy != call.userId()) {
        throw NoSuchElementException("Work order not found")
    }
    call.respond(wo)
}

patch("/work-orders/{id}") {
    if (call.isTicketsOnly()) {
        throw IllegalArgumentException("tickets cannot modify work orders")
    }
    // existing logic...
}
```

- [ ] **Step 5: Run tests — expect PASS**

- [ ] **Step 6: Commit, push, watch Action**

```bash
git commit -m "feat: WO createdBy/description and tickets create/list isolation"
git push && gh run watch
```

---

### Task 4: api-gateway — ACL for tickets

**Repo:** `api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt`
- Modify/create tests: `src/test/kotlin/pro/masterdoc/gateway/WorkOrderProxyRoutesTest.kt` (+ assets/sites if needed)

**Interfaces:**
- Produces: tickets in read/write for `/work-orders`; tickets in read for `/assets` and `/sites`

- [ ] **Step 1: Failing tests**

Extend `WorkOrderProxyRoutesTest`:

- GET `/work-orders` allowed with `tickets`
- POST `/work-orders` allowed with `tickets`
- GET `/work-orders` forbidden with only `charts`

Add or extend assets proxy test: GET `/assets` allowed with `tickets`.

- [ ] **Step 2: Run — expect FAIL**

- [ ] **Step 3: Update EquipmentRoutes**

```kotlin
proxyPrefix(
    "/sites",
    config.catalogServiceBaseUrl,
    client,
    deps,
    readFeatures = listOf("equipment", "admin", "tickets"),
    writeFeatures = listOf("equipment", "admin"),
)
proxyPrefix(
    "/assets",
    config.catalogServiceBaseUrl,
    client,
    deps,
    readFeatures = listOf("equipment", "tickets"),
    writeFeatures = listOf("equipment"),
    scopeFilterHint = true,
)
proxyPrefix(
    "/work-orders",
    config.dashboardServiceBaseUrl,
    client,
    deps,
    readFeatures = listOf("board", "engineer", "tickets"),
    writeFeatures = listOf("board", "engineer", "tickets"),
    scopeFilterHint = true,
)
```

Note: PATCH still hits writeGate (`tickets` allowed at gateway); dashboard Task 3 rejects tickets-only PATCH.

- [ ] **Step 4: Run tests — PASS; commit; push; watch**

```bash
git commit -m "feat: allow tickets feature on work-orders and asset/site reads"
```

---

### Task 5: client-app — WorkOrder DTO + repository `createdBy`

**Repo:** `client-app`

**Files:**
- Modify: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt`
- Modify: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt` (if present; else commonTest)

**Interfaces:**
- Produces: `WorkOrderDto.createdBy`, `WorkOrderDto.description`; `CreateWorkOrderRequest.description`; `list(assigneeId, createdBy)`

- [ ] **Step 1: Extend DTOs**

```kotlin
val createdBy: String? = null,
val description: String? = null,
// CreateWorkOrderRequest:
val description: String? = null,
```

```kotlin
suspend fun list(assigneeId: String? = null, createdBy: String? = null): List<WorkOrderDto> {
    val params = buildList {
        assigneeId?.takeIf { it.isNotBlank() }?.let { add("assigneeId=$it") }
        createdBy?.takeIf { it.isNotBlank() }?.let { add("createdBy=$it") }
    }
    val query = if (params.isEmpty()) "" else params.joinToString("&", prefix = "?")
    // ...
}
```

- [ ] **Step 2: Unit-test query building if repository is hard to mock; otherwise skip to compile via CI**

Prefer a small pure helper if needed:

```kotlin
fun workOrdersListQuery(assigneeId: String?, createdBy: String?): String
```

Test both params, one param, none.

- [ ] **Step 3: Commit; push**

```bash
git commit -m "feat: work order createdBy/description client models"
```

---

### Task 6: client-app — `TicketsScreen`

**Repo:** `client-app`

**Files:**
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/TicketsScreen.kt`
- Create: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/TicketsPartitionTest.kt` (pure partition helper)
- Modify: `composeApp/.../MainShellContent.kt`
- Modify: `StubDestinationScreen.kt` only if title still needed

**Interfaces:**
- Consumes: `WorkOrdersRepository.list(createdBy)`, `create`, `EquipmentRepository.listAssets`
- Produces: `TicketsScreen(repository, equipmentRepository, currentUserId)`

- [ ] **Step 1: Pure partition helper + test**

```kotlin
fun partitionCustomerTickets(orders: List<WorkOrderDto>): Pair<List<WorkOrderDto>, List<WorkOrderDto>> {
    val active = orders.filter { it.status == "new" || it.status == "in_progress" }
    val done = orders.filter { it.status == "closed" }
    return active to done
}
```

Test: mixed list → correct buckets.

- [ ] **Step 2: Implement TicketsScreen**

Structure (mirror MyWorkOrders patterns):

- Load assets + `list(createdBy = currentUserId)`
- Form: ExposedDropdownMenuBox assets; AppTextField multiline description; AppButton «Создать»
- On create: `title = description.lineSequence().first().trim().take(120).ifBlank { "Заявка" }`, `type=emergency`, `siteId=asset.siteId`, `dueAt=today ISO date`, `description=description`
- Sections «Активные» / «Завершённые»
- Tap → `WorkOrderDetailScreen(..., readOnly = true)` (no mentor)

- [ ] **Step 3: Wire MainShellContent**

```kotlin
NavDestinationId.Tickets ->
    if (workOrdersRepository != null && equipmentRepository != null) {
        TicketsScreen(
            repository = workOrdersRepository,
            equipmentRepository = equipmentRepository,
            currentUserId = session.user?.id,
        )
    } else StubDestinationScreen(active.destination)
```

- [ ] **Step 4: Commit; push; watch**

```bash
git commit -m "feat: TicketsScreen for customer emergency requests"
```

---

### Task 7: client-app — admin «Привязка заказчиков»

**Repo:** `client-app`

**Files:**
- Modify: `EngineerScopeScreen.kt` — generalize filter
- Modify: `UsersScreen.kt` — button + host editor
- Modify: `MainShellContent.kt` — pass `userScopesRepository` into UsersScreen
- Modify: `EngineerScopeFilterTest.kt` — tickets filter test

**Interfaces:**
- Produces: `filterUsersForScopeBinding(users, requiredFeature: String)`; `EngineerScopeScreen(..., requiredFeature, title)`

- [ ] **Step 1: Tests**

```kotlin
@Test
fun filterTicketsUsers() {
    val users = listOf(
        AdminUser(id = "1", features = listOf("tickets"), ...),
        AdminUser(id = "2", features = listOf("engineer"), ...),
    )
    assertEquals(listOf("1"), filterUsersForScopeBinding(users, "tickets").map { it.id })
}
```

Keep existing engineer filter via `filterUsersForScopeBinding(users, "engineer")` (replace `filterEngineersForScopeBinding` body to delegate).

- [ ] **Step 2: Implement filter + screen params**

```kotlin
internal fun filterUsersForScopeBinding(users: List<AdminUser>, requiredFeature: String): List<AdminUser> =
    users.filter { requiredFeature in it.features }

internal fun filterEngineersForScopeBinding(users: List<AdminUser>): List<AdminUser> =
    filterUsersForScopeBinding(users, "engineer")
```

`EngineerScopeScreen(..., requiredFeature: String = "engineer", title: String = "Привязка инженеров")` — use `filterUsersForScopeBinding(users, requiredFeature)`.

- [ ] **Step 3: UsersScreen entry**

Add optional `userScopesRepository: UserScopesRepository?`. Button «Привязка заказчиков» opens scope screen with `requiredFeature = "tickets"`, `title = "Привязка заказчиков"`.

Wire from MainShellContent like board already wires scopes.

- [ ] **Step 4: Commit; push; watch**

```bash
git commit -m "feat: admin binding for tickets customers via user-scopes"
```

---

### Task 8: client-app — show `description` on detail

**Repo:** `client-app`

**Files:**
- Modify: `WorkOrderDetailScreen.kt`

- [ ] **Step 1: After title/status block, if `wo.description` not blank, show AppText**

```kotlin
wo.description?.takeIf { it.isNotBlank() }?.let { desc ->
    AppText(text = "Описание", style = AppTextStyle.Label)
    AppText(text = desc)
}
```

- [ ] **Step 2: Commit; push; watch**

```bash
git commit -m "feat: show work order description on detail screen"
```

---

### Task 9: Spec coverage smoke + docs note

**Repo:** `masterdoc` (optional short note in design status)

- [ ] **Step 1: Mark design status implemented-in-progress / done in design file after CI green**
- [ ] **Step 2: Verify acceptance checklist from spec manually or via smoke if env available**
- [ ] **Step 3: Commit docs status if changed**

---

## Self-review (plan vs spec)

| Spec requirement | Task |
| --- | --- |
| Feature `tickets` in catalog | 1 |
| Zitadel expected key | 2 |
| `createdBy` + `description` on WO | 3, 5 |
| Create emergency + board visibility | 3, 6 |
| Own-only lists; active/closed split | 3, 6 |
| Read-only detail | 6, 8 |
| Scoped assets via user-scopes | 4 + existing scope filter; 7 admin bind |
| Gateway ACL | 4 |
| Photos deferred | no task (explicit) |
| Admin customer binding | 7 |

No TBD placeholders. PATCH for tickets-only rejected in dashboard (Task 3), not gateway method-split (YAGNI).
