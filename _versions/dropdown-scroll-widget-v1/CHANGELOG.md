# dropdown-scroll-widget-v1 — revision history

The file at `Widgets\minimal\dropdown-scroll-widget-v1.html` always holds the latest revision.

## rev1-initial.html — current

Second of four brainstormed dropdown-function variants, built alongside dropdown-visibility
and dropdown-filter so Ben can compare all four and pick a direction (dropdown-widget-v1,
navigate-on-select, was built and picked first - see its own CHANGELOG).

- Same comma-separated `"Label|value"` parsing as dropdown-widget-v1, with the value being an
  anchor id instead of a URL.
- Deliberately uses `document.getElementById`, not `widget.findElement` - the target is meant
  to be OUTSIDE this widget's own root, elsewhere on the page. architecture.md's warning
  against `document.querySelector` is about reaching elements WITHIN one's own widget instance
  (where id collisions across multiple instances are the risk); reaching another widget's slot
  on purpose is a different, intended case.
- `AnchorId` is documented (header comment, option label) as expected to match a Custom ID set
  on the target's slot via the Advanced tab (see architecture.md, "Slot-level styling") - ties
  this widget directly to a platform capability found in the same session, rather than
  inventing a separate id-assignment mechanism.
- Native `scrollIntoView({behavior: 'smooth', block: 'start'})`, no configurable offset for a
  sticky header - a real, common need for anchor navigation, deliberately left out of this
  first pass rather than guessed at unprompted.

## Not yet tested live in SDX.
