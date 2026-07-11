# Notes for reviewing engineers — Stockroom API v1.5.0

Three additions, all additive. Existing 8 operations are byte-for-byte
unchanged except two new optional fields on `Product` (see below).

## Inherited conventions (Rule 6 — the existing contract's choices win)

Everything new in this diff reuses what was already in `current-spec.yaml`
rather than introducing my own defaults: kebab-case paths, snake_case
properties, `cursor`/`limit` pagination, the existing `Sort` parameter, the
existing `Problem` schema (not a renamed/rebuilt error type), UUID IDs,
`date-time` timestamps, and the spec's inline `{data, pagination}` list
envelope shape (used verbatim for the new movements list rather than
introducing a separate named envelope schema, to match the other three list
endpoints exactly).

One addition — `Idempotency-Key` — has no precedent in the existing spec.
It's new because nothing in the current contract needed retry-safety before;
flagging it as a new convention rather than silently bolting it onto only
one endpoint. It is currently used only by
`POST /product-price-updates`; I did not retrofit it onto the existing
`POST /products` per Rule 6 (don't rewrite a working spec's conventions
during an edit) — that's a separate decision for a separate change if
wanted.

## 1. Movement history — `GET /products/{productId}/movements`

Read-only, matching "movements are written by our ingestion pipeline, not
this API" — no `POST`. Paginated (`cursor`/`limit`) since big SKUs run to
tens of thousands of rows; filterable by `warehouse_id` and `type`
(`inbound`/`outbound`/`correction`); sortable via the existing `Sort`
param. `quantity_delta` is signed (negative for outbound) rather than a
separate direction field plus unsigned quantity, since that's a smaller,
already-net representation of the same fact.

## 2. Discontinue a product — `PUT /products/{productId}/discontinuation`

Modeled as a singleton sub-resource behind `PUT`, not as a `status` value
addable through `PATCH /products/{productId}`. Two reasons:

- The legal requirement is that discontinuation is **irreversible through
  the API**. If `status` were patchable, "cannot be un-discontinued"
  would only be enforced by server-side logic silently rejecting
  `status: active` — a trap for the next engineer who adds a field to
  `ProductUpdate`. Keeping it off `ProductUpdate` entirely makes the
  one-directional rule structural: there is no path in the spec that can
  set `status` back to `active`.
- "Discontinuing an already-discontinued product should be handled
  gracefully" maps directly onto `PUT`'s idempotent semantics (Rule 4):
  first call → `201`, every later call → `200` with the unchanged original
  record. This is the same operation both times, not two different
  actions returning different codes for different reasons — flagging this
  explicitly since Rule 8's audit item 9 ("same action → same status code
  everywhere") could otherwise look violated at a glance.

`Product` gained three new **optional** fields for the UI's "who/when/why":
`discontinued_at`, `discontinued_by`, `discontinued_reason` — all
`null` while `status` is `active`. Additive, so existing consumers
deserializing `Product` are unaffected (confirmed via `oasdiff`, see below).

## 3. Bulk price update — `POST /product-price-updates`

- **Partial success is not an HTTP error.** The endpoint returns `201` (or
  a replay) as long as the request is well-formed; each row's outcome is
  reported inside the response body (`PriceUpdateBatch.rows[].status` +
  `.error`). This was the one design choice most likely to be gotten wrong
  — a naive implementation would 422 the whole request because 3 of 500
  rows were bad, which is exactly what merchandising said they don't want.
- **Retry-safety** ("must not apply twice") is handled the same way as any
  other non-idempotent `POST` under Rule 4: an `Idempotency-Key` header,
  required. Reusing a key with the same body replays the original batch
  (`Idempotent-Replayed: true`, `200`-equivalent result inside a `201`
  envelope); reusing it with a different body is `409` — new `Conflict`
  response component, added to the shared responses since the existing spec
  had no conflict case yet.
- **Added `GET /product-price-updates/{batchId}`**, beyond what the brief
  asked for. Rationale: `201` responses conventionally carry a `Location`
  header per Rule 4, and a `Location` pointing at nothing retrievable is
  worse than not having one. This also gives merchandisers a way to look up
  a past upload's row results later without having kept the original
  response, and gives the idempotency mechanism a natural resource to
  reference. If this is unwanted scope, it can be dropped without touching
  anything else — it's fully independent of the `POST`.
- `unit_price_cents: minimum: 0` rejects negative prices at the schema
  level for defense in depth, but a negative-price row still shows up as a
  row-level `failed` result with `code: invalid_price`, not a `400` — a
  malicious or buggy client sending negative prices for 3 of 500 rows
  should not be able to fail the other 497.

## Validation

- `npx @redocly/cli lint revised-spec.yaml`: **0 errors, 0 warnings.**
- `oasdiff breaking current-spec.yaml revised-spec.yaml`: **no breaking
  changes** ("but the specs are different" — expected, confirms the tool
  saw the diff and still found nothing breaking).

## Breaking changes

None. Two new optional, nullable fields on `Product`; three new paths; no
removed/renamed paths, operations, or properties; no property became
required; no type/format narrowing; no status code changes on existing
operations; no auth changes.

## Consistency audit (Rule 8, self-check)

1. Property casing uniform (snake_case) — yes, including new fields.
2. All collection paths plural nouns (`movements`, `product-price-updates`)
   — yes. `discontinuation` is a deliberate singular exception: it's a
   singleton sub-resource (a product has at most one), not a collection —
   consistent with REST practice for 1:1 sub-resources (cf. `.../profile`
   style patterns), not a violation of the plural-collections rule.
3. Zero verbs in paths — yes.
4. All new 4xx/5xx `$ref` the existing shared `Problem` schema, same content
   type — yes.
5. Pagination identical on the new list endpoint — yes, reused verbatim.
6. ID formats uniform (all `uuid`, including new `discontinued_by`,
   `recorded_by`, `batchId`) — yes.
7. Timestamps uniform (`date-time`) — yes.
8. `operationId` pattern uniform — yes (`listProductMovements`,
   `discontinueProduct`, `createProductPriceUpdate`,
   `getProductPriceUpdate`, matching the existing `{verb}{Resource}`
   shape).
9. Same action → same status code everywhere, with the one explained,
   deliberate exception (discontinuation's `201`-then-`200` idempotent
   create) — yes.
10. Sort/filter conventions uniform — yes, `movements` reuses the existing
    `Sort` param and the `status`-style single-value filter pattern already
    used by `listProducts`/`listStockLevels`.
