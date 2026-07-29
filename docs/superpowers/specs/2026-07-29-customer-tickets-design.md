# Customer «Заявки» (requester / tickets) — Design

**Date:** 2026-07-29  
**Status:** approved (session decisions + docs)  
**Repos:** `client-app`, `feature-service`, `api-gateway-service`, `dashboard-service`, `catalog-service` (scopes reuse), `masterdoc-zitadel`  
**Amends:** features-only catalog (`tickets` was deferred); TOIR `requester` / `userRequests` → wire feature **`tickets`** (already in client `FeatureId.Tickets`)

## Goal

Заполнить stub навигации **«Заявки»** (`FeatureId.Tickets` / wire `tickets`): заказчик, привязанный к цеху(ам), создаёт аварийную заявку по оборудованию из своего scope, описывает проблему, видит свои активные и завершённые заявки, открывает read-only деталку.

## Product decisions (settled)

| Topic | Choice |
| --- | --- |
| Nav / feature | Existing stub `tickets` («Заявки»), not a new destination |
| Depth | End-to-end (catalog + ACL + WO + UI) |
| Photos | **Out of this plan** — separate follow-up (document-service is PDF-only today) |
| Site binding | Reuse `user-scopes` (same as engineer); admin binds requester → site(s) |
| List ownership | Only WOs **created by** this user |
| Create | Immediate `emergency` WorkOrder on dispatcher board; `createdBy` = caller |
| Detail | Read-only (`WorkOrderDetailScreen(readOnly = true)`); no start/close/assign/cancel |
| Architecture | One screen + extend WO model (no separate Ticket entity / `/tickets` API) |

## Non-goals

- Photo / image attachments (follow-up subtask when storage allows non-PDF)
- AI «Приёмщик» draft path
- Cancel / edit of own request
- Chat with engineer, SLA widgets, push
- Giving `tickets` users the `equipment` catalog UI (nav item) — only scoped asset picker on Tickets screen
- Changing engineer/dispatcher WO lifecycle

## Domain

### Feature wire

| Wire | Title RU | Who |
| --- | --- | --- |
| `tickets` | Заявки | Customer / requester portal |

Add to:

- `feature-service` `FeatureCatalog.ENTRIES`
- `masterdoc-zitadel` `terraform/expected.yaml` (+ terraform role if required by apply)
- Admin invite catalog (comes from feature-service `GET /features`)

Legacy docs mentioning `userRequests` / IdP role `requester` are superseded by features-only: access = grant `tickets`.

### WorkOrder extensions (`dashboard-service`)

| Field | Type | Rules |
| --- | --- | --- |
| `createdBy` | `String?` | Set on create from `X-User-Id` (never from client body). Required for tickets-originated WOs; backfill `null` for existing WOs |
| `description` | `String?` | Optional; customer form maps «Описание проблемы» here. Trim; max length 4000 |

Create request body (client → gateway → dashboard): existing `CreateWorkOrderRequest` + optional `description`. Type forced to `emergency` for tickets UI (client sends `emergency`). `dueAt` default = today (UTC date). `source` = `manual`. `title`: first line / truncated description (max ~120 chars), or explicit short title if UI keeps both — **MVP: one multiline field → `description`; `title` = truncated description** (non-blank).

List: `GET /work-orders?createdBy={userId}` (and existing `assigneeId`). When both omitted → org list (board/admin as today).

Statuses (unchanged): `new` \| `in_progress` \| `closed`.

| UI section | Statuses |
| --- | --- |
| Активные | `new`, `in_progress` |
| Завершённые | `closed` |

### Scope

- Admin binds users with feature `tickets` via `PUT /user-scopes/{userId}` (same payload: `siteIds` / optional `assetIds`).
- Gateway already sets `X-Scope-Filter: 1` for callers without `board`/`admin` → catalog assets and dashboard WO lists respect scope.
- Tickets create: asset must be in caller scope (dashboard/catalog covers check); reject otherwise.

## ACL (api-gateway)

| Route | Read features | Write features |
| --- | --- | --- |
| `/work-orders` | `board`, `engineer`, **`tickets`** | `board`, `engineer`, **`tickets`** |
| `/assets` | `equipment`, **`tickets`** | `equipment` (unchanged; tickets cannot mutate assets) |
| `/sites` | `equipment`, `admin`, **`tickets`** (read) | unchanged writes |
| `/user-scopes` | unchanged from admin-engineer-scope-binding (`admin`\|`board` read; `admin` write) | admin binds both engineers and customers |

