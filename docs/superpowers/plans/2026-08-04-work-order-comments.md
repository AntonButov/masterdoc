# Work order comments Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Append-only work-order comment feed (text + optional one photo) via new `comment-service`, gateway proxy, and detail UI.

**Architecture:** `comment-service` (port 8103) stores comment metadata (file-backed). Photos upload to existing `attachment-service`; comments store `attachmentId` only. Gateway proxies `/comments` with the same feature ACL as `/attachments`. Client shows feed + composer on `WorkOrderDetailScreen`.

**Tech Stack:** Kotlin 21, Ktor 3, Compose Multiplatform, GitHub Actions Compose deploy (mirror `attachment-service`).

**Spec:** `masterdoc/docs/superpowers/specs/2026-08-04-work-order-comments-design.md`

## Global Constraints

- UI: names only — `formatAssigneeLabel` / «Пользователь»; never raw user/attachment ids in copy.
- Text required, trim, max **2000**; optional one photo (jpeg/png/webp via attachment-service).
- No edit/delete.
- Multi-repo: commit/push in the repo each task touches; watch Actions; no heavy local full builds.
- Prefer targeted route/unit tests; CI builds/deploys.

## File map

| Path | Role |
| --- | --- |
| `comment-service/` (new) | Store + list/create comments |
| `api-gateway-service/.../GatewayConfig.kt` | `commentServiceBaseUrl` |
| `api-gateway-service/.../EquipmentRoutes.kt` | Proxy `/comments` |
| `client-app/.../CommentsRepository.kt` | Client API |
| `client-app/.../WorkOrderDetailScreen.kt` | Feed + composer |
| `client-app/.../MainShellContent.kt` (etc.) | Wire repository |

---

### Task 1: Bootstrap `comment-service` + route tests

**Repo:** `comment-service` — create under `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/comment-service`, mirror `attachment-service` (Gradle Ktor JVM, Dockerfile, compose, CI). Port **8103**, image `masterdoc-comment-service`, remote `/opt/comment-service`.

**Files:**
- Create: Gradle wrapper/project, `Application.kt`, `CommentStore`, tests, Dockerfile, `deploy/docker-compose.yml`, `.github/workflows/ci.yml`

**Interfaces:**
- `GET /health` → `{status:ok}`
- `GET /comments?workOrderId=` → `List<CommentDto>` ascending by `createdAt`
- `POST /comments` body `CreateCommentRequest(workOrderId, text, attachmentId?)` → `CommentDto` (201)
- Headers: `X-Org-Id`, `X-User-Id` (author)
- Reject blank/whitespace text; reject text length > 2000

- [ ] **Step 1:** Scaffold from `attachment-service` (drop image upload; keep Ktor JSON). `mainClass` = `pro.masterdoc.comment.ApplicationKt`. Env `PORT` default `8103`, `COMMENT_STORAGE_DIR`.

- [ ] **Step 2:** Failing tests `CommentRoutesTest.kt`: create + list round-trip; blank text → 400; oversize text → 400; org isolation (org-a comments not visible to org-b for same workOrderId).

- [ ] **Step 3:** Implement `CommentStore` — disk JSON list per `{storage}/{orgId}/{workOrderId}.json`.

- [ ] **Step 4:** Tests PASS.

- [ ] **Step 5:** Dockerfile + Compose + CI (health `http://127.0.0.1:8103/health`).

- [ ] **Step 6:** `gh repo create masterdoc-app/comment-service` if needed; commit; push `main`; watch deploy.

```bash
git commit -m "feat: bootstrap comment-service for work order comments"
git push -u origin main && gh run watch
```

---

### Task 2: Gateway proxy `/comments`

**Repo:** `api-gateway-service`

**Files:**
- Modify: `GatewayConfig.kt` (+ `testDefaults`)
- Modify: `EquipmentRoutes.kt`
- Create/extend: `CommentProxyRoutesTest.kt` (mirror `AttachmentProxyRoutesTest`)

**Interfaces:**
- Env `COMMENT_SERVICE_BASE_URL` default `http://127.0.0.1:8103`
- `proxyPrefix("/comments", ..., readFeatures/writeFeatures = board|engineer|tickets)`

- [ ] **Step 1:** Failing proxy ACL tests (no feature → 403; with tickets → forwarded).

- [ ] **Step 2:** Config + proxy.

- [ ] **Step 3:** Tests PASS; commit/push/watch.

```bash
git commit -m "feat(gateway): proxy /comments for tickets/engineer/board"
```

---

### Task 3: Client `CommentsRepository`

**Repo:** `client-app`

**Files:**
- Create: `auth/.../CommentsRepository.kt`
- Create: `auth/.../CommentsRepositoryTest.kt` (jvmTest RecordingGatewayHttpClient)

**Interfaces:**
```kotlin
@Serializable
data class WorkOrderCommentDto(
    val id: String,
    val orgId: String,
    val workOrderId: String,
    val authorId: String,
    val text: String,
    val attachmentId: String? = null,
    val createdAt: String,
)

@Serializable
data class CreateWorkOrderCommentRequest(
    val workOrderId: String,
    val text: String,
    val attachmentId: String? = null,
)

class CommentsRepository(...) {
    suspend fun list(workOrderId: String): List<WorkOrderCommentDto>
    suspend fun create(request: CreateWorkOrderCommentRequest): WorkOrderCommentDto
}
```

- [ ] **Step 1:** Failing JVM tests for GET query + POST body.

- [ ] **Step 2:** Implement.

- [ ] **Step 3:** Commit `feat(auth): CommentsRepository for work order feed`.

---

### Task 4: Detail UI — feed + composer

**Repo:** `client-app`

**Files:**
- Modify: `WorkOrderDetailScreen.kt`
- Wire `CommentsRepository` (+ existing `AttachmentsRepository`) from MyWorkOrders / Tickets / Board detail callers
- Optional small unit test for empty/composer enable rules if helpers extracted

**UI:**
- Section «Комментарии» under photos
- List: author label (`formatAssigneeLabel`), relative/ISO time, text, optional «Фото» thumb (reuse photo open pattern; label «Фото», not id)
- Empty: «Нет комментариев»
- Composer: `AppTextField` + disk/camera for one pending photo + «Отправить» enabled when text non-blank and not sending
- Flow: upload pending photo if any → `create` → clear composer → reload list
- Works when `readOnly = true` for lifecycle (gate on repository presence)

- [ ] **Step 1:** Load + render feed.

- [ ] **Step 2:** Composer + submit.

- [ ] **Step 3:** Wire repositories in shell callers.

- [ ] **Step 4:** Commit/push/watch deploy.

```bash
git commit -m "feat(work-orders): comment feed and composer on detail"
```

---

### Task 5: Smoke

**Skill:** `/smoke-test`

- [ ] All related Actions green on default branches.
- [ ] Org **Fixaverse Smoke**; open WO detail → see «Комментарии»; add text comment; optional photo if picker works.
- [ ] Report PASS/FAIL/PARTIAL.

---

## Spec compliance checklist

- [ ] comment-service deployed on 8103
- [ ] gateway `/comments` ACL
- [ ] feed chronological; names not ids
- [ ] text required; one optional photo via attachment-service
- [ ] no edit/delete
