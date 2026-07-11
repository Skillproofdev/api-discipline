# Pressroom CMS API design review

Reviewed `spec-under-review.yaml` (OpenAPI 3.1, `Pressroom CMS API`
v1.2.0). A linter pass (Spectral, default OpenAPI ruleset) turns up no
structural errors — every `$ref` resolves, the document is valid YAML/JSON
Schema. That's the easy 20%. The real defects in this spec are semantic:
several operations don't do what their HTTP method, status code, or stated
behavior claims they do. Those are what I focused this review on, since
they're the ones that will actually corrupt data, misfire alerts, or
surprise a client author who trusted the HTTP spec instead of this API's
prose.

Findings below, most severe first.

## Critical — method/behavior mismatches

### 1. `PATCH /articles/{articleId}` is not a PATCH — it's an undocumented destructive PUT

The operation's own description (lines 153-156): "Replaces the full
article resource with the representation in the request body. **Fields
omitted from the request are reset to their default values.**" That is
full-replace semantics, which is what `PUT` means — `PATCH` means "apply
this partial change, everything else stays as it is." As written, a
client that PATCHes `{"status": "published"}` to publish an article will
silently wipe `title`, `slug`, and `body_markdown` back to their defaults,
because those fields were "omitted" and get reset. This is a
data-loss bug waiting to happen, not a naming quibble: every REST client
library and every engineer who has ever used PATCH correctly elsewhere
will call this expecting partial-update semantics and get a destructive
overwrite instead. Fix: either (a) make this a real `PATCH` — only
fields present in the body change, ideally as `application/merge-patch+json`
per RFC 7396 to make the contract explicit — or (b) if full replacement
really is the intended operation, rename it `PUT` and document it as
replace, matching how `POST /articles/{articleId}/content` already
(correctly) uses replace-style language for its own resource.

### 2. `GET /articles` carries request-defining state in a request body

Lines 48-54: `GET /articles` declares a `requestBody` (`FilterExpression`)
for "complex filter expressions." Per HTTP semantics, a GET request's body
has no defined meaning — RFC 9110 explicitly says a body on GET has no
generally defined semantics, many HTTP client libraries (including
browsers' `fetch` in several configurations), proxies, CDNs, and API
gateways drop or reject bodies on GET, and intermediate caches key on URL
alone and won't account for a body that varies the response. A client
behind a caching proxy filed this correctly and then got someone else's
cached article list back. This isn't a style preference, it's a
correctness bug against the transport layer this API runs on. Fix: either
expose complex filters as a single encoded query parameter (URL-safe JSON
or a small filter DSL), or add a dedicated `POST /articles/search`
"search" endpoint that takes the filter tree in its body and returns the
same paged envelope — the common, well-understood pattern for
"GET-shaped-but-too-complex-for-query-params."

As a secondary issue on the same operation: it's unspecified what happens
if a caller supplies *both* the `status` query filter and a
`FilterExpression` body — AND'd together, body wins, param wins? That
interaction needs a documented rule regardless of which fix above is
chosen.

### 3. `POST /comments` returns `500` for ordinary client validation failures

Lines 339-346: the `500` response is documented to fire "when the comment
in the request body fails validation (missing body text, unknown article,
banned words)." Every one of those is a client error — bad or missing
input — and `500 Internal Server Error` is reserved for the server's own
failures. Misusing it here has real operational consequences: on-call gets
paged for a 5xx-rate alert every time a user leaves the comment box empty;
retry logic that correctly treats 5xx as "safe to retry" will retry a
comment that will *never* succeed no matter how many times it's resent
(the input is invalid, not the server); and API monitoring dashboards will
show this endpoint as unreliable when it's actually working exactly as
designed. Fix: `400` for malformed input (missing body text) or `422` for
semantically invalid-but-well-formed input (unknown article, banned
words) — consistent with how `POST /articles` already correctly
distinguishes `400` from `422` for the same two failure classes. Right
now `POST /comments` documents *no* `400` or `422` at all, so this isn't
just a wrong code, it's the only place a validation failure is documented,
and it's wrong.

### 4. `POST /articles/{articleId}/content` uses POST for a stated-idempotent replace

Lines 216-223: the operation is literally named "Replace article content"
and its own description says "This operation is idempotent: sending the
same body twice leaves the article in the same state." That is exactly
what `PUT` means, and `PUT` is the verb HTTP-aware infrastructure
(load balancers, retry-safe HTTP clients, some caches) already knows is
safe to retry automatically. Using `POST` for an idempotent replace throws
that safety away for no reason — a generic HTTP client has no way to know
this particular `POST` is safe to retry after a timeout, so it won't,
even though the API guarantees it would be fine. Fix: `PUT
/articles/{articleId}/content`.

## Major — inconsistent conventions across the API surface

### 5. `Comment.id` is an integer; every other resource ID in the API is a UUID string

`Article.id`, `Author.id` are both `type: string, format: uuid`.
`Comment.id` (line 539-541) is `type: integer, format: int64`. Nothing in
the domain explains why comments alone get a different identifier
strategy — and a client building one generic "fetch by ID" helper across
resources now has to special-case comments. If there's an infra reason
(e.g. comments live in a different, auto-increment-PK datastore), it needs
to be documented; otherwise, align it with everything else.

### 6. `Comment.created_at` is a Unix-epoch integer; every other timestamp in the API is an RFC 3339 string

`Article.created_at`/`updated_at` and `Author.created_at` are all `type:
string, format: date-time`. `Comment.created_at` (lines 549-551) is `type:
integer` with a description clarifying "Unix epoch seconds." Same
resource family, same concept, two representations — a shared
deserializer that parses `created_at` as an ISO 8601 string across the API
breaks specifically on comments. Standardize on one (I'd keep RFC 3339
given it's the majority convention already in this document and is
timezone-explicit, where epoch seconds silently assume UTC).

### 7. `/comments` error responses use `application/json` instead of `application/problem+json`

Every error response in `/articles/*` and `/authors/*` is served under
`application/problem+json` (RFC 9457), consistent with the API's own
error-body schema (`Problem`, an RFC 9457 problem-details object). But
`/comments`' `400`/`401`/`500` responses (lines 300-303, 307-309, 335-338,
343-346) are all declared as plain `application/json` even though they
use the exact same `Problem` schema. A client that content-negotiates or
dispatches on media type (which is the entire point of using
`application/problem+json` instead of generic JSON) will mishandle
comment errors specifically.

