# Web GIS App — CLAUDE.md

Terse index of the rules. **Reasoning, measurements, and phase history live in the
read-on-demand docs below — not here.** See "Keeping this file small".

## What this is

A pnpm monorepo of single-page TypeScript / Vite / OpenLayers web GIS apps sharing
one map layer and one application shell. Each app is _configuration + composition
only_: its layers, basemaps, default view, widget list, and strings. Everything
reusable lives in the two packages (`packages/map-core`, `packages/app-shell`).

| app                 | purpose                                                                                                                        |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `apps/web-gis-app`  | Ottawa-area reference app — the shape to copy. Ships the projection switcher.                                                   |
| `apps/victoria-app` | CRD (Victoria BC) relocation suitability. No projection switcher. Permanently not deployed (licence). See `docs/VICTORIA_PLAN.md`. |
| `apps/arctic-app`   | Canadian Arctic demo. Default view **not** Web Mercator; app-scoped contrast exemption. See `docs/ARCTIC_PLAN.md`, `docs/arctic-gis-layers.md`. |

Naming one app is an example, not a restriction: the rules apply to every app in
`apps/`, and shared behaviour belongs in a package so a second app can reuse it.

## Guiding principle

Be pragmatic — no over-engineering. Add structure or abstraction only when a
concrete, present need forces it, never speculatively. Be pragmatic but not
careless: readable, testable, maintainable code.

**Existing abstractions here were each forced by a measured failure** (recorded in
the plan docs) and stay. This principle governs _new_ work; it is not a licence to
retro-strip working, tested machinery that already earned its place.

## This file is guidance, not enforcement

Lint and tests are the real safety net. Where a rule is enforced automatically,
state it in a line and trust the check:

- `ol` imports confined to `map/openlayers/` — `no-restricted-imports` (`noOpenLayers`).
- Extent reprojection only via the densifying helper — `noRawTransformExtent` / `noRawApplyTransform`.
- One-way deps; no cross-app import; no app→package — `noCrossProjectImports`, `noAppShellFromMapCore`.
- Symbology contrast, ΔE, casing, radius, monotonic ramps — `symbology-contrast.test.ts` (map-core + one per app).
- Layer completeness / paging — the loader rejects a layer it cannot prove complete.

Carried **only by prose** (tests do not catch these — keep them explicit): **data
honesty** and **licensing / redistribution**. Both summarized below, detailed in
`docs/layers.md`.

## Keeping this file small

CLAUDE.md is loaded every session; every line is paid on every task. It holds
**only** terse rules and pointers. It does **not** hold rationale, measurements,
phase history, worked examples, or a rule's derivation — those go in a
read-on-demand brief or a plan doc.

When a phase establishes or amends a rule: put a one-line invariant here _only if
it is cross-cutting_, and put the reasoning and measurements in the relevant brief.
If lint or a test fully enforces it, cite the check and stop. A paragraph of
justification is the signal that it belongs in a brief, not here.

## Read on demand

Read the relevant doc before working in its area rather than guessing.

- `docs/architecture.md` — **when:** boundaries, dependency direction, monorepo layout, adding or moving a widget/component, state & events, the MapAdapter contract, map config, per-app storage, the widget catalogue.
- `docs/projections.md` — **when:** anything touching a projection, extent reprojection, or saved views / bookmarks.
- `docs/layers.md` — **when:** adding or changing a layer; rasters, tile grids, paging/completeness, service metadata, load scope, data honesty, licensing/export.
- `docs/symbology.md` — **when:** any on-map colour, contrast, legend, ΔE, casing/size, or exemption.
- `docs/conventions.md` — **when:** i18n strings, or component CSS/styling.
- `docs/PROMPT_PLAN.md`, `docs/VICTORIA_PLAN.md`, `docs/ARCTIC_PLAN.md` — phase records and measurements (base / Victoria / Arctic tracks). Referenced by the briefs for derivations.
- `.claude/skills/add-map-layer/`, `.claude/skills/add-map-widget/` — step-by-step for the two common tasks.

## Stack & non-negotiables

