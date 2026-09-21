# Library Catalog

All libraries attach onto the shared `window.sdx` namespace. Load order (ascending
`loadIndex`, see architecture.md) matters — a library can only use packages defined by an
already-loaded, lower-index library.

## sdx-lib-v1 (loadIndex implicit/0) — `sdx.Widget`
The foundational class. See architecture.md for the full lifecycle. No other package.

## api-lib-v1 (loadIndex 10) — `sdx.api`
Wraps `window.parent.Application.api` (aliased internally as `dlApi`) — the actual SDX/DL
backend API object. **Never call `dlApi` methods directly in a widget** — always go through
`sdx.api` wrappers below, or add a new wrapper here if something's missing (ask the user first
if unsure it belongs here vs. inline in the widget).

| Method | Returns / does |
|---|---|
| `api.getApi()` | raw `window.parent.Application.api` (the underlying `dlApi`) |
| `api.getSessionUser()` | current logged-in user object (`{id, display, ...}`) |
| `api.getCurrentItemId()` | id of the content currently open (detail/preview context) |
| `api.getContentById(id)` | `dlApi.contents.get(id)` → Content metadata, see content-api.md |
| `api.streamContentById(id, preview=true, cacheBust=false)` | `fetch()`s the raw file bytes as a Response (credentials included, no-store). **Has a real bug**: `preview = preview \|\| true` forces `preview` to `true` even when explicitly passed `false` — see content-api.md |
| `api.isSuperAdmin()` | boolean |
| `api.getSpaceUuid()` | current space's UUID parsed from `window.parent.location.pathname` via `/spaces/([0-9a-f-]{36})/i`, or `null` outside a space. **Fragile/`@todo`-flagged in source** — relies on URL shape, no stable API for this yet |
| `api.isDeveloperMode()` | reads `sessionStorage['developerMode'] === 'true'` on the parent window — used to skip caching, and widgets often `console.log` extra debug info when true |
| `api.keepLatestVersions(contents)` | dedupes an array of Content by `parentId+name`, keeping the newest `created` |
| `api.findLatest(query)` | `dlApi.contents.find(query)` then `keepLatestVersions` on `response.result` |
| `api.findByLogicalType(logicalType, limit=100, hidden=false)` | shorthand for `dlApi.contents.find({contentLogicalType, contentHidden, limit})` + dedupe |
| `api.findByName(nameLike, limit=100, hidden=false)` | shorthand for `dlApi.contents.find({contentName, contentHidden, limit})` + dedupe — `contentName` is a SQL LIKE match (`%.jpg`), see content-api.md. Added upstream (Ben) 2026-08-26; jpg-gallery-widget predates it and still builds this query by hand via `api.findLatest` |
| `api.setBackgroundImage(element, imageId)` | sets `element.style.backgroundImage` to `dlApi.contents.streamAsGetURL(imageId, true)` |

**Don't hand-roll what api.findLatest already does.** jpg-gallery-widget-v1's first two
revisions called `dlApi.contents.find(query).then(r => r.result)` directly instead of
`api.findLatest(query)` — which does exactly that, plus version dedup, in one call. This is
easy to slip into when building a query with options `api.findLatest`/`api.findByLogicalType`
don't obviously cover (e.g. multiple LIKE patterns merged together) — the fix is still to call
the wrapper per-pattern and merge/dedupe the *results*, not to reimplement the wrapper's own
body inline. If a genuinely new query shape is needed that no existing wrapper covers, add a
new wrapper to api-lib-v1 rather than calling `dlApi` ad hoc inside a widget — but only with an
explicit go-ahead (SKILL.md rule 8). **Drafted but NOT applied**: `api.getPreviewURL(id)`,
`api.getDownloadURL(id)`, `api.getUsersByIds(ids)` and `number.formatFileSize(bytes)` exist only
in `_versions/_shared-libraries/*.proposed-extensions.html` — they were briefly live for
jpg-gallery rev6, then reverted (rev7). Do not call them from a widget; the live libraries don't
have them. See `_versions/_shared-libraries/README.md` for the drafts and CHANGELOG.md for what
they would add.

## l10n-lib-v1 (loadIndex 10) — `sdx.l10n`, plus `Widget.prototype` extensions
- `l10n.getCurrentLocale()` — `window.parent.currentLocale || localStorage['i18n.locale'] ||
  'en'`, wrapped in try/catch defaulting to `'en'`.
- `l10n.translate(path, fallback)` — resolves a dotted path against
  `window.parent.Application.customLocalization[locale]`; returns `fallback` if missing.
