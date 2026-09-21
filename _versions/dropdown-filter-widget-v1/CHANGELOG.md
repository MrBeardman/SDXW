# dropdown-filter-widget-v1 — revision history

The file at `Widgets\minimal\dropdown-filter-widget-v1.html` always holds the latest revision.

## rev1-initial.html — current

Fourth of four brainstormed dropdown-function variants - the one originally flagged as "bigger
scope, needs a small library." Built as a standalone foundation instead: `postMessage` with a
generic `dropdown-selection` message type and an author-set `broadcast-key`, so ANY future
widget can listen and react however it wants (filter a list, recolor something, anything),
rather than only the fixed show/hide behavior dropdown-visibility-widget-v1 implements.

- **No shared library created** - mirrors the module-level shared-state pattern
  `pv-performance-lib` uses (`getSelection`/`setSelection`/`onSelectionChange`, see
  architecture.md) conceptually, but via `postMessage` instead of a real library, so this
  widget stays self-contained per SKILL.md rule 8 (library work needs its own explicit
  go-ahead, separate from widget work).
- Broadcasts the starting selection on load, not just on change - a listener widget that
  finishes loading after this one still learns the initial state rather than only future
  clicks.
- **Honest limitation, not hidden**: this is genuinely the least immediately testable of the
  four variants on its own - nothing in `minimal/` currently listens for `dropdown-selection`
  messages, so selecting an option here has no visible effect yet beyond the message itself
  (visible in the browser console). A matching "listener" widget would be natural follow-up
  work if this is the direction Ben prefers, not something built speculatively here.

## Not yet tested live in SDX.
