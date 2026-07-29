# Engineer assign push notifications Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a dispatcher assigns an engineer to a work order, that engineer’s Android app receives an FCM push; tap opens the work-order detail.

**Architecture:** New Ktor `notification-service` stores FCM device tokens and sends pushes. `api-gateway` proxies authenticated `/device-tokens`. `dashboard-service` best-effort calls `/internal/notify` after assignee change. `client-app` Android uses KMPNotifier 1.x to register tokens and deep-link on tap.

**Tech Stack:** Kotlin 2.1.10 / Ktor 3.0.3 (services), Java 21, FCM HTTP (PushSender interface), Compose Multiplatform client + `io.github.mirzemehdi:kmpnotifier:1.6.1`, GitHub Actions deploy.

**Spec:** [docs/superpowers/specs/2026-07-29-engineer-assign-push-design.md](../specs/2026-07-29-engineer-assign-push-design.md)

## Global Constraints

- Platforms v1: **Android only**; Wasm/desktop must not register tokens or require Firebase.
- Trigger: notify **only** when `assigneeId` changes to a **non-null** value different from previous.
- Assign PATCH must **succeed** even if notify/FCM fails (log + continue).
- Tap payload key exactly `workOrderId`.
- Push title RU: `Новая заявка`.
- Port for notification-service: **8097**.
- Internal auth header: `X-Internal-Token` / env `NOTIFICATION_INTERNAL_TOKEN` (same pattern as black-box).
- Client Kotlin is **2.2.21** → use KMPNotifier **1.6.1** (`NotifierManager`), **not** 2.0 (requires Kotlin 2.4+).
- No heavy local Wasm/full Gradle builds; targeted JVM/Android unit tests; CI builds on push.
- Multi-repo: commit + push each affected repo when its tasks finish; watch Actions for product repos.
- Do not expose `/internal/notify` on the public gateway.

## File map

| Area | Files |
|------|--------|
| notification-service (new) | `build.gradle.kts`, `settings.gradle.kts`, `gradlew*`, `Dockerfile`, `deploy/docker-compose.yml`, `.github/workflows/ci.yml`, `README.md`, `Application.kt`, `DeviceTokenStore.kt`, `PushSender.kt`, tests |
| api-gateway-service | `GatewayConfig.kt`, `EquipmentRoutes.kt` (or new `DeviceTokenRoutes.kt`), proxy-auth helper, tests |
| dashboard-service | `NotificationClient.kt`, `Application.kt` (assign hook), `deploy/docker-compose.yml`, tests |
| client-app | `libs.versions.toml`, `composeApp` Android deps + MainActivity, `DeviceTokensRepository`, `PushDeepLink`, `MyWorkOrdersScreen` / shell wiring, AuthModule |
| masterdoc | this plan + spec (already) |

---

### Task 1: Scaffold `notification-service` — health + device tokens

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/notification-service` (create new git repo)

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradle/wrapper/*` (copy from `black-box-service`), `src/main/kotlin/pro/masterdoc/notification/Application.kt`, `DeviceTokenStore.kt`
- Test: `src/test/kotlin/pro/masterdoc/notification/DeviceTokenRoutesTest.kt`
- Create: `README.md` (port 8097, env vars)

**Interfaces:**
- Produces: `DeviceTokenStore.upsert(userId, token, platform)`, `remove(userId, token)`, `tokensFor(userId): List<StoredToken>`
- Produces: `POST /device-tokens`, `DELETE /device-tokens?token=`, `GET /health`
- Consumes: headers `X-User-Id` (required for token routes)

- [ ] **Step 1: Scaffold project from black-box-service**

Copy `black-box-service` Gradle wrapper + structure. Set:

```kotlin
// settings.gradle.kts
rootProject.name = "notification-service"
```

```kotlin
// build.gradle.kts — match black-box-service deps; mainClass:
application { mainClass.set("pro.masterdoc.notification.ApplicationKt") }
```

Package: `pro.masterdoc.notification`. Default `PORT=8097`.

- [ ] **Step 2: Write failing route tests**

