---
generated: '2026-08-26'
method: generated
name: Serve Near Space Labs tiles in a web map
description: Wire Near Space Labs XYZ tiles into a Leaflet/MapLibre/OpenLayers map, including the survey-free `basemap` keyword and the static-key pattern.
api: openapi/near-space-labs-tile-service.json
operations: ['POST /oauth/static_key', 'GET /tile/v2/{mosaic_id}/{z}/{x}/{y}.{ext}']
source: >-
  Grounded in openapi/near-space-labs-tile-service.json,
  https://docs.nearspacelabs.com/retrieving-tiles and https://docs.nearspacelabs.com/authentication.
  The contract declares no operationIds, so operations are identified by their verbatim method + path.
---

# Serve Near Space Labs tiles in a web map

The tile surface is the ordinary XYZ / slippy-map contract, so any standard map client consumes it with no bespoke connector.

## Tile URL template
```
https://api.nearspacelabs.net/tile/v2/{surveyid}/{z}/{x}/{y}
```
- `z` 14-21, 256x256 px, PNG or JPEG (select the format with a `.png` / `.jpeg` suffix).
- Substitute the literal keyword `basemap` for `{surveyid}` to get the most recent tile the server holds for each `z/x/y` across all surveys — pan and zoom a continuous map without choosing a survey first. This keyword is documented in prose only; it is not in the contract.

## Auth for a browser client
A browser map cannot hold an `NSL_SECRET` and cannot refresh a 60-minute token per tile. Mint a static key server-side once:

1. `POST https://api.nearspacelabs.net/oauth/static_key` with `{"client_id":..., "client_secret":...}`.
2. Append `?api_key=<api_key>` to the tile template.

**Read this before you do it.** The key is a bearer credential valid for one year, it travels in the URL where every proxy and log sees it, Near Space Labs publishes no revocation endpoint, and the docs' only remediation for a leak is "reissue it". Do not ship one to a public page you cannot rotate. Prefer a server-side tile proxy that holds the secret and attaches a short-lived Bearer token, and use the static key only when a proxy genuinely is not possible.

## Handling gaps in coverage
A tile where no imagery exists returns `404`. Render a transparent placeholder, skip it, or fall back to another basemap layer — do not treat it as a failure. To avoid the 404s entirely, precompute valid coordinates with `GET /tile/v2/{mosaic_id}/coverage` (see `skills/near-space-labs-find-imagery-for-an-area.md`).

## Caching
Send `If-None-Match` on tile requests to avoid re-downloading unchanged tiles. Cache survey footprints indefinitely; geometry rarely changes once published.

## Version
Use the `/tile/v2/` paths. The un-versioned `/tile/` routes are deprecated, emit deprecation warnings, and return legacy-formatted timestamps. No removal date has been published — see `lifecycle/near-space-labs-lifecycle.yml`.
