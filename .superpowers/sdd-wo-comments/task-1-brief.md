### Task 1: Bootstrap `comment-service` + route tests

**Repo:** `comment-service` — create under `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/comment-service`, mirror `attachment-service` (Gradle Ktor JVM, Dockerfile, compose, CI). Port **8103**, image `masterdoc-comment-service`, remote `/opt/comment-service`.

**Files:**
- Create: Gradle wrapper/project, `Application.kt`, `CommentStore`, tests, Dockerfile, `deploy/docker-compose.yml`, `.github/workflows/ci.yml`

**Interfaces:**
- `GET /health` → `{status:ok}`
- `GET /comments?workOrderId=` → `List<CommentDto>` ascending by `createdAt`
- `POST /comments` body `CreateCommentRequest(workOrderId, text, attachmentId?)` → `CommentDto` (201)
- Headers: `X-Org-Id`, `X-User-Id` (author)
- Reject blank/whitespace text; reject text length > 2000

- [ ] **Step 1:** Scaffold from `attachment-service` (drop image upload; keep Ktor JSON). `mainClass` = `pro.masterdoc.comment.ApplicationKt`. Env `PORT` default `8103`, `COMMENT_STORAGE_DIR`.

- [ ] **Step 2:** Failing tests `CommentRoutesTest.kt`: create + list round-trip; blank text → 400; oversize text → 400; org isolation (org-a comments not visible to org-b for same workOrderId).

- [ ] **Step 3:** Implement `CommentStore` — disk JSON list per `{storage}/{orgId}/{workOrderId}.json`.

- [ ] **Step 4:** Tests PASS.

- [ ] **Step 5:** Dockerfile + Compose + CI (health `http://127.0.0.1:8103/health`).

- [ ] **Step 6:** `gh repo create masterdoc-app/comment-service` if needed; commit; push `main`; watch deploy.

```bash
git commit -m "feat: bootstrap comment-service for work order comments"
git push -u origin main && gh run watch
```

---

