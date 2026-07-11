# Notes for reviewing engineers — Ledgerline API v1

## Conventions (declared once, applied everywhere)

- **Path casing**: kebab-case (`webhook-endpoints`, `secret-rotations`).
  **Property casing**: snake_case.
- **Pagination**: `cursor` + `limit` (default 20, max 100) — charge volumes
  are described as "huge," so offset pagination was rejected up front.
- **List envelope**: `{ data: [...], pagination: { next_cursor, has_more } }`
  on every collection.
- **Error schema**: RFC 9457 Problem Details (`application/problem+json`)
  everywhere, one shared `Error` component, extended with two
  payments-specific fields (`available_amount`, field-level `errors`) rather
  than per-endpoint custom shapes.
- **Versioning**: URL prefix `/v1`; breaking changes get `/v2`, deprecated
  operations get a `Deprecation` header, minimum 6-month sunset.
- **Auth**: bearer token (merchant secret key).
- **IDs**: opaque, prefixed strings (`ch_`, `re_`, `we_`, `dl_`) rather than
  bare UUIDs — a deliberate, uniform choice (Rule 8 requires one ID format,
  not "uuid or int64 specifically"); prefixes make IDs self-describing in
  logs and support tickets, which matters a lot for a payments API. Every
  resource follows the same `{prefix}_{opaque}` shape.
- **Timestamps**: `date-time`, UTC, on every resource.
- **`status` vocabulary reused**: charges, refunds, and deliveries all use
  the same three-value shape `pending` / `succeeded` / `failed` rather than
  each inventing its own state names.

## How the API prevents double-charging (hard requirement 1)

`POST /charges` requires an `Idempotency-Key` header, generated once by the
client per logical charge attempt and resent unchanged on every retry of
that same attempt:

- Same key + same request body → the original charge is returned (`201`,
  `Idempotent-Replayed: true` header), **no new charge is created**, even if
  the original charge already succeeded, failed, or is still pending.
- Same key + a **different** body → `409`, forcing the client to notice the
  mismatch rather than silently charging with mixed-up parameters.
- Keys are merchant-scoped and expire after 24h (documented on the
  `IdempotencyKeyRequired` parameter).

I generalized the same mechanism to `POST /charges/{chargeId}/refunds`
(hard requirement 1 only names charges, but a refund is also a non-idempotent
money-movement POST subject to the same client timeout/retry behavior —
leaving it unprotected would just relocate the existential bug one endpoint
over). Flagging this generalization explicitly since the brief didn't ask
for it by name.

## Endpoint config: two edit styles (hard requirement 2)

- **Full re-submit** → `PUT /webhook-endpoints/{endpointId}` (all three
  fields required, full replace).
- **Single-field toggle** → `PATCH /webhook-endpoints/{endpointId}`
  (`application/merge-patch+json`, partial).

Neither needs an idempotency key: both are HTTP-idempotent by definition —
submitting the same `PUT` or `PATCH` body twice leaves the endpoint in
exactly the same state both times, unlike `POST /charges` which creates a
new resource on every call unless deduplicated explicitly. This is Rule 4
(PUT/PATCH semantics), not the idempotency-key mechanism, doing the work.

## Refund over-amount is a clean 4xx (hard requirement 3)

`POST /charges/{chargeId}/refunds` returns `422` (not `500`, not `409`) when
`amount` exceeds `charge.amount - charge.refunded_amount`. The response
carries a specific `type` URI
(`.../refund-amount-exceeds-available`) and an `available_amount` extension
field with the current refundable balance, so the client can correct and
retry in one round trip without re-parsing prose.

## Secret rotation (hard requirement 4)

`POST /webhook-endpoints/{endpointId}/secret-rotations` — a sub-resource
creation, not a `POST .../rotateSecret` verb, and not a field on `PATCH`
(rotation isn't a value the client can propose; the server generates it).
`201` returns the endpoint with the new secret shown in full, one time,
exactly like on initial creation. The old secret stops verifying signatures
immediately. No delete/recreate of the endpoint involved, and
`event_types`/`url`/`active`/`id` are unchanged by rotation.

## Other decisions worth flagging

- **Delivery retry is async** (`POST .../deliveries/{deliveryId}/retries` →
  `202`, not `201`/`200`): actually sending the webhook takes real network
  time and the automatic 24h backoff process already runs asynchronously, so
  keeping manual retries consistent with that (rather than blocking the
  request) avoids a special case. Returns `409` if the target delivery isn't
  currently `failed`.
- **Refunds and deliveries are nested resources**
  (`/charges/{chargeId}/refunds/...`,
  `/webhook-endpoints/{endpointId}/deliveries/...`) since neither exists
  independent of its parent.
- **`secret` is only ever present in the response body of create and
  rotate** — omitted from `GET`/list responses entirely (not masked, not
  present as `null`) via a separate `WebhookEndpointWithSecret` schema used
  only on those two operations.

## Validation

Ran `npx @redocly/cli lint spec.yaml`: **0 errors, 1 warning** (recommends
`license.url`/`license.identifier` — cosmetic, no real license URL given in
the brief).

## Consistency audit (Rule 8, self-check)

1. Property casing uniform (snake_case) — yes.
2. All collection paths plural nouns (`charges`, `refunds`,
   `webhook-endpoints`, `deliveries`, `secret-rotations`, `retries`) — yes.
3. Zero verbs in paths — yes (`secret-rotations` and `retries` are
   sub-resource nouns, not verbs, per Rule 1).
4. All 4xx/5xx `$ref` the shared `Error` schema, same content type
   (`application/problem+json`) — yes.
5. Pagination param names/types identical on every collection — yes.
6. ID formats uniform (all prefixed opaque strings) — yes.
7. Timestamps uniform (`date-time`, UTC) — yes.
8. `operationId` pattern uniform (`listX`/`getX`/`createX`/`updateX`/
   `deleteX`, plus `replaceX` for the one `PUT`) — yes.
9. Same action → same status code everywhere (all creates `201` except the
   intentionally-async retry at `202`, all deletes `204`, all full/partial
   updates `200`) — yes, and the one exception is explained above.
10. Sort/filter param conventions uniform (`status` filter reused verbatim
    across charges and deliveries; `sort` takes a bare key or `-`-prefixed
    key) — yes.
