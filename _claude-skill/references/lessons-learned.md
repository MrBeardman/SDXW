# Lessons Learned — JPG Gallery Widget (2026-08-26)

Reliability notes from building `jpg-gallery-widget-v1.html`, the first widget built from this
skill rather than from scratch. Kept as a reference for anyone extending the SDX widget system
next, human or Claude.

## Summary

The widget-authoring layer (the `<widget>` tag format, lifecycle, libraries) was fully
learnable from static inspection of the existing files — no guessing required. The `dlApi`
layer underneath it was not: several gaps could only be closed with live API introspection, and
one gap was closed with an inference that happened to be correct but was not fully confirmed.

## What was fully derivable from static inspection

The following required no guessing and no live testing — reading the existing 27 files was
sufficient:

- The `<widget>` tag format and its five-block structure (config / l10n / style / tags / script)
- The `sdx.Widget` lifecycle (`onLoad`, `onUpdate`, `ready`)
- The CSS-variable config channel (`--widget-<key>` → `widget.getConfig(key)`)
- The `Template` class for rendering data-driven lists
- The l10n mechanism (`widget-l10n` block + `resolveL10N`)

Every widget shares one author's conventions closely enough that four or five examples were
enough to generalize the rest. No clarifying question in this project touched this layer.

## What required live API introspection

api-lib-v1 wraps only the `dlApi` calls that some existing widget happened to need already — it
documents **usage**, not the underlying **contract**. Three gaps followed directly from this:

1. No example filters content by file type or mime type.
2. No example resolves a `ownedBy` / `createdBy` UUID to a display name.
3. No example distinguishes a real file download from a preview stream.

None of these could be resolved by reading the codebase. What resolved them was a live browser
session and `dlApi.contents.getQueryDimensions()` — a real, callable, self-describing
introspection method that returns the exhaustive list of queryable dimensions. **This method is
not referenced or used anywhere in the 27 existing files.** Its existence was previously
undocumented in the codebase itself; it only became known once a browser was inspected live.

## What was resolved with a fallback, not a confirmation

`dlApi.users` was confirmed to exist as a sibling package to `dlApi.contents`, but its method
names (`get` / `getMany`) were **inferred** from the parallel shape of the `contents` API, not
observed directly — the browser session was lost before `users.getQueryDimensions()` returned.

The widget code reflects this uncertainty honestly: a try/catch plus a per-id fallback, rather
than a single asserted method name. This turned out correct on the first run, but its confidence
level was, and should have been, lower than anything pattern-matched from static examples — and
that distinction was preserved in code comments rather than left implicit.

## Structural properties of the codebase that increase guessing risk

- **Two unrelated architectures share one flat folder** with no naming or directory convention
  distinguishing them: the current `<widget>` tag format, and an older standalone-document /
  `SspaLib` prototype (`news-template.html`, `performance-report-widget-all-projects.html`).
  Nothing marks the latter as superseded except noticing the divergence on inspection.
- **Personal scratch/test widgets** (`test-bs-v1.html`, `test-jm-v1.html`,
  `jm-test-widget-v1.html`) sit alongside canonical examples with no signal that they're
  lower-quality references — one contains a duplicate `hrid` copy-pasted from
  welcome-widget-v1.
- **At least one config option type (`select`) is unconfirmed** — it appears only as a
  commented-out example in one file, with no way to tell from the code alone whether it's
  planned, abandoned, or half-built.
- **The real `dlApi` surface is more self-documenting than the widget layer exposes.**
  `getQueryDimensions()` is the clearest case: a genuinely useful introspection tool that exists
  only at runtime and has to be rediscovered through a live session each time, since no widget
  author has ever written its results down.

## Recommendation

The widget-authoring layer is solid enough to onboard from examples alone. The `dlApi` layer
underneath it is not — its actual contract should be documented once, ideally by running the
introspection calls for `contents`, `users`, and any other package in active use and recording
the results, rather than left to be re-derived through guesswork or live debugging each time a
new widget needs something api-lib hasn't wrapped yet.
