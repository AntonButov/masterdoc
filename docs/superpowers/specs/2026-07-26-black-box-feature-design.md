# Black-box as a product feature (journal by user)

**Date:** 2026-07-26  
**Status:** approved (ready for plan)  
**Supersedes (UI):** Admin tab «Журнал» from [2026-07-24-sites-black-box-design.md](./2026-07-24-sites-black-box-design.md)

## Problem

Audit (black-box) is readable only as a global list under **Админ → Журнал**. Admins cannot pick a user. Journal access is tied to `admin`, not a dedicated product feature.

## Goal

- Journal is a **separate product feature** «Чёрный ящик» (`black_box`).
- On that screen: dropdown **Все | пользователь** → show that user’s events (or all).
- Any org user who has `black_box` can open the journal for **any** user in the org (read-only).

## Non-goals

- New storage or schema in black-box-service (filter `userId` already exists).
- Writing audit from the client feature screen (writes stay gateway fire-and-forget).
- Per-event redact UI / export / date range filters (can follow later).
- Giving `black_box` invite/delete/user-edit powers.

## Product model

| Wire id     | Title RU       | Nav |
|-------------|----------------|-----|
| `black_box` | Чёрный ящик    | Own primary nav item |

- **Админ** keeps: Пользователи | Площадки only (no «Журнал»).
- Invite / set-features checkbox list includes «Чёрный ящик».

## UX

Screen title: **Чёрный ящик**.

1. Dropdown «Пользователь»: first option **Все**, then org users (display: name · email; value: user id).
2. Button **Обновить**.
3. Event list (existing Russian titles via `AuditEventDescription`):
   - Filter **Все**: each row shows actor label (name/email if known, else truncated `userId`) under or beside the title.
   - Filter specific user: only that user’s events; actor label optional (already implied by filter).
4. Empty: «Пока нет событий».
5. Errors: same humanized gateway messages as Admin where applicable.

Default filter: **Все**. Changing dropdown reloads (or Reload uses current selection).

## API / auth

Keep path `GET /admin/audit` (no rename in this slice).

| Endpoint | Who |
|----------|-----|
| `GET /admin/audit?limit=&userId=` | feature `black_box` (not `admin`) |
| `GET /admin/users` (list only) | `admin` **or** `black_box` (dropdown needs the list) |
| Invite / set-features / delete / resend | `admin` only (unchanged) |

- Omit `userId` → all events for org (current black-box behavior).
- Pass `userId` → filter (already implemented in gateway + black-box-service).
- Client `listAudit(limit, userId: String? = null)` appends query when set.

Recording `audit.list` after successful read stays as today (actor = caller).

## Client changes

Repos: `client-app`

- `FeatureId.BlackBox("black_box")`, `NavDestinationId.BlackBox`, nav catalog entry (order near Admin, e.g. 65–75).
- `FeatureLabels` / titles: «Чёрный ящик».
- New screen (extract journal UI from `UsersScreen` `JournalTab`; Admin drops Journal tab + `AdminTab.Journal`).
- Wire in `MainShellContent` like other feature pages.
- `AdminUsersRepository.listAudit(userId: String? = null)`.

## Backend / IdP changes

| Repo | Change |
|------|--------|
| `feature-service` | Catalog entry `black_box` / «Чёрный ящик» |
| `api-gateway-service` | `ProductFeatures` entry; `GET /admin/audit` → `requireFeature(black_box)`; `GET /admin/users` → allow `admin` **or** `black_box`; other admin routes stay `requireAdmin` |
| `masterdoc-zitadel` | Project role key `black_box`; `expected.yaml` + Smoke/Demo invite role keys include `black_box` |
| `black-box-service` | No behavior change required |

## Data flow

```mermaid
sequenceDiagram
  participant UI as BlackBoxScreen
  participant GW as api-gateway
  participant FS as feature-service
  participant Users as ZitadelAdmin
  participant BB as black-box-service

  UI->>GW: GET /admin/users
  GW->>FS: features includes black_box?
  GW->>Users: list users
  Users-->>UI: items

  UI->>GW: GET /admin/audit?userId=
  GW->>FS: features includes black_box?
  GW->>BB: GET /events?orgId&userId&limit
  BB-->>UI: items
```

## Testing

- feature-service: catalog contains `black_box`; filter grants.
- gateway: audit 403 without `black_box`; 200 with; `userId` forwarded; users list 200 with only `black_box`; invite still 403 without `admin`.
- client: unit/UI tests for dropdown → query param; Admin has no Journal tab; nav shows feature when granted.
- Smoke (Fixaverse Smoke): grant `black_box`, open «Чёрный ящик», filter Все / one user, Админ without Журнал.

## Migration

1. Deploy feature-service + gateway + client + Zitadel role.
2. Add `black_box` to Smoke (and Demo if needed) user grants so journal remains usable after Admin tab removal.
3. Existing admins do **not** automatically see the nav item until granted `black_box` (explicit grant; document in runbook / ensure-smoke workflow).

## Related

- [2026-07-24-sites-black-box-design.md](./2026-07-24-sites-black-box-design.md) — audit store + former Admin journal tab  
- feature-service `FeatureCatalog` / gateway `ProductFeatures` must stay aligned  
