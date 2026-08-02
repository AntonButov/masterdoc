# Equipment QR → ticket — Design

date: 2026-08-02  
repos: `feature-service`, `catalog-service`, `api-gateway-service`, `client-app`

## Goal

Админ печатает QR-наклейку на единицу оборудования. Заявитель и инженер сканируют её (камера телефона или сканер в приложении), после логина видят **название** оборудования и создают аварийную заявку на этот актив.

## Product decisions (session-settled)

| Decision | Choice |
| --- | --- |
| Who generates / prints QR | Separate product feature **`asset_qr`** («QR оборудования») — **not** bundled into `admin` / `equipment` |
| Default grant | Only role **`admin`** gets `asset_qr` in seed defaults; other roles do not |
| Who scans | **Requester** (`tickets`) and **engineer** (`engineer`) |
| Scan channels | **Both**: HTTPS deep link on sticker + in-app camera scanner |
| Post-scan UX | **Same for both roles**: equipment **name** → «Создать заявку» |
| Unauthenticated sticker open | **Login only** — no public name or asset data |
| Token model | **Opaque `qrToken`** on asset (not raw `assetId` in QR payload) |
| Public name without login | Out of scope v1 |
| Batch PDF label print | Out of scope v1 (single-asset QR + print/download from UI) |
| NFC | Out of scope v1 |

## Problem (current)

- Assets live in `catalog-service` (`Asset`: id, org, site, name, …) with no sticker identity.
- Deep links exist: `#/equipment/{assetId}` — opens equipment detail for users with `equipment`, not a ticket-create flow.
- Requester creates emergency WO from `TicketsScreen` by picking an asset from a dropdown.
- Engineer has `MyWorkOrders`, not Tickets nav; still needs QR → create emergency WO.
- UI rule: never show raw ids in user-facing copy — names only.

## Architecture (approach B — opaque token)

```
Caller with feature asset_qr (seed: admin role only)
  EquipmentDetail QR block → POST /assets/{id}/qr
  catalog stores Asset.qrToken
  client renders QR image from https://app.fixaverse.ru/#/qr/{token}
  → print / sticker on equipment

Requester / engineer
  A) Phone camera opens #/qr/{token}
  B) In-app «Сканировать QR» reads same URL/token
  → if not logged in: OIDC login (no asset name yet)
  → GET /assets/by-qr/{token} (auth + org + scope)
  → screen: asset name + description field + Create
  → POST /work-orders (type=emergency, assetId, siteId, …)
```

### Components

1. **`catalog-service`**
   - Add optional `qrToken: String?` on `Asset` (URL-safe, unique per org when non-null).
   - `POST /assets/{id}/qr` — generate or rotate token; returns `{ qrToken, qrUrl }` (and updated asset). Gateway requires feature **`asset_qr`**.
   - `GET /assets/by-qr/{token}` — resolve within caller's `X-Org-Id`; return minimal DTO: `assetId`, `name`, `siteId` (and site name if cheap). 404 if unknown/rotated. Apply same scope rules as asset visibility used for WO create (user must be allowed to create WO on that asset).
   - Include `qrToken` on asset GET for callers with `asset_qr` (print UI); do not show QR block to requester/engineer.

2. **`feature-service`**
   - Add catalog entry `asset_qr` / «QR оборудования».
   - Default role seed: `admin` → `["admin", "black_box", "equipment", "asset_qr"]`; other roles unchanged.
   - Backfill: orgs that already have `admin` role row get `asset_qr` appended to that role's features if missing (same pattern as requester backfill — only additive).

3. **`api-gateway-service`**
   - Proxy `POST /assets/{id}/qr` with feature gate **`asset_qr`** only (not plain `equipment` / not `admin` alone).
   - Proxy `GET /assets/by-qr/{token}` for `tickets` ∪ `engineer` ∪ `asset_qr` ∪ `admin`.
   - Mirror `asset_qr` in gateway `ProductFeatures` / client `FeatureId` lists used by `/me`.

4. **`client-app`**
   - Deep link: `#/qr/{token}` → `AppDeepLink.AssetQr(token)` → dedicated screen (not full equipment passport).
   - Screen copy: equipment **name** only; CTA «Создать заявку»; reuse create payload shape from `TicketsScreen` (`emergency`, title from first line of description, `dueAt` today).
   - In-app scanner entry: visible when features include `tickets` or `engineer` (e.g. button on Tickets and My Work Orders, or shell action). Camera reads QR → parse `#/qr/{token}` or bare token → same screen.
   - On `EquipmentDetailScreen`: QR print block **only if** `FeatureId.AssetQr` — show QR image + link; «Сгенерировать» / «Перевыпустить» / print; confirm on rotate (old stickers break).
   - QR image: generate client-side from URL (KMP/Wasm-capable library or platform canvas). No server-side PNG required in v1.
   - After successful create: navigate to ticket/WO detail or tickets list (same pattern as Tickets).

5. **Auth / features**
   - Wire id: **`asset_qr`** (titleRu: «QR оборудования»).
   - Generate / rotate / print UI: **`asset_qr`**.
   - Scan + create: `tickets` or `engineer` (engineer does not need Tickets nav item).
   - Org admins can later grant `asset_qr` to another role via Roles UI — product allows it; seed default remains admin-only.

### URL format

```
https://app.fixaverse.ru/#/qr/{qrToken}
```

Token: opaque URL-safe string (e.g. 22+ chars from secure random). Rotating replaces token; old QR returns 404 after resolve.

### UI names (workspace rule)

- Marker/sticker label and post-scan header: **asset name** (fallback «Оборудование»), never UUID / token / truncated id.
- Token appears only in URL / admin copy-link technical field if needed; prefer showing QR image + «Скопировать ссылку» without emphasizing token.

## Error / edge cases

| Case | Behavior |
| --- | --- |
| Unknown / rotated token | 404 → «Код не найден или устарел» |
| Logged in, wrong org / no scope | 403/404 → generic «Нет доступа» |
| Asset `draft` | Do not resolve for create (404 «Оборудование недоступно») — only `active` assets; `asset_qr` may generate QR only for `active` assets |
| User has `equipment` but not `asset_qr` | Sees equipment passport; **no** QR print block; `POST .../qr` → 403 |
| Engineer without `tickets` nav | QR screen still creates WO via API |
| Camera permission denied | Message + fallback: paste link / open from phone camera |

## Out of scope v1

- Public unauthenticated asset name
- Batch label PDF / ZPL print
- NFC / barcode symbologies other than QR
- Separate QR microservice
- Auto-assign engineer from scan

## Success criteria

1. User with `asset_qr` (default: admin role) can generate/print QR encoding `#/qr/{token}`; without `asset_qr` the UI and `POST .../qr` are denied.
2. Requester and engineer, after login, reach name + create-ticket from sticker link and from in-app scan.
3. Guest sees only login; no asset name leaked.
4. Rotate invalidates old stickers.
5. Created WO is `emergency` bound to that asset; UI shows asset **name**.

## Test plan (summary)

- feature-service: catalog `asset_qr`; admin seed includes it; other roles do not.
- Catalog: token uniqueness, rotate, by-qr happy/404, scope denial.
- Gateway: `asset_qr` required for POST qr; tickets/engineer for by-qr.
- Client: QR block gated by `AssetQr`; deep link parse; create WO; names not ids.
- Smoke (Fixaverse Smoke): admin (`asset_qr`) generate → open `#/qr/{token}` as requester/engineer → create WO → see name.
