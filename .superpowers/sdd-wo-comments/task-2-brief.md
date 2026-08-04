### Task 2: Gateway proxy `/comments`

**Repo:** `api-gateway-service`

**Files:**
- Modify: `GatewayConfig.kt` (+ `testDefaults`)
- Modify: `EquipmentRoutes.kt`
- Create/extend: `CommentProxyRoutesTest.kt` (mirror `AttachmentProxyRoutesTest`)

**Interfaces:**
- Env `COMMENT_SERVICE_BASE_URL` default `http://127.0.0.1:8103`
- `proxyPrefix("/comments", ..., readFeatures/writeFeatures = board|engineer|tickets)`

- [ ] **Step 1:** Failing proxy ACL tests (no feature → 403; with tickets → forwarded).

- [ ] **Step 2:** Config + proxy.

- [ ] **Step 3:** Tests PASS; commit/push/watch.

```bash
git commit -m "feat(gateway): proxy /comments for tickets/engineer/board"
```

---

