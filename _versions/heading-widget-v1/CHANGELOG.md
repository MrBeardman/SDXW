# heading-widget-v1 — revision history

The file at `Widgets\minimal\heading-widget-v1.html` always holds the **latest** revision,
ready to (re-)upload/test in SDX. This folder keeps a copy of every revision along the way.

## rev1-initial.html — current

**Confirmed working live in SDX, 2026-09-17** (user tested the upload): the `onLoad` tag swap,
the `@property` + `not style()` optional font size, bold-default-on, and `select` level all
behave as intended. The three "to verify" items below are therefore closed.

Third member of the minimal widget library, 2026-09-17. Built from label-widget-v1 rev3; the
three decisions below were asked and answered before implementation.

- **7 options**, in the order given: level, text, font size, bold, italic, color, alignment.
- **Heading level via JS (user's choice over six pre-rendered headings toggled by CSS).** The tag
  name is the one option CSS cannot express, so this is the first widget in `minimal/` with an
  `onLoad` handler: 5 lines that create the chosen `h1`-`h6`, copy the placeholder's already
  filled `innerHTML` across, and `replaceWith` it. Works because sdx-lib runs the preload
  callbacks (where data-attributes-lib applies `widget-text`) *before* `onLoad`. The placeholder
  is an `h2` (the default level) so the widget is still a heading if the swap never runs.
- **Empty font size = the page's own size for that level (user's choice over always-explicit).**
  Done in CSS, not JS: `--heading-font-size` is registered via `@property` with
  `syntax: "<length>"; initial-value: 0px`, so an empty option (which makes the `calc()` invalid)
  computes to `0px` instead of breaking the declaration; then
  `@container not style(--heading-font-size: 0px)` is the only place `font-size` is set. With
  the option empty no `font-size` is declared at all, so the h-tag keeps whatever the SDX page
  (or the UA) gives that level. **Most speculative part of this widget - verify live first.**
  If it misbehaves, the 1-line JS fallback is `if (size) heading.style.fontSize = size + 'px'`
  inside the existing `onLoad`.
- **Bold defaults to `true`** (headings are bold by default). Base rule is `font-weight: normal`
  and the `style(--widget-font-bold: true)` query sets bold, so the toggle is a real two-way
  switch - turning it off gives a non-bold heading.
- Alignment, italic, color, `margin: 0`, localized labels: identical to label-widget-v1 rev3.

**To verify on first upload** (in addition to the still-open `select` rendering and `@container
style()` items tracked in label-widget-v1's CHANGELOG):
1. `@property` inside a widget's `<style id="widget-styles">` is honoured by the host (it sits
   outside `@scope`, at the top level of the style element).
2. An empty text option leaves the heading at the page's natural size for that level; a value
   like `40` overrides it.
3. Changing the level in the editor re-renders with the new tag (depends on how the editor
   re-runs `onLoad` on config change - if it only re-applies CSS vars, a level change may need a
   page refresh to show).

## rev2-boolean-defaults-as-strings.html — current

Same comparison pass as button-widget-v1 rev6, against Ben's freshly-downloaded
`example-heading-simple.html` (`SDE-Download-2026-09-21 17-37-02/`).

- **Adopted**: `font-bold`/`font-italic` boolean defaults changed from JSON `true`/`false` to
  the strings `"true"`/`"false"`. Confirmed via label-widget-v1's own history (see its
  CHANGELOG) - Ben's saved `example-label-simple.html` from an earlier sync had these as JSON
  booleans; today's copy has them as quoted strings, everything else nearly unchanged. Most
  plausibly a fix Ben made after hitting it in live testing, not a style preference.
- **NOT adopted**: Ben's alignment mechanism (flex + `justify-content`, values `flex-start`/
  `flex-end`) - this widget already uses the better-justified `text-align` + `start`/`end`
  approach (see label-widget-v1's CHANGELOG for the multi-line-text reasoning). No change
  needed here since this widget was already consistent, unlike button-widget-v1 rev1-5, which
  had drifted to `left`/`right` values.
- **NOT adopted**: Ben's `font-size` label ("Font Size", no hint) over this widget's own
  ("Font Size (px, empty = default for level)") - ours is clearer about the empty-means-
  inherit behavior specific to this widget.
- **NOT adopted**: Ben's `document.createElement(widget.getConfig('level'))` with no fallback -
  this widget's own `|| 'h2'` fallback is a real defensive improvement Ben's copy lacks.
- Everything else (the `@property`/`not style()` font-size pattern, the `@container style()`
  bold/italic pattern, the onLoad tag-swap) matched almost verbatim between both copies,
  confirming the technique is stable and shared, not independently reinvented.
