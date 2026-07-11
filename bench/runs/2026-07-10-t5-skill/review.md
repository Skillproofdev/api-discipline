# API design review — Crmly API (`spec-under-review.yaml`)

## Summary

Ran `npx @redocly/cli lint spec-under-review.yaml`: **1 error, 2 warnings**
(broken `$ref`, missing `license`, one operation with zero documented 4xx
responses — all three corroborate findings below). Beyond what the linter
catches mechanically, the spec has real design-discipline problems: two
different pagination schemes for the same concept, two different error
shapes (one of them per-endpoint), a `PUT` that is actually a `PATCH`, a
`POST` that returns `200` instead of `201`, a verb-shaped path, and no
versioning strategy at all. 13 distinct defects found, grouped below by
which rule they break, each with location and fix. Not adopting this
contract as-is — every item below needs a decision before integration
work starts against it.

---

## 1. Resource naming and paths (Rule 1)

**1.1 — `GET /getUser` is a verb-shaped path, not a resource.**
Location: `paths./getUser`.
Fix: fold this into the collection as a filter — `GET /users?email=...` —
reusing the existing `/users` list endpoint instead of a bespoke lookup
endpoint. If email lookup must return exactly one result semantically,
document that `email` is unique and the list is expected to contain 0 or 1
items when filtered by it, rather than inventing a second retrieval path.

**1.2 — `/order` and `/order/{orderId}` are singular; every other
collection (`/users`) is plural.**
Location: `paths./order`, `paths./order/{orderId}`.
Fix: rename to `/orders` and `/orders/{orderId}`. This also means
`operationId`s and any client code generated from the old paths will
change — call this out as a breaking rename if this spec has ever shipped.

**1.3 — Property casing is mixed within the same resource.**
Location: `components.schemas.User.properties.firstName` (camelCase) next
to `last_name`, `phone_number` (snake_case) in the same object; also
`User.createdAt` (camelCase) vs. `Note.created_at` / `Order.created_at`
(snake_case) — the same concept named two different ways across schemas.
Repeated in `UserCreate` and `UserUpdate` (`firstName` vs. `last_name`).
Fix: pick one casing (the majority of the spec is already snake_case) and
rename `firstName` → `first_name`, `createdAt` → `created_at` everywhere.
This is a breaking change to `User`/`UserCreate`/`UserUpdate` if live —
flag it as such rather than silently changing field names in a "review."

---

## 2. Error model (Rule 2)

**2.1 — `POST /users` invents a one-off error shape for `400` instead of
reusing the shared `Error` schema.**
Location: `paths./users.post.responses.400` — inline schema
`{error_code, msg, field_errors}`, structurally and semantically different
from `components.schemas.Error` (`{code, message}`) used everywhere else.
Fix: delete the inline schema; `$ref` the shared `Error` schema like every
other error response. If field-level validation detail is genuinely
needed, add it as an optional array on the *shared* `Error` schema (so
every endpoint gets it, not just this one) — don't fork the shape per
endpoint.

**2.2 — The shared `Error` schema is not RFC 9457 Problem Details, and
responses aren't served as `application/problem+json`.**
Location: `components.schemas.Error` (`{code, message}`, no `type`,
`title`, `status`); every error response's `content` key is
`application/json`.
Fix: either adopt RFC 9457 (`type`/`title`/`status`/`detail`/`instance`,
content type `application/problem+json`) as the skill's default
recommends, or explicitly document a different standardized shape in
`info.description` — right now the spec does neither, so there's no
stated contract for what an error looks like, only what one example
happens to look like.