### 8. Two different sort-parameter conventions for the same concern

`GET /articles` uses a single `sort` query param taking a comma-separated,
`-`-prefixed field list (line 33-40, e.g. `-created_at`).
`GET /comments` instead uses a pair of params, `order_by` (bare field
name) + `direction` (`asc`/`desc` enum) (lines 263-275). Same job — order
a list — two incompatible shapes in the same API, and unlike pagination
(which correctly shares `Cursor`/`Limit` as reusable `components/parameters`
across every list endpoint), sorting was never factored out the same way.
Pick one convention and share it the way pagination already is.

### 9. `operationId: FetchAuthor` breaks the API's own camelCase convention

Every other `operationId` in the document is lower-camelCase:
`listArticles`, `createArticle`, `getArticle`, `updateArticle`,
`deleteArticle`, `replaceArticleContent`, `listComments`, `createComment`,
`listAuthors`. `GET /authors/{authorId}` (line 392) is `FetchAuthor` —
both a different casing (PascalCase) and a different verb (`Fetch` vs. the
`get` used for the equivalent single-resource-lookup operations on
articles). Generated SDK method names will be inconsistent
(`client.articles.get(...)` next to `client.authors.FetchAuthor(...)` or
similar) purely because of this one outlier.

### 10. Creation responses inconsistently document `Location`

`POST /comments` documents a `Location` header pointing at the created
comment (lines 323-328). `POST /articles`, creating an equally new
resource with its own `201`, does not (lines 92-98). Both are `201`
creates; only one tells the client where the created resource now lives.

## Worth flagging, lower confidence

### 11. `Comment.author_name` is free text with no link to `/authors`

`Article.author_id` is a foreign key into `/authors`. `Comment` instead
carries a bare `author_name` string (lines 545-546, 559-560) with no
`author_id` and no relationship to the `Author` resource at all. This may
be intentional — comments could be from unauthenticated/unregistered
commenters who aren't "authors" in this system's sense — but if
commenters actually are drawn from the same author pool (e.g. staff
replying to reader comments), this is a modeling inconsistency: one
resource in the API references people by ID, the other by an unverified
free-text name with no referential integrity. Worth confirming intent
either way; if commenters genuinely aren't authors, a rename to something
like `display_name` would make the "these are not the same kind of person
as `Author`" distinction clearer than reusing "author" in the field name
at all.
