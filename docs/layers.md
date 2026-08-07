# Layers, Data Honesty & Licensing

Read this before adding or changing a layer, working on a raster, tile grid, paging, or
anything touching export/licence. The `add-map-layer` skill is the step-by-step; this is
the schema and the rules with their reasoning. **Data honesty and licensing are carried
by prose, not tests — keep them explicit.**

## LayerConfig schema

`packages/map-core/src/layers/layer-config.ts`. Each layer config module exports a
`LayerConfig` and a `localize(nls)`. Config is the single source of truth (Rule 3).

**There are two kinds of layer, split at the layer level.** Both extend
`LayerConfigBase` (id, title, visibility, attribution, redistribution):

- `LayerConfig` — features, queried from a service. Everything below describes this.
- `RasterLayerConfig` (`layers/raster-layer.ts`) — a tile cache of pixels the provider
  already rendered. No `geometryType`, `style`, `popupFields`, `tableFields`, or `join`
  (for a raster those are unanswerable, not merely unused), and no `legend` (its legend
  is **derived from its published palette**). It adds `palette`, `projections`,
  `opacity`.

The discriminant is `kind`, **optional on the vector side** (`kind?: 'vector'` vs.
`kind: 'raster'`), so every layer written before the split is unchanged. Narrow with
`isVectorLayer` / `isRasterLayer`; the registry offers `list()` (both), `listVector()`
(features), `getVector(id)`. A widget that needs features asks for the vector list rather
than checking per layer. Registering a raster runs the **admissibility gate** — see
"Rasters" in `docs/symbology.md`.

## A declared tile grid must be bounded

A cache outside Web Mercator declares its own grid — `origin`, `resolutions`,
`tileSize`. Those three make a correct grid **and an unbounded one**: OpenLayers fills a
`TileGrid`'s tile ranges from `sizes` or `extent` and nothing else, and
`withinExtentAndZ` reads a grid with no ranges as one with no limits, so every off-grid
coordinate a panned or oversized viewport computes becomes a real request. NASA GIBS
answers those **400** — 930 of 1,019 requests in one arctic-app session, before the fix.

So a declared grid also declares its bounds, in the shared `map/tile-grid-bounds.ts`
fields: **`matrixSizes`** (a WMTS matrix set's `MatrixWidth`/`MatrixHeight`, one entry
per resolution) or **`gridExtent`** (source-projection units). `map/openlayers/
tile-grid.ts` is the only place a `TileGrid` is constructed — rasters and basemaps both
go through it.

- **A raster data layer must declare bounds** — `assertRegistrableRaster` refuses one
  that doesn't. Its tile failures are user-facing (the TOC notice), and a request that
  should never have been made must not be able to reach that sentence.
- **A basemap declares bounds when its provider publishes them — and declares the
  _finding_ when it doesn't.** A bound that contradicts the service is worse than none:
  NRCan's CBMT serves tiles _outside_ its published `fullExtent` and answers 200 for
  every index tried, so bounding it by that would blank strips of a working basemap.
  Such a cache declares `boundsUnavailable` carrying the evidence, not the claim: the
  reason, the day measured, the out-of-grid tile URL actually requested, and the status
  it answered. **Silence still fails** — a comment is not a declaration (AR6 shipped a
  gate that refused prose-only measurements and two apps lost basemaps; see
  ARCTIC_PLAN › AR6a).
  The allowance is narrow: a raster may not use it at all; the probed status must be
  **200 or 404**, so a cache answering **400** (rejecting the index — the GIBS behaviour
  the rule exists for) cannot be declared away and must be bounded or not shipped; and
  each app's `basemaps.config.test.ts` **pins which of its caches declare it**.
