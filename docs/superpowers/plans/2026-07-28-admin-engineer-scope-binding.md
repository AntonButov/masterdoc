# Admin Engineer Scope Binding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move engineer Site/Asset binding ownership from the dispatcher board to Admin → Users, with gateway write ACL = `admin` and read ACL = `admin|board`.

**Architecture:** Reuse existing `EngineerScopeScreen` + catalog `user-scopes`. Change only UI entry, engineer picker filter (`equipment`), and api-gateway proxy feature gates. No catalog/dashboard model changes.

**Tech Stack:** Ktor api-gateway (Kotlin), Compose Multiplatform client-app, kotlin.test; CI via GitHub Actions (no heavy local full builds).

**Spec:** `masterdoc/docs/superpowers/specs/2026-07-28-admin-engineer-scope-binding-design.md`

## Global Constraints

- Engineer identity = feature `equipment` (no new `engineer` feature).
- Scope model unchanged: Sites and/or Assets via `PUT /user-scopes/{userId}`.
- Board keeps `userScopesRepository` for WO assignee `candidates` (dispatcher GET).
- Multi-repo: commit/push in the repo each task touches; watch Actions.
- Targeted tests only locally if needed; prefer push → CI.

## File map

| File | Role |
| --- | --- |
| `api-gateway-service/src/main/kotlin/.../EquipmentRoutes.kt` | Split `/user-scopes` read/write features |
| `api-gateway-service/src/test/kotlin/.../UserScopeProxyRoutesTest.kt` | ACL tests |
| `client-app/.../EngineerScopeScreen.kt` | Filter picker to `equipment` users |
| `client-app/.../UsersScreen.kt` | Entry button + host scope editor |
| `client-app/.../BoardScreen.kt` | Remove binding UI |
| `client-app/.../MainShellContent.kt` | Pass `userScopesRepository` into Users |
| `masterdoc/docs/plans/2026-07-27-001-feat-engineer-scope-binding-plan.md` | Amend R8 owner |

---

### Task 1: Gateway `/user-scopes` admin write, board|admin read

**Repo:** `api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt` (user-scopes `proxyPrefix`)
- Modify: `src/test/kotlin/pro/masterdoc/gateway/UserScopeProxyRoutesTest.kt`

**Interfaces:**
- Consumes: existing `proxyPrefix(..., readFeatures=, writeFeatures=)`
- Produces: PUT requires `admin`; GET requires `admin` or `board`

- [ ] **Step 1: Update failing tests for new ACL**

Replace / extend `UserScopeProxyRoutesTest` so expectations match the design:

```kotlin
@Test
fun `PUT user-scopes forbidden without admin feature`() = testApplication {
    application {
        module(
            GatewayConfig.testDefaults(),
            GatewayDeps(
                featureClient = featureClientWith("board"),
                backendClient = BackendProxyClient { _, _, _, _ -> error("unused") },
                tokenValidator = TokenValidator.accepting(),
            ),
        )
    }
    val response =
        client.put("/user-scopes/user-1") {
            header(HttpHeaders.Authorization, "Bearer good")
            contentType(ContentType.Application.Json)
            setBody("""{"siteIds":[],"assetIds":[]}""")
        }
    assertEquals(HttpStatusCode.Forbidden, response.status)
}

@Test
fun `PUT user-scopes proxied with admin feature`() {
    // same catalog HttpServer stub as current `PUT user-scopes proxied with board feature`
    // but featureClientWith("admin") instead of "board"
    // assert OK + X-Org-Id / X-User-Id forwarded
}

@Test
fun `GET user-scopes covers allowed with board feature`() {
    // catalog stub returning {"covers":true}
    // featureClientWith("board") → assert OK
}

@Test
fun `GET user-scopes covers allowed with admin feature`() {
    // same stub; featureClientWith("admin") → assert OK
}

@Test
fun `GET user-scopes covers forbidden without board or admin`() = testApplication {
    application {
        module(
            GatewayConfig.testDefaults(),
            GatewayDeps(
                featureClient = featureClientWith("equipment"),
                backendClient = BackendProxyClient { _, _, _, _ -> error("unused") },
                tokenValidator = TokenValidator.accepting(),
            ),
        )
    }
    val response =
        client.get("/user-scopes/user-1/covers/asset-1") {
            header(HttpHeaders.Authorization, "Bearer good")
        }
    assertEquals(HttpStatusCode.Forbidden, response.status)
}
```

Remove obsolete tests that assert PUT success with only `board`, or rewrite them as the forbidden case above.

- [ ] **Step 2: Run tests — expect RED (PUT with admin still fails until Step 3, or board PUT still passes incorrectly)**

```bash
cd api-gateway-service
./gradlew test --tests 'pro.masterdoc.gateway.UserScopeProxyRoutesTest' --no-daemon
```

Expected: FAIL (e.g. PUT with `admin` forbidden while still gated on `board`, and/or PUT with `board` still OK).

