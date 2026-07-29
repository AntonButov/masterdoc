# Dispatcher cannot take WO into work — Design

date: 2026-07-29  
extends: KD7 in `masterdoc/docs/plans/2026-07-28-001-feat-wo-assistant-core-plan.md`  
(`Dispatchers do not perform work orders — hard enforce`)

## Goal

Диспетчер не берёт заявку в работу и не закрывает её. Статус меняет только **назначенный** инженер (`caller userId == assigneeId`). Диспетчер назначает исполнителя и правит поля доски (длительность и т.п.).

## Product decisions (session-settled)

| Decision | Choice |
| --- | --- |
| Who may change status | Only assigned engineer (`X-User-Id == assigneeId`) |
| Unassigned WO | Nobody can start/close until assignee is set |
| Dispatcher (`board`) | Assign + board fields; no «В работу» / «Закрыть» |
| Engineer path | «Мои заявки» — start/close when assigned to self |
| Dual-role (`board` + `engineer`) | No special board UX (rare); perform via «Мои заявки» when assigned |
| Enforcement | UI hide + API hard reject (approach A) |

## Problem (current)

- `BoardScreen`: `readOnly = !dispatcherMode` → диспетчер видит «В работу» / «Закрыть».
- `dashboard-service` PATCH: status ACL skipped when caller has `board` → API allows dispatcher (and any board user) to change status, including with `assigneeId == null`.

## API (`dashboard-service`)

On `PATCH /work-orders/{id}` when body includes `status`:

1. Require `call.userId() == current.assigneeId` (assignee non-null and equal).
2. Otherwise `400` with message like `Only the assigned engineer may change status`.
3. **Remove** the bypass `if ("board" !in callerFeatures)` around this check — `board` no longer grants status changes.

Unchanged:

- Assignee patch still requires `board`.
- Assignee must have `engineer` + scope cover (existing).
- Legal transitions `new → in_progress → closed` (existing store rules).
- Duration / title / dueAt patches for board callers unchanged by this rule.

## UI (`client-app`)

1. **Board (dispatcherMode):** keep `editableAssignee`; do **not** show status action buttons («В работу», «Закрыть»). Prefer splitting concerns: assignment/duration editable without treating the whole detail as “perform WO” (`readOnly` today conflates both — fix so assignee edit stays, status actions gated off for board).
2. **My work orders (engineer):** keep status buttons when not closed; engineer is the assignee of listed WOs.
3. Dual-role on board: same as dispatcher board — no status buttons; use «Мои заявки» to perform.

## Non-goals

- Changing assignee eligibility (`engineer` + scope)
- Auto-assign on «В работу»
- Special dual-role board UI
- Allowing any engineer (non-assignee) to start an unassigned WO

## Acceptance

- User with only `board`: opens WO on board — no «В работу» / «Закрыть»; can set assignee; PATCH `status` → 400.
- User with `engineer`, WO assigned to them: «Мои заявки» → «В работу» then «Закрыть» succeed.
- PATCH `status` with `assigneeId == null` or caller ≠ assignee → 400 (including callers with `board`).
- Board caller PATCH `assigneeId` / `durationHours` without `status` still works.

## Target repos

- `dashboard-service` — status ACL for all callers
- `client-app` — board detail: hide status actions for dispatcher
- `masterdoc` — this spec
