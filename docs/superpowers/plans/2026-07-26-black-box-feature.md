# Black-box feature (paginated journal by user) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship product feature `black_box` («Чёрный ящик») as its own nav screen with user filter (Все | user) and paginated newest-first journal; remove Admin «Журнал» tab.

**Architecture:** black-box-service gains `offset` on `GET /events`. Gateway gates audit on `black_box`, allows `GET /admin/users` for `admin|black_box`, and forwards pagination. feature-service + Zitadel add the wire/role. client-app adds FeatureId/nav/screen and drops Journal from Admin.

**Tech Stack:** Kotlin, Ktor, Spring (feature-service), Compose Multiplatform, Zitadel Terraform role keys.

**Spec:** [docs/superpowers/specs/2026-07-26-black-box-feature-design.md](../specs/2026-07-26-black-box-feature-design.md)

## Global Constraints

- Wire id exactly `black_box`; RU title exactly `Чёрный ящик`.
- Page size default **30**; UI must not load unbounded journal in one request.
- Order: **newest first** (store `addFirst` + `drop(offset).take(limit)`).
- `GET /admin/audit` requires `black_box` only (not `admin`).
- `GET /admin/users` list: `admin` **or** `black_box`; invite/delete/set-features stay `admin` only.
- Skip recording `audit.list` when `offset > 0`.
- Tenant for smoke: Fixaverse Smoke; grants must include `black_box`.
- Do not run heavy local Gradle full builds for wasm; prefer targeted JVM tests. CI builds on push.
- Multi-repo: commit/push each affected repo when its tasks finish.

## File map

| Area | Files |
|------|--------|
| black-box-service | `Application.kt` (`AuditStore.list`, GET `/events`), `AuditRoutesTest.kt` |
| api-gateway | `ProductFeatures.kt`, `AdminAuth.kt` (shared require), `AdminAuditRoutes.kt`, `AdminUserRoutes.kt`, `BlackBoxClient.kt`, tests |
| feature-service | `FeatureCatalog.kt`, tests, README |
| masterdoc-zitadel | `terraform/main.tf`, `terraform/expected.yaml`, ensure-smoke/demo workflows |
| client-app | `NavModels.kt`, `NavCatalog.kt`, `FeatureLabels.kt`, session/nav tests, `AdminUserModels.kt`, `BlackBoxScreen.kt` (new), `UsersScreen.kt`, `MainShellContent.kt` |

---

### Task 1: black-box-service — offset pagination

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/black-box-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/blackbox/Application.kt`
- Test: `src/test/kotlin/pro/masterdoc/blackbox/AuditRoutesTest.kt`

**Interfaces:**
- Produces: `AuditStore.list(..., limit: Int, offset: Int = 0)` → filtered newest-first, then `drop(offset).take(limit)`
- Produces: `GET /events?offset=` (default 0, `coerceAtLeast(0)`)

- [ ] **Step 1: Write failing test for offset**

Add to `AuditRoutesTest.kt`:

```kotlin
@Test
fun listRespectsOffsetNewestFirst() = testApplication {
    application { module(AuditStore()) }
    repeat(3) { i ->
        client.post("/events") {
            contentType(ContentType.Application.Json)
            setBody("""{"orgId":"o","userId":"u","method":"POST","path":"/p$i","status":201,"action":"a$i"}""")
        }
    }
    val page0 = client.get("/events?orgId=o&limit=2&offset=0")
    val page1 = client.get("/events?orgId=o&limit=2&offset=2")
    assertEquals(HttpStatusCode.OK, page0.status)
    val items0 = json.parseToJsonElement(page0.bodyAsText()).jsonObject["items"]!!.jsonArray
    val items1 = json.parseToJsonElement(page1.bodyAsText()).jsonObject["items"]!!.jsonArray
    assertEquals(2, items0.size)
    assertEquals(1, items1.size)
    // newest first: last posted path /p2 is first
    assertEquals("/p2", items0[0].jsonObject["path"]!!.jsonPrimitive.content)
    assertEquals("/p1", items0[1].jsonObject["path"]!!.jsonPrimitive.content)
    assertEquals("/p0", items1[0].jsonObject["path"]!!.jsonPrimitive.content)
}
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
cd /Users/antonbutov/Documents/MYPROJECTS/Fixaverse/black-box-service
./gradlew test --tests 'pro.masterdoc.blackbox.AuditRoutesTest.listRespectsOffsetNewestFirst'
```

Expected: FAIL (offset ignored or compile error on signature).

- [ ] **Step 3: Implement offset**

In `AuditStore.list`, add `offset: Int = 0` and change terminal ops to:

```kotlin
.drop(offset.coerceAtLeast(0))
.take(limit)
.toList()
```

In `get("/events")`:

```kotlin
val limit = call.request.queryParameters["limit"]?.toIntOrNull()?.coerceIn(1, 500) ?: 100
val offset = call.request.queryParameters["offset"]?.toIntOrNull()?.coerceAtLeast(0) ?: 0
// ...
items = store.list(orgId = orgId, userId = userId, from = from, to = to, limit = limit, offset = offset),
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
./gradlew test --tests 'pro.masterdoc.blackbox.AuditRoutesTest'
```

Expected: PASS

- [ ] **Step 5: Commit and push**

```bash
git add src/main/kotlin/pro/masterdoc/blackbox/Application.kt src/test/kotlin/pro/masterdoc/blackbox/AuditRoutesTest.kt
git commit -m "feat(audit): support offset pagination on GET /events"
git push
```

Watch Actions; report result.

---

### Task 2: feature-service — catalog `black_box`

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/feature-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/feature/features/FeatureCatalog.kt`
- Modify: `src/test/kotlin/pro/masterdoc/feature/features/FeatureCatalogTest.kt`
- Modify: `src/test/kotlin/pro/masterdoc/feature/api/FeaturesControllerTest.kt` (expected id list if asserted)
- Modify: `README.md` (feature table)

