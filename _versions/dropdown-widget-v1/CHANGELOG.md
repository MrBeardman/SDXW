# dropdown-widget-v1 — revision history

The file at `Widgets\minimal\dropdown-widget-v1.html` always holds the latest revision.

## rev1-initial.html — current

First build, direction chosen from four brainstormed options (navigate-on-select, scroll-to-
anchor, postMessage row-visibility, cross-widget shared-selection filter) - navigate-on-select
picked as the most directly precedented, reusing the same "click to go somewhere" idea as
button-widget-v1's `link`/`link-new-tab`, just for a menu of destinations instead of one link.

- **Genuinely needs JS, unlike the other four `minimal/` widgets so far**: `type: "select"` in
  widget-config is for fixed, author-defined choices - it can't represent end-user-typed,
  variable options. Options are authored as comma-separated `"Label|URL"` pairs in a single
  `text` option and parsed into real `<option>` elements in `onLoad`. This is the second
  sanctioned JS case in this family (the first was heading-widget-v1's tag swap) - genuinely
  nothing CSS/data-attributes-lib can do here, not a shortcut taken for convenience.
- **`window.open()` needs an explicit `'noopener'`, unlike `<a target="_blank">`.** The MDN
  behavior behind button-widget-v1 rev5's correction (`target="_blank"` implicitly blocking
  `window.opener`) is specific to the HTML attribute on `<a>`/`<area>`/`<form>` elements - it
  does not extend to the `window.open()` JS API, which needs `'noopener'` passed explicitly as
  its third argument. Got this right on the first pass by treating the earlier correction as a
  reason to double-check the adjacent case, not as "already handled."
- **No alignment/color/spacing options** - applying the newly-documented finding
  (architecture.md, "Slot-level styling") that the SDX layout builder's own Style tab already
  covers margin/padding/width/height/position/alignment for every widget slot. Kept only
  `padding: 8px 12px; cursor: pointer;` on `#dropdown` itself - the control's own inner
  spacing, not the slot's, same category as button-widget-v1's own internal button padding.
- Default-option matching: an empty or non-matching `default-option` value falls back to the
  browser's own native behavior (first `<option>` selected) - no extra code needed for that
  case.
- Known limitation, not engineered around: a URL containing a literal comma would break the
  `.split(',')` parsing. Acceptable for a simple author-typed field; flag if it becomes a real
  problem.

## Open questions

- Whether a leading blank/prompt option ("-- Choose a page --") would be better UX than
  starting on a real default - not built, since the brief specifically described a "default
  option," not a blank prompt state. Worth asking about if this gets real use.
- Not yet tested live in SDX.
