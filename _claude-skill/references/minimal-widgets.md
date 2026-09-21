# Minimal widget library — `Widgets/minimal/`

The reference set of AI-written widgets built to the minimal-widget bar. Copy from these, not
from the older feature widgets, when starting a new simple widget. Each one uploads to SDX under
`category: "AI Examples/Basic"` — the AI-written counterpart of Ben's `"Examples/Basic"`
(`_versions/label-widget-v1/example-label-simple.html` is one of his), so both sets sit side by
side in the Globals panel. Revision history lives in `Widgets/_versions/<hrid>/`, not in the
`minimal/` folder itself.

## The bar every widget here meets

- Every option is consumed **declaratively**: by data-attributes-lib in the markup, or by CSS
  through its `--widget-<key>` custom property. No `widget.onLoad` handler, no DOM writes -
  with one sanctioned exception: an option that is genuinely impossible to express in
  markup/CSS (so far only a **tag name**, heading-widget-v1) gets the smallest `onLoad` that
  does just that one thing, and everything else in the widget stays declarative.
- **No** loading/active/error state markup and **no** l10n block — these widgets are
  synchronous and config-only, nothing can fail or wait (patterns.md, "Exception: a purely
  synchronous, config-only widget").
- The script is exactly `widget.ready();` (plus the `sdx`/`widget` boilerplate), except for
  the case above.
- Option keys are **kebab-case**, labels are **localized** `{ "en": ..., "de": ... }` objects on
  every option and every `select` choice.
- A `boolean` option's `"default"` is a **quoted string**, `"true"`/`"false"` — not a JSON
  boolean literal. Confirmed via a real edit Ben made between two syncs of the same file
  (`example-label-simple.html`): an earlier copy had `font-bold`/`font-italic` as JSON
  booleans, a later one had them as quoted strings, nothing else in the file changed — most
  plausibly a fix he hit in his own live testing, not a style choice. All four widgets here
  now follow this; see button-widget-v1 rev6 / heading-widget-v1 rev2 / label-widget-v1 rev4
  for the correction.
- Any option that becomes a CSS length is a **bare number** with the unit appended in CSS via
  `calc(var(--widget-x) * 1px)` (patterns.md, "type: text must be a single unbroken CSS token").
- A short `<!-- ... //-->` header comment above `<widget>` says what the widget is.
- Don't bake generic slot-level layout (margin, padding, width, height, position, alignment of
  the whole widget within its row) into a widget's own options — the SDX layout builder already
  provides a Style tab and Advanced tab for every widget slot, entirely separate from the
  widget's own config (see architecture.md, "Slot-level styling"). Only options that affect the
  widget's own inner content/behavior belong here. **Open question raised by this**:
  divider-widget-v2's `spacing-before`/`spacing-after` may now be redundant with the slot's own
  Padding fields — not yet resolved, not changed on this note alone.

## Option type → zero-JS mechanism (the lookup table)

| Need | Option type | How it reaches the DOM/CSS | Example |
|---|---|---|---|
| Text content | `text` | `widget-text="<key>"` on the element (sets `innerHTML`) | both |
| Link target | `text` | `widget-link="<key>"` sets `href` | button |
| Open in new tab | `boolean` | `widget-link-new-tab="<key>"` sets `target="_blank"` (implies noopener) | button |
| Show/hide | `boolean` | `widget-visible="<key>"` | data-attributes-test-widget-v1 |
| Background image id | `text` | `widget-background-image="<key>"` | data-attributes-test-widget-v1 |
| Color | `color` | `color: var(--widget-<key>)` | both |
| Length in px | `text` (bare number) | `calc(var(--widget-<key>) * 1px)` | all |
| Length in % | `text` (bare number) | `calc(var(--widget-<key>) * 1%)` | divider |
| One of N keywords | `select` | value IS the CSS keyword: `text-align: var(--widget-<key>)` — or `justify-content:` for placing a block narrower than its parent; `start`/`center`/`end` are valid for both | label, heading (text-align); divider (justify-content) |
| On/off styling | `boolean` | `@container style(--widget-<key>: true) { ... }` nested in the element's rule | label, heading |
| Mode switch (N non-CSS values) | `select` | one `@container style(--widget-<key>: <value>) { ... }` block per non-default mode, holding every rule that mode changes; nest it in `#widget-body` when several elements/axes change at once | divider (orientation) |
| Optional length, empty = leave unset | `text` (bare number or empty) | `@property --x { syntax: "<length>"; inherits: true; initial-value: 0px }` + `--x: calc(var(--widget-<key>) * 1px)` + `@container not style(--x: 0px) { prop: var(--x) }` — the registered property turns the empty-option `calc()` failure into a queryable `0px` | heading |
| Element tag name | `select` | one of the sanctioned JS cases: `onLoad` does `createElement(getConfig(key))`, copies the placeholder's `innerHTML` (already filled by `widget-text`, since preload runs before `onLoad`), `replaceWith` | heading |
| Variable, end-user-typed options | `text` (`"Label\|URL"`, comma-separated) | the other sanctioned JS case: `select` can only hold *fixed, author-defined* choices, not choices someone else types in later — parse the delimited string in `onLoad`, build real `<option>` elements | dropdown |
| Navigate on change | n/a (JS event) | `select.addEventListener('change', ...)` — `location.href = value` in the same tab; `window.open(value, '_blank', 'noopener')` for a new tab. `window.open()` needs `'noopener'` passed explicitly — the implicit protection `target="_blank"` gets on a plain `<a>` does NOT extend to this JS API | dropdown |

