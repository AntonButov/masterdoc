# Tickets equipment list + WO photos Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the tickets create dropdown with an equipment list step, then a form with description + real photo upload via a new `attachment-service`, wired through dashboard `attachmentIds` and gateway.

**Architecture:** New Ktor `attachment-service` (port 8094) stores jpeg/png/webp blobs. `dashboard-service` keeps ordered id lists on work orders (JSON text column, max 10). Gateway proxies `/attachments` with the same feature ACL as `/work-orders`. Client adds `TicketsCreateStep.EquipmentList`, photo pickers (disk + camera), and detail-screen gallery/add.

**Tech Stack:** Kotlin 21, Ktor 3, Flyway, Compose Multiplatform, existing `MultipartBody` / `GatewayHttpClient.postBytes`, GitHub Actions Compose deploy.

**Spec:** `masterdoc/docs/superpowers/specs/2026-08-04-tickets-equipment-list-and-photos-design.md`

## Global Constraints

- UI copy: names/titles only — never raw UUIDs / user ids (fallback «Оборудование» / «Заявка»).
- Photos optional; max **10** per WO; types `image/jpeg|png|webp`; max **8 MiB** per file.
- No delete-after-save in MVP; remove only local previews before create.
- QR create path unchanged in this plan (may reuse attach helper later).
- Multi-repo: commit/push in the repo each task touches; watch Actions; no heavy local full builds.
- Prefer targeted unit/route tests; CI builds/deploys.

## File map

| Path | Role |
| --- | --- |
| `attachment-service/` (new repo clone under workspace) | Upload/store/serve images |
| `dashboard-service/.../V3__work_order_attachments.sql` | `attachment_ids` column |
| `dashboard-service/.../WorkOrderStore.kt` | Model, create, append |
| `dashboard-service/.../Application.kt` | `POST /work-orders/{id}/attachments` |
| `api-gateway-service/.../GatewayConfig.kt` | `attachmentServiceBaseUrl` |
| `api-gateway-service/.../EquipmentRoutes.kt` | Proxy `/attachments` |
| `client-app/.../WorkOrderModels.kt` | `attachmentIds` on DTO/request + attach API |
| `client-app/.../AttachmentsRepository.kt` | Upload + content URL |
| `client-app/.../TicketsCreateFlow.kt` | `EquipmentList` step |
| `client-app/.../TicketsScreen.kt` | List + form UI |
| `client-app/.../platform/PickImage.*` | Disk + camera pickers |
| `client-app/.../WorkOrderDetailScreen.kt` | Show + add photos |

---

### Task 1: Bootstrap `attachment-service` + route tests

**Repo:** `attachment-service` (create new git repo under `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/attachment-service`, mirror `document-service` layout: Gradle Ktor JVM, Dockerfile, `deploy/docker-compose.yml`, `.github/workflows/ci.yml`).

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradlew*`, `src/main/kotlin/pro/masterdoc/attachment/Application.kt`
- Create: `src/test/kotlin/pro/masterdoc/attachment/AttachmentRoutesTest.kt`
- Create: `Dockerfile`, `deploy/docker-compose.yml`, `.github/workflows/ci.yml` (port **8094**, image `masterdoc-attachment-service`, remote `/opt/attachment-service`)

**Interfaces:**
- Produces:
  - `POST /attachments` → `AttachmentMetaDto(id, orgId, contentType, sizeBytes, createdAt)`
  - `GET /attachments/{id}` → meta
  - `GET /attachments/{id}/content` → bytes + Content-Type
  - `GET /health` → `{status:ok}`
- Headers: `X-Org-Id` (default `default-org`), `X-User-Id` optional
- Reject non-image and size > 8 * 1024 * 1024

- [ ] **Step 1: Scaffold Gradle project** from `document-service` (copy gradle wrapper, drop pdfbox dependency). `mainClass` = `pro.masterdoc.attachment.ApplicationKt`. Port env `PORT` default `8094`, storage `ATTACHMENT_STORAGE_DIR`.

- [ ] **Step 2: Write failing tests** in `AttachmentRoutesTest.kt`:

```kotlin
@Test
fun uploadJpegAndGetContent() = testApplication {
    val dir = Files.createTempDirectory("att-test")
    application { module(AttachmentStore(dir)) }
    val jpeg = byteArrayOf(0xFF.toByte(), 0xD8.toByte(), 0xFF.toByte(), 0xD9.toByte()) // minimal marker bytes ok for test
    val response = client.submitFormWithBinaryData(
        url = "/attachments",
        formData = formData {
            append("file", jpeg, Headers.build {
                append(HttpHeaders.ContentType, "image/jpeg")
                append(HttpHeaders.ContentDisposition, "filename=photo.jpg")
            })
        },
    ) { header("X-Org-Id", "org-a") }
    assertEquals(HttpStatusCode.OK, response.status)
    val id = Json.parseToJsonElement(response.bodyAsText()).jsonObject["id"]!!.jsonPrimitive.content
    val content = client.get("/attachments/$id/content") { header("X-Org-Id", "org-a") }
    assertEquals(HttpStatusCode.OK, content.status)
    assertContentEquals(jpeg, content.bodyAsBytes())
}