**Interfaces:**
- Produces: catalog id `black_box`, titleRu `Чёрный ящик`

- [ ] **Step 1: Update failing/asserting tests**

In `FeatureCatalogTest`, extend expected sorted ids to include `"black_box"`:

```kotlin
assertEquals(
    listOf("admin", "black_box", "board", "charts", "copilot", "equipment"),
    catalog.catalog().map { it.id },
)
```

Add:

```kotlin
@Test
fun `includes black_box definition`() {
    val item = catalog.catalog().single { it.id == "black_box" }
    assertEquals("Чёрный ящик", item.titleRu)
}
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
./gradlew test --tests 'pro.masterdoc.feature.features.FeatureCatalogTest'
```

- [ ] **Step 3: Add catalog entry**

```kotlin
FeatureDefinition("admin", "Админ"),
FeatureDefinition("black_box", "Чёрный ящик"),
FeatureDefinition("board", "Доска"),
// ...
```

Update README feature table row for `black_box`.

- [ ] **Step 4: Run all feature-service tests — PASS**

```bash
./gradlew test
```

- [ ] **Step 5: Commit and push**

```bash
git add src/main/kotlin/pro/masterdoc/feature/features/FeatureCatalog.kt \
  src/test/kotlin/pro/masterdoc/feature/features/FeatureCatalogTest.kt \
  src/test/kotlin/pro/masterdoc/feature/api/FeaturesControllerTest.kt \
  README.md
git commit -m "feat: add black_box feature to catalog"
git push
```

---

### Task 3: api-gateway — ProductFeatures, auth, audit offset, users list

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/ProductFeatures.kt`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/AdminAuth.kt`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/BlackBoxClient.kt`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/AdminAuditRoutes.kt`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/AdminUserRoutes.kt`
- Modify: `src/test/kotlin/pro/masterdoc/gateway/ProductFeaturesTest.kt` (if any)
- Modify: `src/test/kotlin/pro/masterdoc/gateway/AdminAuditRoutesTest.kt`
- Modify: `src/test/kotlin/pro/masterdoc/gateway/AdminUserRoutesTest.kt`
- Modify: all `BlackBoxClient` fakes in tests (`listEvents` signature + offset)
- Modify: `README.md` audit row

**Interfaces:**
- Produces: `suspend fun ApplicationCall.requireAnyOfFeatures(deps, features: List<String>): ValidatedToken?` (same shape as `requireAdmin`, any match)
- Produces: `BlackBoxClient.listEvents(orgId, userId, limit, offset)`
- Consumes: Task 1 offset query

- [ ] **Step 1: Failing tests**

Update `AdminAuditRoutesTest`:

```kotlin
@Test
fun `GET admin audit requires black_box not admin`() = testApplication {
    // featureClientWith("admin") only → Forbidden
    // ...
    assertEquals(HttpStatusCode.Forbidden, response.status)
}

