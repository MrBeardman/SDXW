# SDX Widget Architecture

## The file format

Every widget/library is ONE `.html` file whose root element is a custom `<widget>` tag (not
real HTML — it's the SDX editor's source format, injected into the host document at runtime).
Up to five children, always in this order, any of which may be omitted:

```html
<widget>

	<!-- CONFIG (required) -->
	<script type="application/json" id="widget-config">
	{ ... see "widget-config" below ... }
	</script>

	<!-- L10N (optional) -->
	<script type="application/json" id="widget-l10n">
	{ "de": { "key": "German $var text" }, "en": { "key": "English $var text" } }
	</script>

	<!-- STYLE (optional) -->
	<style id="widget-styles">
		@scope { #widget-body { /* rules, always nested under #widget-body */ } }
	</style>

	<!-- TAGS (optional — omit entirely for "controller" widgets, see patterns.md) -->
	<div id="widget-body"> ... </div>

	<!-- SCRIPT (required) -->
	<script id="widget-script">
		(function () { /* IIFE, see below */ })();
	</script>

</widget>
```

Indentation in real files is **tabs**, not spaces (a few files mix spaces — tabs is the norm to
follow for new code).

## widget-config fields

```json
{
  "hrid": "kebab-case-unique-id-v1",
  "display": "Human Readable Name V1",
  "description": "One sentence, shown as a tooltip in the widget picker.",
  "category": "Files | General | News | Projects | PV | Test | Testing (widgets); News | PV | SDX (libraries)",
  "type": "library | widget",
  "layoutUnique": true,
  "loadIndex": 0,
  "icon": "mdi-icon-name-without-mdi-prefix",
  "iconColor": "#RRGGBB",
  "options": [
    { "key": "accentColor", "label": "Accent color", "type": "color", "default": "#96C21F" },
    { "key": "limit", "label": "Max files", "type": "text", "default": "50" },
    { "key": "showLegend", "label": "Show legend", "type": "boolean", "default": true }
  ]
}
```

- `hrid` must be globally unique and match the filename convention `<name>-v<N>`.
- `category` — confirmed real values, split by `type` since libraries and widgets have settled
  on a different (though overlapping) set in practice, not a hard constraint on the field
  itself. **Libraries** (`type: "library"`): `News`, `PV`, `SDX` (SDX holds the foundational/
  generic libraries — api/date/number/chart/data-attributes/templates/xlsx-lib, and sdx-lib
  itself). **Widgets** (`type: "widget"`): `Files`, `General`, `News`, `Projects`, `PV`, `Test`,
  `Testing` — `Test` and `Testing` are two distinct categories, not a typo of each other
  (`Test` for personal scratch widgets like test-bs-v1/test-jm-v1, `Testing` for more
  deliberate feature-probe widgets like charts-test-widget-v1/data-attributes-test-widget-v1).
  There is no generic "Basic"/"Elements"/"Components" category for foundational atomic widgets
  (buttons, text, headings) — `General` is the closest existing fit for these today. Confirmed
  via a live screenshot of the Globals panel's widget browser (2026-08-27), not guessed: Library
  Widgets 13 total (News 1, PV 3, SDX 9), UI 19 total (Files 6, General 1, News 1, Projects 1,
  PV 4, Test 2, Testing 4). **Confirmed 2026-09-17**: a `/` in the category does NOT nest — `"AI Examples/Basic"`
  renders as one flat group literally named "AI Examples/Basic" in the Globals panel
  (screenshot). It's purely a naming convention, used so the AI-written minimal widgets sort
  next to Ben's "Examples/Basic".
- `type: "library"` — reusable code only, no visible UI of its own, appears grouped under
  "Library Widgets" in the Globals panel. `type: "widget"` — a placeable, visible widget.
- `layoutUnique: true` means only one instance of this widget/library can exist per page/layout
  (all libraries set this; most single-purpose widgets like the welcome widget do too).
  `layoutUnique: false` allows multiple instances on the same page (e.g. news-widget-v1,
  report-widget-v1 — anything that can reasonably be placed more than once).
- `loadIndex` controls load order for **libraries**, ascending. Lower-level libraries load
  first: sdx-lib-v1 (core Widget class, implicitly index 0) → api/l10n/date/number/chart-lib
  (loadIndex 10) → templates/xlsx-lib (loadIndex 20) → news-lib (loadIndex 30) →
  pv-investor-lib (loadIndex 30) → pv-performance-lib (loadIndex 40) → pv-select-lib
  (loadIndex 50). A library can only use `sdx.*` packages defined by a lower loadIndex.