- **One unusable basemap must not take the others with it.** A config that can't be built
  is **excluded and stated**, never fatal: `basemapConfigProblem` asks instead of
  throwing, `basemapsForProjection` drops it and reports to the **console** once (a
  defect here is a developer's bug, not a sentence to show the user), and the projection
  falls back to blank + labelled graticule. `MapView` contains a failed `setBasemap` for
  the same reason — it runs inside a `projection-changed` handler, and `EventBus.emit`
  walks subscribers in a plain loop, so an escaping exception abandons every later
  subscriber mid-switch.
- **The notice is only for tiles the grid has.** `classifyTileLoadError` sends an
  out-of-grid failure to `console.warn` — it's an application bug, not missing data, and
  "parts may be missing" over a fully-rendered layer is false the way a wrong legend is.

## Service metadata: coded-value domains and `outFields`

Esri layers get one extra request — `<url>?f=json`, fetched once per layer and memoized
in `layers/layer-metadata.ts` — supplying two things the features don't carry. Both
degrade to prior behaviour if that request fails; a metadata outage must never fail a
layer load.

- **Coded-value domains.** Services store codes (`'PVD'`), not labels (`'Paved'`). The
  rule everywhere: **display uses the label, queries use the code.** `resolveDomainValue`
  resolves for the popup, the attribute table (as rows are built, so filtering and
  sorting act on the label the user sees), the Filter widget's suggestions, and exports;
  `domainCodeForLabel` maps back before a `where` clause. A code the domain doesn't cover
  displays as the raw code, never blank. Exports carry the label **and** a `<field>_code`
  companion column so the file is readable and rejoinable — see `export-data-model.ts`.
  Pure helpers live in `layers/field-domains.ts`, importable as
  `@gis-app/map-core/field-domains`. A config needs nothing special: naming the field is
  enough.
- **Minimal `outFields`.** The loader requests the union of `popupFields` and
  `tableFields` plus the object-ID field, not `*`. It only requests fields the service
  declares (Esri fails the whole query on one unknown name), and falls back to `*` when
  the config declares no fields or metadata is unavailable.
- **`supportsPagination`.** Tri-state: explicit `false` is refused, `null` (undeclared or
  metadata unavailable) pages anyway and proves the result against the row count.

## Completeness: a partial layer is never rendered

An Esri query reaching the service's `maxRecordCount` answers **HTTP 200 with a
well-formed but partial `FeatureCollection`** (with `exceededTransferLimit`).
`layers/sources/esri-paging.ts` + `esri-feature.ts` make completeness part of the
loader's contract: one request when the layer fits (unchanged, no count query), and when
it doesn't, page through `resultOffset`/`resultRecordCount` until the service's own
`returnCountOnly` total is in hand. Anything that leaves completeness unproven
**rejects** — no OL layer is created, and the failure shows on the layer's TOC row. Never
loosen this into rendering what arrived: **a hazard polygon that was never sent reads as
absence of hazard.** Caps `MAX_LAYER_FEATURES` (50,000) and `MAX_PAGES` (100) bound a
runaway loop. Full rationale, including which servers put the flag where: PROMPT_PLAN ›
Phase 11.

## Load scope and redistribution

Two optional config fields for provincial/licensed data:

- **`EsriFeatureSource.extent`** — `[minLon, minLat, maxLon, maxLat]` WGS84, sent as an
  intersects envelope. For a service published far wider than the app's region (BC's
  DCRRA liquefaction is 921,433 polygons province-wide, 8,637 inside the CRD): without it
  the query is absurd and, past `MAX_LAYER_FEATURES`, refused. **An active AOI supersedes
  it** — Esri `/query` takes one geometry parameter, and the AOI is the narrower,
  user-chosen constraint.
- **`LayerConfig.redistribution`** — `'permitted'` (default) or `'restricted'`. Access to
  a live service is not permission to reproduce its data: BC's "Access Only" tier and the
  CRD/Province tsunami service's no-distribution clause both allow viewing and forbid
  handing the data on. A restricted layer is excluded from Export Data's picker and
  **named there with the reason**, in both locales. Not a security boundary and not
  presented as one; it stops the app's own features from doing the redistributing. State
  the licence tier and required credit in the layer config's own comment.
  **That naming requirement is scoped to _licence_ restrictions.** The rule it came from
  — "a silently missing entry reads as a bug" — is about a layer the user can see and
  would expect to export, held back by a term they can't infer. It does **not** extend to
  a layer with nothing to export: a raster has no features, so it is absent from Export
  Data silently and is **not** listed as restricted, because saying "the licence forbids
  this" about a layer that simply has no rows would misstate the licence. The two cases
  stay distinguishable — one is a restriction, the other is a data type.

## Data honesty (non-negotiable, not test-caught)

- **Never render knowingly partial data.** Enforced structurally by the completeness
  contract above for Esri; the principle is general.
- **"No data" is visually distinct from zero** and from a low class — a distinct symbol
  state, never a standard marker or omission. This extends to protected/withheld
  location data (e.g. a withheld wreck coordinate gets a visually distinct protected
  state, not a standard marker and not silent omission).
- **Derived ratios from randomly rounded counts are suppressed below a denominator
  threshold** (150; derivation documented — StatsCan publishes no guidance for this). See
  VICTORIA_PLAN › V2b for the `renterSharePct` correction (a band-provided tenure
  category was omitted, overstating every reserve).
- **Contested dataset boundaries** require source attribution and version tracking.
- **Any suitability/area analysis computes in the appropriate projected CRS, never in view
  coordinates** — EPSG:26910 for the CRD (Web Mercator's distortion at 48.5°N is ~1.5×
  linear / ~2.3× areal). Never EPSG:3413 or 4326 for CRD suitability.
- **Mark unverifiable things as failed, not optimistically confirmed.** Run a verification
  session before any build phase that depends on external data.