@Test
fun `GET admin audit allows black_box and forwards offset`() = testApplication {
    // featureClientWith("black_box")
    // blackBoxClient.listEvents asserts limit=30, offset=30, userId="u9"
    client.get("/admin/audit?limit=30&offset=30&userId=u9") { header(..., "Bearer good") }
    assertEquals(HttpStatusCode.OK, response.status)
}
```

Add/adjust users list test: `featureClientWith("black_box")` → `GET /admin/users` OK; invite still Forbidden without admin.

Update every fake `listEvents` to add `offset: Int` parameter.

- [ ] **Step 2: Run AdminAuditRoutesTest — expect FAIL**

```bash
./gradlew test --tests 'pro.masterdoc.gateway.AdminAuditRoutesTest'
```

- [ ] **Step 3: Implement**

`ProductFeatures.ENTRIES` add `FeatureDefinitionDto("black_box", "Чёрный ящик")`.

`AdminAuth.kt` — add `requireAnyOfFeatures` by copying `requireAdmin` body and replacing the single-feature check with:

```kotlin
if (features.none { it in granted }) {
    logAuthDenied("forbidden", feature = features.joinToString(","))
    respondText("Forbidden", status = HttpStatusCode.Forbidden)
    return null
}
```

`BlackBoxClient.listEvents(..., limit: Int, offset: Int = 0)` and HTTP query `&offset=`.

`AdminAuditRoutes.kt`:

```kotlin
val validated = call.requireAnyOfFeatures(deps, listOf("black_box")) ?: return@get
val userId = call.request.queryParameters["userId"]?.takeIf { it.isNotBlank() }
val limit = call.request.queryParameters["limit"]?.toIntOrNull()?.coerceIn(1, 100) ?: 30
val offset = call.request.queryParameters["offset"]?.toIntOrNull()?.coerceAtLeast(0) ?: 0
val upstream = deps.blackBoxClient.listEvents(orgId, userId, limit, offset)
// respond...
if (upstream.status.value in 200..299 && offset == 0) {
    deps.blackBoxClient.recordAsync(/* audit.list */)
}
```

`AdminUserRoutes.kt` `get("/admin/users")`:

```kotlin
val validated = call.requireAnyOfFeatures(deps, listOf("admin", "black_box")) ?: return@get
```

- [ ] **Step 4: Run gateway tests touching admin/audit/features — PASS**

```bash
./gradlew test --tests 'pro.masterdoc.gateway.AdminAuditRoutesTest' \
  --tests 'pro.masterdoc.gateway.AdminUserRoutesTest' \
  --tests 'pro.masterdoc.gateway.ProductFeaturesTest' \
  --tests 'pro.masterdoc.gateway.FeaturesRoutesTest'
```

Fix all `listEvents` override compile breaks in other test fakes.

- [ ] **Step 5: Commit and push**

```bash
git commit -m "feat(gateway): gate audit on black_box; paginate; users list for black_box"
git push
```

---

### Task 4: masterdoc-zitadel — role key + Smoke/Demo grants

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/masterdoc-zitadel`

**Files:**
- Modify: `terraform/main.tf` (`local.roles`)
- Modify: `terraform/expected.yaml`
- Modify: `.github/workflows/ensure-smoke-org.yml` (`INVITE_ROLE_KEYS`)
- Modify: `.github/workflows/ensure-smtp-demo.yml` (`INVITE_ROLE_KEYS`)
- Modify: verify tests if they hardcode role lists (`verify/src/test/...`)
- Modify: `docs/RUNBOOK.md` (mention `black_box`)

**Interfaces:**
- Produces: Zitadel project role_key `black_box`

- [ ] **Step 1: Add role to terraform locals**

```hcl
roles = {
  admin     = "Admin"
  black_box = "Black box"
  board     = "Board"
  charts    = "ППР"
  copilot   = "Copilot"
  equipment = "Equipment"
}
```

`expected.yaml` roles list include `black_box`.

Workflows:

```yaml
INVITE_ROLE_KEYS: board charts copilot equipment admin black_box
```

Update verify stubs’ `roleKeys` to include `black_box` where full catalog is expected.

- [ ] **Step 2: Commit and push**

```bash
git commit -m "feat(zitadel): add black_box project role and smoke grants"
git push
```

- [ ] **Step 3: Apply platform roles**

