# Conventions — i18n & Styling

Read this before adding user-facing strings or component CSS.

## i18n — the `Nls` library component

`packages/app-shell/src/i18n/nls.ts` provides `Nls`: a small localized-string registry.
`nls.add(id, locale, text)` registers a string for `'en' | 'fr'`; `nls.get(id, subs?)`
retrieves it for the current locale with `{0}`/`{1}`/… substitution; `nls.setLocale(locale)`
switches; `nls.has` checks registration; `Nls.formatNumber(value, locale)` gives Canadian
en/fr number formatting. Already implemented (`nls.ts` + `nls.test.ts`) — don't reimplement.

`packages/app-shell/src/i18n/index.ts` exports `nls`, the **single shared instance**. There
is no central locale file — every widget, sub-component, and layer config implements:

```ts
localize(nls: Nls): void {
  nls.add('measure.title', 'en', 'Measure');
  nls.add('measure.title', 'fr', 'Mesurer');
  nls.add('measure.lengthResult', 'en', 'Length: {0} {1}');
  nls.add('measure.lengthResult', 'fr', 'Longueur : {0} {1}');
}
```

During bootstrap (`main.ts`), every registered widget's and every layer config's
`localize(nls)` is called once against the shared `nls`, before first render. Components then
call `nls.get('measure.title')` wherever they need text. On `language-changed` (emitted by
`AppState` on the language toggle), call `nls.setLocale(language)` — a few lines of glue in
each app's `main.ts`. Components that display text subscribe to `language-changed` and
re-resolve via `nls.get(...)`.

**Registering only one locale is a bug** — add both `en` and `fr` in the same `localize`
change.

**ID convention:** `<widget-or-layer-id>.<name>`, e.g. `measure.title`, `toc.legendLabel`,
`layers.ottawaZoning.fields.zoneCode`.

### Where string catalogs live across the packages

Rule 5 doesn't change when a widget moves into a package: **the catalog travels with the code
that renders the string.** No central locale file.

- A stock widget in app-shell registers its own `en`/`fr` pairs in its static
  `localize(nls)`, under its own `<widget-id>.` namespace. The app doesn't supply, copy, or
  wrap those — it lists the widget in its `WidgetRegistration`s, and
  `WidgetRegistry.localize(nls)` calls each registration's `localize` once at bootstrap.
- App-shell chrome strings belonging to no single widget (`core.*`, `toolbar.*`) live in
  `packages/app-shell/src/i18n/core-strings.ts`; the app calls `localizeCore(nls)` at
  bootstrap.
- The app owns only app-specific catalogs: its layer configs (`layers.<layerId>.*`), its
  basemap list, and its chrome (`app.title`), registered from `main.ts` the same way.
- Bootstrap order in `main.ts`: `localizeCore` → `LanguageSwitcher` → layer localizers →
  basemaps → `widgets.localize` → app chrome → `nls.setLocale`. `Nls.add` overwrites, so **an
  app can override any shell string** by re-registering that id (both locales) after the
  shell's localizers run — the supported customization hook (e.g. re-wording
  `location-search.inputLabel` for a non-Canadian gazetteer). Registering only one locale is
  still a bug.

## Styling

- **Every component/widget owns its base CSS.** Each custom element (and the OL adapter's
  on-map overlays) has its own co-located `*.css` next to its module — `toolbar.ts` ↔
  `toolbar.css`, `widgets/filter/filter-widget.ts` ↔ `widgets/filter/filter.css` — imported
  as a side-effect at the top of that module (`import './filter.css';`, before the other
  imports). The file holds only that component's styles; no component reaches into another's
  CSS, the same decoupling the TS follows. (A component that adds no styles of its own beyond
  reused primitives — e.g. Location Search, just a `Typeahead` — needs no CSS file.) The
  `*.css` ambient module declaration each package carries (`src/types/css.d.ts`) lets these
  side-effect imports type-check under `tsc`; Vite bundles them.
- **App overrides win by source order.** Each app's `src/style.css` holds the global
  foundation (design tokens on `:root`, the reset, shared utilities like `.visually-hidden`)
  and is that app's override hook. It is imported **last** in `main.ts` (after every
  component has pulled in its own base CSS transitively), so at equal specificity its rules
  win — to tweak a component's look within one app, add a rule to that app's `style.css`. A
  change every app should get belongs in the component's base CSS in the package instead. Keep
  selectors at the same specificity as the base rule; rely on load order, not `!important`.
  (Deliberately **not** `@layer`: an environment that doesn't honour the at-rule drops the
  whole `@layer { … }` block, silently unstyling a component — plain cascade + source order
  degrades gracefully. Component base CSS is unlayered; so is third-party CSS like
  `ol/ol.css`.)
- **Use native CSS nesting** (`&`, nested selectors, nested media queries) wherever it
  improves readability — nest a widget's internal element selectors under its custom element's
  tag selector, nest `:hover`/`:focus`/state under their base rule. Keeps each component's
  styles scoped and co-located. No extra tooling — Vite supports native nesting directly;
  don't add a PostCSS nesting plugin.
- **Prefer relative units** (`rem`, `em`, `%`, `ch`) over `px`, especially for font sizes,
  spacing, and layout — a WCAG 1.4.4 (Resize Text) concern: `rem`-based sizing respects the
  user's browser font-size and zoom, where fixed `px` doesn't scale consistently. Acceptable
  `px` exceptions: hairline borders (`1px`), and values tied to a fixed external constraint
  (e.g. an OpenLayers control's pixel-based hit target).
