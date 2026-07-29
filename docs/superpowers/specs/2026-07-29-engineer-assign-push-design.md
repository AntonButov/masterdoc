# Engineer assign push notifications — Design

date: 2026-07-29  
library: [KMPNotifier](https://github.com/mirzemehdi/KMPNotifier) (`kmpnotifier-push-firebase`)

## Goal

Когда диспетчер назначает инженера на заявку, Android-клиент инженера получает FCM push. Тап открывает карточку этой заявки.

## Product decisions (session-settled)

| Decision | Choice |
| --- | --- |
| Platforms v1 | **Android only** (desktop/Wasm: no push) |
| Web Push | Out of scope v1 (possible later via Web Push API + Service Worker, not KMPNotifier) |
| Trigger | **Only on assign** — `assigneeId` changed to a non-null user |
| Token store + FCM send | Dedicated **`notification-service`** |
| Tap action | Open **work-order detail** (`workOrderId` in payload) |
| Client library | KMPNotifier + Firebase Cloud Messaging |
| Assign vs notify failure | Assign succeeds even if FCM/notify fails (best-effort) |

## Problem (current)

- Feature `engineer` exists; WO assign lives in `dashboard-service`.
- `client-app` has Android + desktop + Wasm; no push/notification stack.
- No device-token registry and no FCM sender in the platform.

## Architecture (approach A — thin notification-service, sync hook)

```
Android client (KMPNotifier)
  → POST /device-tokens (via api-gateway, user JWT)
  → notification-service stores (userId, token, platform)

Dispatcher assigns WO
  → dashboard-service PATCH assigneeId succeeds
  → dashboard → POST internal notify { userId, title, body, data }
  → notification-service → FCM → Android

Tap notification
  → payload.workOrderId → WorkOrderDetailScreen
```

### Components

1. **`client-app` (Android)**  
   - Init KMPNotifier with Firebase push.  
   - Request notification permission; obtain FCM token; register with backend when user is authenticated and has feature `engineer` (or always when logged in — tokens are user-scoped; send path filters by assignee).  
   - On token refresh: re-register.  
   - On logout: unregister / delete token.  
   - Push click listener: navigate to existing `WorkOrderDetailScreen` by `workOrderId`.  
   - Wasm/desktop: no-op (do not call register).

2. **`notification-service`** (new repo, Compose/Ktor pattern like siblings)  
   - Persist device tokens: `(userId, token, platform, updatedAt)`; upsert by token; allow multiple devices per user.  
   - Public (via gateway): `POST /device-tokens`, `DELETE /device-tokens` (body or query token).  
   - Internal (service-to-service only): `POST /internal/notify` with `{ userId, title, body, data: Map }`.  
   - Look up tokens for `userId`, send FCM data+notification message; drop invalid tokens on FCM error.  
   - Secrets: Firebase service account / FCM credentials via env.

3. **`dashboard-service`**  
   - After successful assignee change where new `assigneeId != null` and value actually changed: call notification-service internal notify.  
   - Payload example: title «Новая заявка», body short WO title/id, data `{ "workOrderId": "<id>" }`.  
   - Do **not** fail the PATCH if notify fails; log and continue.  
   - No notify on clear assignee (`null`) or unchanged assignee.

4. **`api-gateway-service`**  
   - Proxy authenticated `POST/DELETE /device-tokens` → notification-service (require logged-in user; inject/pass `X-User-Id`).  
   - Do **not** expose `/internal/notify` to clients.  
   - Dashboard reaches notification-service via internal base URL (same pattern as other service-to-service calls).

## API sketch

### `POST /device-tokens` (user)

```json
{ "token": "<fcm-token>", "platform": "android" }
```

Auth: user JWT. Server binds token to caller `userId`.

### `DELETE /device-tokens` (user)

```json
{ "token": "<fcm-token>" }
```

### `POST /internal/notify` (service)

```json
{
  "userId": "<assigneeId>",
  "title": "Новая заявка",
  "body": "<wo title or id>",
  "data": { "workOrderId": "<id>" }
}
```

## Non-goals (v1)

- Web Push / Wasm notifications  
- iOS (API shape may allow `platform: "ios"` later; no client work in v1)  
- Push on status change, comments, or unassign  
- Retry queue / outbox (sync best-effort only)  
- In-app notification center / user mute settings  
- Rich media / action buttons on the notification  

## Acceptance

- Engineer on Android with registered token: dispatcher assigns them → receives push with WO context.  
- Tap opens that work order’s detail screen.  
- Assign still succeeds if notification-service or FCM is down.  
- Unassign / re-assign to same user without change does not spam.  
- Re-assign to another engineer notifies only the new assignee.  
- Wasm/desktop builds do not register tokens or crash on missing Firebase.  
- Unauthenticated client cannot register tokens; `/internal/notify` not reachable from public gateway routes.

## Target repos

- `notification-service` — new: tokens + FCM send + deploy CI  
- `dashboard-service` — assign hook  
- `api-gateway-service` — device-tokens proxy  
- `client-app` — KMPNotifier Android + deep link  
- `masterdoc` — this spec  
- `masterdoc-zitadel` / deploy docs — only if new env secrets need documenting  

## Open for implementation (not product blockers)

- Exact Firebase project ownership and secret names in GitHub Actions / VPS.  
- Whether register is gated on feature `engineer` or any authenticated user (recommend: any authenticated user; send only on engineer assign).  
- Internal auth between dashboard and notification-service (shared secret / network-only — follow existing sibling pattern).
