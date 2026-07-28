# Admin owns engineer scope binding — Design

date: 2026-07-28  
amends: `masterdoc/docs/plans/2026-07-27-001-feat-engineer-scope-binding-plan.md` (R8 / actor A2 ownership)

## Goal

Перенести владение привязкой инженера (цеха и/или оборудование) с диспетчера на **админа**. Логика `user-scopes` и фильтрация списков инженера не меняются — меняются UI-вход и write ACL.

## Product decisions (session-settled)

| Decision | Choice |
| --- | --- |
| Owner of binding | Admin (`admin` feature) |
| UI entry | Админ → Пользователи → «Привязка инженеров» |
| Board entry | Removed completely |
| Engineer picker | Only users with `engineer` |
| Scope model | Sites and/or Assets (existing MF) |

## UI

1. `UsersScreen` (tab Пользователи): secondary button «Привязка инженеров».
2. Opens existing `EngineerScopeScreen` (same save → `PUT /user-scopes/{userId}`).
3. `BoardScreen`: remove «Привязка инженеров» button and `showScopeEditor` path. Keep `userScopesRepository` on the board for WO assignee `candidates` (dispatcher read).
4. Wire `userScopesRepository` (+ equipment) into `Users` page from `MainShellContent` for the binding editor.
5. Picker: when admin user list is loaded, show only users whose `features` contain `engineer`. Manual ID fallback stays for environments without admin directory (unchanged).

## ACL (api-gateway)

`/user-scopes` proxy today: `features = listOf("board")` for all methods.

Change to split gates:

| Method | Required feature(s) |
| --- | --- |
| `PUT` (and other non-GET) | `admin` |
| `GET` (`/{userId}`, `/covers/…`, `/candidates/…`) | `admin` **or** `board` |

Rationale: admin writes bindings; dispatcher still needs `candidates` / read when assigning WOs on the board. Catalog ↔ dashboard S2S `covers` stays direct (not via gateway) — no change.

## Non-goals

- Engineer identity = wire feature `engineer` (`equipment` = equipment catalog only)
- Changing Site/Asset union semantics, empty-scope behavior, or engineer list filtering
- Changing WO assign scope checks in dashboard
- Inline binding inside each user row

## Acceptance

- Admin with `admin` opens Админ → Пользователи → Привязка инженеров, sees only `engineer` users, saves sites/assets; engineer lists respect scope as before.
- User with only `board` cannot `PUT /user-scopes` (403); can still `GET …/candidates/{assetId}`.
- Board has no «Привязка инженеров» control.
- User without `engineer` does not appear in the engineer picker.

## Target repos

- `client-app` — move entry, filter picker by `engineer`
- `api-gateway-service` — read/write feature split on `/user-scopes`
- `masterdoc` — this spec; plan note that R8 owner is admin
