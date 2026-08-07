# Architecture

How the code is structured and why. Read this before adding or moving a widget or
component, touching state/events, or working near the map boundary. Cross-cutting
invariants are summarized in `CLAUDE.md`; the reasoning is here.

## The eight architecture rules

1. **No direct component-to-component references.** UI components/widgets only read
   and write `AppState` and subscribe to events on the `EventBus`
   (`packages/app-shell/src/state/`). If widget A needs to react to widget B, B
   emits an event and A subscribes — they must not import each other.

2. **OpenLayers is isolated.** All `ol` imports live under
   `packages/map-core/src/map/openlayers/`. Every other file depends only on the
   `MapAdapter` interface in `packages/map-core/src/map/map-adapter.ts`. Swapping to
   MapLibre/Leaflet later = write `MapLibreMapAdapter` implementing the same
   interface; nothing else changes. Enforced by the core `no-restricted-imports`
   rule (`noOpenLayers` pattern group; the `openlayers/` folder is the one block
   that omits it).

   **2a. Extents are reprojected by densifying, never by their corners.** See
   `docs/projections.md` for the full rule; `transformExtent` and `applyTransform`
   are banned by import name everywhere except `map/openlayers/extent-transform.ts`
   (`noRawTransformExtent` / `noRawApplyTransform`).

3. **Layers are config, not code.** Each map layer is one `LayerConfig` object in
   its own file under the owning app's `src/config/layers/`. The config is the
   single source of truth for source URL, geometry type, symbology, popup fields,
   table fields, and legend. Adding a layer never requires editing widget code —
   see `.claude/skills/add-map-layer/`. Layers are always app-owned: a layer needed
   by two apps is registered by each of them, not promoted into a package. Full
   schema in `docs/layers.md`.

4. **Widgets follow one pattern.** Every widget extends `BaseWidget`
   (`packages/app-shell/src/widgets/base-widget.ts`), is registered in the
   `WidgetRegistry`, and is rendered inside an `OverlayPanel` anchored to its
   toolbar button. A widget may be composed of multiple smaller custom elements,
   but the widget itself is the unit registered with the toolbar. A widget that
   only needs `WidgetContext` (state, events, nls, layers, map, basemaps,
   storageNamespace) is app-agnostic and lives in
   `packages/app-shell/src/widgets/<widget-id>/`; one that hardcodes a layer id,
   extent, or app-specific service stays in that app's `src/widgets/`. Either way
   the app decides which widgets appear, in its `app.config.ts`. See
   `.claude/skills/add-map-widget/`.

5. **No hardcoded user-facing strings.** Every label goes through the shared `Nls`
   instance. Each widget, sub-component, and layer config implements
   `localize(nls): void` and registers its own `en`/`fr` pairs. Registering only
   one locale is a bug. Details in `docs/conventions.md`.

6. **SOLID / DRY.** One responsibility per class/module. Shared behaviour goes in a
   package, not copy-pasted — map/layer behaviour in `packages/map-core`, everything
   else in `packages/app-shell`. Prefer composition over inheritance except for the
   `BaseWidget` template-method base class.

7. **WCAG 2.1 AA.** Toolbar buttons: `aria-label` + `aria-pressed`, keyboard
   operable. Overlay panels follow the dialog pattern: `role="dialog"`,
   `aria-labelledby`, focus moves into the panel on open, `Escape` closes it, focus
   returns to the triggering button. Data tables use semantic
   `<table>`/`<th scope="col">` with `aria-sort`. No colour-only encoding — legend
   entries carry text labels. `<html lang>` updates with the active language. The
   symbology side of AA is in `docs/symbology.md`.

8. **Dependencies point one way.** `apps/<app>` → `packages/app-shell` →
   `packages/map-core`. A package never imports an app, no app imports another app,
   map-core never imports app-shell, and no project reaches into another's files by
   relative path — cross-project imports go through the workspace package name in
   `package.json`. Enforced by `no-restricted-imports` (`noCrossProjectImports`,
   `noAppShellFromMapCore`). When a new app joins the workspace, add its package
   name to `noCrossProjectImports` and prove the boundary against a deliberate
   violation. (`import/no-restricted-paths` was rejected — it silently passes when
   module resolution fails.)