The tag-name and variable-options rows are the two sanctioned JS cases so far. Beyond a change
handler wiring an already-necessary case like the second row, reach for one only when the
option type table above genuinely has no cell for what's needed — a style container query
compares the `true`/`false` token the host writes, evaluated against the element's nearest
ancestor (`#widget-body`, which inherits every `--widget-*` var). Full explanation in
patterns.md, "Pattern: boolean option → CSS via style container query".

For `select`: pick option **values** that are already valid CSS keywords for the property they
drive, so no mapping is needed. Prefer `start`/`end` over `left`/`right` for alignment — they
follow the reading direction, which is the rtl argument otherwise used to justify a flex
container.

## Skeleton

```html
<!--
One line on what the widget is.
//-->
<widget>

	<!-- CONFIG -->
	<script type="application/json" id="widget-config">
		{
		"hrid": "<name>-widget-v1",
		"display": "<Name> Widget V1",
		"description": "...",
		"category": "AI Examples/Basic",
		"type": "widget",
		"layoutUnique": false,
		"icon": "<mdi icon name>",
		"iconColor": "#96C21F",
		"options": [
			{ "key": "display", "label": { "en": "Text", "de": "Text" }, "type": "text", "default": "..." }
		]
		}
	</script>

	<!-- STYLE -->
	<style id="widget-styles">
		@scope {
			#widget-body {
				#element { color: var(--widget-text-color); }
			}
		}
	</style>

	<!-- TAGS -->
	<div id="widget-body">
		<p id="element" widget-text="display"></p>
	</div>

	<!-- SCRIPT -->
	<script id="widget-script">
		(function () {
			const sdx = window.sdx;
			const widget = sdx.Widget.getInstance();
			widget.ready();
		})();
	</script>

</widget>
```

## Adding the next widget here

1. Work through patterns.md's "The minimal-widget bar" checklist first — if the widget needs
   real JS beyond one thing CSS provably can't do, it probably doesn't belong in `minimal/`
   (it's still fine as a normal widget).
2. Copy the closest existing file in `minimal/`, keep the option order the user gave, and
   localize every label.
3. Save `_versions/<hrid>/rev1-initial.html` + `CHANGELOG.md` as usual (SKILL.md rule 7).
4. Add a row to `minimal/README.md` and to the techniques table above if a new mechanism
   appears.
5. Mark anything unconfirmed live in the CHANGELOG rather than assuming. As of 2026-09-17,
   `select`, `@container style()`, `@property` inside a widget style block, and the `onLoad`
   tag swap are all confirmed working in SDX; whether a `/` in `category` nests in the
   Globals panel is settled: it does NOT nest - the panel shows one flat group literally named
   "AI Examples/Basic" (screenshot 2026-09-17). The name is still worth keeping so the
   AI-written set sorts next to Ben's "Examples/Basic".

## Current members

- **button-widget-v1** — 9 options. The origin of the bar: 5 revisions from 18 options and
  ~65 lines of JS down to `widget.ready();`. Shows `widget-link`/`widget-link-new-tab`, hover
  color pairs, and `select` → `text-align`.
- **label-widget-v1** — 6 options. Merged from an AI first pass and Ben's
  `example-label-simple`. Shows boolean → CSS via `@container style()`, `text-align:
  start/end`, and localized labels throughout.
- **divider-widget-v2** — 8 options. A single `<hr>`, no data attributes, every option a CSS
  variable. Shows a `select` driving `border-style` (line drawn as one border side), an
  orientation `select` switching several rules through one nested `@container style()`
  block, `select` → `justify-content` for placing a partial-length block, and a percent
  length via `calc(* 1%)` horizontally and `calc(* 1px)` vertically (percent heights never
  resolve inside a widget - see architecture.md's host-DOM gotcha).
- **heading-widget-v1** — 7 options. label-widget-v1 plus a level `select` (h1–h6) applied by
  a 5-line `onLoad` tag swap, an optional font size that leaves the heading at the page's own
  size when empty (`@property` + `not style()`), and bold defaulting on. Its CHANGELOG lists
  what to verify on first upload.
- **dropdown-widget-v1** — 3 options. The first widget here needing JS for a reason besides
  heading's tag swap: its options are author-typed as comma-separated `"Label|URL"` pairs
  (`select` can't hold end-user-variable choices), parsed into real `<option>` elements, then
  navigated on `change`. Also the first to deliberately leave out alignment/spacing options,
  applying the slot-level-styling finding earlier in this file. Not yet tested live.
- **dropdown-scroll-widget-v1** — 2 options. Same options-parsing as dropdown-widget-v1, values
  are anchor ids instead of URLs; `document.getElementById` (deliberately not
  `widget.findElement`, since the target is meant to be outside this widget's own root) +
  `scrollIntoView({behavior:'smooth'})`. Not yet tested live.
- **dropdown-visibility-widget-v1** — 2 options. Reuses the confirmed cross-widget
  `postMessage({type:'widget-set-row-visibility'})` pattern (see architecture.md) as a view
  switcher — an option's `hide` list is computed as every OTHER option's rows, not
  author-typed, so selections are mutually exclusive by construction. What counts as a valid
  row id is still unconfirmed — see its CHANGELOG. Not yet tested live.
- **dropdown-filter-widget-v1** — 3 options. Broadcasts a generic `postMessage
  ({type:'dropdown-selection'})` with an author-set `broadcast-key`, for some future listener
  widget to filter by however it likes — mirrors `pv-performance-lib`'s shared-selection
  pattern conceptually, without creating a shared library (rule 8). Honestly the least
  immediately testable of the four dropdown variants on its own, since nothing in `minimal/`
  listens for it yet. All four dropdown variants exist side by side deliberately, so a person
  can compare and pick a direction rather than committing to one up front.
