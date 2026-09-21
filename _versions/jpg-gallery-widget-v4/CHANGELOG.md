# jpg-gallery-widget — revision history

The file at `Widgets\jpg-gallery-widget-v4.html` (one level up) always holds the **latest**
revision, ready for you to upload into SDX yourself. This folder keeps a copy of every revision
along the way so we can track what changed and roll back if needed.

Note: rev1-rev3 were built and tested under hrid `jpg-gallery-widget-v1`. rev4 renamed the hrid
to `jpg-gallery-widget-v3` to avoid overwriting that original v1 widget already live in SDX.
rev7 (edited in place, see below) then bumped the hrid again to `jpg-gallery-widget-v4` - not
because of a collision this time, but because the hrid/display had gone stale across several
revisions in a row, silently causing every re-upload to overwrite the last one in SDX. Going
forward, the hrid/display gets bumped for every revision meant to be (re-)uploaded, independent
of whether that revision also gets a new `revN-...html` file here - see SKILL.md rule 9. This
folder and the canonical file are renamed to match each time the hrid changes.

## rev1-initial.html — first working version

- Grid of JPG thumbnails, queried via two `contentName` LIKE queries (`%.jpg`, `%.jpeg`) merged
  and double-checked against `mimeType`.
- Client-side name search box (separate pass from the query, no extra network calls).
- Owner resolution via `dlApi.users` (with a per-id fallback), download link, config options
  for accent color, logicalType, limit, show-owner toggle.
- Confirmed working live on the first try — screenshot showed thumbnails, owner, and download
  all rendering correctly.

## rev2-filters-and-polish.html

Built on user feedback after rev1 was live:

- **Fixed**: file name was rendering but invisible (white text inherited from SDX's dark theme,
  on a white card) — now has an explicit dark color. See patterns.md "Gotcha: always set an
  explicit text color".
- **Fixed**: owner lookup now properly batches via `dlApi.users.getMany()`, unwrapping its
  `{type, value}` typed-collection response instead of falling through to N individual calls.
- **Added**: dev-mode-gated `console.log` of both the raw query result and the fully prepared
  card data, so the full field set is always inspectable without re-deriving it.
- **Added**: file size + upload date (each independently toggleable), shown as a chip over the
  thumbnail.
- **Added**: an owner filter and a 5-way sort (name/newest/oldest/largest/smallest), both pure
  client-side over already-fetched data, built with the existing `pv-select` component.
- **Added**: shimmer skeleton loading state, staggered card fade-in, image zoom + card-lift +
  cursor-tilt on hover, glow on the download link — hand-rolled CSS/vanilla JS (no external
  component libraries; the runtime has no build step to install one into).

## rev3-sizing-cleanup-and-empty-state.html

User feedback after rev2 was live (confirmed working, but visually rough at the edges):

- **Fixed**: search input looked cut off against its pill-shaped border. Padding was too tight
  for the curve — same class of fix as `pv-select-button`'s asymmetric pill padding, just never
  applied here. Also added `box-sizing: border-box` and `appearance: none` defensively.
- **Fixed**: cards in the same row weren't the same height. CSS Grid stretches the immediate
  grid child (the cloned template wrapper) by default, but the visible `.jpg-card` inside that
  wrapper never inherited that stretched height — added `height: 100%` on `.jpg-card`, plus a
  2-line `-webkit-line-clamp` on the file name so a long name can't make one card taller than
  its row-mates in the first place (same pattern `.description` already uses in
  overview-widget-v1). Also pinned the download link to the bottom of the card via
  `margin-top: auto` on a flex `.jpg-body`, so it lines up across cards regardless of name/owner
  line count.
- **Fixed**: a lone remaining card (e.g. after filtering) stretched to fill the whole row width.
  `grid-template-columns` used `auto-fit`, which collapses empty tracks and redistributes their
  width to whatever's left; switched to `auto-fill`, which reserves the tracks instead, so
  remaining cards keep their normal width.
- **Removed**: the `logicalType` config option and all query-time logicalType narrowing — the
  widget now purely matches file name/type, nothing else.
- **Fixed**: `_loadJpgFiles` was hand-rolling `dlApi.contents.find(query).then(r => r.result)`,
  duplicating exactly what `api.findLatest()` already does — this violated the skill's own
  documented rule ("always go through sdx.api wrappers, never call dlApi directly"). Replaced
  with a direct call to `api.findLatest()`, which also removed the now-unnecessary
  `_queryByNamePattern` wrapper function entirely.
- **Removed**: the `throw new Error(...noFilesFound...)` on an empty result. An empty gallery
  isn't an error - now `_applyFilters` shows a friendly inline message ("no files found" for a
  genuinely empty gallery, "no matches" once a filter narrows a non-empty one down to nothing)
  instead of triggering the widget's error state.