## Dependencies

Keep the dependency tree small and deliberate — this is a small, auditable reusable
library, not a pile of npm packages glued together.

- Approved runtime dependencies: `ol` (mapping) and `proj4` (CRSs OL doesn't ship —
  EPSG:26910 for the CRD services), both confined to `map/openlayers/` per Rule 2.
  Register a new CRS by adding it to `PROJECTION_DEFINITIONS` in that folder's
  `projections.ts`; no app imports `proj4` itself. `@types/geojson` is dev/type-only.
- **proj4 is opt-in.** An app that needs a non-default CRS — or offers the projection
  switcher — imports `@gis-app/map-core/projections` and calls
  `registerProjections()` before constructing the adapter. Nothing else imports that
  module, which keeps proj4 (~131 kB raw / ~45 kB gzipped) out of every app that
  doesn't ask for it. Don't re-add a side-effect import to the adapter. Offering a
  switcher therefore has a real bundle cost, and is a per-app decision.
- Before adding any other runtime dependency — date formatting, debounce/throttle, a
  "lightweight" state library, a UI/dialog/table component library, CSV parsing, etc.
  — **stop and ask**. If it's small enough to write (a 20–30 line debounce, a tiny
  CSV row parser, a dialog focus trap), write it in the package that owns the concern
  instead.
- Don't add dev tooling (extra ESLint plugins, a different test runner) beyond
  SETUP.md without asking.
- If you do add a dependency, name it explicitly in your summary with the reason — it
  shouldn't arrive silently inside a larger change.

## Monorepo layout

```
gis-app/
├── CLAUDE.md
├── docs/                      # this brief and the others
├── .claude/
│   ├── skills/add-map-layer/SKILL.md
│   ├── skills/add-map-widget/SKILL.md
│   └── agents/code-reviewer.md
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── eslint.config.js
├── packages/
│   ├── map-core/                  # the map layer — publishable later
│   │   └── src/
│   │       ├── map/
│   │       │   ├── map-adapter.ts          # MapAdapter interface (lib-agnostic)
│   │       │   ├── types.ts                # Extent, MapFeature, SelectedFeature…
│   │       │   ├── basemap-config.ts       # BasemapConfig schema
│   │       │   ├── drawing-chrome.ts       # measure/AOI/graticule chrome constants
│   │       │   ├── symbology-contrast.ts   # contrast standard + basemap swatches
│   │       │   ├── projection-catalogue.ts # curated projection data (ol-free)
│   │       │   ├── tile-grid-bounds.ts     # matrixSizes / gridExtent / boundsUnavailable
│   │       │   └── openlayers/             # ALL `ol` imports live here
│   │       └── layers/
│   │           ├── layer-config.ts         # LayerConfig + source schemas
│   │           ├── raster-layer.ts         # RasterLayerConfig + admissibility gate
│   │           ├── layer-registry.ts
│   │           ├── layer-metadata.ts       # Esri f=json: domains + outFields
│   │           ├── field-domains.ts        # pure domain helpers
│   │           ├── classification.ts       # choropleth breaks + monotonic lightness
│   │           └── sources/                # wfs.ts, esri-feature.ts, esri-paging.ts, geojson.ts, filter-translation.ts
│   └── app-shell/                 # reusable application shell — depends on map-core
│       └── src/
│           ├── state/             # EventBus, AppState, AppEvent union
│           ├── i18n/              # nls.ts, index.ts (shared instance), core-strings.ts
│           ├── components/        # app-shell, toolbar, map-view, popup, language-switcher, select, typeahead
│           └── widgets/
│               ├── base-widget.ts / widget-registry.ts / overlay-panel.ts / widget-context.ts
│               └── toc/ attribute-table/ measure/ filter/ aoi/ location-search/
│                   bookmarks/ coordinates/ export-data/ basemap/
└── apps/                          # every app has the identical shape
    ├── web-gis-app/
    │   ├── index.html                  # single page (incl. the icon sprite)
    │   ├── a11y/                       # per-app axe-core + Playwright suite
    │   └── src/
    │       ├── main.ts                 # bootstrap: registries, localize, mount
    │       ├── style.css               # design tokens + override hook (loaded last)
    │       └── config/
    │           ├── app.config.ts       # registers layers + widgets
    │           ├── basemaps.config.ts  # selectable basemaps (config, not code)
    │           ├── map.config.ts       # default/Home view
    │           └── layers/             # one file per layer (Rule 3)
    │                                   #   + symbology-contrast.test.ts + basemaps.config.test.ts
    ├── victoria-app/              # same shape; see docs/VICTORIA_PLAN.md
    └── arctic-app/                # same shape; see docs/ARCTIC_PLAN.md
```

