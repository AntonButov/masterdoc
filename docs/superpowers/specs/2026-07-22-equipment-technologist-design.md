# Equipment + Technologist Agents (draft-only)

**Date:** 2026-07-22 · **Updated:** 2026-07-27  
**Status:** implementation contract (synced with services) · **GTM:** feature-эксперимент «ИИ-наполнение ТОиР» — [ai-technologist-bootstrap-experiment.md](../../ai-technologist-bootstrap-experiment.md)  
**Related:** [TOIR_AI_SYSTEM_DESIGN.md](../../TOIR_AI_SYSTEM_DESIGN.md), [B2B_MVP_SCOPE.md](../../B2B_MVP_SCOPE.md), [maintenance-map practices](2026-07-22-maintenance-map-practices.md), [maintenance-service extraction](2026-07-27-maintenance-service-extraction-design.md)

## Product invariant

AI agents **never** publish to the operational ledger. MCP writes always create:

- `status: draft`
- `source: ai_generated`

A human with feature `equipment` confirms (`draft` → `active`) or rejects in the client.

## Agent pipeline

One PDF → one equipment unit → one maintenance map (many items). Host: `technologist-service` :8095 (OpenRouter `google/gemini-2.5-pro`, native PDF). Org **Fixaverse Smoke** (`383177088934346755`) uses deterministic mocks (no OpenRouter).

```text
upload PDF → POST /ai/document-validator → (optional replace) → POST /ai/equipment-card
  → human approves card → POST /ai/technologist {documentId,siteId,assetId} → draft PPR
  → human confirms asset + map
```

| Agent | Route | MCP | Result |
|-------|-------|-----|--------|
| Document validator | `POST /ai/document-validator` · `POST .../confirm-replace` | `document-mcp` | `ok` / `reject{explanation}` / `needs_replace{obsoleteDocumentIds}` |
| Equipment card | `POST /ai/equipment-card` | `technologist-mcp` `create_draft_asset` | draft asset |
| Technologist | `POST /ai/technologist` | `maintenance-mcp` `create_draft_maintenance_map` | draft map for existing `assetId` |

## Auth

Client calls gateway with `Authorization: Bearer <JWT>` and feature `equipment`. Services trust `X-Org-Id` / `X-User-Id` (default `default-org` / `unknown` if absent).

## Public API

Paths below are what the client hits on the gateway base URL; each maps 1:1 to the owning service.

### Documents · `document-service` :8093

| Method | Path | Response |
|--------|------|----------|
| POST | `/documents` | multipart PDF → `Document` 201 |
| GET | `/documents?folder=` | `{ items: Document[] }` — folder = storageKey prefix; default = `orgId` |
| GET | `/documents?sha256=` | `{ items }` matching hash in org |
| GET | `/documents/{id}` | `Document` |
| GET | `/documents/{id}/content` | PDF bytes (`Content-Type: application/pdf`, inline disposition) |
| GET | `/documents/{id}/text` | `{ text }` — UTF-8 extract / sidecar |
| DELETE | `/documents/{id}` | 204 |
| POST | `/documents/from-text` | `{ text, filename? }` → fake PDF fixture 201 |

Storage: `{DOCUMENT_STORAGE_DIR}/{orgId}/{id}.pdf` + sidecar `{id}.meta.json`. Restart recovers metas from disk.

### Assets · `catalog-service` :8091

| Method | Path | Response |
|--------|------|----------|
| POST | `/assets` | `CreateAssetRequest` → `Asset` |
| GET | `/assets` | `{ items: Asset[] }` |
| GET | `/assets/{id}` | `Asset` |
| POST | `/assets/{id}/confirm` | `Asset` (active) |
| POST | `/assets/{id}/reject` | 204 |
| POST | `/assets/{id}/unlink-documents` | `{ documentIds }` → asset without those ids |

### Maintenance maps · `maintenance-service` :8098 (Postgres)

Gateway feature gates: `equipment` + `charts`. Persist in Postgres; MCP writes always `draft` + `ai_generated`.

| Method | Path | Response |
|--------|------|----------|
| POST | `/maintenance-maps` | create (asset must exist in catalog) |
| GET | `/maintenance-maps?assetId=&status=` | `{ items }` |
| GET | `/maintenance-maps/{id}` | map |
| PATCH | `/maintenance-maps/{id}` | update draft fields |
| POST | `/maintenance-maps/{id}/confirm` | active (+ `activatedAt`, then dashboard scheduler tick) |
| POST | `/maintenance-maps/{id}/reject` | 204 |
| GET | `/internal/active-maps?orgId=&mapId=` | active maps for dashboard PPR→WO scheduler (private network) |

### Work orders · `dashboard-service` :8092 (gateway feature `board`)

