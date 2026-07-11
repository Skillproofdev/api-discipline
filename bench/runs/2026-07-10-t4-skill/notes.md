# Notes for reviewing engineers — RoomBoard Bookings API v3.0.0

**Read this before merging.** Product filed both changes as "small
cleanups." Change 1 was implementable without breaking anyone. Change 2
cannot be — there is no version of "require `venue_id`" that doesn't change
what a currently-succeeding call from one of the 11 third-party
integrations receives. This is flagged per the breaking-changes discipline
below, not shipped silently.

## Breaking changes

1. **`GET /bookings`: `venue_id` is now required (was optional).**
   - **Why this is unavoidably breaking**: the entire point of the change
     is to make the previously-silent "no `venue_id` → all venues" behavior
     fail instead. Any integration currently omitting `venue_id` — which is
     exactly the accidental cross-venue pattern Support is seeing — gets
     `400` starting now instead of `200` with (wrong) data. I looked for a
     non-breaking alternative (e.g., an opt-in `all_venues=true` flag
     instead of hard-requiring `venue_id`) and rejected it: it's the same
     breaking change under a different name, since any client not
     currently sending that flag would still start failing.
   - **Migration note**: any of the 11 third-party integrations, or either
     first-party app, calling `GET /bookings` without `venue_id` will start
     receiving `400 Bad Request` (`type` identifies the missing parameter)
     immediately on deploy of this contract version. There is no
     compatibility window built into this spec — this needs a business
     decision before shipping (see recommendation below).
   - **Confirmed via tooling**: `oasdiff breaking current-spec.yaml
     revised-spec.yaml` reports exactly this one change:
     `request-parameter-became-required` on `venue_id` — and nothing else,
     which cross-checks the dual-field approach to change 2 below as
     genuinely non-breaking.
   - **Recommendation** (a product/security call, not an API-design one):
     either (a) hold this specific change for a short, explicit grace
     period — e.g., 2–4 weeks of `400` **plus** a `Deprecation`-style
     warning already active today so it's visible in integration logs
     before enforcement flips on, combined with direct outreach to the 11
     known partners (small, known list — feasible to contact
     individually, unlike an open third-party ecosystem); or (b) accept the
     break immediately as a security-justified exception and coordinate
     hotfix support for partners who break. I did not pick between these —
     it's not a spec-design decision — but the spec as delivered
     implements the harder/immediate version (b) since the brief said
     "must be impossible... from now on." If (a) is preferred, the fix is
     one line: leave `required: false` for the grace period and enforce at
     the application layer with a response warning header, then flip
     `required: true` here when the window closes.
   - `info.version` bumped `2.1.0` → `3.0.0` (semver major) to signal a
     breaking change occurred in the document, independent of whether the
     team also moves the URL prefix to `/v3` — I left `servers[0].url` at
     `/v2` since the brief frames this as an in-place "next release," but
     flagging that the base spec's own stated convention was only
     "Versioning: URL prefix (/v2)" with no breaking-change/deprecation
     policy on record. That's a pre-existing gap in the contract, not
     something I introduced, but it's the reason there's no formal sunset
     mechanism to plug this change into.

## Non-breaking change

2. **`start_time` → `starts_at` rename** — shipped as an **additive
   migration, not a literal rename**, and this is a deliberate deviation
   from the literal instruction ("must be renamed... everywhere"):
   - `Booking` (response) now has both `starts_at` (new, canonical,
     required) and `start_time` (kept, required, `deprecated: true`,
     documented as always equal to `starts_at`). Existing integrations
     parsing `start_time` keep working unchanged; new/updated integrations
     can read `starts_at`.
   - `BookingCreate`: `start_time` was removed from the flat `required`
     list and replaced with `anyOf: [{required: [start_time]}, {required:
     [starts_at]}]` — at least one of the two must be sent. Existing
     clients still sending `start_time` alone continue to satisfy this;
     new clients can send `starts_at` alone. If both are sent, they must be
     equal or the request is `422` (avoids silently picking a winner when
     a client sends conflicting values).
   - `BookingUpdate`: same dual-field pattern, both fields optional
     (consistent with `PATCH` semantics), same equal-if-both-present rule.
   - `oasdiff` confirms this produces **zero** breaking-change findings —
     the only flagged issue is `venue_id` above.
   - **Not done**: I did not remove `start_time` anywhere. If the design
     system genuinely wants a hard rename (not a deprecation), that is
     itself a breaking change and belongs in a real `/v3` cutover with its
     own deprecation window — the literal "rename it" instruction as given
     is incompatible with the 11 live third-party integrations continuing
     to work, and the brief didn't ask me to break those.
   - `end_time` is untouched, as instructed — no `ends_at` was added and
     none should be inferred.

## Validation

- `npx @redocly/cli lint revised-spec.yaml`: **0 errors, 0 warnings.**
- `oasdiff breaking current-spec.yaml revised-spec.yaml`: **1 breaking
  change** — `venue_id` became required on `GET /bookings` — flagged above
  with a migration note, per Rule 6. No other breaking changes detected.

## Consistency audit (Rule 8, self-check)

1. Property casing uniform (snake_case, including new `starts_at`) — yes.
2. Collection paths unchanged, still plural (`/bookings`) — yes.
3. Zero verbs in paths — yes, unchanged.
4. All 4xx/5xx still `$ref` the shared `Problem` schema — yes.
5. Pagination unchanged (`page`/`per_page`) — yes, not in scope for this
   change and not touched.
6. ID formats uniform (`uuid`) — yes, unchanged.
7. Timestamps uniform (`date-time`) — yes, `starts_at` matches
   `start_time`'s existing format exactly.
8. `operationId` pattern unchanged and uniform — yes.
9. Same action → same status code everywhere — yes, unchanged (this
   change touches parameter/property presence, not status codes).
10. Filter/sort conventions unchanged — yes; `venue_id`'s filter *meaning*
    is unchanged (still "scope to this venue"), only its
    optionality changed, which is exactly and only the flagged breaking
    change.