- `options[].type` confirmed in real, working widgets: `"text"`, `"color"`, `"boolean"`,
  `"select"`. `select` takes an `options: [{label, value}]` sub-array, and needs
  **localized-object labels** (`{"en": ..., "de": ...}`) on both the top-level option and every
  nested choice to render correctly — confirmed live (2026-09-17): button-widget-v1,
  label-widget-v1, and heading-widget-v1 all use this exact shape, tested in SDX by the user.
  This was unconfirmed for a while first, evidenced only by a commented-out sketch in
  charts-test-widget-v1.html and a reference widget from a colleague (Ben) using the identical
  shape — worth remembering as a calibration case: that same colleague's widget also used
  `widget-link`/`widget-link-new-tab` attributes that were unconfirmed at the time (the library
  hadn't implemented them yet, later confirmed real once it was updated) and an implicit
  `rel="noopener"` assumption that turned out correct per MDN once actually checked (see
  libraries.md's data-attributes-lib-v1 entry) — "looks clean and considered" isn't the same as
  "independently verified," even from a trusted source, though a colleague's draft can also
  turn out to be exactly right once the surrounding pieces catch up.
- Every option becomes a CSS custom property `--widget-<key>`, read back via
  `widget.getConfig('<key>')` (see below). This is how the SDX editor's per-instance
  settings panel reaches the widget's JS — there is no other config channel. **Where the
  host actually sets them (confirmed via DevTools, 2026-09-17)**: as an inline `style` on
  `div.sdx-widget-renderer__host`, an ancestor of the `<widget>` element — not on `<widget>`
  itself. They reach everything below by normal custom-property inheritance, which is why
  `getComputedStyle(<widget>)` and `@container style()` queries at any depth all see them.

  **Host DOM chain and its sizing gotcha.** Each widget instance renders as:
  `div.sdx-widget-renderer__host` (the `--widget-*` vars; had `height: 120px` in the
  screenshot) → `div.sdx-widget-renderer__content` (`position: relative; contain: layout;
  container-type: inline-size`, height **0**) → `body` (`min-height: 0; height: auto`,
  height **0**) → `<widget>` → your `#widget-body`. The two wrappers between the sized host
  and the widget are content-sized, so **a percentage `height` anywhere inside a widget
  resolves to nothing**, however tall the host slot is (divider-widget-v2's vertical mode hit
  this: `height: 100%` on `#widget-body` + `height: <n>%` on the line rendered 0px). Anything
  that needs a height must state it in px or get it from its own content. Percentage
  *widths* are fine — the chain is full-width (916px in the screenshot).

## The sdx.Widget class (from sdx-lib-v1, the foundational library)

Every widget script does:

```js
(function () {
	const sdx = window.sdx;           // global namespace, shared by ALL widgets on the page
	const api = sdx.api;               // (etc — whichever libs this widget needs)

	const widget = sdx.Widget.getInstance();  // binds to THIS widget's own <widget> root

	widget.onLoad(async function () { /* first render */ });
	widget.onUpdate(async function () { /* re-render on auto-update tick, optional */ });
	widget.setAutoUpdate(intervalMs);          // optional, only if onUpdate is used

	widget.ready();  // ALWAYS call this last — nothing runs without it
})();
```

Key instance methods (all defined in sdx-lib-v1, extended by other libs — see libraries.md):

| Method | Purpose |
|---|---|
| `widget.getRoot()` | the `<widget>` root element |
| `widget.findElement(sel)` / `findElements(sel)` | **scoped** query inside this widget's own root — never use `document.querySelector` directly, since many widget instances share one host document and ids are only unique per-instance |
| `widget.getConfig(key)` | reads `--widget-<key>` CSS var (the option value) as a trimmed string |
| `widget.addClassToElements(sel, cls)` / `removeClassFromElements(sel, cls)` | bulk class helpers, used internally for state switching |
| `widget.onLoad(fn)` / `widget.onUpdate(fn)` / `widget.setAutoUpdate(ms)` | lifecycle hooks |
| `widget.ready()` | kicks off the lifecycle — must be the last line of every widget script |
| `widget.getL10N()` / `widget.resolveL10N(path, refObj)` | from l10n-lib, see patterns.md |
| `widget.createTemplate(sel)` | from templates-lib, returns a `Template` instance, see patterns.md |
| `widget.createChart(options, sel)` | from chart-lib, wraps ECharts init + resize |
| `widget.applyStaticLocalization()` / `keepStaticLocalization()` | from l10n-lib, applies `data-loc-key` elements |

`Widget.onPreload(callback)` (static) lets a library register a step that runs for **every**
widget on the page right before its own `onLoad`. Used by data-attributes-lib to auto-apply
`widget-text` / `widget-visible` / `widget-background-image` / `widget-link` /
`widget-link-new-tab` attributes to every widget.

## Lifecycle / state machine

`widget.ready()` schedules `onReady()` (on window `load` and also via `setTimeout(0)`, whichever
fires):
1. Run every registered global preloader (`Widget._preloadCallbacks`, in registration order).
2. Run this widget's own `onLoad` callback.
3. On success: hide `[widget-state="loading"]` and `[widget-state="updating"]`, show
   `[widget-state="active"]`, hide `[widget-state="error"]`.
4. On thrown error (from preload OR onLoad): show `[widget-state="error"]`, and write
   `error.message` into `[widget-error-message]`. `console.error` logs it too.
5. If `setAutoUpdate(ms)` was called with `ms > 0`, schedule `onUpdate` to run again after
   `ms`, showing `[widget-state="updating"]` for ~1s (`_postUpdateVisibilityDelayMs`) around
   each tick, then reschedule itself (self-perpetuating `setTimeout` chain, not `setInterval`).

**Convention:** the `<div id="widget-body">` almost always contains up to four **sibling**
blocks marked with a `widget-state` attribute, all starting `d-none` except loading:

```html
<div widget-state="loading">Loading…</div>
<div class="d-none" widget-state="active"> ... real content ... </div>
<div class="d-none" widget-state="updating"><span class="mdi mdi-refresh-circle mdi-spin"></span></div>
<div class="d-none widget-error" widget-state="error">
	<small>An error occured in widget</small>
	<h2 widget-error-message></h2>
</div>
```
`widget-state="updating"` is usually absolutely positioned as a small spinner badge, not a
full-screen block (see report-widget-v1 / portfolio-widget-v1 / charts-test-widget-v1 CSS).
Throwing an `Error` with a localized message inside `onLoad` is the standard way to show an
empty/error state (e.g. news-widget-v1 throws when there are zero items) — **but see
patterns.md for when the whole state machine can be skipped entirely** (button-widget-v1 has no
loading/active/error markup at all, since it's synchronous and can't fail).

## "Controller" widgets (no template/style block)

A widget MAY omit the TAGS and STYLE blocks entirely and keep `layoutUnique: true` — it then
has no visible markup of its own and instead reaches into OTHER widgets already placed on the
layout by id, using plain `document.querySelector` (not `widget.findElement`, since it targets
elements outside its own root) and ids configured via its own `options`. Example:
preview-test-widget-v1.html. Say so with a leading HTML comment:
`<!-- This is a controller widget. It contains only a l10n and a script block... -->`.

## Shared/global widget CSS variables (from sdx-lib-v1)

```css
widget {
	--widget-border-radius: 15px;
	--widget-default-background-color: #fff;
	--widget-default-error-background-color: #FFD4D4;
	--widget-default-error-color: #AA0000;
}
```
Use `var(--widget-border-radius)` / `var(--widget-default-background-color)` for card corners
and backgrounds so every widget stays visually consistent; don't hardcode these.

## Cross-widget communication

- `window.postMessage({type: 'widget-set-row-visibility', show: [rowId...], hide: [rowId...]},
  '*')` — used by overview-widget-v1 to show/hide OTHER layout rows (by a row-id configured via
  options) based on this widget's own state (e.g. superadmin tabs). This implies the SDX editor
  listens for this message type at the layout/host level.
- Module-level (not instance-level) shared state inside a library is the pattern for state
  shared across multiple widget instances on one page — see `pvPerformance.getSelection` /
  `setSelection` / `onSelectionChange` (a plain closure variable + callback array inside
  pv-performance-lib, not attached to any one widget). Use this pattern for e.g. "this filter
  widget controls that list widget" scenarios, same as chooser-widget-v1 → pv-performance-lib →
  report/portfolio-widget.
- No confirmed way for one widget to directly reference/call into another (a wished-for
  `widget.getOtherWidget("id").onClick(...)`-style API does not exist anywhere in this
  codebase) — the two patterns above (`postMessage`, module-level shared state in a library)
  are the only confirmed mechanisms. Treat any other cross-widget interaction idea as something
  to test for, not assume.

## Slot-level styling and advanced settings (provided by the host, not the widget)

Every widget placed on a layout gets its own Style tab and Advanced tab in the editor,
generated by the SDX layout builder itself — not by the widget's own `options[]`. These apply
to the widget's outer slot/wrapper, entirely independent of whatever the widget's own config
defines (confirmed via screenshot):

**Style tab**: Color, Alignment, Border radius, Margin (top/right/bottom/left, linkable),
Padding (top/right/bottom/left, linkable), Align self, Order, Width, Height, Position. A
"Mockup configuration" sub-section (background color, height) affects only how the slot
previews in the layout builder, not the live widget.

**Advanced tab**: Custom ID, Custom Classes, Custom CSS — with the editor's own instructions
telling the author to scope rules to `#slot-id { ... }` so they don't leak onto the rest of the
page — and Z-Index.

**Implication for widget authoring**: don't bake generic slot-level layout — margin, padding,
width, height, position, alignment of the whole widget within its row — into a widget's own
`options[]`. The host already provides all of this, for every widget, with zero config needed
on the widget's part. A widget's own options should cover only things specific to its own
inner content/behavior (a button's own border-radius on the button element itself is fine —
that's not the same as the slot's border-radius around it). See minimal-widgets.md for an open
question this raises about one existing widget's options.