```kotlin
package pro.masterdoc.notification

import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.testing.*
import kotlin.test.*

class DeviceTokenRoutesTest {
    @Test
    fun healthOk() = testApplication {
        application { module(DeviceTokenStore()) }
        val r = client.get("/health")
        assertEquals(HttpStatusCode.OK, r.status)
    }

    @Test
    fun registerRequiresUserId() = testApplication {
        application { module(DeviceTokenStore()) }
        val r = client.post("/device-tokens") {
            contentType(ContentType.Application.Json)
            setBody("""{"token":"t1","platform":"android"}""")
        }
        assertEquals(HttpStatusCode.BadRequest, r.status)
    }

    @Test
    fun registerAndDelete() = testApplication {
        application { module(DeviceTokenStore()) }
        val created = client.post("/device-tokens") {
            header("X-User-Id", "eng-1")
            contentType(ContentType.Application.Json)
            setBody("""{"token":"t1","platform":"android"}""")
        }
        assertEquals(HttpStatusCode.Created, created.status)
        val deleted = client.delete("/device-tokens?token=t1") {
            header("X-User-Id", "eng-1")
        }
        assertEquals(HttpStatusCode.NoContent, deleted.status)
    }
}
```

- [ ] **Step 3: Run tests — expect FAIL**

```bash
cd /Users/antonbutov/Documents/MYPROJECTS/Fixaverse/notification-service
./gradlew test --tests 'pro.masterdoc.notification.DeviceTokenRoutesTest'
```

Expected: FAIL (classes missing) or compile errors.

- [ ] **Step 4: Implement store + routes**

```kotlin
// DeviceTokenStore.kt
data class StoredToken(
    val userId: String,
    val token: String,
    val platform: String,
    val updatedAt: String,
)

class DeviceTokenStore {
    private val byToken = java.util.concurrent.ConcurrentHashMap<String, StoredToken>()

    fun upsert(userId: String, token: String, platform: String): StoredToken {
        require(userId.isNotBlank()) { "X-User-Id required" }
        require(token.isNotBlank()) { "token required" }
        require(platform.isNotBlank()) { "platform required" }
        val row = StoredToken(userId, token, platform, java.time.Instant.now().toString())
        byToken[token] = row
        return row
    }

    fun remove(userId: String, token: String): Boolean {
        val existing = byToken[token] ?: return false
        if (existing.userId != userId) return false
        byToken.remove(token)
        return true
    }

    fun tokensFor(userId: String): List<StoredToken> =
        byToken.values.filter { it.userId == userId }
}
```

In `Application.kt`: Netty on `PORT` default 8097; `POST /device-tokens` receives `{token, platform}`, binds to `X-User-Id`; upsert; 201. `DELETE /device-tokens?token=` → 204 if removed else 404. `GET /health` → `{status:ok}`. StatusPages for `IllegalArgumentException` → 400.

- [ ] **Step 5: Run tests — expect PASS**

```bash
./gradlew test --tests 'pro.masterdoc.notification.DeviceTokenRoutesTest'
```

- [ ] **Step 6: Commit + create GitHub repo + push**

```bash
git init
git add .
git commit -m "feat: notification-service device token registry"
gh repo create Fixaverse/notification-service --private --source=. --remote=origin --push
```

(Use the same GitHub org as `dashboard-service` if not `Fixaverse` — check with `gh repo view` on a sibling.)

---

### Task 2: `notification-service` — internal notify + PushSender

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/notification-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/notification/PushSender.kt`
- Modify: `Application.kt`
- Test: `src/test/kotlin/pro/masterdoc/notification/NotifyRoutesTest.kt`

**Interfaces:**
- Consumes: `DeviceTokenStore.tokensFor`, `PushSender.send`
- Produces: `POST /internal/notify` body `{ userId, title, body, data: Map<String,String> }`
- Produces: `interface PushSender { fun send(tokens: List<String>, title: String, body: String, data: Map<String, String>): Int }` (returns success count)
- Produces: `RecordingPushSender` for tests; `LoggingPushSender` default when FCM creds absent

- [ ] **Step 1: Write failing notify tests**

```kotlin
@Test
fun notifySendsToAllUserTokens() = testApplication {
    val store = DeviceTokenStore()
    val sender = RecordingPushSender()
    application { module(store, sender = sender, internalToken = "secret") }
    client.post("/device-tokens") {
        header("X-User-Id", "eng-1")
        contentType(ContentType.Application.Json)
        setBody("""{"token":"t1","platform":"android"}""")
    }
    val r = client.post("/internal/notify") {
        header("X-Internal-Token", "secret")
        contentType(ContentType.Application.Json)
        setBody("""{"userId":"eng-1","title":"Новая заявка","body":"WO-1","data":{"workOrderId":"wo-1"}}""")
    }
    assertEquals(HttpStatusCode.OK, r.status)
    assertEquals(listOf("t1"), sender.lastTokens)
    assertEquals("wo-1", sender.lastData["workOrderId"])
}

