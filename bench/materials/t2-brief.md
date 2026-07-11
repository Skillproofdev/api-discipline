# Product brief: Ledgerline — payments API (v1)

Ledgerline processes card payments for online merchants. We are designing the
public merchant-facing HTTP API. Merchants integrate server-to-server; several
will be high-volume (thousands of requests/minute) with aggressive client-side
timeout and retry logic.

## Domain

- **Charges.** A merchant charges a customer's stored payment method: amount
  (integer minor units), currency, payment method reference, optional
  free-text statement descriptor, status (`pending`, `succeeded`, `failed`),
  failure reason when failed, timestamps. Merchants list charges (huge
  volumes — filtering by status and date range, sorting by creation time) and
  fetch a single charge.
- **Refunds.** A charge can be refunded, fully or partially, possibly multiple
  times up to the charged amount. A refund has amount, status
  (`pending`, `succeeded`, `failed`), reason (`duplicate`,
  `fraudulent`, `requested_by_customer`), timestamps. Merchants list refunds
  for a charge and fetch a single refund.
- **Webhook endpoints.** A merchant registers HTTPS URLs to receive event
  notifications (charge succeeded, refund succeeded, ...). An endpoint config
  has: URL, list of subscribed event types, an active/disabled flag, a signing
  secret we generate (shown in full only when it is created or rotated).
- **Deliveries.** For each event sent to each endpoint we record delivery
  attempts: response code, timing, success/failure. Failed deliveries are
  retried automatically with backoff for 24h. Merchants need to (a) list
  deliveries for an endpoint (filter: failed only), and (b) manually trigger a
  re-send of a specific failed delivery after they fix their receiver.

## Hard requirements from engineering

1. **Double-charge is an existential bug.** Merchant clients WILL time out and
   retry charge creation with identical payloads. The API contract must make
   retried creation safe — a retry must never produce a second charge. Spell
   out exactly how the API achieves this from the client's point of view.
2. Merchants manage endpoint configs two ways in their dashboards: sometimes
   they re-submit the whole config form (URL + event types + active flag),
   sometimes they toggle just the active flag or swap just the URL. The API
   should support both editing styles — and behave predictably if the same
   config edit is submitted twice.
3. Refunding more than the remaining un-refunded amount must be a clean,
   machine-readable client error, not a 500.
4. The signing secret must be rotatable without deleting/recreating the
   endpoint (and rotation must invalidate the old secret).

## Deliverable

The API design as an OpenAPI 3.1 YAML document, plus whatever short notes you
think the reviewing engineers need.
