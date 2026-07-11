# Crmly API design review

Reviewed `spec-under-review.yaml` (OpenAPI 3.1, `Crmly API` v1.0.0) end to
end: every path, every schema, every response. Findings below, grouped by
severity. Each entry names the exact location, the problem, why it matters
for a public/production API, and the fix I'd make.

I also ran a spec linter (Spectral, default OpenAPI rules) as a
cross-check; it independently confirmed finding #1 (a hard structural
error) and flagged the missing `operation-description` warnings on nearly
every operation, which I'm not itemizing individually below since it's the
same gap repeated ~9 times — worth a blanket pass before ship, but it's a
docs-quality issue, not a design defect.

## Blocker

### 1. `GET /order/{orderId}` references a schema that doesn't exist

`responses.200.content.application/json.schema` is `$ref:
'#/components/schemas/OrderDetails'` (line 292). No `OrderDetails` schema
is defined anywhere in `components/schemas` — only `Error`, `User`,
`UserCreate`, `UserUpdate`, `Note`, `Order` exist. This isn't a style nit:
the document doesn't resolve, so every code-generation tool, mock server,
and validator that consumes this spec breaks on this one operation. Given
the summary says "Get an order with line items," the missing schema
presumably needs to be `Order` plus a `line_items` array — it needs to
actually be defined, not just referenced.

## Major — HTTP method / status semantics violated

### 2. `PUT /users/{userId}` claims full-replace but implements partial update

The operation's own description says: "Partially updates a user. Only the
fields present in the request body are changed; omitted fields keep their
current values." That is PATCH semantics, not PUT. `PUT` means "replace
this resource with what I'm sending" — a client that omits `phone_number`
because it doesn't have a new value would, under correct `PUT` semantics,
be understood as "clear phone_number," but this API's actual behavior is
"leave it alone," which is exactly backwards for `PUT` and exactly right
for `PATCH`. This will bite the first client author who reads the HTTP
spec instead of this API's docs. Fix: make this `PATCH` (ideally with
`Content-Type: application/merge-patch+json` to make the merge semantics
explicit), and reserve `PUT` — if you want it at all — for true full
replacement.

### 3. `POST /users` returns `200` instead of `201` on creation, no `Location` header

`responses.200` on `createUser` (line 82) should be `201 Created`. A
`201` with a `Location` header pointing at `/users/{userId}` is how a
client is supposed to learn the new resource's canonical URL without
having to know your path scheme in advance. Right now a client has to
infer the URL from `User.id` in the body — it works, but it's not the
protocol doing its job, and it's inconsistent with typical REST practice
this API otherwise seems to be reaching for (e.g., the delete-vs-204
pattern used correctly elsewhere).

### 4. Two different DELETE response conventions for no stated reason

`DELETE /users/{userId}` returns `204 No Content` (correct, standard).
`DELETE /order/{orderId}` returns `200` with a body: `{deleted: boolean,
order: Order}` (lines 303-314). Same HTTP verb, same kind of operation,
two incompatible response shapes in the same API. A shared client can't
write one `deleteResource()` helper. Pick one — I'd standardize on `204`
for both, since the deleted representation is rarely useful once the
resource has just been told to stop existing; if there's a real reason
callers need the final order body back (e.g., for a receipt/confirmation
screen), then apply the `200 + body` pattern to `/users/{userId}` too, and
document why deletes return bodies in this API.

## Major — resource design / naming

### 5. `GET /getUser` is an RPC-shaped endpoint bolted onto a REST API

`/getUser` (line 109) breaks the resource-oriented convention the rest of
the API follows (`/users`, `/users/{userId}`). It's verb-named, camelCase
in a path where every other segment is a lowercase noun, and it duplicates
"fetch a single user" as a second, structurally different mechanism from
`GET /users/{userId}`. The intent — look a user up by email instead of ID
— is a filter, not a different operation. Fix: `GET /users?email=...`,
returning the existing paged list envelope (with `email` as an exact-match
filter, since email is presumably unique, this degenerates cleanly to
0-or-1 results). This also removes the need for a bespoke 404-only error
contract on this one endpoint.

### 6. Inconsistent pluralization: `/users` vs. `/order`

`/users` and `/users/{userId}` are plural; `/order` and `/order/{orderId}`
(lines 228, 273) are singular. Should be `/orders` and `/orders/{orderId}`.
Small on its own, but exactly the kind of inconsistency that makes a
generated SDK or a new engineer's mental model unreliable across the two
resources this API has.

## Major — inconsistent conventions across the API surface

### 7. Two incompatible pagination schemes in one API