@Test
fun notifyRequiresInternalTokenWhenConfigured() = testApplication {
    application { module(DeviceTokenStore(), internalToken = "secret") }
    val r = client.post("/internal/notify") {
        contentType(ContentType.Application.Json)
        setBody("""{"userId":"u","title":"t","body":"b","data":{}}""")
    }
    assertEquals(HttpStatusCode.Unauthorized, r.status)
}
```

- [ ] **Step 2: Run — expect FAIL**

```bash
./gradlew test --tests 'pro.masterdoc.notification.NotifyRoutesTest'
```

- [ ] **Step 3: Implement PushSender + route**

```kotlin
fun interface PushSender {
    /** @return number of tokens accepted by provider */
    fun send(tokens: List<String>, title: String, body: String, data: Map<String, String>): Int
}

class RecordingPushSender : PushSender {
    var lastTokens: List<String> = emptyList()
    var lastData: Map<String, String> = emptyMap()
    override fun send(tokens: List<String>, title: String, body: String, data: Map<String, String>): Int {
        lastTokens = tokens
        lastData = data
        return tokens.size
    }
}

class LoggingPushSender : PushSender {
    private val log = org.slf4j.LoggerFactory.getLogger("pro.masterdoc.notification.fcm")
    override fun send(tokens: List<String>, title: String, body: String, data: Map<String, String>): Int {
        log.info("event=fcm_noop count={} title={}", tokens.size, title)
        return tokens.size
    }
}
```

Wire `POST /internal/notify`: require internal token (copy `requireInternalToken` from black-box); resolve tokens; call sender; respond `{ "sent": N }`. If user has zero tokens → 200 `{ "sent": 0 }` (not an error).

Default production sender: `LoggingPushSender` until Firebase credentials exist. Optional later: `FirebasePushSender` reading `FCM_SERVICE_ACCOUNT_JSON` — **out of Task 2 scope** if logging sender satisfies tests; add a TODO comment in README for real FCM wiring + secret.

- [ ] **Step 4: Run — expect PASS; commit; push**

```bash
./gradlew test
git add -A && git commit -m "feat: internal notify endpoint with PushSender"
git push
```

---

### Task 3: `notification-service` — Docker + CI deploy

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/notification-service`

**Files:**
- Create: `Dockerfile`, `deploy/docker-compose.yml`, `deploy/.env.example`, `.github/workflows/ci.yml`, `.gitignore`

**Interfaces:**
- Produces: health on `http://127.0.0.1:8097/health` after deploy
- Env: `PORT`, `NOTIFICATION_INTERNAL_TOKEN`

- [ ] **Step 1: Dockerfile** (from black-box; expose 8097)

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /src
COPY gradlew settings.gradle.kts build.gradle.kts ./
COPY gradle ./gradle
COPY src ./src
RUN chmod +x gradlew && ./gradlew --no-daemon installDist -x test

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN apk add --no-cache curl
COPY --from=build /src/build/install/notification-service/ /app/
ENV PORT=8097
EXPOSE 8097
CMD ["/app/bin/notification-service"]
```

- [ ] **Step 2: compose + ci.yml**

Mirror `black-box-service/.github/workflows/ci.yml` with:

- `REMOTE_PATH: /opt/notification-service`
- `IMAGE_NAME: masterdoc-notification-service`
- `HEALTH_URL: http://127.0.0.1:8097/health`
- runtime.env: `NOTIFICATION_SERVICE_PORT=8097` and pass `NOTIFICATION_INTERNAL_TOKEN` from GitHub secret when present

`deploy/docker-compose.yml`:

```yaml
services:
  notification-service:
    build:
      context: ..
      dockerfile: Dockerfile
    image: ${SERVICE_IMAGE:-masterdoc-notification-service:latest}
    restart: unless-stopped
    ports:
      - "${NOTIFICATION_SERVICE_PORT:-8097}:8097"
    environment:
      PORT: "8097"
      NOTIFICATION_INTERNAL_TOKEN: "${NOTIFICATION_INTERNAL_TOKEN:-}"
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://127.0.0.1:8097/health"]
      interval: 5s
      timeout: 3s
      retries: 12
```

- [ ] **Step 3: Commit; push `main`/`master`; watch Action**

```bash
git add -A
git commit -m "ci: test and deploy notification-service to VPS"
git push
gh run watch
```

