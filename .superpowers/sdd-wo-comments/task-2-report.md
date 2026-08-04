# Task 2 report: Gateway proxy `/comments`

## Status

Implementation, push, gateway CI, and gateway deploy completed successfully.

- Repo: `masterdoc-app/api-gateway-service`
- Branch: `main`
- Base: `63cbb3c1ab683bfe484bab3937ea1319333d22f9`
- Commit: `fe2c42d5be1089686cbe48f57bf62741e88cfcf5`
- CI/deploy: https://github.com/masterdoc-app/api-gateway-service/actions/runs/30940734703 — success

## Changes

- Added `GatewayConfig.commentServiceBaseUrl`.
- Added `COMMENT_SERVICE_BASE_URL`, defaulting to `http://127.0.0.1:8103`.
- Added `commentServiceBaseUrl = "http://comment.test"` to `testDefaults()`.
- Added `/comments` proxy with identical read/write feature gates:
  - `board`
  - `engineer`
  - `tickets`
- Mirrored attachment-service VPS wiring in the deploy workflow:
  - bootstrap `.env`: `COMMENT_SERVICE_BASE_URL=http://host.docker.internal:8103`
  - existing VPS `.env`: unconditional `upsert_env COMMENT_SERVICE_BASE_URL 'http://host.docker.internal:8103'`

## TDD evidence

Created `CommentProxyRoutesTest` first with the required ACL cases:

1. no allowed feature (`charts`) → expected `403 Forbidden`
2. `tickets` feature → expected request forwarding

RED:

```text
CommentProxyRoutesTest: 2 tests completed, 2 failed
```

Both tests failed before `/comments` existed.

GREEN:

```text
./gradlew --no-daemon test --tests pro.masterdoc.gateway.CommentProxyRoutesTest
BUILD SUCCESSFUL
```

CI then ran the full Gradle test task successfully.

## Deploy and smoke

- Gateway test job: success
- Gateway deploy job: success
- Org: Fixaverse Smoke (`383177088934346755`)
- URL: `https://api.masterdoc.pro/comments?workOrderId=smoke-gateway-proxy-check`
- Authenticated request reached the new proxy but returned `502 Bad Gateway`.

The `502` is consistent with the upstream comment-service being unavailable, not a gateway ACL rejection. The current comment-service workflow is failing:

- https://github.com/masterdoc-app/comment-service/actions/runs/30940190222 — failure

## Concerns

- End-to-end comments smoke is blocked until `comment-service` deploy succeeds and port `8103` is reachable on the VPS.
- Gateway Actions emit maintenance warnings for `actions/setup-java@v4` and Node.js 20-based actions; they do not affect this deployment.
