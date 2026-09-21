# button-widget-v1 — revision history

The file at `Widgets\minimal\button-widget-v1.html` always holds the **latest**
revision, ready to (re-)upload/test in SDX. This folder keeps a copy of every revision along
the way.

## rev1-initial.html

First build — testing how far the confirmed option types (`text`/`color`/`boolean`) can
stretch to cover the full breadth of what a genuinely rich button needs: content (text, MDI
icon, link, open-in-new-tab), color (text/background with hover pairs, border), typography
(size, weight), sizing (border width/radius, padding vertical/horizontal), and behavior
(alignment, hover effect, full width) - 18 options total, no `select` used anywhere.

Two things carried over directly from earlier findings rather than being rediscovered:
- Every px-based option is a bare number, unit appended via `calc(var(--widget-x) * 1px)` in
  CSS - the fix for the confirmed "`12 px` silently fails" bug (see patterns.md), applied
  consistently instead of just described.
- Alignment/hover-effect (naturally select-shaped fields) implemented as `text` + a defensive
  JS mapping function (`_resolveOptionClass`) instead of gambling on the unconfirmed `select`
  type - same visual result, zero risk.

**Immediately corrected after delivery** (not its own revision, just a same-day fix once the
real category taxonomy was confirmed via a live screenshot): `category` was `"Basic"`, which
doesn't exist anywhere in the real system - changed to `"General"`, the closest real match for
a foundational/atomic widget. See architecture.md for the full confirmed taxonomy.

## rev2-simplified-select-test-label-grouping.html

User feedback after testing rev1 live in the SDX editor:

- **Removed**: the loading/active/error state machine and l10n block entirely, and the
  separate `#btn-wrap` div - confirmed unnecessary for a widget that's purely synchronous and
  config-driven, with nothing to wait for and no realistic failure mode. `widget.onLoad`/
  `widget.ready()` are still required (framework-level, unrelated to whether visual state
  markup exists), but the visible state-machine `<div>`s are not. `#widget-body` now serves
  directly as both the flex/alignment container and the button's only wrapper - the TAGS block
  shrank from ~19 lines to 6. Generalized into patterns.md as an exception to the standard
  loading/active/error pattern.
- **Changed**: `alignment` and `hoverEffect` are now genuinely `type: "select"`, not `text` -
  the first real test of this still-unconfirmed option type. Deliberately using two different
  label conventions to maximize what a single test round teaches us:
  - `alignment` uses **localized-object labels** (`{"en": ..., "de": ...}`, both at the
    top-level option label and on every nested choice), matching the pattern the user
    independently reproduced when drafting their own select-type example.
  - `hoverEffect` uses **plain string labels** throughout, matching the convention used by
    every other confirmed-working option type in the entire codebase.

  If only one renders as a working dropdown in the SDX editor, that tells us which label
  format is actually correct. If neither does, `select` still isn't implemented/working at
  all. `_resolveOptionClass`'s defensive mapping was kept either way - `widget.getConfig()`
  returns the same plain string regardless of whether the value came from a real dropdown or
  a fallback text field, so the safety net costs nothing and covers both outcomes.
- **Changed**: every option label now carries a group prefix (`Color — `, `Typography — `,
  `Size — `, `Behavior — `) for scannability. With no confirmed way to add section headers or
  route options into the native builder's Content/Style/Advanced tabs (everything a custom
  widget defines renders flat under one Content-equivalent list - see architecture.md), a
  consistent label prefix is the only available way to give an 18-item flat list visual
  structure.
- **Changed**: the `icon` option's label now includes a concrete example
  (`e.g. "arrow-right"`) instead of just "(optional)", since a bare MDI icon name with no
  example is easy to get wrong with no visible error if it's mistyped.

## rev3-simplified-per-ben-feedback.html

Ben (BS) reviewed rev2 and gave two things: a screenshot of the exact 9 options he wants kept
(down from 18), and a complete, separately-written reference widget (`example-button-simple`)
to inspire simplification from. Both applied — see below for exactly what was adopted from
Ben's example vs. what was deliberately not, and why.

