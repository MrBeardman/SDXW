# dropdown-visibility-widget-v1 — revision history

The file at `Widgets\minimal\dropdown-visibility-widget-v1.html` always holds the latest
revision.

## rev1-initial.html — current

Third of four brainstormed dropdown-function variants. Directly reuses the ALREADY-confirmed
`postMessage({type: 'widget-set-row-visibility', show, hide}, '*')` pattern (overview-widget-v1
/ architecture.md), rather than inventing a new mechanism - the whole reason this option was
worth building first among the less-precedented three.

- Options are `"Label|rowId1+rowId2"` pairs - `+` joins multiple row ids under one option,
  since a single "view" often spans more than one row.
- **Mutual exclusivity is computed, not authored**: `hide` for a given option is every row id
  mentioned by ANY option minus that option's own `show` list. This makes the widget behave as
  a genuine view switcher (pick one, others hide) without needing the author to type out hide
  lists by hand - a deliberate design choice, not the only valid one (an author wanting
  overlapping/non-exclusive views would need a different widget).
- Applies the default selection on load, not just on the first manual change - so the page
  starts in the correct state (only the default view's rows visible) rather than showing
  everything until the first interaction.
- **Genuinely unconfirmed**: the exact shape/source of a valid "row id" - architecture.md notes
  the message TYPE and payload shape as confirmed (overview-widget-v1 sends it), but not
  documented anywhere is how a row gets ITS id in the first place, or whether it's the same
  Custom ID mechanism used for widget slots (Advanced tab) applied at the row level instead.
  Needs confirming before this is used for real, not just for Ben's comparison.

## Not yet tested live in SDX.
