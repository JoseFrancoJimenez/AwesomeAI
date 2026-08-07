# Projections

Read this before touching anything that reprojects, switches projection, or stores a
view. The one-line invariants are in `CLAUDE.md` / `docs/architecture.md`; the reasons
and the measured failures are here.

## Rule 2a — extents are reprojected by densifying, never by their corners

Reprojecting a bounding box by transforming its four corners is exact _only_ for a
cylindrical projection pair; for any other it silently undershoots, because the true
transformed boundary is curved and the corners miss the extrema. The whole-Arctic box
comes back from EPSG:3413 **64.6% too small — and as the wrong quadrant**, not merely a
small box (Phase 12).

So `transformExtent` (`ol/proj`) and `applyTransform` (`ol/extent`) are banned by
import **name** everywhere except the one module that wraps them,
`map/openlayers/extent-transform.ts`:

- `transformExtentDensified(extent, source, target, stops?)` samples `stops` points
  along each edge (default 32, clamped to ≥2 so corner-only is unreachable) and returns
  the bbox of the transformed set. It degenerates to exactly the corner answer for a
  cylindrical pair, so **call it unconditionally** — there is no "this extent is small
  enough" judgement to make at a call site.
- Sampling only walks the boundary, so a pole _inside_ the extent still needs explicit
  handling: `lonLatExtentOfProjected`, in the same module.

The helper lives under `openlayers/` rather than beside `projection-catalogue.ts`
because that module is deliberately `ol`-free (app-shell's switcher reads it). Enforced
by `noRawTransformExtent` / `noRawApplyTransform`. Measurements: PROMPT_PLAN › Phase 13.

## Three rules that keep projection state from leaking

The view can run in more than one projection, and switching is runtime state.

1. **The catalogue is curated data, not a registry.**
   `packages/map-core/src/map/projection-catalogue.ts` holds every projection a view
   may run in — today EPSG:3857, 3978 (Canada Atlas Lambert), 3573 (Arctic LAEA,
   Canada), 3413 (NSIDC sea ice). No EPSG lookup, no user-entered codes. Each row
   declares its proj4 definition, `extent` (projected units), `worldExtent` (lon/lat —
   `ol/layer/Graticule` won't draw without one), the EPSG-published `areaOfUse`, units,
   and a `titleId`. The module is `ol`-free, so the switcher UI reads it without pulling
   in the map runtime. Adding a projection means adding a row and verifying it against
   the EPSG registry — never a runtime lookup. Keep the catalogue reviewable as a table.

2. **Which projections an app _offers_ is the app's decision**, declared in its
   `map.config.ts` as a subset of the catalogue and passed to widgets as
   `WidgetContext.projections`. Fewer than two means no switcher renders — how an app
   opts out. `victoria-app` does, staying in EPSG:3857. Conversely an app may exclude
   EPSG:3857: `arctic-app` does, because Web Mercator has no pixels above 85.06°N and
   that is inside its subject, not at the edge. Offering a switcher is also what makes
   proj4 a real dependency of that app's bundle: `main.ts` must call
   `registerProjections()` before constructing the adapter.

3. **Basemaps are projection-scoped.** `BasemapConfig.projections` lists the projections
   a tile cache is published for, and the switcher offers only compatible ones. **A
   projection with no compatible basemap is normal, not an error**: `MapAdapter` falls
   back to `BLANK_BASEMAP` (white, no app declares it) and shows the labelled graticule
   over it. OpenLayers _can_ resample a Web Mercator cache into a polar view — don't; a
   Mercator source has no pixels above 85.06°N, so it puts a hole exactly where a polar
   map is about. A cache outside Web Mercator must declare its grid (`projection`,
   `origin`, `resolutions`, `tileSize`): an ArcGIS or WMTS matrix set is not
   automatically an OL default tile grid, and NRCan's CBMT runs on round scales rather
   than powers of two. Tile-grid bounds are in `docs/layers.md`.

## Switching preserves centre and scale, not zoom level

A zoom index means a different ground resolution in every projection. When the current
centre falls outside the target's area of use (Ottawa → EPSG:3413), the view fits the
area of use instead — proj4 won't object on its own, it answers with a finite coordinate
thousands of kilometres from anywhere the CRS is defined.

## A stored view is a PortableView, never an extent

`PortableView = {center: [lon, lat], groundResolution (m/px), projection}`. Transforming
a lon/lat box between projections samples its corners — exact only for a cylindrical pair
(the whole-Arctic box comes back 64.6% too small covering one quadrant), and a polar view
containing the pole has lon/lat bounds of (−180 … 180, 90) however tightly it's zoomed.
Bookmarks use this (with a v1 migration). `MapAdapter.projectExtent` / `getViewExtent`
densify their transforms and handle the pole explicitly, both through
`extent-transform.ts` (Rule 2a).

## A feature the active projection cannot represent is left out and counted, not drawn

Clipping invents a boundary the data doesn't have, and drawing it anyway is worse:
OpenLayers clamps Web Mercator northings rather than failing, so an 86°N polygon renders
as a spike leaving the map. The adapter reports `LayerFitReport` through
`MapAdapter.onLayerFit`; `MapView` turns it into the layer's Table-of-Contents notice
(the same non-fatal channel a partial join uses); switching to a projection that can
show the feature brings it back.