- **Cleaned up**: CSS now defines local custom properties (`--jpg-gap`, `--jpg-pill-padding`,
  `--jpg-thumb-height`, `--jpg-skeleton-height`) instead of repeating literals, and every
  singular (non-repeated) element is styled by its existing `id` directly instead of carrying a
  parallel, redundant class - only the truly repeated per-card elements (`.jpg-card`,
  `.jpg-thumb`, `.jpg-name`, etc.) still use classes, since they can't be unique ids.

## rev4-hrid-v3-and-hover-fix.html

User feedback after rev3 was live, plus real API documentation supplied by the user:

- **Changed**: hrid `jpg-gallery-widget-v1` → `jpg-gallery-widget-v3`, display name to match, so
  re-uploading creates a distinct widget instead of overwriting the one already live in SDX.
  This folder and the canonical file were renamed to match.
- **Fixed**: the hover zoom on the thumbnail wasn't visibly animating. Root cause: the CSS
  transitioned `background-size` from the keyword `cover` to a percentage (`112%`) - browsers
  generally can't smoothly interpolate a keyword into a percentage, so the "animation" either
  snapped or did nothing. Replaced with `transform: scale(1.08)` on `.jpg-thumb`, which is
  always smoothly animatable (GPU-accelerated) and gets cleanly clipped by `.jpg-card`'s
  existing `overflow: hidden`. Captured as a general pattern in `patterns.md`.
- **Documentation**: the user supplied the real SDX API docs and client source
  (`api.documentation.html`, `api.js`). Everything in `content-api.md` and `libraries.md` that
  was previously "confirmed via live introspection" (i.e. reverse-engineered by calling
  `getQueryDimensions()` and inspecting results) has been upgraded to "confirmed via official
  documentation" where the docs covered it, and several previously-open questions are now
  resolved outright - see content-api.md for the full detail. Highlights:
  - `contents.stream(id, preview, locale)` / `streamByName(fullname, preview, locale)` -
    `preview` really is a real, documented parameter, defaulting to `false` at the API level
    (not `true` as api-lib-v1's own `streamContentById` wrapper JSDoc claims - that wrapper has
    a real bug, see content-api.md).
  - The mystery third argument in `dlApi.contents.streamAsGetURL(id, preview, undefined,
    cacheBust)` is `locale` - the "AsGetURL" variant of any method takes that method's own
    params plus a trailing cachebust boolean, not a bespoke signature.
  - `contents.getMany`/`users.getMany` both officially return a Java `List` - combined with the
    already-confirmed `{type, value}` wrapping on `users.getMany`, this generalizes to: any
    dlApi method returning a Java Collection (List/Set) is wrapped this way over the wire, not
    just `users.getMany` specifically.
  - `users.getByDisplayName(String)`, `users.getByEmail(String)`, `users.getByLogin(String)`
    exist as exact-match single-user lookups, an alternative to `find`/`get`/`getMany` worth
    knowing about for future widgets.

## rev5-inset-solid-button-chip-fix-inline-filters.html

User feedback after rev4 was live (screenshots showed real rendering bugs, not just polish):

- **Fixed**: the search focus glow and card hover shadow were both visibly clipped, because the
  widget's host container in SDX gives it almost no surrounding space. Added `padding: 10px` to
  `#widget-body` itself so effects that extend past an element's own box have room to render.
- **Fixed**: the meta chip (size · date, top-right of the thumbnail) drifted outside the image
  on hover. Root cause: it was a *child* of `.jpg-thumb`, so `.jpg-thumb`'s own hover
  `transform: scale(1.08)` (added in rev4) dragged the chip along with it. Moved the chip to be
  a sibling of `.jpg-thumb` instead, anchored via `position: relative` on `.jpg-card` (which
  never transforms), so it now stays pinned to the card regardless of the image zooming
  underneath.
- **Changed**: the download link is now a solid-filled pill button (`background:
  var(--widget-accentColor)`, white text) instead of colored text with a glow-on-hover — the
  glow itself is gone, replaced by a simple `filter: brightness(1.08)` + slight lift on hover.
- **Changed**: the owner/sort filter pills now live in the *same row* as the search box and
  Filters toggle (via `display: contents` on `#jpg-filters-panel`, so its children become direct
  flex items of `#jpg-toolbar`) instead of a separate row underneath, with a staggered
  left-to-right fade-in (`@keyframes jpg-filter-in`) each time the panel opens. The toolbar's
  existing `flex-wrap` still lets them drop to a new line if there isn't room.
- **Added**: two new config options, `showOwnerFilter` and `showSortFilter` (both default
  `true`), matching the existing `showOwner`/`showSize`/`showDate` naming pattern - lets a given
  placement of the widget turn either filter off. If both are off, the whole Filters button is
  hidden rather than left open onto an empty panel.

## rev6-optimized-clean.html — superseded, kept for reference only