- TypeScript strict, Google TS Style (`tsconfig.base.json` / `eslint.config.js` — do not loosen). Vite for build, Vitest for unit tests.
- Native Web Components, **light DOM only** — never attach a shadow root.
- OpenLayers, isolated behind `MapAdapter`; all `ol` under `packages/map-core/src/map/openlayers/`. → `docs/architecture.md`, `docs/projections.md`
- pnpm workspaces, strict layering `apps/<app>` → `packages/app-shell` → `packages/map-core`. No package imports an app; no app imports another app; map-core never imports app-shell. → `docs/architecture.md`
- Widgets communicate only through `AppState` / `EventBus` — never import each other. → `docs/architecture.md`
- Layers are config, one `LayerConfig` per file, not code. → `docs/layers.md`
- No hardcoded user-facing strings — everything through `Nls`, both `en` and `fr` registered in the same change. → `docs/conventions.md`
- WCAG 2.1 AA throughout; the symbology contrast standard is a hard gate. → `docs/symbology.md`
- **Data honesty:** never render knowingly partial data; "no data" is visually distinct from zero; suppress derived ratios below the documented denominator threshold. → `docs/layers.md`
- **Licensing:** access ≠ redistribution. No repo snapshots and no tile proxying of Access-Only / no-distribution data; `LayerConfig.redistribution` gates Export Data with a stated reason. → `docs/layers.md`
- Dependencies are deliberate: the only approved runtime deps are `ol` and `proj4`, both confined to `map/openlayers/`; `proj4` is opt-in via subpath. Ask before adding any other. → `docs/architecture.md`

## Commands

```bash
pnpm install
pnpm -r build        # all packages + all apps
pnpm -r lint
pnpm -r test
pnpm format          # or format:check — root-level, not recursive
pnpm -C apps/<app> dev
pnpm -C apps/<app> a11y   # axe-core + Playwright; one-time: npx playwright install chromium
```

Preview ports: web-gis-app 4173, victoria-app 4174, arctic-app 4175 (`--strictPort`).
A change in `packages/` reaches every app — lint/test/build **and** a11y all of them.

## Definition of done (every PR-sized change)

- [ ] `pnpm -r lint && pnpm format:check && pnpm -r test` pass; `pnpm -r build` clean (a `packages/` change keeps **every** app compiling).
- [ ] Widget/`OverlayPanel` changes: `pnpm -C apps/<app> a11y` for every app reached, no new critical/serious axe violations.
- [ ] No new `ol` outside `map/openlayers/`; extents reprojected only via `transformExtentDensified` / `lonLatExtentOfProjected`.
- [ ] Dependency direction intact; new shared code in the package that owns the concern; a new app added to `noCrossProjectImports`.
- [ ] No new hardcoded strings; every new `Nls` id registered `en` + `fr` in the same change.
- [ ] New widgets extend `BaseWidget`, are registered, follow the overlay-panel dialog a11y pattern, and live in `app-shell` unless app-specific.
- [ ] New/changed components style via a co-located `*.css`, not an app's `style.css`.
- [ ] New layers are config-only under the owning app's `src/config/layers/`, listed in its `app.config.ts` and `symbology-contrast.test.ts`.
- [ ] Any new/changed on-map colour meets the contrast & size standard with the colour listed in the test; a failure gets a darker ColorBrewer step, not a loosened assertion.
- [ ] Projection changes keep the catalogue reviewable; every new entry has `extent` + `worldExtent`; an app offering <2 projections has no switcher and no proj4 in its bundle.
- [ ] Any new basemap declares its `projections`, ships a swatch set, and either passes contrast or gets a documented named exemption; any self-declared tile grid also declares its bounds.
- [ ] Docs updated where the rule actually lives — the relevant brief or plan doc, **not** CLAUDE.md unless it is a genuinely new cross-cutting invariant.
- [ ] **Committed** — one commit per coherent unit, on a branch (not `main`, don't push), written after the checks pass.

## First words

Every session, your first reply must be exactly **"I've read the docs, I am ready"**,
then list which docs you read (name and path). Always read this file; read the
brief(s) for the area the task touches **before** writing code.
