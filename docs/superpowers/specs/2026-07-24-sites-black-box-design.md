# Sites + asset move + black-box audit

**Date:** 2026-07-24  
**Status:** implemented (MVP)

## Model

- **Site** (catalog-service): `id`, `orgId`, `name`, `address?` — flat functional location (цех / склад). No Area/Line hierarchy in MVP.
- **Asset.siteId**: must reference an existing Site in the same org. Relocate via `POST /assets/{id}/move` `{ "siteId" }` — inventory number and asset id stay immutable.

## Admin UI

Under client-app **Админ** (`admin`): tabs Пользователи | Площадки | Журнал.

## black-box-service

Append-only audit store. **Only api-gateway** writes after successful (2xx) user-facing catalog/admin calls (`POST /events`). Admins read via `GET /admin/audit` → black-box `GET /events`.

Secrets in summaries are redacted; bodies truncated (~4 KB).

## Gateway

- Proxy `/sites`, `/assets` → catalog (`CATALOG_SERVICE_BASE_URL`), JWT → `X-Org-Id`
- Fire-and-forget audit to `BLACK_BOX_SERVICE_BASE_URL` (default `:8096`; optional `BLACK_BOX_INTERNAL_TOKEN`)
- `/sites` allowed for `equipment` or `admin`; other equipment proxies keep their feature gates

## Related

- [B2B_MVP_SCOPE.md](../../../B2B_MVP_SCOPE.md) — Site entity
- [TOIR_AI_SYSTEM_DESIGN.md](../../../TOIR_AI_SYSTEM_DESIGN.md) — catalog owns Site