An app-specific widget (one hardcoding a layer, extent, or service only that app
has) lives in `apps/<app>/src/widgets/<id>/`; there are none today.

## State & events

`AppState`/`EventBus` implementation: `packages/app-shell/src/state/`. AppState is a
**thin** store — business logic lives in map-core. The `AppEvent` union is the
canonical list of cross-component events; when a widget needs a new event type,
extend the union there.

### Notable shared events

Emitted/consumed by more than one widget, so they don't belong to a single widget:

- `zoom-to-extent-requested {extent}` — anything wanting the map to navigate to a
  known extent (Location Search on result select, attribute table on row click).
  `MapView` subscribes once and calls `MapAdapter.fitToExtent`. Sources compute their
  own extent and emit this; don't add per-source zoom logic to `MapView`.
- `language-changed {language}` — emitted by `AppState` on the language toggle;
  triggers `nls.setLocale(...)`.
- `layer-filter-changed {layerId, filter}` — emitted by `AppState.setLayerFilter`
  (Filter widget). `MapView` calls `MapAdapter.setLayerFilter`, which re-queries with
  the filter translated to the source's syntax (`CQL_FILTER` for WFS, `where` for
  Esri, in `sources/filter-translation.ts`); `null` restores the unfiltered set. The
  Filter widget never touches the map — it reads/writes through `AppState`.
