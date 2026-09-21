# Content / File API

Everything in this file is now backed by the real SDX API documentation and client source,
supplied directly by the user (`api.documentation.html`, `api.js`, served from
`/api/dl/dlt/stream?id=...` on the SDX instance itself), cross-checked against live
introspection in a browser session. Where something is still inferred rather than confirmed,
it's marked explicitly.

## The Content shape (from content-shape.json, a real sample)

```json
{
	"id": "eac7479b-b8b1-4ca5-bc88-90f54e4db7d9",
	"name": "IMAGE-JPG.jpg",
	"mimeType": "image/jpeg",
	"size": 415882,
	"logicalType": "",
	"hidden": false,
	"type": "de.jenomics.sde.entities.contents.DefaultFileContent",
	"parentId": "d6bc0706-a2ab-4679-b6f4-eb9c20a9a4d7",
	"spaceId": "d6bc0706-a2ab-4679-b6f4-eb9c20a9a4d7",
	"ownedBy": "32ec023d-1ed6-4ce5-82d3-9492d605b35d",
	"createdBy": "32ec023d-1ed6-4ce5-82d3-9492d605b35d",
	"changedBy": "32ec023d-1ed6-4ce5-82d3-9492d605b35d",
	"created": 1784731825098,
	"changed": 1784731850397,
	"newContent": true,
	"description": "",
	"encoding": "",
	"auditDownloadOrPreview": true,
	"customData": { "...": "arbitrary key/value map, e.g. weather condition tags" }
}
```
`ownedBy` / `createdBy` / `changedBy` are user UUIDs — see the Users service below for how to
resolve them to a display name.

## The `contents` service — confirmed methods relevant to widgets

| Method | Signature | Login required |
|---|---|---|
| `contents.find` | `find(Map query)` → `QueryResult<Content>` | true |
| `contents.get` | `get(UUID id)` → `Content` | **false** |
| `contents.getMany` | `getMany(List ids, boolean allowPartialResolvement)` → `List<Content>` | true |
| `contents.stream` | `stream(UUID id, boolean preview, String locale)` → `StreamResult` | false |
| `contents.streamByName` | `streamByName(String fullname, boolean preview, String locale)` → `StreamResult` | false |
| `contents.getQueryDimensions` | `getQueryDimensions()` → `Set<QueryDimension>` | true |

**`contents.find(query)`** — `query` is a `Map`, whose accepted keys are exactly what
`getQueryDimensions()` returns (below); there's no separate prose spec for the query shape
beyond that. `response.result` is the array of Content objects — this is what `api.findLatest`/
`api.findByLogicalType` already unwrap for you.

**Query dimensions confirmed** (live, via `dlApi.contents.getQueryDimensions()` — the
documentation page lists the method but not a static list of dimension names, since they're
computed at runtime):

| dimension | matchType | type |
|---|---|---|
| `contentHidden` | equal | boolean |
| `containerId` | equal | UUID |
| `contentName` | **like** | String |
| `spaceName` | like | String |
| `contentId` | equal | UUID |
| `contentLogicalType` | like | String |
| `spaceId` | equal | UUID |
| `containerPath` | like | String |
| `created` | equal | Date |

