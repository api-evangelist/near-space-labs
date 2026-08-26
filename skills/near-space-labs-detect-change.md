---
generated: '2026-08-26'
method: generated
name: Detect change in a survey over time
description: Use mosaic_updates and historical capture ids to find what imagery was published for an area in a time window, and compare captures.
api: openapi/near-space-labs-tile-service.json
operations: ['GET /tile/v2/{mosaic_id}/mosaic_updates', 'GET /tile/v2/{mosaic_id}/coverage', 'GET /tile/v2/{mosaic_id}/{mosaic_stac_id}/{z}/{x}/{y}.{ext}']
source: >-
  Grounded in openapi/near-space-labs-tile-service.json and its published response examples, plus
  https://docs.nearspacelabs.com/surveys-and-coverage. The contract declares no operationIds, so
  operations are identified by their verbatim method + path.
---

# Detect change in a survey over time

Near Space Labs refreshes coverage roughly twice a year, and each survey accumulates individual captures. Change detection means comparing two captures of the same `z/x/y`.

## Steps
1. **Poll for new captures** — `GET https://api.nearspacelabs.net/tile/v2/{mosaic_id}/mosaic_updates?since=<ts>&until=<ts>&include_percent_covered=1`. Timestamps on the v2 path are ISO 8601. Each item carries `mosaic_stac_id`, `geospatial_extent`, `capture_date_start`/`_end`, `published_date`, `spatial_resolution_class`, `percent_covered` and a `static_url`.
2. **Record the baseline** — keep the `mosaic_stac_id` for each `z/x/y` you already hold. That id is the address of a specific capture: a timestamp triple plus a base-4 tile path, e.g. `20241101T181751_20241101T182126_20241101T193155_02301310020223`.
3. **Fetch both captures** — `GET /tile/v2/{mosaic_id}/{mosaic_stac_id}/{z}/{x}/{y}.png` for the old id and the new one. Add `?metadata=1` on the un-suffixed form to get the tile's JSON metadata instead of its bytes.
4. **Compare** — diff the two images at the same `z/x/y`. `spatial_resolution_class` may differ between captures (`7.5cm` vs `10.0cm` both appear in the provider's own examples), so normalise before comparing.
5. **Scope by geometry** — use `GET /tile/v2/{mosaic_id}/coverage?wkt=<aoi>&zoom=<z>` to restrict the comparison to your area of interest rather than the whole survey.

## Operational notes
- `mosaic_updates` is **not paginated** and returns an unbounded array. Keep the `since`/`until` window narrow.
- `percent_covered` is only populated when you pass `include_percent_covered=1`; otherwise it is `null`.
- There are no webhooks and no event stream. Change detection here is polling, and you choose the interval — a survey's `last_updated` on `GET /tile/v2/surveys` is the cheapest signal that anything moved at all.
- The legacy `/tile/{mosaic_id}/mosaic_updates` route does the same thing with legacy-formatted timestamps and is deprecated. Use the v2 path.

See `conventions/near-space-labs-conventions.yml` and `data-model/near-space-labs-data-model.yml`.