See [2026-07-25-work-orders-board-design.md](2026-07-25-work-orders-board-design.md): `POST/GET/PATCH /work-orders`, `GET /work-orders/board`.  
PPR→WO scheduler stays in dashboard; loads active maps from maintenance-service over HTTP.

### Technologist jobs · `technologist-service` :8095

| Method | Path | Response |
|--------|------|----------|
| POST | `/ai/document-validator` | `{ documentId, siteId, assetId? }` → validation |
| POST | `/ai/document-validator/confirm-replace` | `{ documentId, obsoleteDocumentIds }` → deletes via document-mcp |
| POST | `/ai/equipment-card` | `{ documentId, siteId }` → `{ draftAssetId, asset }` |
| POST | `/ai/technologist` | `{ documentId, siteId, assetId }` → job 202 |
| GET | `/ai/technologist/jobs/{id}` | job |
| POST | `/ai/technologist/jobs/{id}/confirm-package` | confirms draft map (+ asset if job still holds draftAssetId) |

## DTOs

```json
{
  "Document": {
    "id": "uuid",
    "orgId": "string",
    "filename": "string",
    "contentType": "application/pdf",
    "storageKey": "orgId/id.pdf",
    "sha256": "hex",
    "uploadedBy": "string"
  },
  "Asset": {
    "id": "uuid",
    "orgId": "string",
    "siteId": "string",
    "name": "string",
    "inventoryNo": "string?",
    "category": "string?",
    "description": "string?",
    "status": "draft|active",
    "source": "manual|ai_generated",
    "documentIds": ["uuid"]
  },
  "CreateAssetRequest": {
    "name": "string",
    "siteId": "string",
    "inventoryNo": "string?",
    "category": "string?",
    "description": "string?",
    "documentIds": ["uuid"],
    "source": "manual|ai_generated",
    "asDraft": true
  },
  "MaintenanceMapItem": {
    "id": "uuid",
    "title": "string",
    "kind": "inspection|service|overhaul",
    "interval": { "every": 30, "unit": "days|hours|cycles" },
    "criticality": "low|medium|high",
    "sourceRef": "string?"
  },
  "MaintenanceMap": {
    "id": "uuid",
    "assetId": "uuid",
    "orgId": "string",
    "title": "string",
    "status": "draft|active",
    "source": "manual|ai_generated",
    "items": ["MaintenanceMapItem"]
  },
  "TechnologistJob": {
    "id": "uuid",
    "orgId": "string",
    "documentId": "uuid",
    "siteId": "string",
    "assetId": "uuid",
    "status": "queued|running|succeeded|failed",
    "draftAssetId": "uuid?",
    "draftMapId": "uuid?",
    "error": "string?"
  }
}
```

`ai_generated` or `asDraft=true` → asset starts as `draft`. Confirm/reject only via human (or `confirm-package` after a succeeded job).

## MCP · `technologist-mcp` :8094

`GET /mcp/tools` · `POST /mcp/tools/call` — **no** confirm/reject/publish tools. Asset/document meta only.

| Tool | Required args |
|------|----------------|
| `create_draft_asset` | `name`, `siteId`, `documentIds` (+ optional inventoryNo, category, description) |
| `update_draft_asset` | `assetId` + draft fields |
| `get_document_meta` | `documentId` |

## MCP · `maintenance-mcp` :8099

Draft PPR maps only; upstream `MAINTENANCE_BASE_URL` → maintenance-service.

| Tool | Required args |
|------|----------------|
| `create_draft_maintenance_map` | `assetId`, `title`, `items[]` |
| `update_draft_maintenance_map` | `id` + draft fields; only if `status=draft` |

Item shape: `{ title, kind, interval: { every, unit }, criticality, sourceRef? }`.

## MCP · `document-mcp` :8097

| Tool | Required args |
|------|----------------|
| `get_document_meta` / `get_document` | `documentId` |
| `list_asset_documents` | `assetId` |
| `get_asset` | `assetId` |
| `delete_document` | `documentId` (+ optional `assetId`) |
| `find_documents_by_sha256` | `sha256` |

## Client notes

- Feature nav: `#/equipment`, `#/ppr/{mapId}`.
- Flow: upload → validate → (replace dialog) → equipment-card draft → «В базу» starts technologist → second «В базу» publishes asset+map.
- PDF open: `GET /documents/{id}/content` with Bearer (`OpenAuthenticatedDocument` — web/desktop; Android stub).
- Folder list uses `storageKey` parent (usually `orgId`) so manuals appear even if not yet linked on the asset.

## Ports

| Folder | Port |
|--------|------|
| `catalog-service` | 8091 |
| `dashboard-service` | 8092 |
| `document-service` | 8093 |
| `technologist-mcp` | 8094 |
| `technologist-service` | 8095 |
| `document-mcp` | 8097 |
| `maintenance-service` | 8098 |
| `maintenance-mcp` | 8099 |

## Non-goals

- Intake / mentor / scribe agents
- Spare parts / 1C