`GET /users` paginates with `page` + `page_size` (page-number style,
0-indexed... 1-indexed here, with a `total` count). `GET /order` paginates
with `limit` + `offset` (lines 233-250) and returns no `total`. These are
different pagination models with different client-side math, in the same
API, for the same kind of "list of records" operation. The brief this kind
of API is usually built against wants exactly one pagination convention
that every list endpoint uses identically — pick one (I'd lean
cursor-based if either collection can be written to concurrently while a
client pages through it, otherwise page/page_size is fine) and apply it
everywhere, including the next issue.

### 8. `GET /users/{userId}/notes` isn't paginated at all, and isn't even wrapped

Every other list endpoint returns `{data: [...], <pagination fields>}`.
`listUserNotes` (lines 209-227) returns a bare JSON array. Two problems:
(a) shape inconsistency — a shared client parser that expects `.data`
breaks on this one list endpoint; (b) no `page`/`limit`/cursor parameters
at all, so a customer account that accumulates notes over years (which
CRM note threads do) has no way to be fetched in pages — this endpoint
will eventually return an unbounded, slow, memory-heavy response. Fix:
same envelope, same pagination convention as the rest of the API.

### 9. Two different error-body shapes for the same status code

Nearly every error response in this spec uses the shared `Error` schema:
`{code: string, message: string}`. But `POST /users`'s `400` (lines 88-102)
defines its own one-off inline schema instead: `{error_code, msg,
field_errors}` — different field names for the same concepts (`error_code`
vs `code`, `msg` vs `message`), plus a `field_errors` array of bare
strings with no field-name attribution (a client can't tell *which* field
failed, only that some string describes a failure). This is precisely the
"per-endpoint response parser" tax a shared client wants to avoid. Fix:
extend the shared `Error` schema with an optional, structured
`field_errors: [{field, message}]` array and use it everywhere validation
errors occur, rather than inventing a parallel shape for one endpoint.

### 10. Inconsistent property casing within and across schemas

`User`/`UserCreate` mix `firstName` (camelCase) with `last_name`,
`phone_number` (snake_case) — two casing conventions in the same object.
Separately, `User.createdAt` is camelCase while the semantically identical
field is `created_at` (snake_case) on both `Note` and `Order`. Whatever
casing convention is chosen (this codebase leans snake_case given 5 of 6
timestamp/name fields already use it), it needs to be applied uniformly —
right now a client can't guess a field's casing from any pattern, it has
to check the schema every time.

## Completeness gaps

### 11. Missing `401` response on multiple endpoints despite global auth

Security (`bearerAuth`) is declared globally (`security:` at the document
root, line 10-11), so every operation requires a valid bearer token. Yet
`401` is undocumented on: `GET /users/{userId}` (144), `GET /getUser`
(110), `GET /users/{userId}/notes` (209), `GET /order` (229), `GET
/order/{orderId}` (282), and `DELETE /order/{orderId}` (299). Contrast
with `GET /users`, `POST /users`, and `PUT /users/{userId}`, which do
document it. A generated client/mock built from this spec will treat
unauthenticated calls to the undocumented endpoints as an unspecified
failure mode instead of the `401` they'll actually get.

### 12. `GET /users/{userId}` has no `404` documented

Its siblings on the exact same path — `PUT /users/{userId}` and `DELETE
/users/{userId}` — both document `404` for a nonexistent user (lines
181-186, 194-199). `GET` on the same resource omits it (144-154), which is
inconsistent for literally the same lookup-by-ID failure mode on the same
path.

## Data modeling

### 13. `Order.currency` is optional while `Order.total_cents` is required

`total_cents` (line 405-406) is required; `currency` (407-408) is not. A
monetary amount with no guaranteed unit is a latent bug source — every
consumer either has to assume a currency (wrong for a multi-currency CRM)
or defensively null-check a field that should always be present alongside
an amount. Make `currency` required, or fold both into a single
non-nullable `Money` object (`{amount_cents, currency}`) reused wherever
money appears.

### 14. No API versioning strategy stated anywhere

`servers[0].url` is `https://api.crmly.dev` — no version segment — and
`info.description` says nothing about how breaking changes will be
introduced later. For an API description that (per its own `security` and
`tags`) is clearly meant to be a stable, documented, external-facing
surface, having zero stated versioning policy means the very first
breaking change (a rename, a required-field addition, anything in this
review that gets fixed) has no defined migration path for whoever is
already integrated. Recommend deciding this now — URL-prefixed (`/v1`) is
the simplest to reason about — rather than retrofitting it under pressure
later.

## Lower-confidence note (not counted as a defect, flagging for awareness)

`POST /users` documents no `409 Conflict` for a duplicate `email`. If
email is meant to be unique (plausible for a customer/account system, and
`getUserByEmail` implies email is a lookup key), the create endpoint
should document what happens on a collision instead of leaving it to fall
into the generic `400`/`422` bucket implicitly. If email isn't unique,
disregard.