@Test
fun rejectPdf() = testApplication { /* POST application/pdf → 400 */ }

@Test
fun orgIsolation() = testApplication { /* upload org-a, GET content with org-b → 404 */ }

@Test
fun rejectOversized() = testApplication { /* > 8MiB → 400 */ }
```

- [ ] **Step 3: Run tests — expect FAIL** (no module yet)

Run: `./gradlew test --tests pro.masterdoc.attachment.AttachmentRoutesTest`

- [ ] **Step 4: Implement** `AttachmentStore` + routes (disk: `{storage}/{orgId}/{id}.bin` + `{id}.json` meta). Accept only content types containing `jpeg`/`jpg`, `png`, or `webp` (normalize stored type to `image/jpeg|image/png|image/webp`).

```kotlin
@Serializable
data class AttachmentMetaDto(
    val id: String,
    val orgId: String,
    val contentType: String,
    val sizeBytes: Long,
    val createdAt: String,
)
```

- [ ] **Step 5: Run tests — expect PASS**

- [ ] **Step 6: Add Dockerfile + Compose + CI** (copy from document-service; change names/ports to 8094; health `http://127.0.0.1:8094/health`).

- [ ] **Step 7: Init git remote** on GitHub `masterdoc-app/attachment-service` if missing (`gh repo create`), commit, push `main`, watch Action until deploy success (or document missing secrets).

```bash
git add -A
git commit -m "feat: bootstrap attachment-service for WO photos"
git push -u origin main
gh run watch
```

---

### Task 2: dashboard — `attachmentIds` on work orders

**Repo:** `dashboard-service`

**Files:**
- Create: `src/main/resources/db/migration/V3__work_order_attachments.sql`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/WorkOrderStore.kt`
- Modify: `src/main/kotlin/pro/masterdoc/dashboard/Application.kt`
- Modify/Create tests: `src/test/kotlin/pro/masterdoc/dashboard/WorkOrderAttachmentTest.kt` (or extend `WorkOrderRoutesTest.kt`)

**Interfaces:**
- Consumes: attachment ids as opaque strings (no HTTP call to attachment-service)
- Produces:
  - `WorkOrder.attachmentIds: List<String> = emptyList()`
  - `CreateWorkOrderRequest.attachmentIds: List<String>? = null`
  - `POST /work-orders/{id}/attachments` body `AttachWorkOrderAttachmentsRequest(attachmentIds: List<String>)` → updated `WorkOrder`
  - Cap: if `current.size + new.distinct().size > 10` (counting unique append) → 400 `"Too many attachments"`

- [ ] **Step 1: Migration**

```sql
ALTER TABLE work_orders ADD COLUMN attachment_ids TEXT NOT NULL DEFAULT '[]';
```

- [ ] **Step 2: Failing tests** — create with two ids returns them; append third works; append that would exceed 10 returns 400; default empty list when column `[]`.

- [ ] **Step 3: Implement store**

Encode/decode with `Json.encodeToString(ListSerializer(String.serializer()), ids)` into `attachment_ids` TEXT. Update `INSERT`, `toWorkOrder()`, `createJdbc` to persist request ids (distinct, take at most 10; if request has >10 → throw `IllegalArgumentException`). Add `appendAttachments(orgId, id, ids): WorkOrder`.

- [ ] **Step 4: Route** in `Application.kt`:

```kotlin
post("/work-orders/{id}/attachments") {
    val orgId = call.orgId()
    val id = call.parameters["id"]!!
    val req = call.receive<AttachWorkOrderAttachmentsRequest>()
    try {
        call.respond(store.appendAttachments(orgId, id, req.attachmentIds))
    } catch (e: IllegalArgumentException) {
        call.respondText(e.message ?: "Bad Request", status = HttpStatusCode.BadRequest)
    }
}
```

- [ ] **Step 5: Tests PASS → commit/push/watch**

```bash
git commit -m "feat(work-orders): store attachmentIds on WO (max 10)"
git push && gh run watch
```

---

### Task 3: gateway — proxy `/attachments`

**Repo:** `api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/GatewayConfig.kt` — add `attachmentServiceBaseUrl`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt` — proxy route
- Modify: `.github/workflows/ci.yml` runtime.env — `ATTACHMENT_SERVICE_BASE_URL=http://host.docker.internal:8094`
- Modify: tests that construct `GatewayConfig` / `testDefaults()` to include the new field
- Add/adjust: proxy feature test if pattern exists for documents