If deploy secrets missing on new repo: add `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_PRIVATE_KEY` (same as siblings) and optional `NOTIFICATION_INTERNAL_TOKEN`. Report run result.

---

### Task 4: Gateway — proxy `/device-tokens` (auth, any product feature)

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/GatewayConfig.kt`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt` (add proxy)
- Modify: deploy env / compose if gateway lists base URLs
- Test: `src/test/kotlin/pro/masterdoc/gateway/DeviceTokenProxyRoutesTest.kt` (new)

**Interfaces:**
- Consumes: `NOTIFICATION_SERVICE_BASE_URL` default `http://127.0.0.1:8097`
- Produces: `GatewayConfig.notificationServiceBaseUrl`
- Produces: authenticated proxy `POST|DELETE /device-tokens` → notification-service with `X-User-Id` / `X-Org-Id`
- Does **not** proxy `/internal/**`

- [ ] **Step 1: Write failing test**

Pattern from `WorkOrderProxyRoutesTest.kt`: without auth → 401; with bearer + any feature (e.g. engineer) → forwards (stub upstream or assert not 403 for missing feature if using broad feature list).

Use `proxyPrefix("/device-tokens", config.notificationServiceBaseUrl, ..., features = listOf("engineer", "board", "tickets", "admin", "equipment", "charts", "black_box"))` so any logged-in product user can register (matches spec: any authenticated user with features; users without grants still 403 — acceptable v1).

- [ ] **Step 2: Run — expect FAIL** (missing config field / route)

```bash
./gradlew test --tests 'pro.masterdoc.gateway.DeviceTokenProxyRoutesTest'
```

- [ ] **Step 3: Add config + route**

```kotlin
// GatewayConfig — add:
val notificationServiceBaseUrl: String,
// fromEnv:
notificationServiceBaseUrl =
    System.getenv("NOTIFICATION_SERVICE_BASE_URL") ?: "http://127.0.0.1:8097",
// testDefaults:
notificationServiceBaseUrl = "http://notification.test",
```

In `installEquipmentRoutes`:

```kotlin
proxyPrefix(
    "/device-tokens",
    config.notificationServiceBaseUrl,
    client,
    deps,
    features = listOf(
        "engineer", "board", "tickets", "admin",
        "equipment", "charts", "black_box",
    ),
)
```

Ensure no route exposes `/internal/notify`.

- [ ] **Step 4: Tests PASS; commit; push; watch Action**

```bash
git add -A
git commit -m "feat: proxy device-tokens to notification-service"
git push && gh run watch
```

Wire `NOTIFICATION_SERVICE_BASE_URL` on VPS gateway `.env` to `http://notification-service:8097` or host.docker.internal as used by siblings.

---

### Task 5: Dashboard — notify on assignee change

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/dashboard-service`

**Files:**
- Create: `src/main/kotlin/pro/masterdoc/dashboard/NotificationClient.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt`
- Modify: `deploy/docker-compose.yml` (+ `.env.example` if present)
- Test: extend `WorkOrderRoutesTest.kt`

**Interfaces:**
- Produces: `interface NotificationClient { fun notifyAssigned(userId: String, workOrderId: String, title: String, body: String) }`
- Produces: `HttpNotificationClient(baseUrl, internalToken)` → `POST {base}/internal/notify` with `X-Internal-Token`
- Produces: `NoOpNotificationClient` for existing tests default
- Consumes: previous `current.assigneeId` vs new assignee before/after `workOrderStore.update`

- [ ] **Step 1: Write failing test — assign triggers notify**

```kotlin
class RecordingNotificationClient : NotificationClient {
    val calls = mutableListOf<String>()
    override fun notifyAssigned(userId: String, workOrderId: String, title: String, body: String) {
        calls += "$userId:$workOrderId"
    }
}

@Test
fun assignNotifiesNewAssignee() = testApplication {
    val orders = WorkOrderStore()
    val notifier = RecordingNotificationClient()
    application {
        module(
            /* existing fakes */,
            workOrderStore = orders,
            notificationClient = notifier,
            featureLookup = FakeFeatureLookupClient(setOf("engineer-1")),
            // …match existing test module args
        )
    }
    // create WO as board, then patch assigneeId engineer-1
    // assert notifier.calls == listOf("engineer-1:<id>")
}

@Test
fun clearAssigneeDoesNotNotify() = testApplication { /* assign then clear; calls size unchanged */ }

