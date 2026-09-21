# Recurring Patterns

## The minimal-widget bar (reference: button-widget-v1)

button-widget-v1 went through 5 revisions (`_versions/button-widget-v1/CHANGELOG.md` has the
full history) and ended at 9 options and a single line of custom JS: `widget.ready();`. That's
now the target shape for every widget in this project — not a one-off, not specific to buttons.
Before writing ANY custom JS for a new widget, work through these in order:

1. **Does `data-attributes-lib` already cover it declaratively?** `widget-text` (content from a
   config option), `widget-link` (href), `widget-link-new-tab` (target — and per MDN this alone
   already blocks `window.opener`, no separate `rel="noopener"` needed), `widget-visible`
   (show/hide), `widget-background-image`. Check libraries.md's data-attributes-lib-v1 entry for
   the current, confirmed, complete list rather than assuming from an example — that list has
   already changed mid-project once.
2. **Does CSS alone cover it?** An alignment option doesn't need a flex container + JS
   class-toggling if `text-align` + `display: inline-block` gets the same result with zero JS
   (see the pattern below). A numeric option becoming a CSS length doesn't need string
   concatenation in JS — `calc(var(--widget-x) * 1px)` does it in CSS. A **boolean** option
   doesn't need JS class-toggling either — see "Pattern: boolean option → CSS via style
   container query" below.
3. **Does this widget need the loading/active/error state machine at all?** Skip it entirely
   (not just `d-none` it) for anything synchronous and config-only — see the pattern below.
4. **Is every option actually needed, or just conceivably useful?** button-widget-v1's first
   revision had 18 options, built to deliberately test the full breadth of what confirmed
   option types could cover — that was a worthwhile experiment, but not the target shape for a
   shipped widget. Once real feedback came in, it dropped to exactly 9. Don't keep an
   exploratory version's scope by default once a widget moves past that exploration phase.

Only once all four are genuinely exhausted does custom `widget.onLoad` logic belong in a
widget. A widget whose entire script is `widget.ready();` isn't unfinished — for something this
simple, it's often the correct end state, not a placeholder.

## Rendering a list (Template class)

Standard way to turn an array of data into repeated DOM. Used by news-widget-v1,
overview-widget-v1.