**Interfaces:**
- Produces: `proxyPrefix("/attachments", config.attachmentServiceBaseUrl, client, deps, readFeatures = listOf("board", "engineer", "tickets"), writeFeatures = listOf("board", "engineer", "tickets"))`
- Env default: `http://127.0.0.1:8094`

- [ ] **Step 1: Update `testDefaults()` + any broken compile sites** with `attachmentServiceBaseUrl = "http://127.0.0.1:8094"`

- [ ] **Step 2: Add proxy line** next to `/documents` in `installEquipmentRoutes` (same ACL as `/work-orders`).

- [ ] **Step 3: Add CI env line** for VPS gateway.

- [ ] **Step 4: Targeted test compile/run** `./gradlew test --tests '*Equipment*'` (or full `test` in CI). Commit/push/watch.

```bash
git commit -m "feat(gateway): proxy /attachments for tickets/engineer/board"
git push && gh run watch
```

---

### Task 4: client auth — AttachmentsRepository + WO DTO fields

**Repo:** `client-app`

**Files:**
- Modify: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/WorkOrderModels.kt`
- Create: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/AttachmentsRepository.kt`
- Create: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/AttachmentsRepositoryTest.kt`
- Modify: `auth/src/jvmTest/kotlin/pro/masterdoc/client/auth/WorkOrdersRepositoryTest.kt` (create body includes attachmentIds)
- Wire factory in `AuthModule.kt` if repositories are constructed there

**Interfaces:**
- Produces:
  - `WorkOrderDto.attachmentIds: List<String> = emptyList()`
  - `CreateWorkOrderRequest.attachmentIds: List<String>? = null`
  - `WorkOrdersRepository.attach(orderId: String, attachmentIds: List<String>): WorkOrderDto`
  - `AttachmentsRepository.upload(bytes, filename, contentType): AttachmentMetaDto`
  - `AttachmentsRepository.contentUrl(id: String): String` → `{gateway}/attachments/{id}/content`

- [ ] **Step 1: Failing JVM tests** using `RecordingGatewayHttpClient` — upload posts multipart to `/attachments`; create encodes `attachmentIds`; attach posts `/work-orders/{id}/attachments`.

- [ ] **Step 2: Implement** models + `AttachmentsRepository` mirroring `EquipmentRepository.uploadManualPdf` via `MultipartBody.filePart`.

- [ ] **Step 3: Tests PASS → commit**

```bash
git commit -m "feat(auth): attachments upload + work order attachmentIds"
```

(Do not push until UI tasks land if preferred; or push now — either OK if main stays green.)

---

### Task 5: TicketsCreateFlow — `EquipmentList` step

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/TicketsCreateFlow.kt`
- Modify: `composeApp/src/commonTest/kotlin/pro/masterdoc/client/ui/screens/TicketsCreateFlowTest.kt`

**Interfaces:**
- Produces enum value `EquipmentList`
- `chooseList(Method) → EquipmentList`
- `selectEquipment(EquipmentList) → Form`
- `backFromForm(Form) → EquipmentList`
- `backFromEquipmentList(EquipmentList) → Method`
- `afterSuccessfulCreate(Form) → List`

- [ ] **Step 1: Rewrite failing tests** for the new transitions (replace old `chooseList → Form` / `backFromForm → Method`).

- [ ] **Step 2: Implement helpers** until PASS.

- [ ] **Step 3: Commit** `test+feat(tickets): equipment list create step`

---

### Task 6: Platform image pick (disk + camera)

**Repo:** `client-app`

**Files:**
- Create: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/platform/PickImage.kt`
- Create: `composeApp/src/{androidMain,desktopMain,wasmJsMain}/.../PickImage.*.kt`
- Optional: reuse patterns from `PickPdfFile.*` and `masterdocapp` `ImagePicker` / wasm `fixaverse-camera.js` if already present for client-app QR

**Interfaces:**
- Produces:
```kotlin
data class PickedImage(val bytes: ByteArray, val fileName: String, val contentType: String)

