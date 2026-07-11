# API design review — Pressroom CMS API (`spec-under-review.yaml`)

## Summary

Ran `npx @redocly/cli lint spec-under-review.yaml`: **0 errors, 1 warning**
(missing `info.license` — cosmetic). Structurally this spec is clean: every
`$ref` resolves, error responses consistently `$ref` the shared `Problem`
schema (mostly — see 5 below), versioning is stated, pagination is a
shared component. The real problems here are **semantic**, not structural
— the kind a validator can't catch: an HTTP method used opposite to what
its own description says, a GET with a request body, a `500` for a client
mistake. These are more dangerous than naming nits because client SDKs and
HTTP infrastructure (caches, proxies, retry logic) make hard assumptions
based on method semantics — get these wrong and the failures show up as
"weird caching bugs" and "our retry logic double-posted," not obvious
4xx-in-the-wrong-place cosmetics. 9 distinct defects found, below.

---

## 1. HTTP semantics (Rule 4) — the load-bearing findings

**1.1 — `GET /articles` has a request body. `GET` must never have one.**
Location: `paths./articles.get.requestBody` (the `FilterExpression` body
for "complex filter expressions"). The description explicitly
acknowledges two channels for the same GET: query params for simple
filters, a body for complex ones.
Why it matters: many HTTP clients, proxies, and caches silently drop or
reject bodies on GET requests — this isn't a style rule, `GET` bodies are
explicitly disallowed by the HTTP semantics RFC (9110 §9.3.1: a payload
"has no defined semantics" on GET) and support for them is inconsistent
across the ecosystem. Complex filters will work in some clients and
silently lose the filter in others.
Fix: move complex filtering off `GET`. Cleanest option: a dedicated
search endpoint, `POST /articles/searches` (a sub-resource collection of
search results/queries — not a verb, consistent with Rule 1's
`.../cancellations` pattern), taking the `FilterExpression` in its body
and returning the same paginated envelope `GET /articles` uses. Simpler
alternative if a new endpoint is unwanted: encode the filter as a
URL-safe JSON string in a query parameter (e.g. `?filter=<encoded
FilterExpression>`) — uglier but keeps it a true, cacheable `GET`.

