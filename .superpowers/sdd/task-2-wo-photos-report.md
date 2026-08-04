# Task 2 report: dashboard work-order attachments

## Status

Implemented and verified the dashboard-side attachment ID references.

## Changes

- Added Flyway migration `V3__work_order_attachments.sql` with a non-null `TEXT` column named `attachment_ids`, defaulting to `'[]'`.
- Added `WorkOrder.attachmentIds` and nullable `CreateWorkOrderRequest.attachmentIds`.
- Added `AttachWorkOrderAttachmentsRequest`.
- Added JSON list-of-string encoding/decoding for the PostgreSQL text column.
- Creation deduplicates IDs and rejects more than 10 unique IDs with `Too many attachments`.
- Added `WorkOrderStore.appendAttachments`, which appends distinct IDs, persists the result, and enforces the 10-ID cap.
- Added `POST /work-orders/{id}/attachments`, scoped by `X-Org-Id`, returning the updated work order.
- Added route coverage for create, append/deduplication, cap failure, and the empty default.

## Verification

Commands:

```text
./gradlew test
./gradlew test --tests pro.masterdoc.dashboard.ManagerKpisTest \
  --tests pro.masterdoc.dashboard.WorkOrderStoreContractTest \
  --tests pro.masterdoc.dashboard.MaintenanceMapClientTest
```

The targeted non-container tests pass. The local full suite compiles, but its Testcontainers-backed
classes cannot run because Docker is unavailable (`/var/run/docker.sock` is missing).
GitHub Actions run
[`30918646720`](https://github.com/masterdoc-app/dashboard-service/actions/runs/30918646720)
passed both the full test job and Compose deploy.

## Ship gate and smoke

- `d9d9bc0` added the feature; `0d58737` fixed serialization of empty lists.
- Both commits were pushed to `main`.
- Post-deploy API smoke: `https://api.masterdoc.pro/health` returned `200` and `{"status":"ok"}`.
- Smoke tenant: Fixaverse Smoke (`383177088934346755`); health is tenant-independent and no tenant data was modified.

## Notes

The dashboard stores opaque attachment IDs only and does not call attachment-service, as required. An unrelated pre-existing `.gitignore` modification was left unstaged.

## Follow-up fixes

- Dashboard commit `1641e1d` (`fix(work-orders): tickets-only ownership on attach`) checks the existing work order before accepting attachments and returns 404 for a tickets-only caller attaching to another user's work order. Targeted regression test passed locally.
- Attachment-service commit `581e3f3` (`fix(deploy): bind attachment-service on port 8097`) changes the application default, Docker image port, Compose mapping/healthcheck, and CI runtime configuration to 8097.
- Dashboard CI run [`30919337912`](https://github.com/masterdoc-app/dashboard-service/actions/runs/30919337912) passed tests and deploy.
- Attachment CI run [`30919329393`](https://github.com/masterdoc-app/attachment-service/actions/runs/30919329393) passed tests, but deploy failed because remote port 8097 is already allocated by `deploy-document-mcp-1` (while 8094 remains allocated by `deploy-technologist-mcp-1`). The requested remote health probe on `127.0.0.1:8097/health` returned `{"status":"ok"}`, but it currently reaches document-mcp, not the newly deployed attachment container.
- Task 3 note: gateway default must be `http://127.0.0.1:8097`.