@Composable
expect fun rememberImagePickerLaunchers(
    onResult: (PickedImage?) -> Unit,
    onError: (String) -> Unit = {},
): ImagePickerLaunchers // openGallery, openCamera
```

- [ ] **Step 1: Add expect API** + android/desktop/wasm actuals (ImagePickerKMP if already a dependency; otherwise file picker for gallery + getUserMedia/canvas capture on wasm like QR camera). Prefer minimal: gallery via file input (`image/*`) on wasm/desktop; camera via existing camera JS if available, else gallery-only with clear error string «Камера недоступна» on platforms without cam.

- [ ] **Step 2: Smoke compile via CI** (no local wasm dist). Commit `feat(platform): image picker for ticket photos`.

---

### Task 7: TicketsScreen — EquipmentList + Form with photos

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/TicketsScreen.kt`
- Pass `AttachmentsRepository` from shell wiring (`MainShellContent` / wherever `TicketsScreen` is constructed)

**Interfaces:**
- Consumes: `EquipmentRepository.listAssets()`, `AttachmentsRepository.upload`, `WorkOrdersRepository.create(... attachmentIds)`
- UI:
  - `EquipmentList`: `AppScaffold("Новая заявка", back → Method)`; empty states; `LazyColumn` of `asset.name` (optional site name); onClick set `selectedAsset` + `selectEquipment`
  - `Form`: show selected asset **name** (no dropdown); description; photo strip (previews + remove local); buttons «С диска» / «Камера»; Create enabled when description non-blank and asset set (photos optional); on Create upload all then create; failure keeps previews

- [ ] **Step 1: Wire step UI** for `EquipmentList` and update `chooseList` / backs.

- [ ] **Step 2: Remove dropdown** from Form; show `selectedAsset?.name ?: "Оборудование"`.

- [ ] **Step 3: Photo state** `var pendingPhotos by remember { mutableStateOf<List<PickedImage>>(emptyList()) }` — add via picker; remove by index before submit; disable add at 10.

- [ ] **Step 4: Submit**

```kotlin
val ids = pendingPhotos.map { photo ->
    attachmentsRepository.upload(photo.bytes, photo.fileName, photo.contentType).id
}
repository.create(
    CreateWorkOrderRequest(
        type = "emergency",
        title = description.lineSequence().first().trim().take(120).ifBlank { "Заявка" },
        assetId = asset.id,
        siteId = asset.siteId,
        dueAt = IsoDates.formatEpochDay(localEpochDay()),
        description = description,
        attachmentIds = ids.ifEmpty { null },
    ),
)
```

- [ ] **Step 5: Commit** `feat(tickets): equipment list + photo attach on create`

---

### Task 8: WorkOrderDetailScreen — show + add photos

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/WorkOrderDetailScreen.kt`
- Wire `AttachmentsRepository` into detail from MyWorkOrders / Tickets / Board callers

**Interfaces:**
- Show horizontal/flow of thumbnails: load bytes via authenticated GET content URL (reuse whatever `OpenAuthenticatedDocument` / image decode pattern exists; if none, show count + «Открыть» using platform open with bearer — prefer inline decode if Coil/compose resources already used; else list filenames «Фото N» without ids).
- If `attachmentIds.size < 10` and not forbidden by readOnly-for-actions: allow add photo → upload → `repository.attach(orderId, listOf(id))` → reload. **Note:** tickets detail uses `readOnly = true` for lifecycle actions — still allow photo add (spec: anyone who sees WO). Gate add on presence of `AttachmentsRepository`, not on `readOnly`.

- [ ] **Step 1: Render attachment section** under description (title «Фото», empty → hide or «Нет фото»).

- [ ] **Step 2: Add photo control** with picker + upload + attach.

- [ ] **Step 3: Commit/push client-app main; watch CI+deploy**.

```bash
git commit -m "feat(work-orders): show and add photos on detail"
git push && gh run watch
```

---

### Task 9: End-to-end smoke (after green deploys)

**Repo:** orchestration only — skill `/smoke-test`

- [ ] Confirm `attachment-service`, `dashboard-service`, `api-gateway-service`, `client-app` Actions **success** on default branches.
- [ ] Smoke on `https://app.fixaverse.ru/` org **Fixaverse Smoke** (`383177088934346755`), login `mail+smoke@antonbutov.com`.
- [ ] Checklist:
  1. Заявки → Новая заявка → Список оборудования → list of names
  2. Select asset → form without dropdown; description + add photo
  3. Create → appears in Активные
  4. Open detail → photo visible / add another
  5. Regression: Method QR path still opens
- [ ] Screenshots via Playwright + Read; report PASS/FAIL.

---

## Spec compliance checklist

- [ ] Method → equipment list (not dropdown form)
- [ ] Form: description + photos; create with `attachmentIds`
- [ ] attachment-service deployed; types + 8 MiB enforced
- [ ] dashboard max 10; append endpoint
- [ ] gateway ACL tickets|engineer|board
- [ ] detail show + add
- [ ] no raw ids in UI copy
- [ ] QR path unchanged

## Self-review notes

- Size cap and max count pinned (8 MiB / 10) — matches spec.
- Storage format pinned to TEXT JSON array.
- Image picker task exists (disk + camera); wasm camera may degrade to error message if unavailable — still meets “sources: disk + camera” with platform best-effort.
- QR attach deferred explicitly.