**1.2 — `PATCH /articles/{articleId}` is documented as full replace, which
is `PUT` semantics, not `PATCH`.**
Location: `paths./articles/{articleId}.patch` — description: "Replaces
the full article resource with the representation in the request body.
Fields omitted from the request are reset to their default values." That
is `PUT`'s contract by definition (Rule 4: "`PUT` replaces the full
resource and is idempotent... Using `PUT` for a partial update is a
defect; that's `PATCH`" — this is the same defect in reverse: using
`PATCH` for a full replace). It's also internally inconsistent: `
ArticleUpdate` has zero required properties and no defined defaults for
`status`/`body_markdown`/etc., so "reset to default values" isn't even
implementable against the schema as written.
Fix: if full replacement really is the intent, rename to
`PUT /articles/{articleId}`, make the fields that must always be supplied
`required` on the request schema, and drop `application/merge-patch+json`
framing entirely. If partial update was actually intended (more likely,
given the method name), fix the description to merge-patch semantics
(RFC 7396) and switch the request content type to
`application/merge-patch+json`, matching how `updateProduct`/
`updateBooking`-style PATCH endpoints are normally documented.

**1.3 — `POST /articles/{articleId}/content` is documented as idempotent
full replacement, which is `PUT` semantics, not `POST`.**
Location: `paths./articles/{articleId}/content.post` — description:
"Replaces the entire content block... This operation is idempotent:
sending the same body twice leaves the article in the same state." Rule 4:
`POST` is for non-idempotent creates/triggers; a documented idempotent
full-replace belongs on `PUT`. This is the mirror image of 1.2 — one
endpoint uses `PATCH` for what should be `PUT`, this one uses `POST` for
what should be `PUT`.
Fix: rename to `PUT /articles/{articleId}/content`. No other change needed
— the semantics described are already correct, just attached to the wrong
method.

**1.4 — `POST /comments` documents `500` for client validation failures.**
Location: `paths./comments.post.responses.500` — description: "Returned
when the comment in the request body fails validation (missing body
text, unknown article, banned words)." This is exactly the case Rule 2
calls out by name: "A validation failure is 4xx — never document 500 for
a client mistake." A `500` tells monitoring/on-call this is a server bug;
routing every bad comment submission there will either drown real
incidents in noise or get this alert muted, at which point real 500s go
unnoticed.
Fix: split by validation kind, consistent with the rest of the spec's
error model: structurally malformed input (missing `body` entirely,
wrong types) → `400`; semantically invalid but well-formed input (unknown
`article_id`, banned words) → `422`. Both already exist as response types
elsewhere in this same spec (`createArticle` already documents both `400`
and `422` correctly) — this endpoint should match that pattern, not
invent a third one.

---

## 2. Conventions declared once (Rule 3) / consistency (Rule 8)

**2.1 — Two different sort conventions in the same spec.**
Location: `GET /articles` uses `sort` (comma-separated fields, `-` prefix
for descending — e.g. `-created_at`); `GET /comments` uses `order_by` +
`direction` (separate field name and separate enum `asc`/`desc`). Same
concept ("how do I sort this list"), two incompatible parameter shapes.
Fix: pick one (this review recommends the `sort=-field` form already used
by `articles`, since it composes multi-field sort in one parameter) and
apply it to `comments` too; drop `order_by`/`direction`.

**2.2 — `/comments` error responses use `application/json`; every other
endpoint in the spec uses `application/problem+json` for the same shared
`Problem` schema.**
Location: `paths./comments.get.responses.400`/`401` and
`paths./comments.post.responses.401`/`500` all set `content:
{application/json: {schema: $ref Problem}}`, while `/articles` and
`/authors` consistently use `application/problem+json` for the identical
schema.
Fix: change `/comments`' error response content type to
`application/problem+json` to match the rest of the spec — same schema,
inconsistent media type is still a defect per Rule 8 item 4 ("same content
type").

**2.3 — Mixed ID formats: `Comment.id` is `int64`, every other resource's
`id` is `uuid`.**
Location: `components.schemas.Comment.properties.id` (`type: integer,
format: int64`) vs. `Article.id`, `Author.id` (both `type: string, format:
uuid`).
Fix: standardize on one ID format spec-wide (this spec is already
`uuid`-first — 2 of 3 resources use it) and change `Comment.id` to match,
or if there's a genuine reason comments use an auto-increment integer
(e.g. a legacy comment store), document that exception explicitly in
`info.description` per Rule 8 item 6 ("not mixed without reason") — as
written, there's no stated reason, so it reads as an oversight.

**2.4 — Mixed timestamp representation: `Comment.created_at` is a Unix
epoch integer, every other timestamp in the spec is an ISO 8601
`date-time` string.**
Location: `components.schemas.Comment.properties.created_at` (`type:
integer`, "Creation time as Unix epoch seconds") vs. `Article.created_at`/
`updated_at`, `Author.created_at` (all `type: string, format: date-time`).
Fix: change `Comment.created_at` to `type: string, format: date-time` to
match. Epoch integers lose timezone-explicitness and are a different
parsing path for every client than the rest of the spec already commits
to.

**2.5 — `operationId: FetchAuthor` breaks both the casing and the verb
convention every other operation in the spec follows.**
Location: `paths./authors/{authorId}.get.operationId`. Every other
operationId is camelCase and uses the `{verb}{Resource}` pattern with
`get`/`list`/`create`/`update`/`delete` as the verb (`getArticle`,
`listComments`, `createComment`, ...). `FetchAuthor` is PascalCase and
uses `Fetch` instead of `get`.
Fix: rename to `getAuthor`.

---

## Consistency audit (Rule 8 checklist, run in full)

| # | Check | Result |
|---|-------|--------|
| 1 | Property casing uniform | Pass — snake_case throughout |
| 2 | All collection paths plural nouns | Pass |
| 3 | Zero verbs in paths | Pass |
| 4 | All 4xx/5xx `$ref` shared error schema, same content type | **Fail** — see 2.2; also 1.4 (`500` isn't even a documented client-error case) |
| 5 | Pagination param names/types identical on every collection | Pass — `cursor`/`limit` shared component used everywhere |
| 6 | ID formats uniform | **Fail** — see 2.3 |
| 7 | Timestamps uniform | **Fail** — see 2.4 |
| 8 | `operationId` pattern uniform | **Fail** — see 2.5 |
| 9 | Same action → same status code everywhere | Pass on paper (all creates `201`, all deletes `204`) but see 1.2/1.3 — two operations are *labeled* with the wrong action for what they do, which undermines this check's premise |
| 10 | Sort/filter conventions uniform | **Fail** — see 2.1; also 1.1 (filtering has two channels — query params and a GET body — on one endpoint, and the body channel shouldn't exist at all) |

## Validation

`npx @redocly/cli lint spec-under-review.yaml`: **0 errors, 1 warning**
(missing `info.license`). The clean validator run does not indicate a
sound design — every substantive finding above is a semantic/HTTP-method
defect no schema validator checks for.
