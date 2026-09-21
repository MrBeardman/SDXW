---
name: sdx-widgets
description: Build, read, or edit SDX widgets and widget libraries (Secure Data eXchange's Elementor-style widget system for its SAAS). Use this whenever the user asks to create a new SDX widget or library, modify an existing one, debug a widget, or asks anything about the `sdx.*` JS namespace, the `<widget>` HTML tag format, widget-config/widget-l10n/widget-styles/widget-script blocks, template-* / widget-* data attributes, or files matching *-widget-v1.html / *-lib-v1.html. Push to use this any time SDX, "widget layouts", "widget library", or the SDX editor come up, even if the user doesn't say "widget system" explicitly.
---

# SDX Widget System

SDX (Secure Data eXchange) widgets are small, self-contained HTML fragments injected together
into ONE host document inside the SDX "Widget Layouts" editor (an Elementor-style drag & drop
builder). Widgets are placed into rows/columns on a page, and that page is shown inside an SDX
Space. There is also a **Globals** panel listing every **Library Widget** available to the page.

Read `references/architecture.md` before writing or editing any widget — it explains the
`<widget>` tag format, the Widget lifecycle, and the load order between libraries and widgets.
This is the single most important file; nearly every bug traces back to misunderstanding it.

## When to read which reference

- **references/architecture.md** — REQUIRED FIRST READ. The `<widget>` tag anatomy (config/l10n/
  styles/tags/script), the `sdx.Widget` class, widget-config.json fields, the loading/ready
  lifecycle, `widget-state` attributes, auto-update, and the `--widget-*` CSS variable config
  mechanism. Read this before touching any widget or library file.
- **references/libraries.md** — Catalog of every existing library (`sdx.api`, `sdx.l10n`,
  `sdx.date`, `sdx.number`, `sdx.xlsx`, `sdx.charts`, `sdx.templates`, data-attributes lib,
  pv-* libs, news lib) with their public methods. Read this to know what's already available
  before writing new helper code — most things you need already exist in one of these.
- **references/patterns.md** — Recurring code patterns across widgets: rendering a list with
  the Template class, building a custom dropdown (pv-select), building an ECharts chart,
  l10n usage, config option types, error/loading/updating states, "controller" widgets with
  no markup, and general code-style conventions (so new widgets blend in).
- **references/content-api.md** — The SDX Content object shape (as returned by content
  queries) and everything currently known/unknown about querying, filtering, downloading
  and displaying files (contents). Contains an explicit list of OPEN QUESTIONS that must be
  confirmed with the user before code that depends on them is written — do not guess these.
- **references/minimal-widgets.md** — The minimal widget library in `Widgets/minimal/`
  (button, label, ...): the bar they meet, an option-type → zero-JS-mechanism lookup table,
  a skeleton to copy, and the steps for adding the next one. Read this before starting any
  new simple/atomic widget — it's the fastest route to a widget that needs no custom JS.
- **references/lessons-learned.md** — Reliability retrospective from building the first widget
  off this skill: what was fully derivable from the static files vs. what required live API
  introspection vs. what was an unconfirmed-but-reasonable inference. Read this to calibrate
  how much to trust api-lib-v1 as a complete contract (it isn't) versus a usage log (it is).

## Non-negotiable ground rules for this codebase

0. **The real backend API is documented — use it before guessing.** The SDX instance itself
   serves authoritative docs and client source at `/api/dl/dlt/stream?id=api.documentation.html`
   and `/api/dl/dlt/stream?id=api.js` (e.g. `https://sdx-dev1.jenomics.net/api/dl/dlt/stream?
   id=api.documentation.html`). This is the real Java service contract behind `dlApi` — every
   service (`contents`, `users`, `spaces`, `groups`, ...), every method signature, every login/
   permission requirement. It's one enormous page; don't fetch it whole — use a browser session
   already logged into SDX, `fetch(location.href)` the raw HTML from within that page (bypasses
   the doc viewer's own JS collapsing the detail), then slice out just the `id="service-X-
   method-Y"` sections you need. See content-api.md for what's already been pulled from it.

1. **Never assume — ask.** This codebase has one author style and very deliberate small
   conventions (see patterns.md). If something isn't confirmed by an existing example, ask the
   user rather than inventing an API that doesn't exist in `sdx.*`.
2. **Match the existing style exactly.** ES5-flavored function expressions assigned to `var`/
   `const`, JSDoc block above every function, tab indentation, one `(function () { ... })();`
   IIFE per `<script>`, comments that explain *why* not *what*, minimal/no external deps beyond
   what's already wired up (echarts, xlsx/SheetJS via `window.parent.Application`). Do not
   introduce React, build steps, npm packages, or modern ES module imports.
3. **Minimal code — button-widget-v1 is the concrete bar to match.** Prefer the smallest
   number of lines that solves the problem, reusing library helpers instead of re-implementing
   them. After 5 revisions (see `_versions/button-widget-v1/CHANGELOG.md`), it went from 18
   options and ~65 lines of JS down to 9 options and a single line, `widget.ready();` - that's
   the target shape for every widget going forward, not a one-off. See patterns.md's "The
   minimal-widget bar" for the concrete checklist (data-attributes-lib first, then CSS-only
   techniques, then whether the state machine is even needed, then whether each option is
   actually needed) to work through before any custom JS gets added to a new widget. The
   widgets that meet this bar live in `Widgets/minimal/` (see references/minimal-widgets.md)
   under `category: "AI Examples/Basic"`; a new simple widget starts from one of those.
4. **Two file "eras" exist.** Files under `Widgets/*.html` using the `<widget>` tag are the
   CURRENT architecture — copy these. `news-template.html` and
   `performance-report-widget-all-projects.html` are a legacy standalone-document prototype
   (`<!doctype html>`, `window.parent.SspaLib`, `widget-schema` script) — do NOT copy their
   conventions into new work; they're kept only for historical reference unless the user says
   otherwise.
5. **Test/scratch widgets** (`test-bs-v1.html`, `test-jm-v1.html`, `jm-test-widget-v1.html`)
   are personal sandboxes by team members, not canonical examples — don't copy their sloppier
   bits (e.g. duplicate hrid in jm-test-widget-v1.html).
6. **Never touch the live SDX system.** All work happens on local files only. The person
   uploads/imports changes into SDX themselves — do not attempt to do this via a browser tool,
   even if one is connected, unless explicitly asked to. Read-only browser inspection (checking
   `window.Application.api`, `getComputedStyle`, console output) is fine and encouraged for
   answering "what does the real API/page actually do" questions; anything that would publish,
   save, or modify state in SDX itself is not.
7. **Version every revision, and bump the hrid/display for every revision meant to be
   (re-)uploaded to SDX.** The canonical widget file (e.g. `some-widget-v1.html`) always holds
   the latest revision, ready to be uploaded. Alongside it, keep a
   `_versions/<hrid>/rev<N>-<short-description>.html` copy of every revision plus a
   `_versions/<hrid>/CHANGELOG.md` summarizing what changed and why in each one - do this by
   default from the second revision onward, without being asked again. Separately, bump the
   widget's own `hrid` and `display` fields whenever a revision is meant to be (re-)uploaded or
   tested in SDX, not only when working around an actual collision with something already
   live - leaving them stale means every re-upload silently overwrites the last one in SDX, and
   this has already tripped a project up once. hrid bumps happen on a different, coarser cadence
   than rev-file creation: a hrid bump doesn't require a new rev-file (edits can land in the
   current rev-file in place, if that's what's asked for), and several rev-files can accumulate
   between hrid bumps. Increment the hrid simply and sequentially (`-v3` → `-v4` → `-v5`...),
   independent of the rev-file numbers, which track something finer-grained. Whenever the hrid
   changes, rename the `_versions/<old-hrid>/` folder and the canonical file to match the new
   hrid (move, don't restart) so the revision history stays intact under one name.
8. **Don't modify shared library files (`*-lib-v1.html`) unless explicitly asked to.** Even when
   a widget's own code would clearly be cleaner with a new shared wrapper function (see
   libraries.md's "add a wrapper to api-lib-v1" advice), default to keeping the widget
   self-contained against whatever the libraries currently expose, calling `dlApi` directly with
   a JSDoc note on why no wrapper exists yet, rather than editing a file every other widget
   depends on. Library work is a distinct, larger-blast-radius task from widget work and needs
   its own explicit go-ahead - don't infer permission for it from "use library functions more"
   or "don't duplicate code" style feedback about a single widget. If you do get that go-ahead,
   keep changes purely additive (new methods only, never touching an existing one's behavior),
   and back up the untouched original alongside the new version before editing. In the meantime,
   mark any such local, library-shaped code with the `@todo (library candidate for <lib-name>)`
   comment convention (see patterns.md) so it's easy to find and promote later.
