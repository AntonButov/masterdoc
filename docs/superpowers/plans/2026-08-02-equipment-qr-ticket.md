# Equipment QR → ticket Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Admin with `asset_qr` generates a sticker QR for an active asset; requester/engineer opens `#/qr/{token}` (or pastes/scans it), sees the asset **name**, and creates an emergency work order on that asset.

**Architecture:** Opaque `qrToken` on catalog `assets`; feature-gated generate (`asset_qr`) and resolve (`tickets`/`engineer`/`asset_qr`/`admin`); client deep-link screen + EquipmentDetail QR block; emergency WO via existing dashboard API.

**Tech Stack:** Kotlin/Ktor (catalog, gateway), Spring (feature-service), Compose Multiplatform client, Flyway Postgres, GitHub Actions deploy.

## Global Constraints

- Spec: `masterdoc/docs/superpowers/specs/2026-08-02-equipment-qr-ticket-design.md`
- UI: never show raw UUIDs/tokens as primary labels — asset **name** only (fallback «Оборудование»)
- URL: `https://app.fixaverse.ru/#/qr/{qrToken}`
- QR generate only for `active` assets; resolve only `active`
- `POST .../qr` requires feature `asset_qr` (not plain `equipment`/`admin` alone)
- `GET .../by-qr/{token}` for `tickets` ∪ `engineer` ∪ `asset_qr` ∪ `admin`
- Seed: only **admin** role gets `asset_qr` by default; additive backfill for existing admin rows
- Smoke tenant: Fixaverse Smoke `383177088934346755` only
- Commit/push each repo after its tasks; CI builds on GitHub (no local heavy Gradle)
- Workspace rule: Subagent-Driven execution (do not ask Inline vs Subagent)

## File map

| Area | Create / Modify |
| --- | --- |
| feature-service | `FeatureCatalog.kt`, `ProductRoleService.kt`, tests |
| catalog-service | `V2__asset_qr_token.sql`, `Asset` + DTOs, `JdbcAssetStore`, routes in `Application.kt`, tests |
| api-gateway-service | `ProductFeatures.kt`, custom asset proxy in `EquipmentRoutes.kt`, tests |
| client-app | `FeatureId`, deep link, equipment DTOs/repo, `AssetQrScreen`, EquipmentDetail QR block, shell wiring, optional paste-scan entry |

---

### Task 1: feature-service — `asset_qr` catalog + admin seed/backfill

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/feature-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/feature/features/FeatureCatalog.kt`
- Modify: `src/main/kotlin/pro/masterdoc/feature/roles/ProductRoleService.kt`
- Modify: `src/test/kotlin/pro/masterdoc/feature/features/FeatureCatalogTest.kt`
- Modify: `src/test/kotlin/pro/masterdoc/feature/api/RolesControllerTest.kt` (or equivalent role tests)

**Interfaces:**
- Produces: feature id wire string `"asset_qr"`, titleRu `"QR оборудования"`; admin default features include `asset_qr`

- [ ] **Step 1: Add catalog entry**

```kotlin
FeatureDefinition("asset_qr", "QR оборудования")
```

- [ ] **Step 2: Update admin DEFAULTS**

```kotlin
ProductRole("admin", "Админ", listOf("admin", "black_box", "equipment", "asset_qr")),
```

- [ ] **Step 3: Additive backfill in `listWithSeed`**

After missing-role insert, load roles; if `admin` exists and `"asset_qr" !in features`, `repository.update` with `features = (features + "asset_qr").distinct().sorted()` — do **not** reset other custom features.

- [ ] **Step 4: Update/add tests** — catalog contains `asset_qr`; fresh seed admin includes it; existing admin without it gets additive backfill; customized admin keeps extra features.

- [ ] **Step 5: Commit + push `main`**

```bash
git commit -m "feat(roles): add asset_qr feature for equipment QR stickers"
git push origin HEAD
gh run watch $(gh run list --branch main --limit 1 --json databaseId -q '.[0].databaseId') --exit-status
```

---

### Task 2: catalog-service — schema + store QR token

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/catalog-service`