```html
<div id="list" class="d-none" widget-state="active">
	<div id="item-template" class="d-none">
		<a class="card" template-route="link" template-background-image="image" template-class="flavor">
			<h3 template-text="title">placeholder</h3>
			<span template-visible="hasSubtitle" template-text="subtitle"></span>
		</a>
	</div>
</div>
```
```js
function _prepareData(items) {
	return items.map(function (item) {
		return {
			title: item.title,
			subtitle: item.subtitle,
			hasSubtitle: !!(item.subtitle && String(item.subtitle).trim()), // MUST be a real boolean
			image: item.imageId,
			link: '/spaces/' + item.spaceId,   // or template-route for in-app router navigation
			flavor: item.isFeatured ? 'is-featured' : null
		};
	});
}

widget.onLoad(async function () {
	const data = _prepareData(await loadStuff());
	const container = widget.findElement('#list');
	const template = widget.createTemplate('#item-template');
	template.append(data, container);
});
```
Notes:
- The template root itself needs `class="d-none"` (it's the master, never shown directly);
  `Template.process` removes `d-none` on each **clone**.
- `template-visible` compares strictly against `true`/`'true'` — always compute a real boolean,
  never a truthy-ish value.
- `template-link` sets a plain `href` (full page reload). `template-route` sets `href` too but
  intercepts click to go through `window.parent.Application.router.push(value)` (SPA
  navigation) unless a modifier key is held or the router is unavailable — use `template-route`
  for links to other SDX spaces/pages, `template-link` for truly external URLs.
- `template-class` ADDS one class (doesn't replace); pass `null`/`undefined` to add nothing.
- Re-rendering: templates-lib has no "clear" helper — widgets that re-render on `onUpdate`
  typically rebuild a chart in place (`chart.setOption`) rather than re-running `template.append`
  on the same container repeatedly. If you need to refresh a template-rendered list, clear the
  container's `innerHTML` yourself first (no existing widget does this today — ask if unsure).
- Need a per-clone element reference (e.g. for a client-side filter)? Use `template.process(data,
  callback)` instead of `.append` — see jpg-gallery-widget-v1.html for a worked example.

## Custom dropdown (pv-select)

Not PV-specific despite the name — general reusable dropdown, e.g. for a period/sort filter.
```html
<div id="my-select" class="pv-select pv-select-small"> <!-- omit "-small" for the bigger pill style -->
	<button type="button" class="pv-select-button">
		<span pv-select-label></span>
		<i class="mdi mdi-chevron-down" aria-hidden="true"></i>
	</button>
	<ul class="pv-select-menu" role="listbox"></ul>
</div>
```
```js
pvSelect.fill(
	widget.findElement('#my-select'),
	[{text: 'All', value: ''}, {text: 'Foo', value: 'foo'}],
	currentValue,               // preselects the matching option
	function (value) { /* handle change */ }
);
```
Requires the `.pv-select*` CSS block from pv-select-lib-v1 to be present (it's a shared style,
not per-widget — comes for free once pv-select-lib-v1 is loaded on the page).

## Charts (ECharts via chart-lib)

```js
let chart = null;
function _render() {
	const options = { /* standard ECharts option object */ };
	if (chart === null) {
		chart = widget.createChart(options, '#chart-container');
		_observeSize(chart); // see below — required, chart container is hidden during onLoad
		return;
	}
	chart.setOption(options, true);
}
function _observeSize(chart) {
	new ResizeObserver(function () { chart.resize(); }).observe(widget.findElement('#chart-container'));
}
```
**Always add the ResizeObserver.** The widget body is still `d-none` while `onLoad` runs, so
`echarts.init` measures a zero-size box; `createChart` does wire a generic `window.onresize`,
but that alone won't fire when the box becomes visible without an actual window resize — the
ResizeObserver is what makes the first render size correctly in every real example.

## Config-driven text/visibility/background without JS (data-attributes-lib)

For simple, static-per-instance content driven directly by a widget option (not a data list):
```html
<h1 id="title" widget-text="title">-</h1>
<p id="description" widget-text="description" widget-visible="description-visible">-</p>
```
where `"title"` / `"description-visible"` are OPTION KEYS from `widget-config.json`'s
`options[]` array — no script code needed at all to wire this up; it's applied automatically
via `Widget.onPreload`. Use this instead of manual `widget.onLoad` + `textContent =
widget.getConfig(...)` wiring whenever the value is a static option, not fetched data.

## l10n

```json
// widget-l10n block
{ "de": {"loading": "Wird geladen…", "greet": "Hallo $name"}, "en": {"loading": "Loading…", "greet": "Hello $name"} }
```
```js
widget.findElement('[widget-state="loading"]').textContent = widget.resolveL10N('loading', {});
el.textContent = widget.resolveL10N('greet', {name: user.display});
```
Always provide both `de` and `en`. Every user-facing string in a widget — including the literal
"Loading…" text in the loading state div — gets moved into l10n and resolved in `onLoad`, even
though the markup shows a hardcoded placeholder string for editor-preview purposes.

## Options: config → CSS var → widget.getConfig

Declaring `{"key": "accentColor", "type": "color", "default": "#96C21F"}` makes
`getComputedStyle(widgetRoot).getPropertyValue('--widget-accentColor')` resolve to the
configured (or default) value; `widget.getConfig('accentColor')` is the correct/only way to
read it (trims whitespace, returns `''` if unset — don't read the CSS var manually).
Boolean options come back as the **string** `'true'`/`'false'` — compare with
`widget.getConfig('showLegend') === 'true'`, not a truthy check.

## Loading / active / updating / error state block

Boilerplate present in nearly every visible widget (copy verbatim, adjust content):
```html
<div id="widget-body">
	<div widget-state="loading">Loading…</div>
	<div id="my-content" class="d-none" widget-state="active"> ... </div>
	<div class="d-none" widget-state="updating"><span class="mdi mdi-refresh-circle mdi-spin"></span></div>
	<div class="d-none widget-error" widget-state="error">
		<small>An error occured in widget</small>
		<h2 widget-error-message></h2>
	</div>
</div>
```
```js
widget.onLoad(async function () {
	widget.findElement('[widget-state="loading"]').textContent = widget.resolveL10N('loading', {});
	const data = await loadSomething();
	if (!data.length) {
		throw new Error(widget.resolveL10N('noneFound', {})); // -> shows error state, that's the intended empty-state mechanism
	}
	// render into #my-content
});
```
There's no separate "empty state" UI convention distinct from the error state — an empty result
throws a localized `Error`, which the shared `_handleError` machinery displays. Only add
`widget-state="updating"` and call `setAutoUpdate`/`onUpdate` if the widget's data can go stale
while the page is open (a live chart, "today" stats) — static/one-shot widgets (welcome,
data-attributes-test) skip updating entirely.

**Exception: a purely synchronous, config-only widget doesn't need the state machine at all.**
If a widget has no network/async work and no realistic failure mode (e.g. button-widget-v1 —
everything it does is read a config value and set a DOM property), the loading/active/error
`<div>`s can be dropped entirely, not just left `d-none`. `widget.onLoad(fn)` and
`widget.ready()` are still required — that's the framework's own lifecycle contract, unrelated
to whether visual state markup exists — but `Widget`'s internal state-toggling calls
(`hide loading`, `show active`, etc.) are just `querySelectorAll` + iterate, which silently
no-op when nothing matches. `#widget-body` can then be the widget's actual content directly,
with no state-wrapper `<div>`s inside it at all. This can roughly halve a simple widget's TAGS
block.

## Widget CSS scoping

```css
@scope {
	#widget-body {
		.some-local-class { ... }   /* nest EVERYTHING under #widget-body */
	}
}
```
`@scope` + nesting under `#widget-body` keeps a widget's CSS from leaking onto sibling widgets
in the same host document (they all share one DOM/stylesheet context, not iframes). Always wrap
widget-specific styles this way; only the truly global bits (like `.pv-select*` in
pv-select-lib) live unscoped, and that's a deliberate exception for a shared component.

## General style conventions to match

- `(function () { ... })();` IIFE wraps the whole `#widget-script` body.
- `// Uses` block at the top destructuring needed `sdx.*` packages into `const`s.
- `// Globals` block for widget-local constants (`SCREAMING_SNAKE_CASE`).
- Private helper functions: `function _verbNoun(...) { ... }`, leading underscore, JSDoc block
  above every one (`@param`, `@returns`) even for tiny internal helpers.
- Public library methods: `package.methodName = function (...) { ... };` (not arrow functions
  anywhere in this codebase — stick to `function` keyword throughout, ES5-style, though
  `const`/`let`/template literals/arrow-free destructuring are used freely).
- `// @todo ...` comments mark known rough edges (hardcoded locale, fragile URL parsing) —
  follow this convention for anything you deliberately leave as a known limitation.
- Comments explain **why**, not what the next line obviously does.
- Prefer local CSS custom properties (`--jpg-gap: 14px;` etc, declared once at the top of
  `#widget-body`) over repeating literal values for anything genuinely specific to that widget,
  and style singular (non-repeated) elements by their existing `id` directly rather than adding
  a parallel, redundant class — reserve classes for elements that are genuinely repeated
  (cloned cards, repeated pills), since those can't be unique ids anyway. See the utility-class
  pattern below first, though — plain spacing/layout usually belongs there, not in a custom
  property at all.

## Gotcha: always set an explicit text color

The SDX host app's own ambient/inherited text color is **white** (`rgb(255,255,255)`, confirmed
live — it runs a dark admin theme). Any widget element that relies on inherited color instead of
setting its own will render invisible white-on-white if it sits on a light card background
(e.g. `var(--widget-default-background-color)`, which is `#fff`). This bit jpg-gallery-widget-v1
exactly this way: `.jpg-name` had no explicit `color`, rendered perfectly, and was completely
invisible until diagnosed via `getComputedStyle`. Every text element inside a light-background
card needs an explicit `color` — don't rely on inheritance from the page.

## Gotcha: uniform-height grid items

CSS Grid stretches its immediate children (`align-items: stretch` is the default), but that
stretch only reaches the direct grid child — which, with templates-lib, is the *cloned template
wrapper*, not the visible card markup one level inside it. The wrapper silently grows to match
the tallest row-mate; the visible card inside it does not, unless told to:
```css
.jpg-card { height: 100%; }   /* fills whatever height the (auto-stretched) wrapper ends up at */
```
Pair this with a line-clamp on any free-text field (file names, titles) so one long value can't
make a single card taller than its row to begin with — same `-webkit-line-clamp: 2` pattern
`.description` already uses in overview-widget-v1 and `.news-text` uses in news-widget-v1.

## Gotcha: auto-fit vs. auto-fill in a responsive grid

`grid-template-columns: repeat(auto-fit, minmax(200px, 1fr))` looks identical to `auto-fill`
until there are fewer items than columns — `auto-fit` collapses the empty tracks and hands their
width to whatever's left, so a single remaining card (e.g. after a filter narrows things down)
stretches to fill the entire row. `auto-fill` reserves the tracks instead, so remaining cards
keep their normal width regardless of count. Default to `auto-fill` for any card/thumbnail grid
where the item count can vary at runtime (filtering, searching) — `auto-fit` is only right for
a genuinely fixed, unfiltered set.

## Gotcha: the widget's own container needs its own inset padding — the host gives none

The SDX host page gives a widget virtually no surrounding space. Any effect that visually
extends past an element's own box — a `:focus` glow, a hover `box-shadow`, anything using
`filter: drop-shadow` — gets clipped at the widget's outer edge unless the widget adds its own
padding to create room. Add `padding` directly to `#widget-body` (not just to inner elements) as
a standard defensive step for any widget using hover/focus effects that extend beyond an
element's box.

Don't assume uniform padding is enough, and don't just guess a number — work out roughly how far
each effect actually reaches and pad the affected edge(s) accordingly. For a `box-shadow: <x> <y>
<blur> <spread> <color>`, the effect reaches roughly `y + blur - spread` beyond the box on the
side the offset points toward (and roughly `blur - spread` on the other edges). A shadow like
`0 18px 34px -14px` reaches ~38px below a card but only ~20px to its sides, so
`padding: 10px 10px 28px` (more on the bottom) is the right shape — not `padding: 20px` uniformly
on every side. jpg-gallery-widget needed two separate correction passes for this because the
first fix used a uniform, arbitrarily-sized padding rather than sizing it to the actual shadow.

## Gotcha: an element meant to stay put can't be nested inside something that transforms

Any CSS `transform` (scale, rotate, translate) applied to an element visibly moves every
descendant painted inside it, including an absolutely-positioned overlay badge/chip that's meant
to stay fixed in place. If a badge must stay visually pinned (e.g. top-right corner of a card)
regardless of a sibling image zooming on hover, it needs to be a SIBLING of the transforming
element, not a CHILD — anchored via `position: relative` on a shared ancestor that itself never
transforms (e.g. the outer card), with the badge `position: absolute` against that ancestor.
jpg-gallery-widget's meta chip was originally nested inside `.jpg-thumb`, which gets
`transform: scale(1.08)` on hover (see the background-size gotcha above) — since the chip was a
child of the thing being scaled, it visibly drifted along with it. Moving the chip to be a
sibling of `.jpg-thumb`, both anchored to `.jpg-card`, fixed it.

## Note: check whether a shared component class already covers a style before adding a local override

Before adding a local CSS override for how a shared component (`.pv-select-small`, etc.) looks —
padding, font-size, colors — check what that shared class already provides. jpg-gallery-widget
briefly carried a `#jpg-filters-panel .pv-select-button { padding: ...; font-size: ...; }`
override that duplicated (and slightly diverged from) what `.pv-select-small` already does,
adding clutter for no visual benefit. Trust the shared styling unless there's a genuine,
deliberate reason to diverge from it — and if there is one, say so in a comment.

## Pattern: local code that's really a future library candidate

When a widget needs something no current library wraps (rule 8 — library changes need their
own explicit go-ahead, separate from widget work), write the logic locally rather than
touching the shared file, but mark it clearly so it's easy to find and promote later without
re-deriving the reasoning:
```js
// @todo (library candidate for api-lib-v1): dlApi.contents.streamAsGetURL(id, true) wrapper -
// see _versions/_shared-libraries/api-lib-v1.proposed-extensions.html for a ready-to-apply draft
function _getPreviewURL(id) {
	return api.getApi().contents.streamAsGetURL(id, true);
}
```
Use `@todo (library candidate for <lib-name>): ...` consistently so every instance is
grep-able in one search, distinct from other `@todo`s. If a draft extension for that library
already exists (e.g. in `_versions/_shared-libraries/`), point straight to it rather than
re-explaining the reasoning inline.

## Pattern: use the host page's existing utility classes instead of local spacing CSS

The SDX host page already ships a full Bootstrap-style utility layer, confirmed live and not
documented anywhere in the widget/library source: `mt-0`…`mt-16` / `mb-`/`ml-`/`mr-`/`mx-`/`my-`
(margin, all four sides plus x/y shorthands) and the same set for padding (`pt-`/`pb-`/`pl-`/
`pr-`/`px-`/`py-`), all on a consistent **4px-per-step** scale (`mt-1` = 4px, `mt-4` = 16px,
`mt-10` = 40px, up to `-16` = 64px) — plus `d-none`/`d-block`/`d-flex`, `flex-row`/
`flex-column`, `text-left`/`text-center`/`text-right`, `.rounded` (a fixed 4px radius, not a
scale), `.font-weight-bold`, `w-100`/`h-100`. Every rule carries `!important`, so a utility
class always wins over local widget CSS targeting the same property — don't fight it, use it.

No widget in the codebase currently uses any of this layer; every widget so far hand-rolls its
own local spacing/layout CSS instead. Prefer these classes directly in markup for generic
spacing/layout/alignment needs, and reserve a widget's own `--jpg-*`-style custom properties
and local classes for things genuinely specific to that widget (colors, sizes, effects) — not
for a plain margin or a flex row that the existing utility layer already covers.

## Pattern: boolean option → CSS via style container query (zero JS)

A `type: "boolean"` option reaches the widget as the bare token `true`/`false` in its
`--widget-<key>` custom property. Plain CSS can't branch on that with `var()`, but a **style
container query** can — nest it inside the rule for the element that should change:
```css
#label {
	@container style(--widget-bold: true)   { font-weight: bold; }
	@container style(--widget-italic: true) { font-style: italic; }
}
```
Every element is a style-query container by default (no `container-type` needed). The query
evaluates against the styled element's nearest *ancestor* — `#widget-body` for anything inside
it — which inherits the host-written `--widget-*` properties, so it just works. Match on
`: true` explicitly; a bare `style(--widget-bold)` is only a "has a value" check and would also
match `false`. Introduced in label-widget-v1 rev2, and independently used the same way in Ben's
`example-label-simple.html` (so it's the platform author's intended approach). **Confirmed
working live in SDX on 2026-09-17** (label-widget-v1 and heading-widget-v1 both uploaded and
tested by the user). Support: Chromium 111+, Safari 18+, Firefox more
recent — verify against the browsers SDX users actually run before relying on it for anything
essential. If it silently does nothing, inspect in DevTools what exact token the host writes for
a boolean option (`true` vs `"true"`) before blaming the query.

## Gotcha: a `type: "text"` option value must be a single unbroken CSS token, not "12 px"

Config options feed into `--widget-<key>` custom properties, consumed via e.g.
`border-radius: var(--widget-border-radius);`. A custom property's *declaration* accepts almost
any token stream, so a value like `12 px` (a stray space between the number and the unit) is
stored without complaint — but once it's substituted into a real property via `var()`, CSS
re-parses the result as a `<length>`, and `12`+`px` as two separate tokens isn't a valid one.
The whole declaration becomes invalid and silently falls back to the initial/inherited value —
`border-radius: 0` in this case — with no error or visual sign of what went wrong.

Confirmed empirically:
```css
.a { --x: 12 px; border-radius: var(--x); }  /* computed border-radius: 0px  — broken */
.b { --x: 12px;  border-radius: var(--x); }  /* computed border-radius: 12px — works  */
```
`widget.getConfig()`'s whitespace trim doesn't save this — it only strips leading/trailing
whitespace, not the internal space between the number and the unit.

For any `type: "text"` option whose value needs to become a valid CSS length (border-radius,
sizes, gaps...), either: (a) keep the option's value a bare number with no unit at all, and
append the unit in the widget's own CSS via `calc(var(--widget-x) * 1px)` — robust to
accidental leading/trailing whitespace, since `calc()` still parses cleanly around it; or
(b) if the option genuinely needs to accept arbitrary CSS length units (px/rem/%), make the
"no space between number and unit" requirement explicit in the option's `label`, since a plain
text field gives no other protection against this.

Treat any `type: "text"` option that needs to become valid CSS syntax as fragile until proven
otherwise — same category of risk as `type: "select"` being unconfirmed.

## Gotcha: transition background-size at your peril

CSS transitions generally can't smoothly interpolate a keyword (`background-size: cover`) into
a percentage (`background-size: 112%`) — the base state and the hover state are fundamentally
different value types, so the browser either snaps instantly or doesn't animate at all instead
of the intended smooth zoom. This bit jpg-gallery-widget-v1's hover-zoom effect exactly this
way. Use `transform: scale(...)` for any hover-zoom instead — it's always smoothly animatable
(and GPU-accelerated), and gets cleanly clipped by an ancestor's `overflow: hidden` the same way
`background-size` would have been:
```css
.thumb { background-size: cover; transition: transform .35s ease; }
.card:hover .thumb { transform: scale(1.08); }  /* .card needs overflow: hidden */
```

## Gotcha: pill-shaped inputs need asymmetric-feeling padding

A `border-radius: 999px` input/button with plain `padding: 10px 16px` reads as "cut off" against
the curve — the same reason `pv-select-button` uses `14px 20px 14px 22px` rather than a flat
value. Match that convention (roughly `12px 22px` for a full pill) rather than inventing a new
number per widget, and add `box-sizing: border-box` + `-webkit-appearance: none` defensively on
any custom-styled `<input>`.

## Pattern: empty results are not an error

An empty result set is a normal state, not a failure — don't `throw` on `!data.length` just to
land in the shared error-state machinery (see jpg-gallery-widget-v1's own earlier revision for
the anti-pattern). Instead, drive a dedicated empty-state element from within your own render/
filter function, and distinguish "genuinely nothing to show" from "a filter narrowed a non-empty
set down to nothing" if both are possible — they usually want different copy ("No files found."
vs. "No matches."). Reserve the thrown-`Error`-into-`widget-state="error"` path for things that
actually went wrong (a failed request, a missing dependency), not for an empty-but-successful
query.

## Client-side search/filter over an already-rendered list

New pattern, introduced in jpg-gallery-widget-v1.html — no shared library helper for this yet
(deliberately kept inline for this first widget; consider promoting to templates-lib if a
second widget needs it). Keep the query (server-side data fetch) and the filter (client-side,
zero network calls) clearly separate:
```js
const cardRefs = [];
const template = widget.createTemplate('#item-template');
template.process(data, function (tInst) {   // .process, not .append — gives you the element back
	container.append(tInst);
	cardRefs.push(tInst);                    // cardRefs[i] always matches data[i]
});

searchInput.addEventListener('input', function () {
	const query = searchInput.value.trim().toLowerCase();
	cardRefs.forEach(function (card, i) {
		card.classList.toggle('d-none', data[i].searchName.indexOf(query) === -1);
	});
});
```
