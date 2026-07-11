# Versioning and pagination for our first public API

## Versioning: URL path prefix, `/v1`

Every request goes to `https://api.ourcompany.com/v1/...`. When we ship a
breaking change, it goes out as `/v2` at a new base path; `/v1` keeps
running unchanged until we deliberately retire it (target: minimum 6
months' notice after `/v2` ships, communicated via changelog and, if we
have a developer portal, an in-dashboard banner for API keys still calling
`/v1`).

**Decision, not a menu**: we are not doing header-based versioning
(`Accept: application/vnd.ourapi.v1+json`) or a query parameter
(`?version=1`). Reasons:

- **Visible in every log line, every curl command, every browser
  address bar.** A support engineer looking at an access log can tell
  which contract a request used without cross-referencing headers. This
  matters enormously for a first public API, where most of our early
  integrators will be debugging by hand, not through a polished client
  SDK.
- **Cacheable by URL.** CDNs and HTTP caches key on URL by default; a
  header-based scheme needs `Vary` handled correctly everywhere or you get
  cache poisoning between versions. Path versioning needs nothing extra.
- **Zero ambiguity about what "unversioned" means.** A header-based scheme
  has to define default behavior when the header is absent; that default
  becomes a silent, undocumented version that's easy to break by accident.
  With a path prefix there's no request without a version — the question
  doesn't arise.

**What "a version" covers**: the whole API bumps together, not per
resource or per endpoint. Independent per-resource versions look flexible
early on and turn into a compatibility matrix nobody can hold in their
head by the second year. One version number, one changelog, one deprecation
clock.

**What counts as breaking** (and therefore requires `/v2`, not a same-version
change): removing or renaming a field, changing a field's type or
meaning, changing a status code for an existing situation, adding a newly
*required* request field, or changing default behavior for existing
requests. Adding new optional fields, new endpoints, or new optional query
parameters does not require a version bump and should ship straight into
`/v1`.

## Pagination: cursor-based, `cursor` + `limit`

Every list endpoint takes the same two query parameters and returns the
same envelope shape:

```
GET /v1/things?limit=25&cursor=eyJpZCI6...

{
  "data": [ ... ],
  "pagination": {
    "next_cursor": "eyJpZCI6...",   // null when there are no more pages
    "limit": 25
  }
}
```

- `limit` — page size, 1-100, default 25.
- `cursor` — opaque, base64-encoded, points at "everything after this
  item" per the endpoint's default sort. Never inspect or construct it
  client-side; treat it as a token.
- `pagination.next_cursor` — pass straight back as `cursor` to get the next
  page; `null` means you're on the last page.

**Decision, not a menu**: we are not doing offset/limit (`?offset=50&limit=25`)
or page-number (`?page=3&per_page=25`) pagination anywhere in this API, even
for endpoints that feel small today. Reasons:

- **Correct under concurrent writes.** Offset pagination is defined
  relative to the *current* result set at query time. If a row is
  inserted or deleted ahead of a client's current offset while they're
  paging through, they skip or double-see rows on the next request. This
  is silent and hard to reproduce in a bug report — cursor pagination,
  anchored to a specific row rather than a position, doesn't have this
  failure mode.
- **One convention, applied everywhere, from day one.** The expensive
  version of this mistake is shipping offset pagination on a "this list
  will always be small" endpoint, having it turn out not to stay small,
  and then needing a breaking change (a new version) just to fix
  pagination on that one endpoint. Committing to cursor pagination
  everywhere now means every future list endpoint — including the ones we
  haven't designed yet — is already covered, and a client SDK writes one
  pager helper instead of two.
- **`total` counts are dropped on purpose.** Offset/page schemes tempt you
  into returning a `total` row count, which is expensive to compute
  correctly on a large, actively-written table and is exactly the kind of
  field that quietly becomes "approximate" under load. We don't return
  one. If a UI needs an approximate count for display, that's a separate,
  explicitly-approximate endpoint or field — not smuggled into every list
  response's contract.

**What this costs us, deliberately accepted**: no "jump to page 7"
support, and a cursor can't be constructed by a client from scratch
(you can't say "give me item 500 directly"). For a first public API this
trade is worth it — arbitrary page-jumping is a UI nicety most API
consumers don't actually need, and giving it up buys correctness we'd
otherwise have to retrofit later as a breaking change.
