# Stockroom API — Q3 extension notes

Version bumped `1.4.0` -> `1.5.0`. All eight existing operations and
schemas are unchanged byte-for-byte except for one intentional addition
(three new nullable fields on `Product`, see below) — no existing
integration should notice a difference except by opting into the new
fields/endpoints.

## 1. Stock movement history — `GET /products/{productId}/stock-movements`

Read-only, matching "movements are written by our ingestion pipeline, not
this API." No POST/PATCH/DELETE. Uses the spec's existing cursor
pagination (`cursor`/`limit`) and sort conventions since tens-of-thousands
of movements per SKU is exactly the case cursor pagination exists for;
default sort is newest-first. Added `warehouse_id` and `movement_type`
filters since a warehouse manager auditing one location or one movement
type is the obvious query shape, mirroring how `product_id` already
filters `/warehouses/{warehouseId}/stock-levels`.

`quantity_delta` is a signed integer (not separate `quantity` +
`direction` fields) so inbound/outbound/correction share one field and
summing a product's movements reconstructs its current stock level — one
fewer thing for the reviewing engineer to reconcile between this endpoint
and `/warehouses/{warehouseId}/stock-levels`.

## 2. Discontinue a product — `POST /products/{productId}/discontinue`

Dedicated action endpoint rather than adding `status` to
`ProductUpdate`/`PATCH`. Two reasons:

- **The transition is one-directional and legally sensitive** ("cannot be
  un-discontinued through the API"). A generic merge-patch field lets a
  client accidentally or intentionally flip it back by omission/inclusion
  logic; a dedicated endpoint is the only place that can ever set it, and
  it's structurally impossible to reverse via this API since there's no
  corresponding `undiscontinue` endpoint and `PATCH` explicitly excludes
  the field (documented on `ProductUpdate` and on the `PATCH` operation).
- **It needs its own audit fields** (`discontinued_at`, `discontinued_by`,
  `discontinued_reason`) that a generic field-level patch has no natural
  place to populate consistently.

Idempotent per the brief's "discontinuing an already-discontinued product
should be handled gracefully": a second call returns `200` with the
product's *existing* discontinuation details unchanged (first-write-wins on
`discontinued_at`/`discontinued_by`/`discontinued_reason`), not an error
and not a silently-updated timestamp. This matters because "graceful" is
ambiguous between "no-op" and "re-apply" — I chose no-op so the audit trail
(who/when/why it was *actually* discontinued) stays accurate even if the
discontinue call is retried by a flaky client days later with a different
reason string.

Used `200` rather than `204` on this action since the response body (the
product, with its new discontinuation fields) is the useful thing a caller
wants back, consistent with how `PATCH /products/{productId}` already
returns `200` + body for an update.

## 3. Bulk price update — `POST /price-updates`

Modeled as creating a batch resource (`price-updates` collection, plural
kebab-case per the spec's own convention) rather than a verb-shaped
endpoint like `/products/bulk-price-update`, so it stays consistent with
every other write in this API being a POST-to-collection, and so a batch
can be looked up again later (`GET /price-updates/{priceUpdateId}`) the
same way any other created resource can.

- **Partial success**: the response is always `200` for a well-formed
  request (right shape, <=500 rows), with a `results` array giving each
  row's `status` (`succeeded`/`failed`) and, for failures, a machine
  code (`product_not_found` / `invalid_price`) plus a `detail` string. Row
  failures do not fail the batch or roll back the succeeded rows — this is
  the "3 of 500 rows are bad... while the rest apply" requirement,
  literally. Whole-request malformation (empty `items`, >500 items, wrong
  field types) is instead a `400` `problem+json`, which lets a client
  distinguish "your upload tool sent garbage" from "your data had a few bad
  rows" without inspecting response bodies differently for the same status
  code.
- **Retry safety**: `POST /price-updates` requires the same
  `Idempotency-Key` header pattern I'd use for any money/state-changing
  bulk write — same key + same body replays the stored result
  (`Idempotent-Replay: true`) instead of reapplying prices a second time;
  same key + different body is `409`. This directly satisfies "the whole
  upload must not apply twice if the upload tool retries." I picked this
  over a client-supplied `batch_id` in the request body because it keeps
  retry semantics uniform with how the rest of the API would do
  idempotent writes, rather than inventing a bespoke mechanism just for
  this endpoint.
- **Synchronous processing**: 500 rows of price validation/write is small
  enough to do inline, so the response carries the final per-row results
  directly — no polling loop for the merchandiser's upload tool to
  implement. `GET /price-updates/{priceUpdateId}` exists only so a batch
  can be looked up again after the fact (e.g. from an audit log), not
  because processing is async.

## Pre-existing gaps, left alone (out of scope for this change)

The linter (`.spectral.yaml`) flags 8 pre-existing warnings on the
untouched original operations (missing `operationId`-level descriptions,
missing `info.contact`) — these existed in `current-spec.yaml` before this
change and I left them as-is rather than drive-by-fixing unrelated parts of
a live contract in the same diff as three new capabilities. Flagging in
case the reviewing engineers want a separate docs-cleanup pass.

`ProductUpdate` still has no way to change `sku` or `currency` — unchanged
from the original contract, not something this change request touched.