**This revision briefly depended on new functions added to the shared `api-lib-v1.html` and
`number-lib-v1.html` libraries. Those library changes were reverted almost immediately (see
rev7) once it was clarified that this work package is widget-only for now - the libraries were
never meant to be touched yet.** rev6 is kept here purely as a historical record and as a
reference for what a "use the libraries to the max" pass could look like later; **do not copy
its `api.getPreviewURL`/`api.getDownloadURL`/`api.getUsersByIds`/`number.formatFileSize` calls
into anything live** - those functions don't exist in the real, currently-deployed libraries.
See `_versions/_shared-libraries/README.md` for the drafted (unapplied) library additions this
revision was built against.

Everything else about this revision's *widget-only* cleanup was correct and carried forward
into rev7 - see rev7's entry below for the actual current set of changes.

## rev7-self-contained-cleanup-no-library-changes.html — current

Correction after rev6: the widget goes back to being fully self-contained (only calling
functions that already exist in the real, unmodified `api-lib-v1.html`/`number-lib-v1.html`),
while keeping every part of rev6's cleanup that didn't require a library change:

- **Reverted**: `_loadOwnerNames` back to its own direct `dlApi.users.getMany`/`.get(id)` calls
  (with the same batch + per-id-fallback logic as before) instead of a shared `api.getUsersByIds`
  that no longer exists.
- **Reverted**: `_prepareData` back to building preview/download links via
  `dlApi.contents.streamAsGetURL(content.id, true/false)` directly instead of
  `api.getPreviewURL`/`api.getDownloadURL`.
- **Reverted**: file-size formatting back to a local `_formatFileSize` function instead of a
  shared `number.formatFileSize`.
- **Kept from rev6** (none of these touch the shared libraries, so they stayed):
  - Skeleton loading state removed entirely (see below) in favor of the plain-text loading
    convention every other widget in this codebase uses.
  - More CSS custom properties (`--jpg-pill`, `--jpg-border`, `--jpg-muted`, `--jpg-muted-light`)
    replacing repeated literal values for the pill radius, border, and muted-text colors used
    across the toolbar, chip, download button and empty-state message.
  - Removed a redundant local `#jpg-filters-panel .pv-select-button` CSS override - the
    `.pv-select-small` class already applied from pv-select-lib-v1 sizes it correctly, matching
    every other widget that uses this component.
- **Removed**: the shimmer skeleton loading state (`#jpg-skeleton-grid`, `.jpg-skeleton-card`,
  `@keyframes jpg-shimmer`) - it wasn't rendering correctly for the user for undetermined
  reasons. Replaced with the plain `<div widget-state="loading">Loading…</div>` + `widget.
  findElement('[widget-state="loading"]').textContent = widget.resolveL10N('loading', {})`
  pattern already used by every other widget in this codebase (see patterns.md's "Loading /
  active / updating / error state block").

**Edited again, same rev7 file (deliberately no new rev-file for this round):**

- **Changed**: hrid `jpg-gallery-widget-v3` → `jpg-gallery-widget-v4`, display name to match.
  This folder and the canonical file were renamed to match, same as the rev4 hrid bump. See the
  top-of-file note and SKILL.md rule 9 for why this now happens more proactively.
- **Removed**: the "Filters" toggle button and the `#jpg-filters-panel` wrapper div entirely.
  The owner/sort dropdowns are now always-visible plain pills directly in the toolbar row,
  each still individually hidden via `showOwnerFilter`/`showSortFilter` if its config option is
  off - there's no longer a show/hide interaction or an "empty panel" case to special-case.
  Also dropped the now-unused `filters` l10n key.
- **Fixed**: the card hover shadow was still clipped at the *bottom* of the widget even with the
  rev5 padding fix. `#widget-body`'s padding is now asymmetric - `10px 10px 28px` - since a
  `0 18px 34px -14px` box-shadow reaches roughly `18 + 34 - 14 =~ 38px` below the card, well past
  the uniform 10px that was enough for the smaller top/side effects (the 2px focus glow, the
  6px chip inset).

## Known open items (see also `_claude-skill/references/content-api.md`)

- The exact SQL LIKE wildcard/case-sensitivity syntax for `contentName` is confirmed to exist
  (via official docs) but still not independently stress-tested against real data (e.g. an
  uppercase `.JPG` extension).
- The precise behavioral difference between `preview=true` and `preview=false` (thumbnailing?
  watermarking? `Content-Disposition` headers?) is still not spelled out anywhere in the docs
  pulled so far - only that `preview` is a real, documented boolean parameter.
- Whether/when to actually apply the library additions drafted for rev6 (now parked in
  `_versions/_shared-libraries/*.proposed-extensions.html`) is a separate, not-yet-started work
  package.
- Why the shimmer skeleton loading state didn't render correctly was never root-caused (rev7
  just removes it in favor of the plain-text convention) - if a future widget wants a skeleton
  loading state, this is worth investigating rather than assuming the same approach will work.