- **Reduced to exactly 9 options**, matching Ben's screenshot: alignment, text color (+hover),
  background color (+hover), border radius, open-in-new-tab, link, button text. Everything
  else from rev2 — icon, font size/weight, border width, padding, hover effects, full width,
  the `Color —`/`Typography —`/`Size —`/`Behavior —` label-group prefixes (no longer needed at
  9 items) — is dropped, not just hidden.
- **Option keys switched from camelCase to kebab-case** (`textColor` → `text-color`, etc.),
  matching Ben's convention. Since every key becomes a `--widget-<key>` CSS custom property,
  kebab-case is the more idiomatic choice for CSS either way — this is very likely worth
  adopting as the standing convention for all future widgets, not just this one.
- **Alignment re-implemented as `text-align` on `#widget-body` + `display: inline-block` on the
  button**, replacing rev2's flex container + per-alignment modifier class + JS. `left`/
  `center`/`right` are already valid `text-align` keywords, so the select value flows straight
  into CSS with zero JavaScript involved — objectively simpler than the flex approach for a
  single inline element, adopted from Ben's example.
- **`select` is used with localized-object labels** (`{"en": ..., "de": ...}`) throughout,
  matching Ben's convention exactly. This is now independently evidenced by two separate
  sources (the original charts-test-widget-v1.html sketch, and Ben's widget) pointing at the
  same label shape for `select` specifically — worth real confidence, though see the caveat
  below about how much to trust an unverified reference wholesale.
- **`widget-link` / `widget-link-new-tab` data attributes were NOT adopted**, despite being
  used in Ben's example. Verified by reading `data-attributes-lib-v1.html`'s actual current
  source directly: `Widget.onPreload` there only wires `widget-text`, `widget-visible`,
  `widget-background-image` — there is no code anywhere that reads or applies a `widget-link`
  or `widget-link-new-tab` attribute. As written, against the currently-deployed library,
  Ben's button's `link`/`link-new-tab` options would not actually take effect - the `href`
  would stay `"#"` regardless of what's configured. This widget keeps manual JS for that
  wiring instead (still just a few lines), and flags the gap as a real, concrete candidate for
  extending `data-attributes-lib` later (see `_versions/_shared-libraries/README.md`).
  **Worth raising with Ben** - his reference example has at least this one demonstrably
  non-functional part, which is also a useful data point for how much to trust the *rest* of
  it (like the `select` label format above) as verified-working vs. plausible-but-untested.
- **`widget-text="display"` WAS kept** - unlike `widget-link`, this one is confirmed real
  (present and correctly wired in the actual `data-attributes-lib-v1.html` source), so the
  button's own label still needs zero custom JS.
- **No transitions/hover motion** - matching Ben's minimal example, hover is an instant color
  swap, no `transition:` property at all. A deliberate simplification, not an oversight.

Net result: JS shrank from ~65 lines (rev2, including `_resolveOptionClass` and its two call
sites) to ~15 lines (just the link/new-tab wiring), and the whole file from 227 to 119 lines.

## rev4-declarative-link-attrs-now-confirmed-real.html

The user updated `data-attributes-lib-v1.html` itself (incorporating Ben's `widget-link`/
`widget-link-new-tab` additions, now actually implemented) and asked for it to be re-verified.
Re-read directly from source, confirmed real this time: `ATTRIBUTE_WIDGET_LINK`/
`ATTRIBUTE_WIDGET_LINK_NEW_TAB` constants exist, `_applyLink`/`_applyLinkNewTab` functions
exist, and both are now genuinely wired into `Widget.onPreload` alongside the original three.

- **Adopted `widget-link="link"` / `widget-link-new-tab="link-new-tab"`** in the TAGS block,
  removing the manual `button.href = ...` / `button.target = ...` JS from rev3 - this is what
  rev3's CHANGELOG flagged as "not adoptable yet," now genuinely adoptable.
- **Claimed (incorrectly - see rev5) a gap in `_applyLinkNewTab`**: that it should also set
  `rel="noopener"` alongside `target="_blank"`, and patched that attribute back in with a few
  lines of widget-level JS as a result. This claim was wrong - see rev5 for the correction.
- JS dropped to 3 lines (the since-removed noopener patch), down from rev3's ~15.

## rev5-noopener-correction-matches-bens-original.html — current

**Correction to rev4.** The user pointed out, quoting MDN directly, that rev4's `noopener`
claim was wrong: setting `target="_blank"` on `<a>`/`<area>`/`<form>` elements **already**
implies the same `window.opener`-blocking behavior as an explicit `rel="noopener"`, in every
currently-relevant browser - this has been standard browser behavior for several years
specifically to close this exact footgun by default. `_applyLinkNewTab` was correct and
complete as originally updated; there was never a real gap to patch around.

- **Removed** the `rel="noopener"` JS patch from rev4 entirely - `widget.onLoad` is gone, the
  script is now nothing but `widget.ready();`. This is exactly Ben's original design from the
  very first message in this whole thread - it turns out to have been correct in full once
  `data-attributes-lib-v1.html` actually implemented `widget-link`/`widget-link-new-tab` (see
  rev4). Two separate things needed to be true for that original design to work as written:
  the attributes needed to exist (they didn't at first, now they do - rev4), and the missing
  `rel="noopener"` needed to not actually matter (it doesn't - this revision).
- **The skill's `libraries.md` entry for `data-attributes-lib-v1` has been corrected** to
  remove the "real gap, not yet fixed" framing and reflect the accurate MDN-sourced behavior,
  so this mistake doesn't get repeated or passed on.

## Edited in place, 2026-09-17 (no new rev-file)

- **Moved** to `Widgets/minimal/` - the minimal widget library folder (see
  `_claude-skill/references/minimal-widgets.md`). Same file, same hrid.
- **Changed**: `category` `"General"` -> `"AI Examples/Basic"`, mirroring Ben's
  `"Examples/Basic"` so the AI-written minimal widgets sit next to his examples in the Globals
  panel. Whether the `/` actually nests is still unconfirmed - this upload will answer it.
- hrid deliberately NOT bumped: re-uploading is meant to update the widget already live in
  SDX (category only), not create a second one. Bump to `-v2` if a separate copy is wanted.



## rev6-full-localization-start-end-string-booleans.html — current

Ben sent a fresh, separately-authored copy of his own button reference
(`SDE-Download-2026-09-21 17-37-02/example-button-simple.html`). Compared line-by-line against
this widget rather than assumed newer-by-default - some of his choices were adopted, some
weren't, each for a stated reason:

- **Adopted**: every option label is now localized (`{"en": ..., "de": ...}`) - only
  `text-align` had this before, the other 8 options were plain strings. Matches
  minimal-widgets.md's own stated bar, which this widget wasn't actually meeting.
- **Adopted**: the missing `<!-- ... //-->` header comment - present on label/heading/divider,
  missing here since rev1.
- **Adopted**: `link-new-tab`'s boolean default changed from JSON `false` to the string
  `"false"`. Confirmed, not just matched for consistency: label-widget-v1's own saved
  `example-label-simple.html` (an earlier sync) had `font-bold`/`font-italic` as JSON
  booleans; today's copy of the same file has them as quoted strings, with the rest of the
  file otherwise near-identical - meaning Ben changed exactly this, most plausibly after
  hitting it in his own live testing. heading-widget-v1 and label-widget-v1 updated to match
  (see their own CHANGELOGs).
- **NOT adopted**: Ben's alignment mechanism (`display: flex; justify-content: var(--widget-
  text-align)`, values `flex-start`/`center`/`flex-end`). His own comment says this is for
  rtl support, but label-widget-v1's existing comment already identifies a real advantage
  `text-align` has that flex + justify-content doesn't: a multi-line label centers every line
  with `text-align`, but only centers the whole block with flex - individual lines inside
  would stay left-aligned. Kept `text-align`, but changed the select's *values* from
  `left`/`right` to `start`/`end` (keeping "Left"/"Right" as the labels) - this was a genuine
  inconsistency with label-widget-v1 and heading-widget-v1, which already used `start`/`end`.
- **NOT adopted**: Ben's color defaults, border-radius default, and option order - stylistic
  choices with no clear technical advantage either way; kept this widget's own SDX-accent-green
  branding and option grouping.

## Recommendations discussed but not yet implemented

None currently open - the category-nesting question is settled (architecture.md, confirmed
`/` does not nest) and kebab-case option keys are now the standing convention across all four
`minimal/` widgets.
