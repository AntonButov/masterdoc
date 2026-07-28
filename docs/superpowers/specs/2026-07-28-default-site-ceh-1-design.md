# Default site «Цех 1» on empty org

**Date:** 2026-07-28  
**Status:** approved design  
**Service:** catalog-service (`GET /sites`)

## Problem

Upload PDF → draft equipment requires `selectedSiteId`. New orgs have zero sites; the Equipment screen disables «Загрузить PDF и создать черновик» and points users to Админ → Площадки.

## Goal

Every org gets one default site without Admin setup: **«Цех 1»**, created lazily when the org first lists sites.

## Behavior

On `GET /sites` for `X-Org-Id`:

1. If the org already has **one or more** sites → return them unchanged (no seed).
2. If the org has **zero** sites → create exactly one site, then return the list:
   - `id`: `ceh-1`
   - `name`: `Цех 1`
   - `address`: `null`
   - `orgId`: from request tenancy header

Idempotent: a second `GET /sites` must return the same single site, not create duplicates.

### Concurrency

Two parallel empty-org `GET /sites` may race on `create` with fixed id `ceh-1`. The loser treats «site id already exists» as success and returns `list(orgId)` (no 500).

### Manual delete

If the admin deletes all sites (including «Цех 1»), the next `GET /sites` seeds again. If any site remains, seed does **not** recreate «Цех 1».

## Non-goals

- Postgres / Flyway seed for catalog (still in-memory `SiteStore`)
- Extracting site name from PDF / technologist MCP `create_site`
- Client-app UI changes (Equipment already auto-selects `sites.firstOrNull()` after `listSites()`)
- Deploy-time curl bootstrap scripts

## Tests (catalog-service)

- Empty org `GET /sites` → 200, one item `ceh-1` / `Цех 1`
- Second `GET /sites` → still one item, same id
- Org that already has a site → seed skipped; existing list unchanged
- Optional: two concurrent GETs → exactly one site

## Related

- [2026-07-24-sites-black-box-design.md](./2026-07-24-sites-black-box-design.md) — Site model
- Client: `EquipmentScreen` — upload enabled when `picked != null && selectedSiteId != null`