@Test
fun reassignSameUserDoesNotNotifyTwice() = testApplication { /* patch same assigneeId again; one call */ }
```

Adapt constructors to current `module(...)` signature — add `notificationClient: NotificationClient = NoOpNotificationClient` at the end with default so old tests compile.

- [ ] **Step 2: Run — expect FAIL**

```bash
./gradlew test --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest.assignNotifiesNewAssignee'
```

- [ ] **Step 3: Implement client + hook**

```kotlin
interface NotificationClient {
    fun notifyAssigned(userId: String, workOrderId: String, title: String, body: String)
}

object NoOpNotificationClient : NotificationClient {
    override fun notifyAssigned(userId: String, workOrderId: String, title: String, body: String) = Unit
}

class HttpNotificationClient(
    baseUrl: String,
    private val internalToken: String?,
    private val client: HttpClient = /* CIO + JSON like FeatureLookupClient */,
) : NotificationClient {
    private val base = baseUrl.trimEnd('/')
    override fun notifyAssigned(userId: String, workOrderId: String, title: String, body: String) {
        runBlocking {
            try {
                client.post("$base/internal/notify") {
                    if (!internalToken.isNullOrBlank()) header("X-Internal-Token", internalToken)
                    contentType(ContentType.Application.Json)
                    setBody(
                        NotifyBody(
                            userId = userId,
                            title = title,
                            body = body,
                            data = mapOf("workOrderId" to workOrderId),
                        ),
                    )
                }
            } catch (e: Exception) {
                log.warn("event=notify_failed userId=$userId workOrderId=$workOrderId cause=${e.message}")
            }
        }
    }
}
```

In PATCH handler, **before** update capture `val previousAssignee = current.assigneeId`. After successful `update(...)`:

```kotlin
val updated = workOrderStore.update(...)
if (assigneePresent && updated.assigneeId != null && updated.assigneeId != previousAssignee) {
    runCatching {
        notificationClient.notifyAssigned(
            userId = updated.assigneeId!!,
            workOrderId = updated.id,
            title = "Новая заявка",
            body = updated.title,
        )
    }.onFailure { e -> log.warn("event=notify_hook_failed", e) }
}
call.respond(updated)
```

`main()`: read `NOTIFICATION_SERVICE_BASE_URL` (default `http://127.0.0.1:8097`) and `NOTIFICATION_INTERNAL_TOKEN`; wire `HttpNotificationClient`.

- [ ] **Step 4: Tests PASS; commit; push; watch**

```bash
./gradlew test --tests 'pro.masterdoc.dashboard.WorkOrderRoutesTest'
git add -A
git commit -m "feat: notify engineer on work-order assign"
git push && gh run watch
```

---

### Task 6: Client — DeviceTokensRepository + Android KMPNotifier register

**Repos:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/client-app`

**Files:**
- Modify: `gradle/libs.versions.toml`, `composeApp/build.gradle.kts`, AndroidManifest if needed
- Create: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/DeviceTokensRepository.kt`
- Create: `composeApp/src/androidMain/.../PushBootstrap.android.kt` (or in MainActivity)
- Create: `composeApp/src/commonMain/.../PushDeepLink.kt`
- Modify: `auth/.../AuthModule.kt`, `MainActivity.kt`, logout path
- Test: `auth/src/jvmTest/.../DeviceTokensRepositoryTest.kt`

**Interfaces:**
- Produces: `DeviceTokensRepository.register(token, platform)`, `unregister(token)` via `POST/DELETE /device-tokens`
- Produces: `object PushDeepLink { var pendingWorkOrderId: String? }` (or `MutableStateFlow`)
- Android: `NotifierManager.initialize` + `getPushNotifier().getToken()` + listener → register; onPayload/click → set `PushDeepLink.pendingWorkOrderId`
- Wasm/desktop: no Firebase init

- [ ] **Step 1: Repository test (fake HTTP)**

```kotlin
@Test
fun registerPostsJson() = runBlocking {
    val http = RecordingHttp() // capture method/url/body like other auth tests
    val repo = DeviceTokensRepository(config, http, tokenStore)
    repo.register("fcm-1", "android")
    assertTrue(http.lastUrl!!.endsWith("/device-tokens"))
    assertTrue(http.lastBody!!.contains("fcm-1"))
}
```

Use `http.postForm` with `Content-Type: application/json` (existing client pattern).

Unregister: `http.delete("${base()}/device-tokens?token=${urlEncode(token)}")`.

- [ ] **Step 2: Run — expect FAIL; implement repository; PASS**