**Files:**
- Create: `src/main/resources/db/migration/V2__asset_qr_token.sql`
- Modify: `src/main/kotlin/pro/masterdoc/catalog/Application.kt` (`Asset` + DTOs)
- Modify: `src/main/kotlin/pro/masterdoc/catalog/JdbcAssetStore.kt`
- Test: extend `AssetPersistenceTest` / new `AssetQrTest`

**Interfaces:**
- Produces:
  - `Asset.qrToken: String? = null`
  - `JdbcAssetStore.rotateQrToken(orgId, id): Asset` — only if `status == active`, else 400/404 via route
  - `JdbcAssetStore.findActiveByQrToken(orgId, token): Asset?`
  - Token: URL-safe Base64 without padding, ≥22 chars from `SecureRandom`

```sql
-- V2__asset_qr_token.sql
ALTER TABLE assets ADD COLUMN IF NOT EXISTS qr_token TEXT;
CREATE UNIQUE INDEX IF NOT EXISTS assets_org_qr_token_uq
  ON assets (org_id, qr_token)
  WHERE qr_token IS NOT NULL;
```

- [ ] **Step 1: Write failing persistence/route tests** for rotate + find + uniqueness
- [ ] **Step 2: Migration + mapper/SQL for `qr_token`**
- [ ] **Step 3: Implement rotate/find; commit + push; watch CI**

---

### Task 3: catalog-service — HTTP routes POST qr / GET by-qr

**Repo:** catalog-service

**Files:**
- Modify: `Application.kt` routes
- Test: `AssetRoutesTest.kt` (or new `AssetQrRoutesTest.kt`)

**Interfaces:**
- Produces:
  - `POST /assets/{id}/qr` → `AssetQrResponse(qrToken, qrUrl, asset)` where `qrUrl = "https://app.fixaverse.ru/#/qr/$qrToken"` (base URL constant ok)
  - `GET /assets/by-qr/{token}` → `AssetQrResolveResponse(assetId, name, siteId, siteName?)`
  - Non-active generate → 400; unknown token / non-active resolve → 404
  - When `X-Scope-Filter` enabled: resolve must `scopes.covers(orgId, userId, asset)` or 404/403 (prefer 404 to avoid leakage)

- [ ] **Step 1: Failing route tests**
- [ ] **Step 2: Implement routes**
- [ ] **Step 3: Commit + push; watch CI**

---

### Task 4: api-gateway — ProductFeatures + asset QR proxy gates

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/api-gateway-service`

**Files:**
- Modify: `src/main/kotlin/pro/masterdoc/gateway/ProductFeatures.kt`
- Modify: `src/main/kotlin/pro/masterdoc/gateway/EquipmentRoutes.kt`
- Modify: `ProductFeaturesTest.kt`
- Create/Modify: asset QR proxy tests (mirror `UserScopeProxyRoutesTest` style)

**Interfaces:**
- Replace generic `/assets` `proxyPrefix` with `installAssetProxyRoutes` that routes:
  - `POST /assets/{id}/qr` → require `asset_qr`
  - `GET /assets/by-qr/{token}` → require any of `tickets`,`engineer`,`asset_qr`,`admin`
  - other GET → existing `equipment`/`tickets` (keep engineer if currently missing for list — do not regress tickets)
  - other writes → `equipment`
- For `by-qr` and list/get: keep `scopeFilterHint` behavior; for `asset_qr` org-wide print, `POST .../qr` may use scope filter `0` when caller has `asset_qr` or `admin` (mirror board/admin org-wide)

- [ ] **Step 1: Failing gate tests** (tickets cannot POST qr; tickets can GET by-qr; equipment-only cannot POST qr)
- [ ] **Step 2: Implement**
- [ ] **Step 3: Commit + push; watch CI + deploy**

---

### Task 5: client-app — FeatureId, DTOs, repository methods

**Repo:** `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/client-app`

**Files:**
- Modify: `shared/.../NavModels.kt` — `AssetQr("asset_qr")`
- Modify: `shared/.../FeatureLabels.kt` — `"QR оборудования"`
- Modify: `auth/.../EquipmentModels.kt` — `AssetDto.qrToken`, `AssetQrResponseDto`, `AssetQrResolveDto`
- Modify: `EquipmentRepository` — `generateQr(assetId)`, `resolveQr(token)`
- Update FeatureLabelsTest / related exhaustive when

**Interfaces:**
- Produces:
  - `suspend fun generateQr(assetId: String): AssetQrResponseDto` → `POST /assets/{id}/qr`
  - `suspend fun resolveQr(token: String): AssetQrResolveDto` → `GET /assets/by-qr/{token}`
  - DTOs: `qrToken`, `qrUrl`, resolve: `assetId`, `name`, `siteId`, optional `siteName`

- [ ] **Step 1–4: TDD models/repo + commit** (push can wait until UI tasks if preferred, but prefer push when CI-green package is coherent)

---

### Task 6: client-app — deep link `#/qr/{token}` + AssetQrScreen + login restore