Trigger Terraform Apply / Ensure Smoke Org (`workflow_dispatch`) so Smoke user receives `black_box`. Confirm in run log. If secrets missing, document BLOCKED in handoff — client still ships, journal nav empty until grant.

---

### Task 5: client-app — FeatureId, nav, listAudit API

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/client-app`

**Files:**
- Modify: `shared/src/commonMain/kotlin/pro/masterdoc/client/navigation/NavModels.kt`
- Modify: `shared/src/commonMain/kotlin/pro/masterdoc/client/navigation/NavCatalog.kt`
- Modify: `shared/src/commonMain/kotlin/pro/masterdoc/client/navigation/FeatureLabels.kt`
- Modify: `shared/src/commonTest/kotlin/pro/masterdoc/client/navigation/FeatureLabelsTest.kt` (if exists)
- Modify: `shared/src/commonTest/kotlin/pro/masterdoc/client/navigation/NavMenuBuilderTest.kt`
- Modify: `shared/src/commonTest/kotlin/pro/masterdoc/client/session/ClientSessionFromMeTest.kt`
- Modify: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/AdminUserModels.kt`
- Modify: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/AdminUsersRepositoryTest.kt` (add/extend)

**Interfaces:**
- Produces: `FeatureId.BlackBox("black_box")`, `NavDestinationId.BlackBox`
- Produces: `suspend fun listAudit(limit: Int = 30, offset: Int = 0, userId: String? = null): AuditEventListDto`

- [ ] **Step 1: Failing session/nav tests**

```kotlin
assertEquals("black_box", FeatureId.BlackBox.wireValue)
// fromMe with features containing black_box → FeatureId.BlackBox in session
// NavMenuBuilder includes BlackBox when feature granted; titleRu == "Чёрный ящик"
```

Repository test: request URL contains `limit=30&offset=30&userId=u1` when those args passed.

- [ ] **Step 2: Run targeted tests — FAIL**

```bash
./gradlew :shared:jvmTest --tests 'pro.masterdoc.client.session.ClientSessionFromMeTest' \
  --tests 'pro.masterdoc.client.navigation.*'
```

- [ ] **Step 3: Implement enums, catalog, labels, listAudit**

```kotlin
enum class FeatureId(...) {
    // ...
    BlackBox("black_box"),
    Users("admin"),
}
enum class NavDestinationId { ..., BlackBox, Users, }

// NavCatalog — order 65 between Copilot(60) and Users(70):
NavItemSpec(NavDestinationId.BlackBox, FeatureId.BlackBox, "nav.black_box", "black_box", 65),

fun FeatureId.titleRu() = when (this) {
    FeatureId.BlackBox -> "Чёрный ящик"
    // ...
}

suspend fun listAudit(limit: Int = 30, offset: Int = 0, userId: String? = null): AuditEventListDto {
    val q = buildString {
        append("limit=").append(limit)
        append("&offset=").append(offset)
        if (!userId.isNullOrBlank()) append("&userId=").append(userId)
    }
    // GET "$base/admin/audit?$q"
}
```

- [ ] **Step 4: Tests PASS**

```bash
./gradlew :shared:jvmTest --tests 'pro.masterdoc.client.session.ClientSessionFromMeTest' \
  --tests 'pro.masterdoc.client.navigation.*'
./gradlew :auth:jvmTest --tests 'pro.masterdoc.client.auth.AdminUsersRepositoryTest'
```

- [ ] **Step 5: Commit** (push after Task 6 with UI, or push now if preferred)

```bash
git commit -m "feat(nav): add black_box feature id and paginated listAudit"
```

---

### Task 6: client-app — BlackBoxScreen + remove Admin Journal

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/client-app`

**Files:**
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/BlackBoxScreen.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/UsersScreen.kt` (remove `AdminTab.Journal`, `JournalTab`; keep `formatAuditAt` if still used — move to BlackBoxScreen or shared)
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/shell/MainShellContent.kt`
- Modify: `StubDestinationScreen.kt` if needed
- Move/keep: `formatAuditAt` — prefer `BlackBoxScreen.kt` or small `AuditFormat.kt` next to descriptions; update any tests importing it

**Interfaces:**
- Consumes: `listUsers()`, `listAudit(limit=30, offset, userId)`
- Produces: UI with dropdown Все|user, Обновить, Ещё, newest-first pages

- [ ] **Step 1: Implement `BlackBoxScreen`**