Tickets callers:

- May `POST /work-orders` (emergency + description).
- May `GET /work-orders?createdBy=<self>` — gateway/dashboard **must** enforce: if caller has `tickets` and not `board`/`admin`/`engineer`, force filter `createdBy = X-User-Id` (ignore other users’ ids).
- May `GET /work-orders/{id}` only if `createdBy == self` (or caller has board/engineer/admin).
- Must not `PATCH` status/assignee (write gate can stay shared, but dashboard rejects PATCH for tickets-only / non-assignee — prefer **gateway writeFeatures for PATCH** stay `board`+`engineer` only if proxy can split methods; otherwise dashboard rejects PATCH when caller is not board/engineer). **Preferred:** split proxy: GET/POST allow `tickets`; PATCH only `board`\|`engineer`.

## UI (`client-app`)

### `TicketsScreen` (replaces stub for `NavDestinationId.Tickets`)

Single scrollable screen:

1. **Новая заявка**
   - Dropdown: assets from `listAssets()` (scoped by gateway).
   - Multiline: описание проблемы (required).
   - Submit → `create(emergency, title=trunc(description), description, assetId, siteId from asset, dueAt=today)`.
   - No photo control in this release (omit; follow-up adds attach).
2. **Активные** — list rows (title, status chip, asset name if resolved); tap → detail.
3. **Завершённые** — same, `closed` only.

Empty states: no scope → «Обратитесь к администратору для привязки к цеху»; no assets → «Нет оборудования в вашем цехе».

### Detail

Reuse `WorkOrderDetailScreen(..., readOnly = true)`. Show description when present. No mentor / assign / status buttons.

### Admin binding

Parallel to engineer binding:

- `UsersScreen`: button **«Привязка заказчиков»**.
- Reuse `EngineerScopeScreen` with `requiredFeature = "tickets"` (or `filterUsersForScopeBinding(users, "tickets")`) and title «Привязка заказчиков».
- Picker: users whose `features` contain `tickets`.

### Wiring

`MainShellContent`: `NavDestinationId.Tickets` → `TicketsScreen` with `WorkOrdersRepository`, `EquipmentRepository`, `currentUserId`.

## Error handling

| Case | Behavior |
| --- | --- |
| Create without asset / blank description | Client validation; 400 from API if missed |
| Asset out of scope | 400/403 from dashboard/catalog |
| List without session user id | Show error «Не удалось определить пользователя» (same as My WOs) |
| GET foreign WO as tickets user | 404 |

## Testing

- feature-service: catalog includes `tickets`
- gateway: tickets may GET/POST WO; PATCH forbidden; assets GET allowed; scope filter header for tickets-only
- dashboard: `createdBy` set from header; `?createdBy=` filter; description round-trip; tickets isolation on get-by-id
- client: nav Tickets → real screen; create splits active/closed; detail read-only; admin customer scope filter helper
- zitadel expected.yaml lists `tickets`

## Follow-up (photos)

Separate plan after storage supports images (extend document-service beyond PDF **or** dedicated attachment service): attach photo ids on WO create/update; UI picker on Tickets form; display on detail. Not blocking this design.

## Acceptance

1. User with only `tickets` (+ profile) sees nav «Заявки», not board/equipment.
2. After admin binds site(s), picker shows only that site’s assets.
3. Create emergency WO → appears on dispatcher board; `createdBy` = customer.
4. Active / completed lists show only own WOs; detail is read-only.
5. No photo UI required for acceptance of this feature.
6. Invite catalog offers feature «Заявки» (`tickets`).

## Related

- [2026-07-21-features-only-access-design.md](../../../../feature-service/docs/superpowers/specs/2026-07-21-features-only-access-design.md) — features-only model
- [2026-07-28-admin-engineer-scope-binding-design.md](./2026-07-28-admin-engineer-scope-binding-design.md) — scopes + admin ownership
- [TOIR_AI_SYSTEM_DESIGN.md](../../../TOIR_AI_SYSTEM_DESIGN.md) — requester intent (superseded wire name)
- B2B statuses: новая → в работе → закрыта
