---
generated: '2026-08-26'
method: generated
name: Find imagery for an area of interest
description: Given a location, find which Near Space Labs surveys cover it, over what dates, and get the exact tile URLs to download.
api: openapi/near-space-labs-tile-service.json
operations: ['POST /oauth/token', 'GET /tile/v2/surveys/coverage', 'GET /tile/v2/{mosaic_id}/footprint', 'GET /tile/v2/{mosaic_id}/coverage']
source: >-
  Grounded in openapi/near-space-labs-tile-service.json and
  https://docs.nearspacelabs.com/surveys-and-coverage. The contract declares no operationIds, so
  operations are identified by their verbatim method + path.
---

# Find imagery for an area of interest

The catalog-first path: never guess a tile coordinate. Ask which surveys cover the area, then ask that survey which tiles it holds — the answer already contains fully-formed tile URLs.

## Auth
Bearer token from `POST /oauth/token`. See `skills/near-space-labs-authenticate.md`.

## Steps
1. **Build the AOI** — an OGC WKT geometry in EPSG:4326 (WGS 84). A `POINT (-115.268641 36.173426)` is valid and is what the provider's own Postman collection uses; a `POLYGON ((...))` is the normal case.
2. **Find covering surveys** — `GET https://api.nearspacelabs.net/tile/v2/surveys/coverage?wkt=<wkt>` with optional `observed_start` / `observed_end` ISO 8601 bounds. Each result carries `id`, `capture_date_start`, `capture_date_end`, `footprint` (WKT) and `spectral_range`.
3. **Pick a survey** — choose by capture window. Survey ids are human-readable period + state + place codes (`2024Q4-FL-PTCH`, `2025-P01-CO-DEN`) and are stable.
4. **(Optional) Confirm geometry** — `GET /tile/v2/{mosaic_id}/footprint` returns `{"survey_id":..., "footprint":"POLYGON ((...))", "status":200}`. Cache this: geometry rarely changes once a survey is published. Note the docs call this GeoJSON, but the contract and its own example return a WKT string in a JSON wrapper.
5. **Enumerate the tiles** — `GET /tile/v2/{mosaic_id}/coverage?wkt=<wkt>&zoom=<14-21>&overlap=intersects`. Every item carries `mosaic_stac_id`, `geospatial_extent` ([minx, miny, maxx, maxy]), `capture_date_start`/`_end`, `published_date`, `spatial_resolution_class` (e.g. `7.5cm`), `percent_covered` and — the field that matters — `static_url`, the exact tile URL to fetch.
6. **Fetch** — request each `static_url`. Do not construct tile URLs by hand.

## Why this order
Calling `/coverage` first is what keeps you off the 404 path: requesting a tile where no imagery exists returns `404`, and the provider explicitly recommends enumerating valid coordinates first to avoid them.

## Alternative entry point
If you already know the survey, skip step 2 and use `GET /tile/v2/surveys?page=<n>` to browse the catalog (25 per page, newest first, `has_next` tells you when to stop).

## Conventions and errors
- Zoom must be 14-21; outside that you get `422`, not `400`.
- Malformed WKT or a missing required parameter gives `400`.
- `429` means back off — no `Retry-After` and no published limit, so use exponential backoff with jitter.
- Persist the `x-correlation-id` from any error body for support.

See `conventions/near-space-labs-conventions.yml`, `errors/near-space-labs-problem-types.yml` and `data-model/near-space-labs-data-model.yml`.
