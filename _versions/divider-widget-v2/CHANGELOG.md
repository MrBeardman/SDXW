# divider-widget — revision history

The file at `Widgets\minimal\divider-widget-v2.html` always holds the **latest** revision,
ready to (re-)upload/test in SDX. This folder keeps a copy of every revision along the way.

## rev1-initial.html

Fourth member of the minimal widget library, 2026-09-17. The simplest one so far: no data
attributes at all, every option is a CSS custom property consumed directly by the CSS.

- **5 options**: thickness (px), color, width (%), alignment, spacing above/below (px).
- **Markup is a single `<hr>`** (the semantically right element) with the UA look reset
  (`border: none; margin: 0`) and the line drawn as `height` + `background-color`.
- **Alignment via flex, not text-align.** A line narrower than 100% is a block, so
  `text-align` on the parent can't place it. `display: flex; justify-content:
  var(--widget-align)` does - and `start`/`center`/`end` are valid `justify-content` keywords,
  so the select is byte-identical to the label/heading one. (Ben's example-label-simple used
  flex for its label with `flex-start`/`flex-end` values; the plain `start`/`end` keywords work
  for flex too and keep one select shape across the whole library.)
- **Width in percent** is the first `* 1%` calc in the library (everything before was `* 1px`).
- Spacing is one option applied as `padding-block` on `#widget-body`, not separate top/bottom.
- Not included, deliberately (minimal bar): a line style select (solid/dashed/dotted would need
  `border-top` instead of `background`), separate top/bottom spacing, vertical orientation.
  Any of these is a small additive change if asked for.

Nothing here is new or unverified - every mechanism was already confirmed live in the label and
heading widgets, except `justify-content: start/end`, which is standard CSS.

## rev2-style-orientation-split-spacing.html

The three "left out on purpose" items from rev1, all asked for after seeing rev1 - 8 options
now, still no data attributes and no JS. hrid unchanged (rev1 was not uploaded).

- **Line style** (`select`: solid/dashed/dotted). The line is no longer a `background-color`
  fill but a single **border side** - `border: 0 var(--widget-style) var(--widget-color)` zeroes
  every side, then only the drawn side gets `border-<side>-width: thickness`. The select value
  is a CSS `border-style` keyword, so it flows straight in. (Zeroing all widths first matters:
  `border: none` alone would leave the UA `medium` width on every side, and the style select
  would then paint a 3px box.)
- **Orientation** (`select`: horizontal/vertical). The value isn't a CSS keyword, so a
  `@container style(--widget-orientation: vertical)` block nested in the `#widget-body` rule
  swaps the drawn side (`border-top` → `border-left`), the axis the length applies to (`width`
  → `height`), and the axis the spacing applies to (`padding-block` → `padding-inline`, i.e.
  before/after = above/below when horizontal, left/right when vertical). First style query in
  the library that switches several rules at once, and the first one nested at the
  `#widget-body` level (its container is then the `<widget>` root, which holds the vars).
- **Spacing split** into `spacing-before` / `spacing-after` (was one `spacing`). Named
  before/after rather than top/bottom because they change axis with the orientation.
- `width` renamed to **`length`** for the same reason.
- **Vertical mode's one dependency - verify live**: the line's `height: <length>%` only
  resolves if `#widget-body` has a definite height, so the vertical block sets `height: 100%`
  on it. Whether the SDX column actually gives the widget root a height to fill (a flex row
  with stretched columns would; a content-sized column would not) is unknown from the files.
  If a vertical divider renders with zero height, the fix is to make the vertical length a px
  value (`min-height: calc(var(--widget-length) * 1px)`) - or to add a `min-height` fallback.

## rev3-v2-vertical-length-in-px.html — current

rev2 was uploaded and tested (2026-09-17): horizontal mode, the style select and the spacing
split all work. **Vertical mode rendered nothing**, exactly the dependency rev2 flagged.

- **hrid `divider-widget-v1` → `divider-widget-v2`**, display "Divider Widget V2". The user asked
  for the hrid/display to be bumped on every revision that gets re-uploaded (SKILL.md rule 7,
  now applied on the finer cadence). Canonical file and this folder renamed to match.
- **Vertical length is now px, not %** - `height: calc(var(--widget-length) * 1px)` in the
  vertical block, and the `height: 100%` on `#widget-body` is gone (it had nothing to resolve
  against). One `length` option still serves both modes; its label now says
  "(% horizontal, px vertical)". Default `100` reads as 100% wide or 100px tall.
- **Root cause, from the user's DevTools screenshots** (recorded in architecture.md as a
  gotcha): the host renders each widget as `div.sdx-widget-renderer__host` (this is where the
  `--widget-*` custom properties are actually set, as an inline style; it had `height: 120px`
  in the screenshot) → `div.sdx-widget-renderer__content` (`position: relative; contain:
  layout; container-type: inline-size`, **916×0**) → `body` (`min-height: 0; height: auto`,
  **916×0**) → `<widget>`. So the widget's ancestors are content-sized: a percentage height
  anywhere in a widget resolves to nothing, regardless of how tall the host slot is. Anything
  that needs a height must state it in px (or derive it from its own content).
