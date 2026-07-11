# Change request: Stockroom API — three new capabilities

The attached `t3-base-spec.yaml` is the current production contract for the
Stockroom inventory API (8 operations, live integrations). Product wants
three additions for Q3. Extend the spec.

## 1. Stock movement history

Warehouse managers need an audit trail per product: every stock movement
(inbound delivery, outbound shipment, manual correction) with quantity delta,
the warehouse it happened in, an optional note, who recorded it, and when.
Movements are written by our ingestion pipeline (not this API) — the API only
needs to expose reading them. Big SKUs accumulate tens of thousands of
movements.

## 2. Discontinue a product

Purchasing needs to discontinue a product: it stops being orderable, keeps
its history, and cannot be un-discontinued through the API (legal requirement
— reinstatement goes through a manual process). Discontinuing an
already-discontinued product should be handled gracefully. The UI shows who
discontinued it, when, and an optional reason.

## 3. Bulk price update

Merchandising uploads seasonal price changes for up to 500 products at once
(product id + new unit price each). Partial success matters: if 3 of 500
rows are bad (unknown product, negative price), the merchandisers want to
know exactly which rows failed and why, while the rest apply. The whole
upload must not apply twice if the upload tool retries.

## Deliverable

The updated API contract (however you think a change to an existing contract
is best delivered), plus whatever short notes the reviewing engineers need.
Existing integrations must keep working.