- `l10n.resolveL10N(locale, l10nObj, path, reference)` — resolves a dotted path against a
  passed-in l10n object (i.e. the WIDGET's OWN `#widget-l10n` JSON, not the app-wide one above),
  then does `$var` substitution against `reference` (regex `/\$\w+/g`, no escaping).
- `l10n.applyStaticLocalization(root)` — for every `[data-loc-key]` element under `root`, sets
  `textContent` from `l10n.translate(key, null)` if found. This is for the APP-WIDE
  localization dictionary, not a widget's own l10n block.
- `l10n.onLocaleChange(callback)` — listens for a `locale-changed` CustomEvent on
  `window.parent`, updates `window.parent.currentLocale`, then calls `callback`.
- **Widget instance methods** (added onto `sdx.Widget.prototype`):
  - `widget.getL10N()` — parses this widget's own `#widget-l10n` JSON block.
  - `widget.resolveL10N(path, ref)` — the one actually used everywhere:
    `widget.resolveL10N('loading', {})` or with vars `widget.resolveL10N('chartTitleFor',
    {project: title})` matching `"chartTitleFor": "... $project"` in the l10n JSON.
  - `widget.applyStaticLocalization()` / `widget.keepStaticLocalization()` — app-wide
    `data-loc-key` variant scoped to this widget + auto re-apply on locale change.

  **Two separate localization systems exist** — don't confuse them: (1) each widget's own
  `#widget-l10n` block + `resolveL10N`, used for all widget-authored copy (loading text, empty
  states, labels); (2) the app-wide `data-loc-key` / `customLocalization` system, used only
  rarely (seen in the legacy standalone-document prototypes, not the current `<widget>` widgets)
  for text owned by the host app itself, not the widget. **New widgets should use their own
  `#widget-l10n` block + `resolveL10N`, matching every current example.**

## date-lib-v1 (loadIndex 10) — `sdx.date`
- `date.toDate(value)` — accepts Date, Excel serial number, or date string → `Date|null`.
- `date.startOfDay(dateObj)` → epoch ms of midnight that day.
- `date.toDayKey(dateObj)` → `"YYYY-MM-DD"`.
- `date.toMonthKey(dateOrDayKeyString)` → `"YYYY-MM"`.
- `date.toHours("HH:MM")` → decimal hours (`14:30` → `14.5`) or `null`.
- `date.formatDate(value)` — `toLocaleDateString('de-DE')`. **Locale is hardcoded**, flagged
  `@todo` in source — should really use the user's locale via `l10n.getCurrentLocale()`.
- `date.getDayTimeCategory()` → one of `date.CATEGORY_{NIGHT,MORNING,NOON,AFTERNOON,EVENING}`
  by current hour (22–5 night, 6–10 morning, 11–13 noon, 14–17 afternoon, 18–21 evening).

## number-lib-v1 (loadIndex 10) — `sdx.number`
- Constants: `number.SIGN_CELSIUS` (°), `SIGN_EURO` (€), `UNIT_MWH`, `TIME_MS/TIME_SEC/TIME_MIN`.
- `number.safeDivide(a, b)` → 0 instead of NaN/Infinity on bad input.
- `number.toMS(value, unit)` → e.g. `number.toMS(15, number.TIME_MIN)`.
- `number.formatNumber(value, unit?)` — `toLocaleString('de-DE')`, unit appended with a space.
- `number.formatCurrency(value)` — `de-DE`, hides decimals when the amount is a whole euro,
  otherwise 2 decimals, `€` suffix (`,- €` form for whole numbers).
- `number.formatTemperature(value)` — rounds to whole degrees + `°`, leaves an already-suffixed
  string alone. **Also hardcodes `de-DE`** — same `@todo` as date-lib.

## chart-lib-v1 (loadIndex 10) — `sdx.charts`, plus `Widget.prototype.createChart`
- `charts.getEcharts()` — `window.parent.Application.echarts`.
- `widget.createChart(chartOptions, selector)` — `echarts.init` on `widget.findElement(sel)`,
  `setOption(options, true)`, wires `window.onresize` to `chart.resize()` (note: this
  overwrites `window.onresize` globally rather than adding a listener — fine for one chart
  widget, but watch for collisions if a widget has multiple charts; use `ResizeObserver`
  instead for that case, as report-widget-v1/portfolio-widget-v1 do via their own
  `_observeSize` helper).
- `charts.buildShades(fromColorHex, toColorHex, count)` — builds `count` interpolated hex
  colors between two colors (channel-wise lerp), used for pie/donut palettes.

## templates-lib-v1 (loadIndex 20) — `sdx.templates`, plus `Widget.prototype.createTemplate`
See patterns.md "Rendering a list" for full usage. Data-attribute names:
`template-text`, `template-link`, `template-route`, `template-class`, `template-visible`,
`template-background-image`. `widget.createTemplate(selector)` returns a `Template` bound to
the element at `selector` (which must start `class="d-none"`, as it's the master row that gets
cloned once per data entry). `.append(dataArray, containerElement)` clones+fills+appends one
instance per array entry. `.process(dataArray, callback)` is the lower-level primitive `.append`
is built on — use it directly when you need a reference to each rendered clone (e.g. for a
client-side filter), since it hands you each clone via `callback(tInst, entry)` before you
append it (`entry` = the data object that clone was filled from; added upstream 2026-08-26).

## xlsx-lib-v1 (loadIndex 20) — `sdx.xlsx`
Wraps SheetJS (`window.parent.Application.xlsx`, docs at https://docs.sheetjs.com/).
- `xlsx.fetchSheetRows(contentId, sheetName)` → downloads + parses one sheet → array of row
  objects (first row = keys).
- `xlsx.Workbook.loadInstance(contentId)` → `Promise<Workbook>`, then:
  - `.getSheetAsJson(sheetName)` → array of row objects.
  - `.getSheetAsMap(sheetName)` → object built from column A = key, column B = value (for
    "details"-style key/value sheets).
- Sheet lookup is case-insensitive by name, falls back to the first sheet if not found.

## news-lib-v1 (loadIndex 30) — `sdx.news`
Domain-specific: loads "news" xlsx files by `contentLogicalType`, reads a sheet, applies
from/till date visibility windowing, sorts newest first, falls back to a default background
image. Read the file directly if building something news-related; not general-purpose.

## pv-investor-lib-v1 / pv-performance-lib-v1 / pv-select-lib-v1 (PV category)
Solar-investment-portfolio domain logic (project loading, session-cached "investor" object,
today/month/portfolio aggregation, a shared project-selection singleton, and a styled custom
`<select>`-alike dropdown component `sdx.pvSelect`). Highly domain-specific to the PV
investment product — read the source directly if extending PV widgets. `sdx.pvSelect.fill(...)`
is the one general-purpose, reusable piece (any custom dropdown, not just PV) — see
patterns.md.

## data-attributes-lib-v1 (loadIndex 10) — declarative attrs applied to EVERY widget
Registers a `Widget.onPreload` step (not a package with public methods) that, on every widget,
finds elements with `widget-text` / `widget-visible` / `widget-background-image` /
`widget-link` / `widget-link-new-tab` attributes and applies them by reading the referenced
**CSS config var** (i.e. `widget.getConfig(value)`) — this is how a config option value gets
projected straight into markup with zero JS, see data-attributes-test-widget-v1.html. Distinct
from the very similarly-named `template-*` attributes (templates-lib), which pull from a
**data object**, not a config option — don't mix them up.

**This is the complete list — confirmed by reading the full source, not inferred from
examples.** `sdx.Widget.onPreload` in this file registers five internal functions:
`_applyText`, `_applyVisible`, `_applyBackgroundImage`, `_applyLink`, `_applyLinkNewTab`.

`widget-link` and `widget-link-new-tab` are **confirmed real** (re-verified 2026-08-27, after
being added to the library) — `_applyLink` sets `href`, `_applyLinkNewTab` sets
`target="_blank"` when the referenced option is `'true'`. This was previously flagged in this
skill as non-existent, based on an external example widget that used these attributes before
the library actually implemented them — don't assume that history means they're still missing;
this entry reflects the current, re-verified state.

**`_applyLinkNewTab` does NOT set `rel="noopener"` alongside `target="_blank"` — and it doesn't
need to.** This was initially (incorrectly) flagged in this skill as a real security gap. Per
MDN (developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel/noopener), setting
`target="_blank"` on `<a>`/`<area>`/`<form>` elements already implies the same
`window.opener`-blocking behavior as an explicit `rel="noopener"`, in every currently-relevant
browser — this has been standard behavior for several years specifically to close this exact
footgun by default. `_applyLinkNewTab` is correct and complete as written; no library fix and
no widget-level workaround are needed. button-widget-v1's rev4 briefly patched `rel="noopener"`
back in via JS based on the incorrect assumption; rev5 removed that patch once corrected — see
its CHANGELOG for the full story if the reasoning here needs re-deriving later.

**General lesson, not just about this one attribute pair**: this library has now been checked
twice in one project and given two different, contradictory answers — first "these attributes
don't exist," then "they do, here's what they actually do." Both were correct **at the time
they were checked**. A shared library can change between sessions; re-verify from source again
rather than trusting a previous "confirmed" note indefinitely, especially after being told
something was updated. Separately: **verify claims against authoritative sources (MDN, official
docs) even when they concern general web-platform behavior, not just SDX specifics** — a
plausible-sounding security concern isn't automatically a correct one, and current browser
defaults can make yesterday's best practice unnecessary today.

## dlApi.contents / dlApi.users (raw backend API — see content-api.md for the full method tables)
`dlApi.contents.getQueryDimensions()` is a real introspection call; the confirmed queryable
dimensions are `contentHidden`/`containerId`/`contentId`/`spaceId`/`created` (all "equal") and
`contentName`/`spaceName`/`contentLogicalType`/`containerPath` (all "like", SQL-style `%`
wildcard). No `mimeType` or owner dimension exists — see content-api.md for what this means for
filtering by file type and resolving owners. `dlApi.users` methods (`get`, `getMany`, `find`,
`getByDisplayName`/`getByEmail`/`getByLogin`, `getQueryDimensions`) are now confirmed via the
official SDX API docs — see content-api.md's "users service" table for signatures. Widgets
should reach for `api.getUsersByIds()` (api-lib-v1) rather than calling `dlApi.users.*` directly
for the common "resolve a batch of user ids" case; call `dlApi.users.*` directly only for a need
that wrapper doesn't cover.