```kotlin
@Composable
fun BlackBoxScreen(repository: AdminUsersRepository, modifier: Modifier = Modifier) {
    val pageSize = 30
    var users by remember { mutableStateOf<List<AdminUser>>(emptyList()) }
    var selectedUserId by remember { mutableStateOf<String?>(null) } // null = Все
    var events by remember { mutableStateOf<List<AuditEventDto>>(emptyList()) }
    var offset by remember { mutableStateOf(0) }
    var hasMore by remember { mutableStateOf(false) }
    var loading by remember { mutableStateOf(true) }
    var loadingMore by remember { mutableStateOf(false) }
    var error by remember { mutableStateOf<String?>(null) }
    val scope = rememberCoroutineScope()

    fun actorLabel(userId: String): String {
        val u = users.find { it.id == userId }
        return when {
            u == null -> userId.take(8) + "…"
            else -> listOfNotNull(u.displayNameOrNull(), u.email).joinToString(" · ").ifBlank { userId.take(8) + "…" }
        }
    }

    fun reload() { /* offset=0; events = listAudit(...).items; hasMore = items.size == pageSize */ }
    fun loadMore() { /* offset' = events.size; append; hasMore = page.size == pageSize */ }

    LaunchedEffect(repository) { users = repository.listUsers().items; reload() }
    LaunchedEffect(selectedUserId) { reload() }

    AppScaffold(title = "Чёрный ящик", modifier = modifier) { padding ->
        Column(...) {
            // Material3 ExposedDropdownMenuBox: «Все» + users (name · email)
            AppButton(text = "Обновить", onClick = { reload() }, fillMaxWidth = false)
            // list rows: title, optional actor if selectedUserId==null, formatAuditAt · status
            if (hasMore) AppButton(text = "Ещё", onClick = { loadMore() }, ...)
        }
    }
}
```

Use existing `AdminUser` fields (`email`, name fields as in UsersScreen). Match design-system components (`AppButton`, `AppText`, `AppScaffold`).

- [ ] **Step 2: Wire shell + strip Admin Journal**

`MainShellContent`:

```kotlin
NavDestinationId.BlackBox -> BlackBoxScreen(repository = adminUsersRepository!!)
```

`iconFor`: `Icons.Filled.History` or `Icons.Filled.Inventory2` for BlackBox.

`UsersScreen`: `AdminTab` only `Users, Sites`; delete Journal branch and `JournalTab`.

- [ ] **Step 3: Targeted compile/tests**

```bash
./gradlew :shared:jvmTest :auth:jvmTest
# if compose JVM tests exist for UsersScreen formatAuditAt, move/update
```

Do **not** run `wasmJsBrowserDistribution`.

- [ ] **Step 4: Commit and push client-app**

```bash
git add ...
git commit -m "feat(ui): BlackBox screen with user filter and pagination"
git push
```

Watch Actions; report deploy.

---

### Task 7: Smoke (Fixaverse Smoke)

**Org:** Fixaverse Smoke (`383177088934346755`)  
**Login:** `mail+smoke@antonbutov.com`  
**URL:** `https://app.fixaverse.ru/` after client deploy (or local `wasmJsBrowserRun` if not yet live)

Checklist:

| # | Expectation |
|---|-------------|
| 1 | `/me` features include `black_box` |
| 2 | Nav shows «Чёрный ящик»; Админ has no «Журнал» |
| 3 | Dropdown Все → newest events; actor labels visible |
| 4 | Pick smoke user → filtered list |
| 5 | «Ещё» only if ≥30 events; does not replace page 0 |
| 6 | Board still opens (regression) |

Report PASS/FAIL/BLOCKED per smoke-test skill.

---

## Self-review (plan vs spec)

| Spec item | Task |
|-----------|------|
| Feature `black_box` / title | 2, 4, 5 |
| Own nav, Admin without Journal | 5, 6 |
| Dropdown Все \| user | 6 |
| Pagination 30, newest first, Ещё | 1, 3, 6 |
| Audit gate black_box only | 3 |
| Users list admin\|black_box | 3 |
| Skip audit.list when offset>0 | 3 |
| Zitadel + Smoke grant | 4, 7 |
| No black-box storage rewrite | 1 (offset only) |

No TBD placeholders. Signatures aligned: `listEvents(..., limit, offset)`, `listAudit(limit, offset, userId)`.

---

## Execution

Per workspace rule **plan-execution-subagent-driven**: execute with **Subagent-Driven Development** (fresh implementer per task, review between tasks). Do not ask for Inline unless the user overrides.
