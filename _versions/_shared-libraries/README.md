# Shared library files — not currently modified

This experiment/work package is scoped to the widget only — the shared libraries
(`api-lib-v1.html`, `number-lib-v1.html` etc, one level up in `Widgets/`) are **not**
touched, and jpg-gallery-widget-v3 is written to work entirely with what those libraries
already expose today.

## Files in this folder

- `api-lib-v1.original.html` / `number-lib-v1.original.html` — a snapshot of each library
  exactly as it stood before any of this work started. The live files were never actually
  left in a different state for long (see below), but these are kept as an unambiguous
  reference point regardless.

- `api-lib-v1.proposed-extensions.html` / `number-lib-v1.proposed-extensions.html` — a
  **draft**, not currently applied anywhere, of what it could look like to properly eliminate
  the widget's remaining direct `dlApi.users.*` / `dlApi.contents.streamAsGetURL` calls and its
  local file-size formatter by promoting them into these shared libraries instead:
  - `api.getPreviewURL(id)` / `api.getDownloadURL(id)` — thin wrappers around
    `dlApi.contents.streamAsGetURL`, so no widget needs to reach for `dlApi` directly just to
    build a preview/download link.
  - `api.getUsersByIds(ids)` — batched user lookup (`dlApi.users.getMany`, unwrapping its
    `{type, value}` response) with a per-id fallback if the batch call fails, returning a plain
    `id -> User` map.
  - `number.formatFileSize(bytes)` — a B/KB/MB/GB formatter alongside `formatCurrency`/
    `formatTemperature`, reusing `formatNumber` internally for the same de-DE formatting.

  These were briefly live on disk during development of jpg-gallery-widget-v3, then reverted
  once it was clarified that library changes are a separate work package. Nothing currently
  live depends on them. When that work package is ready, these two files show a concrete,
  already-drafted, JSDoc'd, style-matched starting point — diff them against the real
  `api-lib-v1.html`/`number-lib-v1.html` to see exactly what would be added (each is purely
  additive, no existing method touched).