- [ ] **Step 3: Add dependency**

```toml
# libs.versions.toml
kmpnotifier = "1.6.1"
# libraries:
kmpnotifier = { module = "io.github.mirzemehdi:kmpnotifier", version.ref = "kmpnotifier" }
```

```kotlin
// composeApp androidMain.dependencies
implementation(libs.kmpnotifier)
```

Do **not** add to commonMain/wasm (avoid Firebase on web). Shared register API can live in common via expect/actual or call repository only from androidMain.

- [ ] **Step 4: Android bootstrap in MainActivity**

After Koin start, if session will be ready later — register when user is logged in (observe session or call from shell `LaunchedEffect`):

```kotlin
// Pseudocode — use real NotifierManager 1.6 API from docs:
NotifierManager.initialize(
    NotificationPlatformConfiguration.Android(
        notificationIconResId = R.drawable.ic_notification, // add white silhouette icon
        // …
    )
)
NotifierManager.addListener(object : NotifierManager.Listener {
    override fun onNewToken(token: String) { /* launch register */ }
    override fun onPayloadData(data: PayloadData) {
        val id = data["workOrderId"] as? String
        if (!id.isNullOrBlank()) PushDeepLink.pendingWorkOrderId = id
    }
})
```

Also request notification permission (Android 13+). On logout (`AuthRepository.logout` / App logout): unregister last token if stored in memory/`TokenStore` companion.

Firebase: add `google-services` plugin + `google-services.json` **only when available**; until then document that local Android push needs the file — CI may skip Android assemble if secret absent. Prefer wiring placeholder path in README without committing secrets.

- [ ] **Step 5: Deep link into MyWorkOrders**

Modify `MyWorkOrdersScreen` to accept `initialOrderId: String? = null` and `onConsumedInitialOrder: () -> Unit = {}`:

```kotlin
LaunchedEffect(initialOrderId) {
    if (!initialOrderId.isNullOrBlank()) {
        selectedId = initialOrderId
        onConsumedInitialOrder()
    }
}
```

In `MainShellContent` for `MyWorkOrders` page: read `PushDeepLink.pendingWorkOrderId`, pass as `initialOrderId`, clear on consume. If pending id set while on another tab: select MyWorkOrders nav destination first (call existing select API on `MainShellComponent` if available; otherwise set pending and switch page index).

- [ ] **Step 6: Targeted tests; commit; push; watch client CI**

```bash
./gradlew :auth:jvmTest --tests 'pro.masterdoc.client.auth.DeviceTokensRepositoryTest'
# avoid full wasm build locally
git add -A
git commit -m "feat: Android FCM token register and WO deep link"
git push && gh run watch
```

---

### Task 7: Wire secrets + end-to-end smoke notes

**Repos:** masterdoc + VPS env (docs only if code already done)

**Files:**
- Modify: `notification-service/README.md`, optionally `masterdoc/docs/...` short runbook note
- Ensure gateway + dashboard compose env include notification base URL + shared internal token

- [ ] **Step 1: Document env matrix**

| Var | Where |
|-----|--------|
| `NOTIFICATION_INTERNAL_TOKEN` | notification-service, dashboard-service, (same value) |
| `NOTIFICATION_SERVICE_BASE_URL` | gateway, dashboard |
| `FCM_SERVICE_ACCOUNT_JSON` / Firebase | notification-service (when real sender added) |
| `google-services.json` | client-app Android (local/CI secret) |

- [ ] **Step 2: Manual smoke checklist (no automated e2e required in v1)**

1. Engineer Android build with Firebase → login → token registered (check notification-service logs).
2. Dispatcher assigns WO → engineer receives push «Новая заявка».
3. Tap → work order detail opens.
4. Stop notification-service → assign still 200.

- [ ] **Step 3: Commit docs if any; done**

---

## Self-review (plan vs spec)

| Spec item | Task |
|-----------|------|
| Android-only client | Task 6 |
| Assign-only trigger | Task 5 |
| notification-service tokens + FCM send | Tasks 1–3 (`LoggingPushSender` first; real FCM when creds) |
| Gateway device-tokens, no internal expose | Task 4 |
| Deep link `workOrderId` | Task 6 |
| Best-effort notify | Task 5 |
| Web Push / iOS out of scope | Not scheduled |

**Note:** Real FCM delivery needs Firebase project credentials after Task 2’s logging sender; treat production `FirebasePushSender` as follow-up once secrets exist — token registry + notify path remain testable.
