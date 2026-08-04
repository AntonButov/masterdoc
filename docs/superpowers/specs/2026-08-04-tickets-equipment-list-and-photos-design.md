# Tickets: equipment list + photo attachments — Design

**Date:** 2026-08-04  
**Status:** approved (spec)  
**Repos:** `attachment-service` (new), `dashboard-service`, `api-gateway-service`, `client-app`  
**Amends:** `2026-07-29-customer-tickets-design.md` (photos were out of scope); client-app `2026-08-03-tickets-create-method-picker-design.md` (list path was dropdown+description)

## Goal

На пути «Список оборудования» заменить dropdown отдельным экраном выбора актива; следующий шаг — описание проблемы + фото (диск/камера). Backend: отдельный `attachment-service` и `attachmentIds` на work order.

## Product decisions

| Topic | Choice |
| --- | --- |
| Approach | Separate `attachment-service` + list step before form |
| List contents | Equipment in requester scope (replaces Form dropdown) |
| Photos | Real upload (not UI-only stub) |
| Photo count | Optional, up to 10 per WO |
| Sources | Disk + camera (Wasm / Android / Desktop) |
| Delete after save | No (MVP); remove only before create submit |
| Detail | Show attachments + allow add for anyone who can see the WO |
| QR path | Unchanged in this plan (AssetQr create); may share attachmentIds later |
| UI names | Asset/WO titles only — never raw ids |

## Flow

```text
Заявки (list)
  [Новая заявка]
       │
       ▼
Method (QR hero / Список оборудования)
       │
       ├─ QR → existing AssetQr path
       └─ Список → EquipmentList → Form (описание + фото + Создать)
                 back: Form → EquipmentList → Method → Заявки
                 success → Заявки (reload)
```

## Architecture

### attachment-service (new)

Ktor service, pattern like `document-service` (Compose on API VPS, volume for blobs).

| Endpoint | Behavior |
| --- | --- |
| `POST /attachments` | multipart; accept `image/jpeg`, `image/png`, `image/webp`; return `{ id, contentType, sizeBytes }` |
| `GET /attachments/{id}` | metadata (org-scoped) |
| `GET /attachments/{id}/content` | binary |
| `GET /health` | liveness |

Suggested port: `8094` (next free after document `8093`).  
Reject unknown content types and bodies over **8 MiB**.  
Org id from gateway-forwarded header (same tenant pattern as siblings).

### dashboard-service

- Add `attachmentIds: List<String>` on `WorkOrder` (default empty).
- Flyway: store as JSON array column on `work_orders` (ordered list, max 10 elements).
- `CreateWorkOrderRequest.attachmentIds` optional.
- `POST /work-orders/{id}/attachments` body `{ "attachmentIds": ["…"] }` — append; **400** if total would exceed 10.
- Does **not** store file bytes; only references ids from attachment-service.
- No delete API in MVP.

### api-gateway-service

- Proxy `/attachments` → attachment-service.
- ACL: same feature gates as work-order read/write for tickets / engineer / board (and admin if already required for WO).
- Env: `ATTACHMENT_SERVICE_BASE_URL` (default `http://127.0.0.1:8094`).
- CI deploy job on `main` like sibling services.

### client-app

| Step | UI |
| --- | --- |
| `TicketsCreateStep.EquipmentList` (new) | Lazy list of assets by **name** (+ site label if available); empty states NoScope / NoEquipment |
| `TicketsCreateStep.Form` | Selected asset name (read-only), description, photo strip, Create |
| Detail | Thumbnails via content URL; «Добавить фото» → upload → `POST …/attachments` |

Create sequence: upload each photo → collect ids → `POST /work-orders` with `attachmentIds`. If any upload fails, do not create WO; show error.

Extend `TicketsCreateFlow` helpers:

- `chooseList` → `EquipmentList` (not Form)
- select asset → `Form`
- `backFromForm` → `EquipmentList`
- `backFromEquipmentList` → `Method`
- `afterSuccessfulCreate` → `List`

## Error handling

| Case | Behavior |
| --- | --- |
| Unsupported type / too large | attachment-service 4xx; client message without raw ids |
| Partial upload before create | Abort create; keep local previews so user can retry |
| Attach over cap 10 | dashboard 400; client disable add when at limit |
| Missing scope / no assets | Message on EquipmentList (same copy as today on Form) |

## Testing

- Flow unit tests for new step transitions.
- attachment-service: accept/reject types, org isolation, content round-trip.
- dashboard: create with ids, append, cap 10.
- gateway: proxy + feature gate smoke tests if pattern exists.
- client: create payload includes `attachmentIds`; list shows names not ids.

## Out of scope

- Delete attachments after save
- Video / HEIC / arbitrary files
- Client-side image compression pipeline (optional later)
- Changing QR create UX beyond optional shared attach helper
- Engineer «Мои заявки» separate create path

## Success criteria

1. Method → «Список оборудования» opens an asset list, not a dropdown form.
2. Selecting an asset opens description + photo form; Create persists WO with uploaded ids.
3. Detail shows photos; users who see the WO can add more (≤10).
4. No raw UUIDs/user ids in user-facing copy.
5. New service deploys via GitHub Actions on default branch.
