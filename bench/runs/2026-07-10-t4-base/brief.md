# Change request: RoomBoard Bookings API — naming cleanup + mandatory venue scoping

The attached `t4-base-spec.yaml` is the current production contract for the
RoomBoard Bookings API (v2, live: two first-party mobile apps and 11
third-party calendar integrations).

Two changes came out of the platform review. Product considers both "small
cleanups" and wants them in the next release:

## 1. Rename `start_time` to `starts_at`

The design-system team standardized on `*_at` suffixes for all timestamps
across RoomBoard products. The `Booking` object's `start_time` field must be
renamed to `starts_at` everywhere it appears (responses, create payload,
update payload). (`end_time` will follow in a later ticket — do NOT rename it
now.)

## 2. Require venue scoping on booking lists

Support keeps seeing accidental cross-venue data pulls: integrations call
`GET /bookings` without `venue_id` and receive every venue's bookings for the
tenant. Security wants listing to be impossible without an explicit venue:
the `venue_id` query parameter on `GET /bookings` must be required from now
on.

## Deliverable

The updated API contract (however you think a change to an existing contract
is best delivered), plus whatever short notes the reviewing engineers need.
