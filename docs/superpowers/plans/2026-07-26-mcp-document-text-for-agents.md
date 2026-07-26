# MCP Document Text for Agents Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop sending PDF base64 to OpenRouter; agents read all equipment documents via document-mcp `get_document_text` after a skeleton draft asset exists.

**Architecture:** Validator creates (or reuses) a draft asset, then LLM uses document-mcp tools only. Equipment-card enriches via `update_draft_asset`. OpenRouter messages are text + tools; `/documents/{id}/text` must return real extracted text (PDFBox), not raw PDF bytes.

**Tech Stack:** Kotlin 2.1 / Ktor 3, PDFBox, OpenRouter function tools, multi-repo Fixaverse services + client-app.

**Spec:** `masterdoc/docs/superpowers/specs/2026-07-26-mcp-document-text-for-agents-design.md`

## Global Constraints

- No PDF/`file`/`file-parser` in OpenRouter request bodies after Task 5.
- Draft writes stay `status=draft`, `source=ai_generated`.
- On validator reject of a **newly created** skeleton: call catalog `POST /assets/{id}/reject`; keep the uploaded document.
- Default text chunk: offset=0, limit=16000, hard cap 32000.
- Smoke org mocks unchanged (no OpenRouter).
- Per Fixaverse rules: commit + push each repo task; watch Actions; no heavy local Gradle unless a tiny targeted test.
- Workspace roots are separate git repos under `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/`.

## File map

| Repo | Files |
|------|--------|
| document-service | `build.gradle.kts`, `Application.kt` (`readText`), tests |
| catalog-service | `Application.kt` (PATCH asset, link docs), tests |
| document-mcp | `Application.kt` (`get_document_text`), tests |
| technologist-mcp | `Application.kt` (`update_draft_asset`), tests |
| technologist-service | `Agents.kt`, `Application.kt`, tests |
| client-app | `EquipmentModels.kt`, `EquipmentScreen.kt` |

---

