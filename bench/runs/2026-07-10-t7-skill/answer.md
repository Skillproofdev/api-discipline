# Versioning and pagination for our first public API

## The decision

- **Versioning: a URL path prefix, `/v1`.** Every route ships under
  `https://api.ourcompany.com/v1/...`. Breaking changes get `/v2`; non-breaking
  additions (new endpoints, new optional fields, new optional query params)
  ship into `/v1` directly, no version bump.
- **Pagination: cursor-based, two parameters — `cursor` and `limit`.**
  `cursor` is an opaque, server-issued string; omit it for the first page.
  `limit` bounds page size (1–100, default 25). Every collection endpoint
  returns the same envelope: `{ "data": [...], "pagination": { "next_cursor":
  ..., "has_more": ... } }`.

Both are declared once, in the spec's shared components, and every endpoint
reuses them verbatim — no endpoint invents its own pagination parameter
names or its own version signal.

## Why URL-prefix versioning, not a header or media type

- It's visible everywhere it matters without inspecting headers: in browser
  tabs, curl commands, error logs, support tickets, and API docs URLs. For a
  first public API, the people debugging integration issues are as often
  third-party developers as our own team — an approach that's legible by
  eyeball beats one that technically "belongs" in a header.
- It plays well with edge caching and routing: `/v1` vs `/v2` can be
  routed, cached, and rate-limited as distinct upstreams with zero
  content-negotiation logic, which a header- or media-type-based scheme
  doesn't give you for free.
- It sets an unambiguous contract from day one: nothing is unversioned, so
  there's no future migration to retrofit versioning onto live traffic —
  the single most common way public APIs end up breaking clients
  accidentally.

**Deprecation policy, stated now rather than improvised later:** when `/v2`
ships, `/v1` gets a `Deprecation` header (RFC 8594) on every response and a
minimum 6-month sunset before shutdown, announced in the changelog and by
direct email to registered API keys. This is a policy decision as much as
a technical one — write it down before the first breaking change forces
the question under time pressure.

## Why cursor pagination, not page/offset

- **Correctness under concurrent writes.** Offset pagination
  (`page`/`per_page` or `limit`/`offset`) skips or duplicates rows when
  items are inserted or deleted between page fetches — a client paging
  through results while new data arrives will silently see a corrupted
  view. Cursor pagination, anchored to a specific row rather than a
  position count, doesn't have this failure mode.
- **Performance at scale.** Offset pagination gets slower with each page
  (`OFFSET 50000` still requires scanning and discarding 50,000 rows in
  most databases). Cursor pagination stays roughly constant-time regardless
  of how deep a client pages, which matters once any collection grows past
  a trivial size — and for a public API, some integration will eventually
  page deep into a large collection.
- **It's the right default even before we know which endpoints will get
  large.** Deciding this once, up front, avoids a two-tier API where old
  endpoints use `page`/`per_page` and new ones use `cursor`/`limit` — that
  split is exactly the kind of per-endpoint inconsistency that makes
  client SDKs and documentation harder to write and worse to use.

The one real cost of cursor pagination — clients can't jump to an
arbitrary page number, e.g. "page 7 of 20" — doesn't apply to a public API
consumed by integrations, which page forward until `has_more: false`, not
by page-number UI controls. If we later build an internal admin UI that
genuinely needs "jump to page N," that's a case for a separate,
internal-only endpoint or parameter — not a reason to weaken the public
contract's default.

## What every collection endpoint must do

- Accept `cursor` (optional, opaque string) and `limit` (optional, integer
  1–100, default 25) as query parameters — declared once as shared
  OpenAPI components, `$ref`'d everywhere, never redefined per endpoint.
- Return the shared envelope shape (`data` + `pagination.next_cursor` +
  `pagination.has_more`) — not a bespoke shape per resource.
- Never ship unpaginated. A "small" collection today is a production
  incident once a client's account grows.
