### Task 4: Detail UI — feed + composer

**Repo:** `client-app`

**Files:**
- Modify: `composeApp/src/commonMain/kotlin/pro/masterdoc/client/ui/screens/WorkOrderDetailScreen.kt`
- Wire `CommentsRepository` (+ existing `AttachmentsRepository`) from MyWorkOrders / Tickets / Board detail callers (`MainShellContent.kt` etc.)

**UI:**
- Section «Комментарии» under photos
- List: author label (`formatAssigneeLabel`), time, text, optional «Фото» thumb (reuse photo open pattern; label «Фото», not id)
- Empty: «Нет комментариев»
- Composer: `AppTextField` + disk/camera for one pending photo + «Отправить» enabled when text non-blank and not sending
- Flow: upload pending photo if any → `create` → clear composer → reload list
- Works when `readOnly = true` for lifecycle (gate on repository presence)

- [ ] **Step 1:** Load + render feed.
- [ ] **Step 2:** Composer + submit.
- [ ] **Step 3:** Wire repositories in shell callers.
- [ ] **Step 4:** Commit/push/watch deploy: `feat(work-orders): comment feed and composer on detail`

Global: never show raw user/attachment ids; use formatAssigneeLabel / «Пользователь» / «Фото».

Work from: `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/client-app`

Depends on Task 3 CommentsRepository being present (or implement Task 3 first if missing).
