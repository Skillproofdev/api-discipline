# RoomBoard Bookings API — notes on the "naming cleanup + mandatory venue scoping" change

## Both requested changes are breaking. Neither is a "small cleanup."

This API is live with two first-party mobile apps and **11 third-party
calendar integrations** we don't control the release cadence of. Both
requested changes alter the wire contract those integrations already
depend on:

1. **Renaming `start_time` -> `starts_at`.** Every existing third-party
   integration currently reads `start_time` out of `GET`/`POST`/`PATCH`
   responses and writes `start_time` into `POST`/`PATCH` request bodies. If
   we rename the field in place on the live `/v2` contract:
   - Reads break immediately: `booking.start_time` becomes `undefined` in
     every integration's response parsing until they update their code.
   - Writes fail **silently**, which is worse than an error: a client still
     sending `{"start_time": "..."}` to a JSON body that now expects
     `starts_at` doesn't get a 400 — most JSON deserializers just ignore
     unknown fields — so the booking gets created/updated with a missing or
     stale start time and no error surfaces to the caller. This is a data
     integrity risk, not just an inconvenience.
2. **Requiring `venue_id` on `GET /bookings`.** Any integration currently
   calling `GET /bookings` without `venue_id` (which the brief says is
   happening — "support keeps seeing accidental cross-venue data pulls,"
   i.e. it's a real, exercised code path today, not a hypothetical) starts
   getting `400`s the moment this ships. Every such integration's booking
   list view goes blank or errors until someone on their side adds the
   parameter — and we don't have visibility into which of the 11
   integrations do this, or a way to reach all of them before we deploy.

Both changes are backward-incompatible under the API's own stated semver
(`2.1.0`, URL-prefixed `/v2`) and by any standard definition (removing/
renaming a field, narrowing accepted input). I'm flagging this explicitly
because the request as written treats them as same-release cleanups, and
shipping them that way into `/v2` would break both the rename and the
venue-scoping fix in production for external partners with no warning.

## What I delivered instead: both changes ship as `/v3`

`revised-spec.yaml` is a new major version, `/v3`:

- `Booking.start_time` -> `starts_at` in responses, `BookingCreate`, and
  `BookingUpdate`. `end_time` is deliberately left alone, exactly as
  instructed ("do NOT rename it now").
- `GET /bookings` now requires `venue_id` (`400` if omitted, with a
  distinguishable `error.type` of `missing_venue_id` so migrating clients
  can detect this specific failure programmatically instead of just seeing
  a generic 400).
- Every changed operation's `description` is annotated `BREAKING CHANGE
  from v2` inline in the spec, not just in this notes file, so it survives
  into generated docs/SDKs.

`/v2` is **not modified** — it keeps `start_time`, keeps `venue_id`
optional, and keeps returning cross-venue results when `venue_id` is
omitted, exactly as it does today. This is a deliberate recommendation, not
an oversight: I did not touch `t4-base-spec.yaml`'s contract at all, and
`revised-spec.yaml` describes `/v3` only.

## Migration plan I'm recommending to go with this

1. **Publish `/v3` alongside `/v2`**, both live simultaneously.
2. **Deprecate `/v2`** with a stated sunset window — I'd recommend 90 days
   minimum given 11 third-party integrations need to schedule their own
   release, longer if any of the 11 are known to be slow-moving. Announce
   via whatever channel we use for partner integrations (email list, dev
   portal changelog, `Deprecation`/`Sunset` response headers on `/v2` if
   our gateway supports them — this spec doesn't currently document that
   pattern, which is itself a gap worth closing).
3. **Migrate the two first-party mobile apps first** — we control their
   release cadence, so they can move to `/v3` immediately and validate the
   new contract in production before external partners are asked to move.
4. **Give the 11 third-party integrations a migration guide**: field rename
   table (`start_time` -> `starts_at`), and the `venue_id`-required change
   with a note that any integration currently omitting it needs to add
   explicit per-venue calls (loop over venues) to reproduce their old
   all-venue view, since v3 has no all-venue list mode.
5. **Only after the sunset window closes**, retire `/v2`.

## Non-breaking alternatives I considered and rejected

- **Dual-write the rename** (add `starts_at` alongside `start_time` in
  `/v2`, keep both in sync, deprecate `start_time` in docs only): this
  would satisfy "no live break" but not "renamed... everywhere it appears"
  — the brief is explicit that the field must be renamed, not duplicated
  indefinitely, and a permanent dual-field state is exactly the
  inconsistency the design-system standardization is trying to eliminate.
  I judged a real version bump truer to the actual request than a
  permanent shim.
- **Soft-enforce venue scoping** (e.g., require `venue_id` only for new
  API keys issued after a cutoff date, or return a `299 Deprecation`
  warning header for a grace period while still honoring the old
  behavior): this is a legitimate middle path and would avoid an
  immediate `/v3` bump for this change specifically. I didn't take it
  because bundling one breaking and one non-breaking change into the same
  release adds complexity without saving much — once `/v3` exists for the
  rename, putting the venue-scoping fix there too is free and gives
  Security the hard guarantee they asked for ("impossible without an
  explicit venue") instead of a soft one.

## If product wants this in the next release without a new version

That's a real option, but it needs to be a conscious tradeoff made with
eyes open, not the default: ship into `/v2` in place, notify the 11
integration partners of a hard cutover date, and accept that any partner
who hasn't updated by that date sees broken booking lists and possibly
corrupted booking times until they do. I'd push back on this path unless
there's a business reason (e.g. a compliance deadline for the venue-scoping
fix) that outweighs the partner-breakage risk — nothing in the brief
suggests that urgency.
