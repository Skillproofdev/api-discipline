# Ledgerline API — design notes

## Hard requirement 1: retried charge creation must never double-charge

`POST /charges` requires a mandatory `Idempotency-Key` request header
(client-generated, we recommend a UUID per logical charge attempt).

Contract, from the client's point of view:

- Send the same `Idempotency-Key` + same request body again (because the
  first attempt timed out, or the response was lost) -> you get back the
  **same charge**, same `201`, with an `Idempotent-Replay: true` response
  header. No second charge is created.
- Send the same `Idempotency-Key` with a **different** body -> `409
  Conflict` with `error.code = idempotency_key_reused`. The original charge
  is untouched. This catches a client bug (reusing a key across two
  different logical charges) instead of silently doing the wrong thing.
- Keys are scoped per merchant and remembered for 24 hours, which is long
  enough to cover retry/backoff windows well past what a sane client would
  use, then free to reuse.
- This is the same pattern used by Stripe/Adyen/etc., which the "thousands
  of requests/minute, aggressive timeout and retry" merchants integrating
  here will already have client libraries for.

I also put `Idempotency-Key` on `POST /charges/{id}/refunds`, not just
charge creation. The brief's hard requirement only names charge creation,
but refunds move money the same way and are just as likely to be retried by
a timing-out client — leaving them unprotected would just move the
double-refund bug one endpoint over. Flagging this as an addition beyond
the literal ask in case there's a reason (e.g. refunds already go through a
separate ledger with its own dedup) to scope it differently.

## Hard requirement 2: two editing styles for webhook endpoint config, both double-submit-safe

- `PUT /webhook-endpoints/{id}` — full replace. Body requires all three
  fields (`url`, `eventTypes`, `active`). This is what the "re-submit the
  whole config form" flow uses.
- `PATCH /webhook-endpoints/{id}` — partial update. Body requires at least
  one field; only fields present change. This is what "toggle just the
  active flag" or "swap just the URL" uses.

Both verbs are used with their standard HTTP idempotency semantics rather
than needing an idempotency-key mechanism: replaying the identical `PUT` or
`PATCH` body twice converges to the same state and returns `200` both
times — there's no "create a second thing" failure mode here the way there
is for `POST /charges`, because these are pure state-setting operations,
not money-moving creates. I call this out explicitly in each operation's
description since "idempotent" isn't self-evident to every API consumer.

## Hard requirement 3: over-refund is a clean 4xx, not a 500

`POST /charges/{id}/refunds` returns `422` with a dedicated
`RefundAmountError` schema when the requested amount exceeds the charge's
remaining refundable amount (`error.code =
refund_exceeds_remaining_amount`), or when the charge isn't in a refundable
state yet (`error.code = charge_not_refundable`). The error body includes
`remainingAmount` in minor units so the client can self-correct without a
second round trip to look it up. `Charge.refundedAmount` is also exposed on
the charge resource itself so clients can pre-check before calling refund
at all.

I used `422` rather than `400` because the request is well-formed
(syntactically valid JSON, valid field types) but semantically
unprocessable given the charge's current state — a distinction some
merchant integrations key their retry logic on (retry 5xx and some 429s,
never retry 4xx), so getting this "declined due to state" case out of the
5xx family matters operationally, not just stylistically.

## Hard requirement 4: rotatable signing secret, old one invalidated

`POST /webhook-endpoints/{id}/secret/rotate` generates a new secret and
invalidates the old one atomically — no gap where both are valid, no
delete/recreate of the endpoint (which would have orphaned its delivery
history and event-type subscriptions). The secret is returned in full only
in the `create` and `rotate` responses; `GET`/`PATCH`/`PUT`/list responses
never include it, only a `WebhookEndpoint` shape without the `signingSecret`
field. This mirrors how API keys/secrets are handled everywhere else
(shown once, never re-displayed) and forces the merchant to actually
capture it at rotation time.

## Other decisions

- **Money as integer minor units + separate ISO 4217 currency code**, per
  the brief's own field description — no float amounts anywhere, and no
  implicit currency.
- **Response envelope** matches the same `{ data }` / `{ data, meta }` /
  `{ error }` shape used across our other APIs so a shared client SDK can
  parse all of them uniformly.
- **Cursor pagination** on charges given "huge volumes," consistent with
  the note that offset pagination degrades at scale and drifts under
  concurrent writes — charges are written continuously in production.
- **Manual delivery resend is `202 Accepted`, not `200`**, because the
  actual re-delivery attempt happens asynchronously against the merchant's
  endpoint; the response confirms the resend was queued, not that it
  succeeded. The client polls the delivery list (or listens for the
  resulting delivery's own webhook-of-webhooks, if we build that later) to
  see the outcome.
- **IDs are typed with prefixes** (`ch_`, `re_`, `we_`, `del_`) so a
  misplaced ID is obviously wrong at a glance in logs and support tickets,
  and so path parameters can be validated by pattern before hitting the
  database.

## Open items for reviewing engineers

- No rate-limit documentation yet, despite "thousands of requests/minute"
  merchants being explicitly called out — this needs headers
  (`X-RateLimit-*` or similar) and documented limits before launch.
- Event *payload* schemas (what the webhook POST body actually contains per
  event type) aren't specified here — this deliverable only covers the
  merchant-facing management API for endpoints/deliveries, not the
  outbound webhook wire format itself. Worth a follow-up spec.
- Refund `reason` is required on every refund; the brief lists it as a
  Refund field without explicitly saying whether it's mandatory. Made it
  required since `duplicate` / `fraudulent` / `requested_by_customer` reads
  like compliance/reporting data, not an optional annotation — flag if
  that's wrong.
