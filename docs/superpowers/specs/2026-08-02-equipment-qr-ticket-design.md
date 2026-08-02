# Equipment QR → ticket — Design

date: 2026-08-02  
repos: `catalog-service`, `api-gateway-service`, `client-app`

## Goal

Админ печатает QR-наклейку на единицу оборудования. Заявитель и инженер сканируют её (камера телефона или сканер в приложении), после логина видят **название** оборудования и создают аварийную заявку на этот актив.

## Product decisions (session-settled)

| Decision | Choice |
| --- | --- |
| Who generates QR | **Admin only** (`admin` feature) |
| Who scans | **Requester** (`tickets`) and **engineer** (`engineer`) |
| Scan channels | **Both**: HTTPS deep link on sticker + in-app camera scanner |
| Post-scan UX | **Same for both roles**: equipment **name** → «Создать заявку» |
| Unauthenticated sticker open | **Login only** — no public name or asset data |
| Token model | **Opaque `qrToken`** on asset (not raw `assetId` in QR payload) |
| Public name without login | Out of scope v1 |
| Batch PDF label print | Out of scope v1 |
| NFC | Out of scope v1 |

## Problem (current)

- Assets live in `catalog-service` (`Asset`: id, org, site, name, …) with no sticker identity.
- Deep links exist: `#/equipment/{assetId}` — opens equipment detail for users with `equipment`, not a ticket-create flow.
- Requester creates emergency WO from `TicketsScreen` by picking an asset from a dropdown.
- Engineer has `MyWorkOrders`, not Tickets nav; still needs QR → create emergency WO.
- UI rule: never show raw ids in user-facing copy — names only.

## Architecture (approach B — opaque token)

```
Admin (feature admin)
  EquipmentDetail → POST /assets/{id}/qr
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
   - `POST /assets/{id}/qr` — generate or rotate token; returns `{ qrToken, qrUrl }` (and updated asset). Admin-only at gateway.
   - `GET /assets/by-qr/{token}` — resolve within caller's `X-Org-Id`; return minimal DTO: `assetId`, `name`, `siteId` (and site name if cheap). 404 if unknown/rotated. Apply same scope rules as asset visibility used for WO create (user must be allowed to create WO on that asset).
   - Include `qrToken` on asset GET/list for callers that already see the asset (admin UI); do not require exposing token to requester/engineer list UIs.

2. **`api-gateway-service`**
   - Proxy new routes; enforce `admin` on `POST .../qr`.
   - Proxy `GET .../by-qr/{token}` for authenticated users with `tickets` or `engineer` (or any authenticated org member who can create WO — prefer feature gate: tickets ∪ engineer ∪ admin).

3. **`client-app`**
   - Deep link: `#/qr/{token}` → `AppDeepLink.AssetQr(token)` → dedicated screen (not full equipment passport).
   - Screen copy: equipment **name** only; CTA «Создать заявку»; reuse create payload shape from `TicketsScreen` (`emergency`, title from first line of description, `dueAt` today).
   - In-app scanner entry: visible when features include `tickets` or `engineer` (e.g. button on Tickets and My Work Orders, or shell action). Camera reads QR → parse `#/qr/{token}` or bare token → same screen.
   - Admin on `EquipmentDetailScreen`: QR block — show QR image + link if token exists; buttons «Сгенерировать» / «Перевыпустить»; confirm on rotate (old stickers break).
   - QR image: generate client-side from URL (KMP/Wasm-capable library or platform canvas). No server-side PNG required in v1.
   - After successful create: navigate to ticket/WO detail or tickets list (same pattern as Tickets).

4. **Auth / features**
   - No new feature catalog id required for v1.
   - Generate/rotate: `admin`.
   - Scan + create: `tickets` or `engineer` (engineer does not need Tickets nav item).

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
| Asset `draft` | Do not resolve for create (404 «Оборудование недоступно») — only `active` assets; admin may still generate QR on active assets only |
| Engineer without `tickets` nav | QR screen still creates WO via API |
| Camera permission denied | Message + fallback: paste link / open from phone camera |

## Out of scope v1

- Public unauthenticated asset name
- Batch label PDF / ZPL print
- NFC / barcode symbologies other than QR
- Separate QR microservice
- Auto-assign engineer from scan

## Success criteria

1. Admin can generate QR on an asset and see a scannable code encoding `#/qr/{token}`.
2. Requester and engineer, after login, reach name + create-ticket from sticker link and from in-app scan.
3. Guest sees only login; no asset name leaked.
4. Rotate invalidates old stickers.
5. Created WO is `emergency` bound to that asset; UI shows asset **name**.

## Test plan (summary)

- Catalog: token uniqueness, rotate, by-qr happy/404, scope denial.
- Gateway: admin-only POST qr; auth on by-qr.
- Client: deep link parse; QR screen create; admin generate UI; names not ids in copy.
- Smoke (Fixaverse Smoke): admin generate → open `#/qr/{token}` as requester/engineer → create WO → see name.
