# user-card-widget-v1 — revision history

The file at `Widgets\user-card-widget-v1.html` always holds the **latest** revision,
ready to (re-)upload/test in SDX. This folder keeps a copy of every revision along the way.

## rev1-initial.html

First build from Ben's "User Card" request (2026-09-18 list), 2026-10-13.

- **11 options**: user id (text, until Ben's planned `userSelector` editor type exists), layout
  (image left / on top), image shape (circle / rounded / square), image size, show email, show
  phone, show description, accent color, text color, card style (card with shadow / plain), and
  "if the user can't be loaded" (show a message / hide the widget).
- **Data, confirmed live on demo.sdx-app.com, not guessed**: `users.get(id)` for `display`,
  `email`, `description`; the phone lives in `customData.phoneNumber` and the profile image in
  `customData.image` (a content id) - both read from the SDX app's own profile and navbar code.
  The image goes through `api.setBackgroundImage`, the same stream URL the navbar uses.
- **Declarative wherever possible**: layout and image shape are `select` values that are the CSS
  value itself (`flex-direction`, `border-radius`), field toggles use `widget-visible`, card
  style and "hide on error" are style container queries. "Hide on error" needs no JS at all:
  the error block is hidden by CSS and `:has()` drops the shadow padding, so the slot collapses
  to 0px.
- **Errors**: an empty id, an id that isn't a UUID (checked client-side, since the server's own
  reply is English-only), and `ENTITY_NOT_FOUND` (404, confirmed live) each get their own
  localized message. Anything else - including a user without permission to read that user,
  which couldn't be tested from an admin account - gets the generic "couldn't be loaded" message
  and logs the raw error. The typed id is never echoed into a message, because sdx-lib writes
  error messages with `innerHTML`.
- **No profile image**: the initials of the display name show on the accent color.
- `users.get` is called directly via `api.getApi()`, with an `@todo (library candidate for
  api-lib-v1)` note, per SKILL.md rule 8.
- Tested in a local harness (real sdx/api/l10n/data-attributes libs, mocked `Application.api`):
  full user, no image + no phone + long email, stacked plain on a dark background, missing user,
  invalid id, empty id, hidden on error - en and de.

**Known, not this widget's to fix**: sdx-lib's `ready()` runs `onReady` on both window `load`
and `setTimeout(0)`, so `users.get` is called twice per instance (true for every widget).

## rev2-user-by-email-login-or-id.html

Same day, before any upload (so the hrid stays `user-card-widget-v1`). Honza doesn't know other
users' UUIDs, and a `select` can't list server users (its choices are fixed in the config), so:

- The `user-id` option became **`user`**, "User (email, login or ID)". A UUID goes to
  `users.get`; anything without `@` to `users.getByLogin`; anything with `@` to
  `users.getByEmail`, then `getByLogin` if no email matches (some instances use the email as
  the login). Confirmed live on demo: `getByEmail` is case-insensitive, `getByLogin` is exact,
  and all three answer an unknown value with 404 `ENTITY_NOT_FOUND`.
- The client-side "invalid id" message is gone (any non-UUID text is now a valid login to try);
  `noUserId` is now `noUser`.
- Harness: + email in capitals, login, an email-shaped login, unknown email (10 cases, en + de).

**Open**: what `users.get` throws for a non-admin reading someone else's profile - test with a
non-admin login before relying on the generic message.