**There is no `mimeType` (or any file-type/extension) dimension, and no owner/createdBy
dimension.** This is exhaustive. So:
- Filtering by file type MUST go through `contentName` with `matchType: "like"` — e.g.
  `{contentName: '%.jpg'}` (standard SQL LIKE `%` wildcard, per the user's own steer). Run one
  query per extension pattern (`%.jpg`, `%.jpeg`, ...) since a single dimension takes one
  pattern, then merge results and — since LIKE-on-name can only narrow, not guarantee, the
  actual type — double-check each result's real `mimeType` client-side before trusting it.
  **Still unverified**: the exact wildcard/case-sensitivity behavior of this LIKE, and what
  `preview=true` vs `preview=false` change about the streamed response (headers? watermarking?
  a resized thumbnail?) beyond just being a real, documented parameter — the docs give the
  parameter table but no prose description of either.
- Owner-name resolution cannot happen via `contents.find` at all — it must go through the
  separate `users` service (below).
- `spaceId` IS a valid equal-match dimension — confirms the space-scoping pattern used by
  pv-investor-lib/news-lib is the documented way to restrict to one space, and that simply
  omitting it queries across all spaces the session can see (what "all accessible spaces" scope
  means in practice).

**`contents.get(id)` needs no login** — the one exception among the methods above. Every other
content/user method requires an authenticated session (which every widget already has).

**`contents.getMany(ids, allowPartialResolvement)`** — officially returns a plain
`List<Content>` in the Java signature. Combined with what's confirmed for `users.getMany`
(below), this is now believed to return the same `{type: "List", value: [...]}` wrapped shape
over the wire — see "Collection-wrapping convention" below. **Not yet independently
live-tested** for `contents.getMany` specifically, so verify before relying on it.

## Streaming, previewing, and downloading a file

- **`contents.stream(id, preview, locale)`** and **`contents.streamByName(fullname, preview,
  locale)`** are the real, documented methods behind file streaming. `preview` is a genuine
  `boolean` parameter — **its documented default is `false`**, not `true`.
- **The "AsGetURL" convention, confirmed generally**: any `dlApi.<service>.<method>` has a
  matching `<method>AsGetURL(...)` variant that takes exactly the same parameters as the base
  method, plus one extra trailing `boolean` for cache-busting (see the docs' own example:
  `sde.app.activateMaintenanceModeAsGetURL()` vs. `sde.app.activateMaintenanceModeAsGetURL(true)`
  for a no-arg method). This resolves what was previously a mystery: the codebase's
  `dlApi.contents.streamAsGetURL(id, preview, undefined, cacheBust)` call has 4 arguments
  because the real signature is `streamAsGetURL(id, preview, locale, cachebust)` — the 3rd
  argument is `locale` (left `undefined` throughout this codebase, i.e. server default), not an
  unexplained param. Apply this same "base params + trailing cachebust" pattern to any other
  `...AsGetURL` call in the codebase.
- **api-lib-v1 bug, confirmed**: `api.streamContentById`'s implementation is
  `preview = preview || true;` — because of how `||` works in JS, this line **forces `preview`
  to `true` even when the caller explicitly passes `false`**, since `false || true` evaluates to
  `true`. Its own JSDoc ("defaults to true") describes the buggy behavior as if it were
  intentional, but the real API's own documented default is `false`. This bug doesn't affect
  jpg-gallery-widget, which calls `dlApi.contents.streamAsGetURL` directly rather than through
  this wrapper — but don't copy this `x = x || true` pattern, and consider fixing
  `api.streamContentById` itself the next time it's touched (use `preview === undefined ? true :
  preview`, or decide the wrapper's default should actually be `false` to match the real API).
- No documentation seen so far spells out the *behavioral* difference between `preview=true` and
  `preview=false` (thumbnailing vs. full file, response headers, etc.) — only that it's a real
  parameter with a real default. Treat "preview=false → real download" as a reasonable but not
  fully confirmed inference.

## Collection-wrapping convention (confirmed general pattern)

Any `dlApi` method whose Java return type is a `Set` or `List` appears to be serialized over the
wire as `{type: "Set"|"List", value: [...]}` rather than a bare JS array — confirmed live for
`dlApi.contents.getQueryDimensions()` (`Set<QueryDimension>`) and `dlApi.users.getMany()`
(`List<User>`), and now strongly implied for every other `List`/`Set`-returning method
(`contents.getMany`, `groups.getMany`, `spaces.getMany`, etc. — all documented as returning
`List<X>`). **Always unwrap `.value` (e.g. `Array.from(response.value)`) rather than assuming a
plain array** for any method whose documented return type is `List<...>` or `Set<...>`.
Query-style methods (`find`) are a different shape — `{result: [...]}` — don't confuse the two.

## The `users` service — confirmed methods relevant to widgets

| Method | Signature | Login required |
|---|---|---|
| `users.get` | `get(UUID id)` → `User` | true |
| `users.getMany` | `getMany(List ids, boolean allowPartialResolvment=true)` → `List<User>` | true |
| `users.find` | `find(Map query)` → `QueryResult<User>` | true |
| `users.getByDisplayName` | `getByDisplayName(String displayName)` → `User` (exact match) | true |
| `users.getByEmail` | `getByEmail(String email)` → `User` (exact match) | true |
| `users.getByLogin` | `getByLogin(String login)` → `User` (exact match) | true |
| `users.getQueryDimensions` | `getQueryDimensions()` → `Set<QueryDimension>` | true |

`users.getMany`'s second parameter `allowPartialResolvment` **defaults to `true`** — meaning a
call with just `dlApi.users.getMany(ids)` (as jpg-gallery-widget does) will still return
whatever it *could* resolve rather than failing outright if one id is bad. That's the sensible
default and needs no change.

`users.find` query dimensions (live-confirmed): `userId`/`userActive`/`userCreated` (equal),
`userEMail`/`userDisplay`/`userDescription`/`userLogin` (like), and a genuine **range pair** —
`userLastLoginStartDate` (greaterEqual) / `userLastLoginEndDate` (lessEqual) — the first
confirmed example of a range-style dimension pair anywhere in this API, useful precedent if a
future widget needs a date-range filter on anything.

A User object's fields (live-confirmed): `id`, `display`, `email`, `login`, `active`,
`description`, `created`, `changed`, `createdBy`, `changedBy`, `ownedBy`, `lastLogin`,
`loginCount`, `failedLogins`, `customData`, `type`.

`getByDisplayName`/`getByEmail`/`getByLogin` are exact-match single-lookup alternatives to
`find`/`getMany` — worth reaching for directly when you have one of those three values in hand
and don't need a LIKE search or a batch.

## Still-open items

1. **Exact SQL LIKE wildcard/case-sensitivity behavior** for `contentName` and the other `like`
   dimensions — confirmed to exist (now via official docs, not just introspection), but the
   docs don't spell out matching semantics beyond the parameter table, and it hasn't been
   independently stress-tested against real data (e.g. an uppercase `.JPG` extension).
2. ~~**"select" config option type.**~~ Resolved 2026-09-17: `select` works in the editor (see
   architecture.md). Confirmed option types are now `text`/`color`/`boolean`/`select`.
3. **What `preview=true` vs `preview=false` actually changes** about the streamed response
   (thumbnailing, headers, watermarking) — confirmed as a real documented parameter with a real
   default, but no prose description of its effect was found in the docs pulled so far.
