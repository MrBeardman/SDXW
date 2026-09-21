# Shared library changes

This folder holds a pre-edit backup of every shared library file we've modified, plus this
changelog describing exactly what was added. Both edits below are **purely additive** — no
existing method was changed, removed, or had its behavior altered — so they're safe for every
other widget that already depends on these libraries.

Live paths (the actual files SDX would load, one level up): `api-lib-v1.html`,
`number-lib-v1.html`. Backups here: `api-lib-v1.original.html`, `number-lib-v1.original.html`
(the exact content before this round of edits — restore these if anything regresses).

## api-lib-v1.html — 3 new methods

Triggered by jpg-gallery-widget-v3 rev6: the widget was calling `dlApi.contents.streamAsGetURL`
and `dlApi.users.getMany`/`get` directly, which the skill's own ground rules say not to do
("always go through sdx.api wrappers"). No wrapper existed for either need, so these were added
rather than left as ad hoc `dlApi` calls in the widget.

- **`api.getPreviewURL(id)`** — `dlApi.contents.streamAsGetURL(id, true)`, named for what it's
  for rather than requiring the caller to remember the `true`/`false` meaning of `preview`.
- **`api.getDownloadURL(id)`** — same, with `preview: false`.
- **`api.getUsersByIds(ids)`** — resolves multiple users by id in one batched
  `dlApi.users.getMany()` call, unwrapping its `{type, value}` typed-collection response, with a
  per-id `dlApi.users.get()` fallback if the batch call fails (so one bad id can't break the
  rest). Returns a plain `{id: User, ...}` map; deciding which User field to show is left to the
  caller, since that's a widget-specific choice, not something the library should assume.

## number-lib-v1.html — 1 new method

- **`number.formatFileSize(sizeBytes)`** — steps a byte count up through B/KB/MB/GB and formats
  the result through the library's own `formatNumber` (same de-DE formatting as every other
  number here). Identical logic to what was previously duplicated inline in
  jpg-gallery-widget-v3 (`_formatFileSize`); promoted here since any future widget dealing with
  file sizes would want the same thing.

## Why these two libraries and not others

Both are genuinely general-purpose (any widget touching file downloads or byte sizes benefits),
match the existing style and JSDoc conventions of their respective files exactly, and required
no changes to config, l10n, or any other part of the library. Nothing was added to
templates-lib-v1 or pv-select-lib-v1 in this round even though jpg-gallery-widget-v3 uses both —
its usage of those two didn't involve any duplicated logic worth promoting.