- [ ] **Step 3: Implement split gates**

In `EquipmentRoutes.kt`, change the `/user-scopes` proxy from:

```kotlin
proxyPrefix(
    "/user-scopes",
    config.catalogServiceBaseUrl,
    client,
    deps,
    features = listOf("board"),
)
```

to:

```kotlin
proxyPrefix(
    "/user-scopes",
    config.catalogServiceBaseUrl,
    client,
    deps,
    readFeatures = listOf("admin", "board"),
    writeFeatures = listOf("admin"),
)
```

- [ ] **Step 4: Run tests — expect GREEN**

```bash
cd api-gateway-service
./gradlew test --tests 'pro.masterdoc.gateway.UserScopeProxyRoutesTest' --no-daemon
```

Expected: BUILD SUCCESSFUL, all `UserScopeProxyRoutesTest` tests PASS.

- [ ] **Step 5: Commit, push, watch**

```bash
cd api-gateway-service
git add src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt \
        src/test/kotlin/pro/masterdoc/gateway/UserScopeProxyRoutesTest.kt
git commit -m "$(cat <<'EOF'
fix(gateway): admin write, board|admin read for user-scopes

Dispatchers keep GET candidates; only admin may PUT engineer bindings.
EOF
)"
git push
gh run watch
```

---

### Task 2: Filter engineer picker to `equipment` users

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/EngineerScopeScreen.kt`
- Create or extend test: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/EngineerScopeFilterTest.kt` (or add cases to `AssigneeLabelTest.kt` if preferred — prefer dedicated file)

**Interfaces:**
- Consumes: `AdminUser.features: List<String>`
- Produces: `internal fun filterEngineersForScopeBinding(users: List<AdminUser>): List<AdminUser>`

- [ ] **Step 1: Write failing unit test**

```kotlin
package pro.masterdoc.client.ui.screens

import kotlin.test.Test
import kotlin.test.assertEquals
import pro.masterdoc.client.auth.AdminUser

class EngineerScopeFilterTest {
    @Test
    fun filterEngineersForScopeBindingKeepsOnlyEquipment() {
        val users =
            listOf(
                AdminUser(
                    id = "eng",
                    email = "e@x.com",
                    givenName = "E",
                    familyName = "N",
                    features = listOf("equipment"),
                    state = "active",
                ),
                AdminUser(
                    id = "admin-only",
                    email = "a@x.com",
                    givenName = "A",
                    familyName = "D",
                    features = listOf("admin", "board"),
                    state = "active",
                ),
            )
        assertEquals(listOf("eng"), filterEngineersForScopeBinding(users).map { it.id })
    }
}
```

- [ ] **Step 2: Run test — expect RED (unresolved `filterEngineersForScopeBinding`)**

```bash
cd client-app
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.EngineerScopeFilterTest' --no-daemon
```

Expected: FAIL compile or unresolved reference.

- [ ] **Step 3: Implement filter + use in picker**

Add at bottom of `EngineerScopeScreen.kt` (or same package file):

```kotlin
internal fun filterEngineersForScopeBinding(users: List<AdminUser>): List<AdminUser> =
    users.filter { "equipment" in it.features }
```

In the dropdown branch, iterate filtered list:

```kotlin
val engineerUsers = filterEngineersForScopeBinding(users)
// ...
engineerUsers.forEach { user ->
    DropdownMenuItem(
        text = { Text(userLabel(user.id)) },
        onClick = {
            engineerId = user.id
            userMenuExpanded = false
            loadScope(user.id)
        },
    )
}
```

Use `engineerUsers.isNotEmpty()` (not raw `users`) for the dropdown vs manual-ID branch when `hasAdminUsers` is true: if admin directory loaded but no engineers, show empty-state text «Нет пользователей с фичей Оборудование» and do not fall back to listing all users.

- [ ] **Step 4: Run test — expect GREEN**

```bash
cd client-app
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.EngineerScopeFilterTest' --no-daemon
```

Expected: PASS.

- [ ] **Step 5: Commit (client-app; push deferred to Task 3 if same branch work continues in same session — otherwise push now)**

```bash
cd client-app
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/EngineerScopeScreen.kt \
        composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/EngineerScopeFilterTest.kt
git commit -m "$(cat <<'EOF'
feat(client): show only equipment users in engineer scope picker
EOF
)"
```

---

### Task 3: Move binding entry from Board to Admin → Users

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/UsersScreen.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/BoardScreen.kt`
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/shell/MainShellContent.kt`

**Interfaces:**
- Consumes: `UserScopesRepository`, `EquipmentRepository`, `AdminUsersRepository`, `EngineerScopeScreen`, `filterEngineersForScopeBinding`
- Produces: Admin Users hosts editor; Board has no binding button

- [ ] **Step 1: Extend `UsersScreen` signature and host editor**

