# Symbology, Contrast & Colour

Read this before touching any on-map colour — layer style, legend, chrome, basemap, or an
exemption. The standard is a hard gate enforced by `symbology-contrast.test.ts` (map-core
+ one per app). A colour that fails gets a darker ColorBrewer step, never a loosened
assertion.

## Palette recipe

Layer `style` colours are drawn from ColorBrewer (colorbrewer2.org). The hue plan is the
8-class qualitative **Dark2** palette, but every colour is subject to the contrast
standard below — several raw Dark2 colours fail it on these basemaps.

- **Single-class context layers**: pick a colour and declare it in the layer config, with
  a comment naming the ColorBrewer scheme and step. The tests are the gate, not an
  assignment algorithm — a colour is correct if it clears the contrast standard and the
  ΔE floor against every other layer in that app, and no rule can tell you more.

  There **used to be** an automatic rotation ("next unused Dark2 hue; if it fails
  contrast, substitute the sequential dark step in the same family"). Retired in V2b after
  two phases hit its wall from opposite ends (victoria-app exhausted Dark2; web-gis-app's
  Arctic layer found the next two substitutions at ΔE 0 from chrome). A rotation whose
  every output needs manual resolution stopped saving work, and read as an authority that
  had already thought about the palette when it hadn't. The useful part survives as
  guidance: ColorBrewer schemes, dark steps (7–9) for anything that must clear 3:1, hue
  families kept apart. Reuse a verified colour in a different app freely — two apps' layers
  are never on screen together.

- **Hazard layers leave the Dark2 rotation.** A layer whose subject is a modelled hazard
  (inundation, wildfire, ground failure, heat) takes the **darkest step of the ColorBrewer
  sequential palette matching its hazard type** instead of the next qualitative slot: fire
  red, inundation blue, ground grey. Red/orange reading as danger is a real cartographic
  convention; hue still separates hazards from each other, and two hazards of the same
  type deliberately share a family (victoria-app's tsunami and coastal flood are both
  inundation blues). Registration order does **not** determine a hazard layer's colour, so
  adding one never shifts an existing one's. (Two reasons the qualitative rule couldn't
  continue: its next substitutions collide with on-screen chrome, and forcing every hue
  dark for 3:1 compresses Dark2 to ~4 usable slots. Measurements: PROMPT_PLAN › Phase 11.)

- **Palette order is per-app.** Each app walks Dark2 from the start in its own registration
  order; two apps' layers never share a screen, so they needn't agree. The contrast
  standard itself is **not** per-app — every colour in every app must clear it.

