# Task 1 report: Bootstrap comment-service + route tests

## Status

**DONE_WITH_CONCERNS**

The `comment-service` repository was created, implemented, tested, committed, and pushed to `main`. GitHub Actions tests passed. Deployment did not start because the new repository has no access to the required deploy secrets.

## Repository

- Local: `/Users/antonbutov/Documents/MYPROJECTS/fixaverse/comment-service`
- Remote: `https://github.com/masterdoc-app/comment-service`
- Branch: `main`
- Commit: `d8826a4a9b9082a0f4ac505186b92ca5adb462bc`
- Commit message: `feat: bootstrap comment-service for work order comments`

## Implementation

- Gradle/Ktor JVM service with Java 21, Ktor 3, Kotlin serialization, Netty, status pages, and call logging.
- `mainClass`: `pro.masterdoc.comment.ApplicationKt`
- `PORT` defaults to `8103`.
- Persistent storage is configured by `COMMENT_STORAGE_DIR`.
- `CommentStore` persists one JSON list per `{storage}/{orgId}/{workOrderId}.json`.
- Writes are synchronized and replace files atomically.
- Comments are append-only and listed ascending by `createdAt`.
- Comment text is trimmed, required, and limited to 2000 characters.
- Organization and work-order path segments are validated before filesystem use.
- Docker image: `masterdoc-comment-service`
- Compose remote path: `/opt/comment-service`
- Compose healthcheck: `http://127.0.0.1:8103/health`

## Routes

- `GET /health` returns `{"status":"ok"}`.
- `GET /comments?workOrderId=` lists comments for `X-Org-Id`.
- `POST /comments` creates a comment from `CreateCommentRequest(workOrderId, text, attachmentId?)`, taking the author from `X-User-Id`, and returns HTTP 201 with `CommentDto`.
- Missing required headers or work-order ID return HTTP 400.

## TDD evidence

1. Added `CommentRoutesTest.kt` before production route/store code.
2. Initial test run failed at compilation because `module`, `CommentStore`, `CreateCommentRequest`, and `CommentDto` did not exist.
3. Implemented the minimal routes and store.
4. Corrected two test-client decoding setup errors.
5. Final local wrapper run: `./gradlew test --no-daemon` — exit 0.

Test coverage includes:

- create + chronological list round-trip
- text trimming and optional attachment ID
- blank/whitespace text rejection
- text longer than 2000 characters rejection
- organization isolation for the same work order
- health response

## Self-review checklist

- [x] `GET /comments`
- [x] `POST /comments`
- [x] `GET /health`
- [x] organization isolation
- [x] blank text rejected
- [x] oversize text rejected
- [x] port 8103 in application, Dockerfile, Compose, and CI healthcheck
- [x] CI workflow present with test then deploy jobs
- [x] repository clean and tracking `origin/main`

## CI/deploy

- Workflow: `https://github.com/masterdoc-app/comment-service/actions/runs/30940190222`
- `test`: **success**
- `Deploy Compose to VPS`: **failure before build/deploy**
- Cause: `DEPLOY_HOST`, `DEPLOY_USER`, and `DEPLOY_SSH_PRIVATE_KEY` are empty for the new repository.
- Attempting to inspect organization Actions secrets returned HTTP 403 because the current GitHub identity lacks organization-admin / Actions-secrets permission.

## Concern / required follow-up

An organization administrator must grant `masterdoc-app/comment-service` access to the existing deploy secrets (or add repository secrets `DEPLOY_HOST`, `DEPLOY_USER`, and `DEPLOY_SSH_PRIVATE_KEY`) and rerun workflow `30940190222`. Until then, the service is not deployed on the VPS.