```kotlin
@Composable
fun UsersScreen(
    repository: AdminUsersRepository,
    equipmentRepository: EquipmentRepository? = null,
    userScopesRepository: UserScopesRepository? = null,
    currentUserId: String? = null,
    modifier: Modifier = Modifier,
) {
    var tab by remember { mutableStateOf(AdminTab.Users) }
    var showInvite by remember { mutableStateOf(false) }
    var showScopeEditor by remember { mutableStateOf(false) }

    if (showInvite) { /* existing InviteUserScreen */ return }

    if (showScopeEditor && userScopesRepository != null && equipmentRepository != null) {
        EngineerScopeScreen(
            userScopesRepository = userScopesRepository,
            equipmentRepository = equipmentRepository,
            adminUsersRepository = repository,
            hasAdminUsers = true,
            recentAssigneeIds = emptyList(),
            onBack = { showScopeEditor = false },
            modifier = modifier,
        )
        return
    }

    // ... existing scaffold; pass onOpenScopeEditor into UsersTab when repos present
}
```

In `UsersTab`, next to «Пригласить»:

```kotlin
if (onOpenScopeEditor != null) {
    AppButton(
        text = "Привязка инженеров",
        onClick = onOpenScopeEditor,
        variant = AppButtonVariant.Secondary,
    )
}
```

Add imports for `UserScopesRepository` and `EngineerScopeScreen`.

- [ ] **Step 2: Wire `MainShellContent` Users branch**

```kotlin
NavDestinationId.Users ->
    if (adminUsersRepository != null) {
        UsersScreen(
            repository = adminUsersRepository,
            equipmentRepository = equipmentRepository,
            userScopesRepository = userScopesRepository,
            currentUserId = session.user?.id,
        )
    } else {
        StubDestinationScreen(active.destination)
    }
```

- [ ] **Step 3: Remove binding UI from `BoardScreen`**

- Delete `showScopeEditor` state.
- Delete the `if (dispatcherMode && showScopeEditor && …) { EngineerScopeScreen(...) }` block.
- Delete the «Привязка инженеров» `AppButton` in the board column.
- Keep `userScopesRepository`, `adminUsersRepository`, `equipmentRepository`, `hasAdminUsers` for `WorkOrderDetailScreen` assignee flow.
- Remove unused `EngineerScopeScreen` import if unused.

- [ ] **Step 4: Verify compile via targeted desktop test + existing assignee tests**

```bash
cd client-app
./gradlew :composeApp:desktopTest --tests 'pro.masterdoc.client.ui.screens.*' --no-daemon
```

Expected: PASS (including `EngineerScopeFilterTest`, `AssigneeLabelTest`).

- [ ] **Step 5: Commit, push, watch**

```bash
cd client-app
git add composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/UsersScreen.kt \
        composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/BoardScreen.kt \
        composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/shell/MainShellContent.kt \
        composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/EngineerScopeScreen.kt \
        composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/EngineerScopeFilterTest.kt
git commit -m "$(cat <<'EOF'
feat(client): move engineer binding from board to admin Users
EOF
)"
# If Task 2 commit already exists, this commit is UI move only — do not re-add filter files unless uncommitted.
git push
gh run watch
```

---

### Task 4: Amend product plan R8 (admin owner)

**Repo:** `masterdoc`

**Files:**
- Modify: `docs/plans/2026-07-27-001-feat-engineer-scope-binding-plan.md`
- Reference: `docs/superpowers/specs/2026-07-28-admin-engineer-scope-binding-design.md`

- [ ] **Step 1: Update R8 and actor A2 wording**

Change:

```markdown
- R8. Only the dispatcher role/capability path can create or change engineer scope bindings in v1.
```

to:

```markdown
- R8. Only the admin feature path (`admin`) can create or change engineer scope bindings; dispatcher retains read access for WO candidate lists. See `docs/superpowers/specs/2026-07-28-admin-engineer-scope-binding-design.md`.
```

In Key Decisions / Actors, change dispatcher-owns-binding bullet and A2 so binding UI ownership is Admin; dispatcher assigns WOs within scope only.

In A3 Admin: note admin owns binding UI/API write.

- [ ] **Step 2: Commit, push**

```bash
cd masterdoc
git add docs/plans/2026-07-27-001-feat-engineer-scope-binding-plan.md
git commit -m "$(cat <<'EOF'
docs: engineer scope binding owned by admin (R8)
EOF
)"
git push
```

---

## Self-review (plan vs spec)

| Spec requirement | Task |
| --- | --- |
| Admin entry in Users | Task 3 |
| Remove board entry | Task 3 |
| Picker = `equipment` only | Task 2 |
| Sites and/or Assets unchanged | (reuse screen; no task) |
| PUT = admin | Task 1 |
| GET = admin\|board | Task 1 |
| Amend R8 | Task 4 |
| Non-goals (no new feature, no filter model change) | respected |

No placeholders. Types match existing `AdminUser` / `proxyPrefix`.