### Task 1: document-service — real PDF text extraction

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/document-service`

**Files:**
- Modify: `build.gradle.kts` — add `org.apache.pdfbox:pdfbox:3.0.3`
- Modify: `src/main/kotlin/pro/masterdoc/document/Application.kt` — `DocumentStore.readText`
- Modify: `src/test/kotlin/pro/masterdoc/document/DocumentRoutesTest.kt`

**Interfaces:**
- Produces: `GET /documents/{id}/text` → `{"text":"<extracted>"}` with readable text for real PDFs (not `%PDF-` binary garbage). Prefer `.source.txt` sidecar when present; else PDFBox `PDFTextStripper`; if still blank, return empty string (caller/MCP treats as failure).

- [ ] **Step 1: Write failing test** — upload a minimal valid PDF with known text (use existing `MinimalPdf` helpers or write a tiny PDF with ASCII text), `GET /text`, assert body contains that text and does **not** start with `%PDF`.

- [ ] **Step 2: Implement PDFBox extraction in `readText`** — keep sidecar and legacy fake-PDF branch; for real `%PDF` with xref use PDFBox.

- [ ] **Step 3: Run** `./gradlew test --tests 'pro.masterdoc.document.DocumentRoutesTest' --no-daemon -q` — PASS.

- [ ] **Step 4: Commit + push + watch Actions**

```bash
git add build.gradle.kts src/main/kotlin/pro/masterdoc/document/Application.kt src/test/kotlin/pro/masterdoc/document/DocumentRoutesTest.kt
git commit -m "feat(documents): extract PDF text with PDFBox for /text"
git push origin HEAD
gh run watch $(gh run list --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 2: catalog-service — PATCH draft asset + link documents

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/catalog-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/catalog/Application.kt`
- Modify: `src/test/kotlin/pro/masterdoc/catalog/AssetRoutesTest.kt`

**Interfaces:**
- Produces:
  - `PATCH /assets/{id}` body `UpdateAssetRequest(name?, inventoryNo?, category?, description?, documentIds?)` — merge fields; if `documentIds` provided, **union** with existing (do not drop unspecified ids unless explicitly replaced — prefer union for link safety).
  - Or dedicated `POST /assets/{id}/link-documents` `{documentIds:[]}` that unions ids (either is fine; prefer PATCH with optional documentIds union + field patches).
- Reuse existing `POST /assets/{id}/reject` for skeleton cleanup (already deletes draft).

- [ ] **Step 1: Failing tests** for PATCH name/description and linking a documentId onto an asset.

- [ ] **Step 2: Implement** `UpdateAssetRequest` + `AssetStore.update` + route.

- [ ] **Step 3: Targeted test PASS → commit + push + watch.**

```bash
git commit -m "feat(catalog): PATCH assets for draft enrich and document link"
```

---

### Task 3: document-mcp — `get_document_text`

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/document-mcp`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/document/mcp/Application.kt`
- Modify: `src/test/kotlin/pro/masterdoc/document/mcp/McpToolsTest.kt`

**Interfaces:**
- Produces tool `get_document_text` args: `documentId` (required), `offset` (default 0), `limit` (default 16000, max 32000).
- Upstream: `GET {documentBaseUrl}/documents/{id}/text` → parse `text`, slice `[offset, offset+limit)`, return:

```json
{
  "documentId": "...",
  "offset": 0,
  "length": 123,
  "truncated": false,
  "totalChars": 500,
  "text": "..."
}
```

- If upstream fails or `text` blank → `ok:false`, error message.

- [ ] **Step 1: Failing test** — mock upstream returns long text; tool with limit=10 returns truncated=true and length=10; listTools includes `get_document_text`.

- [ ] **Step 2: Implement tool** in `DocumentTools`.

- [ ] **Step 3: PASS → commit + push + watch.**

```bash
git commit -m "feat(document-mcp): add get_document_text chunked tool"
```

---

### Task 4: technologist-mcp — `update_draft_asset`

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/technologist-mcp`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/technologist/mcp/Application.kt`
- Modify: `src/test/kotlin/pro/masterdoc/technologist/mcp/McpToolsTest.kt`

**Interfaces:**
- Produces `update_draft_asset` args: `assetId` (required), optional `name`, `inventoryNo`, `category`, `description`, `documentIds`.
- Upstream: `PATCH {catalogBaseUrl}/assets/{assetId}` with those fields.
- Optional helper tool or reuse: cleanup via catalog reject can stay in technologist-service HTTP client (not required in MCP).

- [ ] **Step 1: Failing test** for update_draft_asset in allowed list + happy path mock.

- [ ] **Step 2: Implement.**

- [ ] **Step 3: PASS → commit + push + watch.**

```bash
git commit -m "feat(technologist-mcp): add update_draft_asset tool"
```

---

### Task 5: technologist-service — text-only LLM + skeleton validator flow

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/technologist-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/technologist/Agents.kt`
- Modify: `src/main/kotlin/pro/masterdoc/technologist/Application.kt`
- Modify: `src/main/kotlin/pro/masterdoc/technologist/TechnologistRunner.kt`
- Modify: `src/test/kotlin/pro/masterdoc/technologist/AgentsTest.kt`
- Modify: `src/test/kotlin/pro/masterdoc/technologist/TechnologistJobTest.kt` (if needed)

**Interfaces:**
- `OpenRouterClient.complete`: remove `file` parts and `file-parser` plugin; send text `system` only; keep tools.
- `DocumentValidatorAgent.validate`: no pdf bytes; tools include `get_document_text`, `list_asset_documents`, `get_document_meta`, `get_asset`, `find_documents_by_sha256`.
- Application validator route:
  1. If `assetId` null → MCP `create_draft_asset` skeleton (`name=Черновик документа`, `siteId`, `documentIds=[documentId]`), remember `createdSkeleton=true`.
  2. Else ensure document linked via catalog PATCH/MCP update.
  3. Run validator with `knownArgs` assetId+documentId.
  4. On reject && createdSkeleton → `POST catalog /assets/{id}/reject` (add thin HttpCatalogClient or call via MCP if you add reject tool; prefer direct HTTP to catalog like documents client).
  5. Respond with `draftAssetId` always when asset resolved.
- `EquipmentCardRequest` add `assetId: String`; agent uses `update_draft_asset` + text tools (not create).
- `TechnologistAgent`: drop pdf parameter; tools + text MCP.
- Smoke paths unchanged.

- [ ] **Step 1: Tests** — OpenRouter request builder has no `file_data`; validator injects get_document_text; equipment card calls update_draft_asset; reject cleans skeleton (unit with fakes).

- [ ] **Step 2: Implement.**

- [ ] **Step 3: PASS targeted tests → commit + push + watch.**

```bash
git commit -m "feat(technologist): MCP document text instead of PDF-in-LLM"
```

---

### Task 6: client-app — pass draftAssetId into equipment-card

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/Fixaverse/client-app`

**Files:**
- Modify: `auth/src/commonMain/kotlin/pro/masterdoc/client/auth/EquipmentModels.kt` — `DocumentValidationResponse.draftAssetId`, `EquipmentCardRequest.assetId`, `createEquipmentCard(..., assetId)`.
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/EquipmentScreen.kt` — `runValidateThenCard` / `runEquipmentCard` pass asset id from validation.

- [ ] **Step 1: Update DTOs and screen flow.**

- [ ] **Step 2: Commit + push + watch** (client CI).

```bash
git commit -m "feat(client): use validator draftAssetId for equipment-card"
```

---

### Task 7: Smoke check on VPS logs

**No code** — after all deploys green:

- [ ] SSH and confirm technologist startup still healthy.
- [ ] Optionally trigger validator once (or ask user to upload); confirm logs show `get_document_text` and **no** large `pdfBytes=` in openrouter_request (or pdfBytes absent / 0).

---

## Spec coverage checklist

| Spec item | Task |
|-----------|------|
| No base64 PDF in OpenRouter | 5 |
| `get_document_text` chunked | 3 (+1 for real extract) |
| Skeleton draft then validate | 5 |
| `update_draft_asset` enrich | 4, 5 |
| Reject deletes skeleton only | 5 (catalog reject) |
| Client passes draftAssetId | 6 |
| `/text` usable for real PDF | 1 |

## Self-review notes

- Placeholder scan: none intended.
- `link` via PATCH union covers “ensure documentId on existing asset”.
- PDFBox added because current `readText` returns binary PDF as UTF-8 — without Task 1, MCP text is useless for real uploads.