- **Multiple categorical classes within one layer**: a ColorBrewer qualitative palette
  sized to the class count (per-count palettes aren't subsets of the 8-class one), then
  contrast-check each class colour.

- **Sequential data (choropleth)**: an **unmodified** ColorBrewer sequential scheme sized
  to the class count (ColorBrewer publishes a separate scheme per count — "5 classes" is
  the published 5-class scheme, not the first five steps of the 9-class one). The ramp's
  light steps can't clear 3:1 and aren't asked to (fill exemption below). The **stroke** is
  one colour for the whole layer and does clear it. Classification method, break
  computation, and the "no data" class live in `layers/classification.ts`; the legend
  states method and breaks (a legend of ranges that doesn't say how they were derived isn't
  reproducible).

### Current assignments

- `web-gis-app`: `ottawa-transit-stations` → Blues-9 s9 `#08306b`, `ottawa-zoning` →
  Purples-9 s9 `#3f007d`, `cwfis-fire-weather-stations` → PuRd-9 s9 `#67001f`,
  `arctic-protected-areas` → Greens-9 s9 `#00441b`. All step-9 darks because this app ships
  the Arctic SDI polar basemap, whose ocean fill is the darkest background anything in this
  workspace renders over.
- `victoria-app` context: `crd-parks` → `#016c59`, `crd-regional-trails` → `#a63603`,
  `crd-municipal-boundaries` → `#6a51a3`, `crd-census-subdivisions` → `#666666` (choropleth
  stroke). Hazards: `dcrra-wildfire` → YlOrRd-9 s8 `#bd0026`, `crd-heat-vulnerability` →
  PuRd-9 s8 `#980043`, `crd-tsunami-hazard` → Blues-9 s9 `#08306b`, `dcrra-coastal-flood` →
  PuBu-9 s8 `#045a8d`, `dcrra-liquefaction` → Greys-9 s8 `#252525`.

The two apps' palettes are independent and may reuse a hex — their layers are never on
screen together.

### Fills, chrome, blank background

- **Polygon fills are exempt from the 3:1 minimum; strokes are not.** Fills use the layer
  colour at ~0.25 opacity with the full-opacity colour as stroke (as in `ottawa-zoning`),
  or a sequential ramp step for a choropleth. The exemption is not a convenience: WCAG 2.1
  SC 1.4.11 covers graphics _required to identify a component_ — the stroke is what makes a
  polygon identifiable and carries the obligation; a translucent fill is data encoding read
  through the legend and mathematically cannot reach 3:1 over a light basemap anyway. **The
  obligation that replaces it is SC 1.4.1 Use of Color**: the value a fill encodes must be
  available without colour — discharged by the legend (method + every class range), the
  popup, and the attribute table, all three carrying the number. An exempt fill without
  those is not exempt, it is unlabelled. Point styles use full-opacity fill with a white
  casing stroke, and points are exempt from nothing.

- **A choropleth's declared colours are not the colours on screen.** A classified layer
  declares the unmodified ColorBrewer ramp (what the legend swatch shows and the "used as
  published" test checks), and the renderer composites it over the basemap at
  `DEFAULT_FILL_OPACITY` (0.75, `classification.ts`) so a region-tiling layer doesn't black
  out the streets. Legend and map differ by that factor — the usual cartographic bargain,
  defensible under the fill exemption. **What this costs is distinctness, so check the
  composited colours:** blending pulls every fill toward the basemap and toward each other.
  The census "no data" grey measured ΔE 23.9 from the nearest ramp step as declared and
  **17.7 once composited** over CARTO Positron's land — the config number was checking a
  colour nobody sees. `compositeOver()` exists for this; each app's suite runs the
  no-data-vs-ramp comparison on blended values over every basemap it ships.

- **Drawing-tool chrome** (measure sketch, AOI outline) is not data and doesn't consume
  palette slots: dashed 2px strokes over a 5px white halo — measure Oranges-9 s9 `#7f2704`,
  AOI BuPu-9 s9 `#4d004b`. (Were `#a63603`/`#ae017e` until V2b's follow-up shipped the
  Arctic SDI basemap whose deep-ocean fill put both around 2.4:1; chrome is gated against
  the union of every guaranteed basemap any app ships. `drawing-chrome.ts` records why the
  AOI changed hue family rather than only darkening.) The **graticule** is chrome by the
  same rule (a property of the projection, not the map) — Greys-9 s8 `#252525` with a white
  label halo, neutral rather than a hue, because a coloured grid over an empty background
  reads as a data layer. Constants in `map/drawing-chrome.ts` (ol-free); the OL adapter
  builds the styles.

- **The blank background** (`BLANK_BASEMAP`, shown when the active projection has no
  compatible cache) is a basemap for contrast purposes and carries its own one-colour
  swatch set. It is white, the same as `PANEL_BACKGROUND`, deliberately: every symbol is
  already audited against white for its legend swatch, so the blank background re-opens no
  palette decision.

## Contrast & size standard (WCAG 2.1 SC 1.4.11 + adopted 2.5.8)

- **Contrast**: the identifying colour of every persistent on-map symbol — point fill,
  line/polygon stroke, chrome stroke, legend swatch — must measure **≥ 3:1** against every
  swatch of every **contrast-guaranteed basemap it can appear over**, and against
  `PANEL_BACKGROUND`.
- **Which basemaps those are is scoped to the app; the rule is not.** A **layer** renders
  only over its own app's basemaps, so that is the set it's judged against
  (`swatchesForBasemaps` + `minContrastAgainst`). **Chrome and the graticule** are map-core
  constants shared by every app, so they're judged against the union of every guaranteed
  basemap in the workspace (`BASEMAP_SWATCHES` + `minBasemapContrast`). Scoping came in when
  web-gis-app added a polar basemap: without it, victoria-app's palette would've been gated
  against an ocean fill it can never render over. Corollary: adding a basemap to one app can
  require re-picking **that app's** palette, and says nothing about the other's. Swatch sets
  live in `BASEMAP_SWATCH_SETS` (`map/symbology-contrast.ts`), sampled from live tiles around
  the default view — re-sample if a basemap's tiles change. `CONTRAST_GUARANTEED_BASEMAPS`
  (OSM, CARTO Positron, NRCan CBMT, Arctic SDI, blank) feed the flattened `BASEMAP_SWATCHES`
  union.
- **Casing**: every persistent symbol carries a **white casing ≥ 1.5px**
  (`SYMBOL_CASING_COLOR`) at ≥ 3:1 against its own colour, so its edge survives basemap
  colours darker than the swatch sets anticipate. Points ring their fill (the config's
  stroke); line/polygon strokes ride a halo of it (adapter's style builder, +1.5px/side);
  chrome always had it.
- **Size**: point radius ≥ 5px. Map-click hit detection pads targets by 6px
  (`CLICK_HIT_TOLERANCE_PX`), making the effective target ≥ 25px — the 24×24 CSS px minimum
  of WCAG 2.2 SC 2.5.8, adopted as a convention (2.2 isn't a project commitment, and map
  pins would qualify for its "essential" exception; the padding costs nothing visually).
  Data strokes ≥ 1.5px, chrome ≥ 2px.
- **Mutual distinguishability (ΔE)**: any two layers' identifying colours in one app must
  measure **≥ 20 CIE76 ΔE** apart (`MIN_LAYER_COLOR_DIFFERENCE` / `colorDifference()`).
  Basemap contrast says a symbol is _visible_; it says nothing about telling two layers
  apart, and the darkness 3:1 forces is exactly what erodes that. 20 is an empirical floor
  just under the closest pair deliberately shipped (victoria-app's two inundation blues,
  21.5) and far above the ~2.3 just-noticeable threshold — a colour under it takes a
  different hue family, not a lowered floor. **The floor is between layers, not within a
  ramp.** A sequential scheme's adjacent steps are _supposed_ to be perceptually close; the
  test there is monotonic lightness (`isMonotonicLightness`) on an unmodified ColorBrewer
  scheme. A classified layer's one identifying colour for the floor is its **stroke**.

## Exemptions — five kinds, none inheritable by default

Every exemption is documented, measured, and pinned by a test. Adding one is a decision to
argue for, not a default to reach for.

1. **Satellite imagery (unbounded pixels).** Esri World Imagery
   (`CONTRAST_EXEMPT_BASEMAPS`) shows arbitrary real-world pixels: its swatch set documents
   the measured range but cannot bound it, and mid-tone imagery defeats both a dark symbol
   core and its white casing. The white casing mitigates as far as practical; switching to
   OSM/Positron is the accommodation — that alternative does **not** make satellite
   compliant and must not be framed as such. Tests keep it explicit: every swatch set is in
   the guaranteed list or the exempt list, never silently skipped.

2. **NOAA bathymetry (a _traded_ exemption, AR0).** `noaa-arctic-bathymetry` (arctic-app's
   EPSG:3995 basemap) is exempt because clearing it costs more than it's worth, **not**
   because nothing could. Its IBCAO/GEBCO ramp is fixed and bounded (deep-ocean blues
   `#4e84cc`…`#5a90d8`); a dark symbol clears them. What fails is the shared measure chrome
   at **2.76:1** on `#548ad2` — and that chrome is already Oranges-9 step 9, so clearing it
   means leaving the Oranges family and repainting the measure tool in every app, to buy a
   basemap for one projection in one app. Declined deliberately. This changed the standing
   rule: an exemption may be granted to a basemap that _could_ be cleared, but only when the
   phase **states the compliant option it declines and what that option costs**, and only
   where an accommodation is reachable. Guardrails enforced: `symbology-contrast.test.ts`
   asserts the exempt basemap is real (measure chrome fails on it) **and** that black clears
   every swatch in it (distinguishing "traded" from "forced"); each app's suite pins its own
   exemptions. The accommodation is derived from the app's own config, never hardcoded.

3. **arctic-app app-scoped exemption (AR1) — the one entry that is not about a basemap.**
   `apps/arctic-app` exempts every basemap it registers.
   - **Scope:** arctic-app only. web-gis-app and victoria-app keep the per-basemap rule.
     map-core's `CONTRAST_GUARANTEED_BASEMAPS` still lists `arctic-sdi`/`cbmt3978` as
     guaranteed; arctic-app's `basemaps.config.ts` opts out of relying on that, and its
     `symbology-contrast.test.ts` asserts the override only runs **downward** (an app may
     decline a guarantee map-core makes; never claim one it doesn't). **A new app inherits
     the workspace rule, not this exemption.**
   - **Justification:** private demo behind Cloudflare Access, no public audience; buys
     basemap coverage for all four projections including EPSG:3413, whose only published
     cache (NASA GIBS `OSM_Land_Water_Map`) can't meet 3:1. That's the whole trade.
   - **Not compliant, never described as such.** Numbers still measured and written down
     (ARCTIC_PLAN › AR1); the suite still sweeps every (symbol, basemap) pair, it just no
     longer fails on one. An undocumented exemption is an omission.
   - **Still gates in that app:** legend swatches vs panel white, casing contrast, point
     radius, ΔE floor. Only the basemap-contrast gate is lifted.
   - **Accommodation wording changes with it:** with no guaranteed basemap in any
     projection, "switch basemap" and "switch projection" are both false, so the widget uses
     `basemap.contrastNoteNoGuarantee`, which states the limitation and casing mitigation and
     stops. Do **not** print an accommodation where none exists.

4. **Per-layer raster exemption (precedent AR4).** For any app **not** arctic-app, shipping
   a luminance-spanning raster data layer requires a **named per-layer exemption** argued on
   its own terms. It **must not inherit** satellite's (unbounded pixels — a published palette
   contradicts that) or arctic-app's app-scoped one (about basemaps, not data layers). It
   states the layer, the measured palette range, the compliant alternative, and its cost, and
   leaves a reachable accommodation or says none exists. arctic-app needs none today only
   because its app-scoped exemption already covers it — a coincidence of scope, not a
   precedent.

5. **A measured correction to how "cannot be cleared" gets claimed (AR1).** The GIBS mask is
   exempt because a _dark_ symbol can't clear it (black reaches 2.52:1 on its `#4e4e4e` land
   tone). It is **not** true that no colour can — white clears the whole mask (4.06:1 water,
   8.32:1 land). What doesn't exist is a single colour clearing that mask **and** the light
   basemaps beside it (white measures 1.03–1.52:1 there). So it's a **palette conflict
   between basemaps**, not an unclearable basemap. General lesson: before writing "no colour
   can clear this", check both ends of the range — the dark end failing is a different claim.

## Rasters: admissible only if the palette is published (AR4)

A layer whose pixels arrive already rendered can't be symbolized here, so the standard's
usual subject — a colour this workspace chose — doesn't exist. The rule that replaces it is
a **gate on registration, in code** (`layers/raster-layer.ts`, `LayerRegistry.register`):

- The provider must publish a **machine-readable palette** and the config must declare it
  (`RasterPalette`: published `sourceUrl`, units, colour stops verbatim). `palette` is
  required, so a palette-less raster doesn't typecheck; `assertRegistrableRaster` re-checks
  at registration. **No published palette, not shippable.**
- This is what stops a raster **inheriting satellite's exemption**. Satellite is exempt
  because its pixels are arbitrary and unbounded _even in principle_; a provider-rendered
  palette is finite, published, enumerable — it gets **measured**, not excused. Two layers
  can be equally awkward to symbolize and make completely different claims about what's
  knowable; never conflate them.
- **The legend is derived from the palette, never hand-written**, and carries **numbers and
  units**. A raster has no popup or table, so the legend is its only channel for SC 1.4.1.
  `LegendEntry` lives on the vector side of the config split for this reason — a raster
  _cannot_ declare a legend, making "derived" unforgeable.
- The palette is a swatch set, so symbol contrast over it is computed and recorded by the
  owning app's suite even where it doesn't gate.
- Where a palette **spans the luminance range**, say so plainly and claim no conformance.
  GIBS' sea-ice ramp runs `#0e000e` to `#ffffff`, so neither black nor white clears 3:1 over
  all of it — a palette conflict, same structure as the AR1 mask finding.

## When a basemap can't be cleared, and polar readiness

- **A basemap no existing symbol can clear is a basemap this workspace does not ship.** Two
  candidate Arctic services (Esri Arctic Ocean Base, Arctic SDI topographic) were rejected in
  Phase 12: mid-tone ocean fills put every layer colour and both chrome colours in web-gis-app
  between 2.0 and 2.7:1. The only other moves are re-picking the whole palette or an exemption,
  and an exemption is reserved for documented cases. Measurements: PROMPT_PLAN › Phase 12.
- **Polar readiness is measured, not required.** `POLAR_OCEAN_SWATCHES` holds two sampled
  ocean fills (`#849cba`, `#84a8cc`); `polarOceanContrast()` measures against them. They're
  deliberately outside `BASEMAP_SWATCHES` — gating on a basemap the workspace doesn't ship
  would fail almost every symbol in the two Mercator apps. What the tests require is that the
  palette rule _can_ produce colours that clear them (ColorBrewer step-8/9 darks clear
  3.4–7.5:1), and each app's suite reports where its own colours stand. Shipping a polar
  basemap means first re-picking that app's palette into the dark range — a phase, not a
  config change.

## How to verify

`pnpm -r test`. `symbology-contrast.test.ts` in map-core audits the chrome constants and the
guaranteed/exempt basemap partition; **each app carries its own copy** under
`src/config/layers/`, auditing that app's registered `LayerConfig`s (fills, strokes, legends,
casing, radius) and `BasemapConfig`s (swatch set exists, `guaranteesContrast` matches the
partition). Every app needs one — a new app copies the file and lists its own layers. A new
colour that fails gets a darker ColorBrewer step, never a loosened assertion. For a one-off
manual check, `contrastRatio()` / `minBasemapContrast()` from
`@gis-app/map-core/symbology-contrast` are importable anywhere (the subpath avoids pulling the
DOM/ol surface into Node).
