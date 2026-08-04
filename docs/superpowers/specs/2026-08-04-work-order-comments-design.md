# Work order comments — Design

**Date:** 2026-08-04  
**Status:** approved (session)  
**Repos:** `comment-service` (new), `api-gateway-service`, `client-app`  
**Depends on:** `attachment-service` (photos already shipped); WO photos design `2026-08-04-tickets-equipment-list-and-photos-design.md`

## Goal

В деталке заявки — append-only лента комментариев (это и есть лог): текст + опционально одно фото. Писать могут все, кто видит заявку. Фото комментария хранится в `attachment-service`; в комментарии только `attachmentId`.

## Product decisions

| Topic | Choice |
| --- | --- |
| Service | Separate `comment-service` |
| Photo storage | Same `attachment-service`; comment holds reference only |
| Log | Feed in detail = audit log of comments |
| Edit / delete | No (MVP) |
| Text | Required (non-blank after trim), max 2000 |
| Photo in comment | Optional, at most one |
| Who | Anyone who can `GET` the WO (`tickets` \| `engineer` \| `board`; admin if already on WO) |
| UI names | Author via `formatAssigneeLabel` — never raw user id |

## Architecture

### comment-service (new)

Port **8103**. File-backed store (same spirit as attachment-service).

| Endpoint | Behavior |
| --- | --- |
| `GET /comments?workOrderId=` | List for org + WO, ascending `createdAt` |
| `POST /comments` | Body `{ workOrderId, text, attachmentId? }`; `authorId` from `X-User-Id`; `orgId` from `X-Org-Id` |
| `GET /health` | `{status:ok}` |

```kotlin
data class CommentDto(
    val id: String,
    val orgId: String,
    val workOrderId: String,
    val authorId: String,
    val text: String,
    val attachmentId: String? = null,
    val createdAt: String,
)
```

Reject blank text; trim; max 2000 chars. `attachmentId` opaque (no call to attachment-service).

### api-gateway

- Proxy `/comments` → `COMMENT_SERVICE_BASE_URL` (default `http://127.0.0.1:8103`)
- ACL: same as attachments — read/write `board` \| `engineer` \| `tickets`

Path is `/comments` (not under `/work-orders`) so it does not collide with dashboard WO proxy.

### client-app

On `WorkOrderDetailScreen` (after photos section):

1. Load `GET /comments?workOrderId=`
2. Feed: author label, time, text, optional photo thumb via attachments content URL
3. Composer: text field + optional one photo (disk/camera) + «Отправить»
4. Submit: upload photo if any → `POST /comments` → reload feed
5. Empty: «Нет комментариев»

## Error handling

| Case | Behavior |
| --- | --- |
| Blank text | Client disable send; API 400 |
| Upload photo failed | Do not create comment; show error |
| Over 2000 chars | Client/API reject |
| No access | Gateway 403; feed empty/error without leaking ids |

## Out of scope

- Edit/delete comments
- Black-box audit duplicate (feed is the log)
- Mentions / reactions / threading

## Success criteria

1. Detail shows chronological comment feed with display names.
2. Any viewer can add text (± one photo).
3. Photo bytes only in attachment-service; comment stores id.
4. No raw user/attachment ids in user-visible copy.
