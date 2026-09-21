# Minimal widget library

Canonical, AI-written reference widgets for SDX, each built to the "minimal-widget bar":
every option wired declaratively through data-attributes-lib and CSS, no loading/error state
markup, and a script that is nothing but `widget.ready();` except where CSS genuinely can't do
the job (heading's tag swap; dropdown's variable option list and its navigate-on-change). They
upload to SDX under the category `AI Examples/Basic`, next to Ben's own `Examples/Basic` widgets.

| Widget | Options | Techniques shown |
|---|---|---|
| `button-widget-v1.html` | 9 | `select` -> `text-align`, `widget-link` / `widget-link-new-tab`, bare-number px option + `calc()`, hover colors |
| `label-widget-v1.html` | 6 | boolean -> CSS via `@container style()`, `text-align: start/end` for rtl, localized labels on every option |
| `divider-widget-v2.html` | 8 | pure CSS-variable widget (no data attributes), line drawn as one border side so a `select` drives `border-style`, orientation switch via a multi-rule `@container style()` block, `select` -> `justify-content`, percent length via `calc(* 1%)` |
| `heading-widget-v1.html` | 7 | tag name from a `select` via a 5-line `onLoad` (the one thing CSS can't do), optional length option via `@property` + `not style()`, bold default on |
| `dropdown-widget-v1.html` | 3 | end-user-typed options ("Label\|URL", comma-separated) parsed into real `<option>` elements in `onLoad` - `select` can't represent variable, author-typed choices; navigates on change, `window.open()` needs explicit `'noopener'` unlike `<a target="_blank">` |
| `dropdown-scroll-widget-v1.html` | 2 | same options-parsing as dropdown-widget-v1, `document.getElementById` (deliberately not `widget.findElement`) to reach a target elsewhere on the page, `scrollIntoView({behavior:'smooth'})` |
| `dropdown-visibility-widget-v1.html` | 2 | reuses the confirmed `postMessage({type:'widget-set-row-visibility'})` pattern from overview-widget-v1 as a view switcher; hide list computed from every OTHER option's rows, not author-typed |
| `dropdown-filter-widget-v1.html` | 3 | generic `postMessage({type:'dropdown-selection'})` broadcast with an author-set key, for a future listener widget to filter by - no shared library, self-contained per rule 8; not yet paired with a listener |

How they are built, and the checklist for adding the next one:
`../_claude-skill/references/minimal-widgets.md`. Revision history per widget: `../_versions/<hrid>/`.