**2.3 — Several operations don't document the errors they can actually
return.**
Location and specifics:
- `GET /users/{userId}` (`getUser`): **zero** 4xx documented — no `401`
  despite bearer auth on every endpoint, no `404` despite taking an
  `{userId}` path parameter that can obviously not exist. (Also flagged
  mechanically: `redocly lint` — "Operation must have at least one 4XX
  response.")
- `GET /getUser` (`getUserByEmail`): has `404`, missing `401`.
- `GET /users/{userId}/notes` (`listUserNotes`): has `404`, missing `401`.
- `GET /order` (`listOrders`): has `400`, missing `401`.
- `GET /order/{orderId}` (`getOrder`): has `404`, missing `401`.
- `DELETE /order/{orderId}` (`deleteOrder`): has `404`, missing `401`.

Fix: since `security: [bearerAuth: []]` applies globally, every operation
can return `401` — add it everywhere it's missing. Add `404` to `getUser`
since it takes a path-scoped ID.

---

## 3. Conventions declared once (Rule 3)

**3.1 — Two incompatible pagination schemes for the same concept.**
Location: `GET /users` uses `page` + `page_size` (response body has flat
`page`/`page_size`/`total` fields); `GET /order` uses `limit` + `offset`
(response body has flat `limit`/`offset`, no `total` at all). Two
collection endpoints, two different parameter pairs, two different
response shapes.
Fix: pick one pair (page-based or cursor-based; if page-based, `page` +
`page_size` is already used twice by name in this spec so it's the path
of least disruption) and define it once as shared parameters, `$ref`'d
from both endpoints. Standardize the response envelope too — every list
response should have the same shape (e.g., `data` + a nested `pagination`
object), not sibling top-level fields that differ in name and presence
per endpoint.

**3.2 — `GET /users/{userId}/notes` isn't paginated at all.**
Location: `paths./users/{userId}/notes.get` — returns a bare
`type: array` with no page/cursor/limit parameters whatsoever.
Fix: apply the same pagination convention chosen in 3.1. An account's
note history is exactly the kind of collection that looks fine in testing
and becomes a timeout in production once a long-lived customer account
accumulates hundreds of notes.

---

## 4. HTTP semantics (Rule 4)

**4.1 — `POST /users` returns `200` instead of `201`, and has no
`Location` header.**
Location: `paths./users.post.responses.200`.
Fix: change to `201`, add a `Location` header pointing at
`/users/{userId}` for the new user, per Rule 4's create semantics.

**4.2 — `PUT /users/{userId}` is documented as a partial update, which
is `PATCH` semantics, not `PUT`.**
Location: `paths./users/{userId}.put` — description literally says "Only
the fields present in the request body are changed; omitted fields keep
their current values," which is merge-patch behavior, and
`UserUpdate` has no required properties, so callers cannot supply a
full resource even if they wanted PUT's actual contract.
Fix: rename the operation to `PATCH /users/{userId}` with
`application/merge-patch+json`, matching the described behavior. If a
true full-replace `PUT` is also wanted, it needs `UserUpdate`'s fields
(or a distinct schema) to be `required`, and the description needs to
actually describe replacement, not merging.

**4.3 — `DELETE /order/{orderId}` returns `200` with a body; `DELETE
/users/{userId}` returns `204` with none.** Same HTTP method, same
conceptual action, two different response shapes.
Location: `paths./order/{orderId}.delete.responses.200` vs.
`paths./users/{userId}.delete.responses.204`.
Fix: Rule 4 allows either `204` no-body or `200` with a receipt body for
deletes — but "the same choice everywhere." Pick one (this review
recommends `204`, since that's what's already used for user deletion and
requires no schema) and apply it to both.

---

## 5. Versioning (Rule 5)

**5.1 — No versioning strategy is stated anywhere.**
Location: `servers[0].url: https://api.crmly.dev` (no version segment);
`info.description` says nothing about versioning or deprecation.
Fix: state one explicitly — a URL prefix (`/v1`) is the path of least
resistance given nothing is versioned yet — plus a deprecation policy for
future breaking changes (e.g., `Deprecation` header, a stated minimum
notice period). Shipping unversioned now guarantees an uncommunicated
breaking change later.

---

## 6. Validator-clean output (Rule 7)

**6.1 — Broken reference: `OrderDetails` is `$ref`'d but never defined.**
Location: `paths./order/{orderId}.get.responses.200.content.application/json.schema`
→ `$ref: '#/components/schemas/OrderDetails'`. No `OrderDetails` schema
exists in `components.schemas` (only `Order` does).
Fix: either define `OrderDetails` (presumably `Order` plus line items,
per the operation summary "Get an order with line items") or point the
`$ref` at `Order` if no richer shape was actually intended. Confirmed via
`npx @redocly/cli lint`: `[no-unresolved-refs] Can't resolve $ref`.

**6.2 — `info.license` missing.** Minor; `redocly lint` warning. Fix:
add a `license` object (name + url/identifier) to `info`.

---

## Consistency audit (Rule 8 checklist, run in full)

| # | Check | Result |
|---|-------|--------|
| 1 | Property casing uniform | **Fail** — see 1.3 |
| 2 | All collection paths plural nouns | **Fail** — see 1.2 (`/order`) |
| 3 | Zero verbs in paths | **Fail** — see 1.1 (`/getUser`) |
| 4 | All 4xx/5xx `$ref` shared error schema, same content type | **Fail** — see 2.1, 2.2 |
| 5 | Pagination param names/types identical on every collection | **Fail** — see 3.1 |
| 6 | ID formats uniform | Pass — all `uuid` |
| 7 | Timestamps uniform (`date-time`, UTC) | Pass in format, but see 1.3 for the `createdAt`/`created_at` naming split |
| 8 | `operationId` pattern uniform | Pass — `list`/`get`/`create`/`update`/`delete` + resource, consistently, aside from the `getUserByEmail` outlier that exists only because of 1.1 |
| 9 | Same action → same status code everywhere | **Fail** — see 4.1 (create), 4.3 (delete) |
| 10 | Sort/filter conventions uniform | N/A — no sorting exists anywhere in this spec to be inconsistent; filtering is limited to the ad hoc `/getUser?email=` lookup (see 1.1) |

## Validation

`npx @redocly/cli lint spec-under-review.yaml`: **1 error** (unresolved
`$ref`, item 6.1), **2 warnings** (missing `license`, item 6.2; zero 4xx
on `getUser`, item 2.3). This is not a validator-clean spec as delivered.
