# label-widget-v1 — revision history

The file at `Widgets\minimal\label-widget-v1.html` always holds the **latest** revision,
ready to (re-)upload/test in SDX. This folder keeps a copy of every revision along the way.

## rev1-initial.html

Built directly to the minimal-widget bar set by button-widget-v1 (see patterns.md), 2026-09-17.

- **4 options**: text, font size, font color, alignment. The originally requested Bold and
  Italic toggles were dropped before the first build - a boolean option arrives in CSS as the
  literal string `true`, so mapping it to `font-weight`/`font-style` would have needed either
  JS class-toggling or `@container style()` queries (browser support unconfirmed for SDX
  users). The user chose to leave them out rather than add JS for them.
- **Zero custom JS** - `widget-text` fills the label (`innerHTML`, so simple inline HTML in
  the text renders), `text-align` on `#widget-body` handles alignment, and font size is a bare
  px number with the unit appended via `calc(var(--widget-font-size) * 1px)`.
- No loading/active/error markup, no l10n block - synchronous and config-only, nothing can
  fail.
- `select` for alignment uses the same localized-label shape as button-widget-v1; whether
  `select` renders in the editor is still tracked in button-widget-v1's CHANGELOG, not here.

## rev2-bold-italic-via-style-container-queries.html

Bold and Italic toggles added back in, still with zero JS - 6 options now, hrid unchanged
since rev1 had not been uploaded yet.

- **`@container style(--widget-bold: true)` / `style(--widget-italic: true)`** nested inside
  the `#label` rule. Every element is a style-query container by default (no `container-type`
  needed), and a style query evaluates against the element's nearest *ancestor* - here
  `#widget-body`, which inherits the host-written `--widget-*` custom properties. The boolean
  option's value is the bare token `true`/`false`, which is exactly what the query compares.
- **First use of style container queries in this codebase.** Support: Chromium 111+, Safari
  18+; Firefox shipped style queries for custom properties more recently - confirm on whatever
  browsers SDX users actually run. If the toggle has no effect in the SDX editor, first check
  in DevTools what form the host writes the variable in (`true` vs `"true"` vs something else)
  before assuming the query itself is the problem - the query must match the computed token
  exactly.

## rev3-merged-with-bens-example-label-simple.html — current

Ben shared `example-label-simple.html` (kept in this folder for reference). It independently
uses the exact same `@container style(--widget-...: true)` technique for bold/italic as rev2 -
good corroboration that this is the intended SDX way, from the platform's own author. Merged the
best of both:

**Adopted from Ben's example**
- Localized `{en, de}` labels on every option, not just the select.
- Option keys `font-bold` / `font-italic` (grouped with `font-size`) and `display` for the text
  (same key button-widget-v1 uses for its text) - consistent naming across the minimal widgets.
- A short header comment above `<widget>` describing the widget.

**Kept from rev2 (deliberately not adopted from Ben's)**
- `text-align` instead of `display: flex; justify-content`. Ben's comment gives rtl support as
  the reason for flex - `text-align: start/end` gets the same direction-awareness with no flex,
  and unlike flex + `inline-block` it aligns every line of a multi-line label, not just the
  block's position. Select values are therefore `start`/`center`/`end` now, not
  `left`/`right`/`flex-start`/`flex-end`.
- `margin: 0` on the `<p>` (Ben's leaves the default 1em paragraph margins).
- Inline `calc(var(--widget-font-size) * 1px)` instead of an intermediate `--label-font-size`
  custom property that's only read once.
- Defaults 16px / `#213B33` / "Label" over Ben's 12px / `#000` / "Display".
- `category: "General"` and icon `format-text` over `"Examples/Basic"` / `hexagon-outline`
  (whether a `/` category actually nests in the Globals panel is still unconfirmed).

**Dropped (copy-paste leftovers in Ben's example, from the button example it was cloned from)**
- `text-decoration: none; display: inline-block;` on a `<p>` that is not a link.
- Header comment claiming the label "can open a url" - there are no link options.
- A `/* border-radius is a bare number */` comment on the font-size line.

hrid unchanged - bump to `label-widget-v2` before uploading if rev2 was already uploaded.

**Same rev3, edited in place:** moved to `Widgets/minimal/` and `category` set to
`"AI Examples/Basic"` (see button-widget-v1's CHANGELOG entry of the same date for why).

**Confirmed working live in SDX, 2026-09-17** - `select` alignment, `@container style()`
bold/italic, localized labels all render and behave as intended.

## rev4-boolean-defaults-as-strings.html — current

Ben sent a fresh copy of his label example today (`SDE-Download-2026-09-21 17-37-02/
example-label-simple.html`) - compared against the version already merged in rev3.

- **Adopted**: `font-bold`/`font-italic` boolean defaults changed from JSON `false` to the
  string `"false"`. This is the confirming evidence for the whole boolean-default finding
  applied across button-widget-v1 (rev6) and heading-widget-v1 (rev2) too: the copy of this
  exact file saved at rev3 time had these as JSON booleans; today's copy has them as quoted
  strings, with everything else nearly identical - meaning Ben changed exactly this one thing,
  most plausibly after it mattered in his own live testing.
- **NOT adopted**: Ben's alignment mechanism (flex + `justify-content`) - unchanged from the
  rev3 decision to keep `text-align` (see that entry for the multi-line-label reasoning this
  widget's own comment already captures).
- Nothing else differed from the rev3 merge - font-size, colors, and option order all matched.

**Standing question resolved**: whether `category: "AI Examples/Basic"` nests in the Globals
panel is no longer open - see architecture.md, confirmed 2026-09-17: it does not nest, it's a
flat group literally named that.
