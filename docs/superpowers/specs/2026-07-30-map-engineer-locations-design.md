# Map engineer locations — design

Date: 2026-07-30  
Status: implemented  
Repos: map-service, feature-service, api-gateway-service, client-app, masterdoc-zitadel

## Goal

Dispatchers with feature `map` see engineers on an OSM map. After an engineer presses «В работу», the Android app posts GPS every 5 minutes until all assigned open work orders are closed.

## Rules

- Start: engineer sets WO `status=in_progress`
- Stop: no assigned WOs in `new`/`in_progress`
- Ping: immediate then every 5 minutes
- Stale: omit locations older than 7 minutes
- Map tiles: OpenStreetMap + Leaflet

## API

Upstream `map-service` (port **8100** on VPS):

- `PUT /engineer-locations/me`
- `GET /engineer-locations`
- `DELETE /engineer-locations/me`

Gateway gates: GET → `map`; writes → `engineer`.

## Feature wire

`map` / «Карта» in FeatureCatalog, ProductFeatures, Zitadel project role.