**Files:**
- Modify: `shared/.../AppDeepLink.kt` + tests
- Create: `composeApp/.../ui/screens/AssetQrScreen.kt`
- Modify: `MainShellContent` / `DefaultMainShellComponent` / `App.kt` for overlay navigation + pending deep link across OIDC

**Behavior:**
- Parse `#/qr/{token}` → `AppDeepLink.AssetQr(token)`
- Screen: loading → show **name** → description field → «Создать заявку» → `CreateWorkOrderRequest(type=emergency, title=first line, assetId, siteId, dueAt=today, description)`
- Errors: 404 «Код не найден или устарел»; 403 «Нет доступа»
- Guest: start login; persist pending hash `fixaverse.pending_deep_link` (or existing pattern) before redirect; restore after token exchange instead of hard `replaceTo("/")`
- Do **not** require Equipment nav destination

- [ ] **Step 1: Deep link unit tests**
- [ ] **Step 2: Screen + shell wiring**
- [ ] **Step 3: Login pending restore**
- [ ] **Step 4: Commit**

---

### Task 7: client-app — EquipmentDetail QR block (generate / rotate / show)

**Files:**
- Modify: `EquipmentDetailScreen.kt` — block when `canManageQr`
- Modify: shell call site — `canManageQr = FeatureId.AssetQr in session.features`
- Add QR image dependency suitable for KMP/Wasm (prefer Compose Multiplatform QR lib, e.g. qrose) **or** minimal platform canvas encoder; show QR for `qrUrl` + «Скопировать ссылку» + «Сгенерировать»/«Перевыпустить» with confirm on rotate
- Only for `active` assets; names not ids

- [ ] **Step 1: UI + generate/rotate**
- [ ] **Step 2: Commit**

---

### Task 8: client-app — scan/paste entry for tickets + engineer

**Files:**
- `TicketsScreen.kt`, `MyWorkOrdersScreen.kt` (or shell): button «Сканировать QR»
- Wasm/web v1: dialog to paste URL or token → navigate to AssetQrScreen (camera optional if cheap; paste satisfies sticker deep-link smoke)

- [ ] **Step 1: Paste entry + parse token from full URL or bare token**
- [ ] **Step 2: Commit + push client-app; fix Wasm CI if MapScreen ICE still red; watch deploy success**

---

### Task 9: Ops grant + smoke (Fixaverse Smoke)

**Checklist:**
1. Ensure feature-service deployed; admin role in Smoke has `asset_qr` (list roles / re-hit seed endpoint).
2. Admin user with `asset_qr` (+ `equipment`) logs into Smoke → open active asset → Generate QR → copy URL / save screenshot of QR.
3. Logout; login as engineer `mail+rustore@antonbutov.com` (or smoke requester) → open `#/qr/{token}` → see **name** → create emergency WO → verify list/detail shows asset name (not id).
4. Rotate QR as admin → old token 404.
5. Report per `/smoke-test` skill (PASS/FAIL, org, SHA, CI URLs, screenshots Read).

---

## Spec coverage

| Spec item | Task |
| --- | --- |
| `asset_qr` feature + admin seed/backfill | 1 |
| `qrToken` + POST/GET routes | 2–3 |
| Gateway gates | 4 |
| Client FeatureId + API | 5 |
| Deep link + create WO + login | 6 |
| Print/generate UI | 7 |
| In-app scan/paste | 8 |
| Smoke | 9 |

## Out of scope (do not implement)

- Public unauthenticated name
- Batch PDF labels
- NFC
- Auto-assign engineer