- `projection-changed {projectionCode}` — emitted by `AppState.setProjection` (Basemap
  widget's projection group). `MapView` rebuilds the view via
  `MapAdapter.setProjection` then re-applies a compatible basemap (`resolveBasemap`,
  fallback `BLANK_BASEMAP`). Layer visibility, filters, AOI, and bookmarks survive —
  they are state, held in EPSG:4326. See `docs/projections.md`.
- `view-restore-requested {view: PortableView}` — restoring a _saved_ view (Bookmarks).
  Distinct from `zoom-to-extent-requested` because an extent can't describe a view
  portably across projections; `MapView` owns the single `MapAdapter.setView` call.
- `basemap-changed {basemapId}` — emitted by `AppState.setBasemap`. `MapView` resolves
  the id against the app's `basemaps.config.ts` and calls `MapAdapter.setBasemap`,
  which swaps only the basemap tile source.
- `aoi-geometry-changed {geometry | null}` — emitted by `AppState.setAoiGeometry`
  (AOI widget draw/upload/clear). Only one AOI exists at a time. `MapView` renders the
  outline then re-applies every layer's attribute filter so each source re-queries
  with **both constraints ANDed** — attribute filter and AOI are independent state
  slices that each narrow the set; clearing one leaves the other applied. Spatial
  predicates translate in `sources/filter-translation.ts` (WFS CQL `INTERSECTS`
  against `WfsSource.geometryName`, default `geom`; Esri `geometry` + `spatialRel`).
  GeoJSON sources are static and ignore both filter kinds.

## MapAdapter contract

Interface: `packages/map-core/src/map/map-adapter.ts`. `OpenLayersMapAdapter`
(`.../map/openlayers/`) is the only file that imports `ol` (Rule 2).

### Map configuration

`MapViewConfig`:

```ts
interface MapViewConfig {
  center: [number, number]; // [lon, lat], EPSG:4326
  zoom: number;
  projection?: string; // default 'EPSG:3857'
  minZoom?: number;
  maxZoom?: number;
}
```

`MapAdapter.mount(target, config)` takes this for the initial view.
`MapAdapter.resetView()` returns to the `config` values — what the Home button calls.
App-level values live in each app's `src/config/map.config.ts`, exporting a single
`mapConfig: MapViewConfig`. This is the only place an app's default
center/zoom/projection are set — apps pick independently.

## Reusable UI primitives

`packages/app-shell/src/components/select/` (`Select`, an ARIA combobox) and
`.../components/typeahead/` (`Typeahead`, built on `Select`) are general-purpose — not
specific to Location Search. The attribute table's layer picker and Filter's
field/operator pickers reuse `Select` rather than writing their own dropdown.

## Widget catalogue

Every widget below is app-agnostic and ships in
`packages/app-shell/src/widgets/<id>/`; each app's `app.config.ts` chooses which
appear and in what order.

| id                | icon         | summary                                                                                                     |
| ----------------- | ------------ | ----------------------------------------------------------------------------------------------------------- |
| `toc`             | layers       | List of configured layers, visibility toggles, expandable legends                                           |
| `attribute-table` | table        | Pick a layer, sortable/filterable/paginated attribute table                                                 |
| `measure`         | ruler        | Draw line/area, show length/area, live update while drawing                                                 |
| `filter`          | funnel       | Build attribute filters against a layer's `filterable` fields                                               |
| `aoi`             | draw-polygon | Draw a polygon or upload GeoJSON to spatially filter layers                                                 |
| `location-search` | search       | Typeahead against the Canadian Geographical Names (GeoGratis) service; zooms to the selected result         |
| `bookmarks`       | bookmark     | Save/restore/delete named views (localStorage; stored as a `PortableView` so a view survives a projection switch) |
| `coordinates`     | crosshair    | Live pointer lat/lon readout (DD or DMS), copy to clipboard, go-to-coordinates navigation                   |
| `export-data`     | download     | Download a layer's current features (filter + AOI applied) as CSV or GeoJSON                                 |
| `basemap`         | map          | Radio list of basemaps; plus the projection radio group when the app offers ≥2 projections (one panel — the basemap list depends on the projection) |

Zoom in/out, Home (calls `MapAdapter.resetView()`), and Export-image (camera; PNG via
`MapAdapter.exportImage()`) are plain toolbar buttons, not widgets (no overlay panel).
The metric scale bar and OL's attribution control are always-on map furniture rendered
by the adapter — neither widgets nor buttons; styled in the adapter's co-located
`scale-bar.css`.

**A basemap may cost the PNG export.** `BasemapConfig.taintsCanvas` declares a provider
that sends no CORS header: its tiles are requested without `crossOrigin`, which taints
the canvas, so `exportImage()` can't read it back. The export button is disabled with a
localized reason while such a basemap is selected — same discipline
`LayerConfig.redistribution` follows for Export Data, because a feature that silently
stops working reads as a bug. Arctic SDI is the one such basemap today. Proxying its
tiles through this origin would remove the taint and is deliberately not done: relaying
a provider's tiles turns access into arguable redistribution.

**Attribution is a licence obligation, not decoration.** A basemap credits via
`BasemapSource.attributions`; a data layer via optional `LayerConfig.attributions`.
Both surface in the attribution control while their layer is visible, and
`exportImage()` composites the control's current text into the PNG. Attribution text is
**not** localized — it is provider-set licence wording, reproduced verbatim in both
locales, not routed through `Nls`. When adding a layer, check whether its provider
requires a credit and set the field.

## Per-app browser storage

A widget that persists to `localStorage` (Bookmarks today) must key it
`` `${context.storageNamespace}.<key>` ``, never a literal. `storageNamespace` is a
`WidgetContext` field each app sets in `main.ts` — `'web-gis-app'`, `'victoria-app'`,
`'arctic-app'` — because apps served from the same origin share one `localStorage` and
would otherwise overwrite each other. An app's namespace is permanent: changing it
orphans everything its users already saved.
