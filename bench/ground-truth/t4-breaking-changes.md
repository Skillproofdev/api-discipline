# T4 ground truth — breaking changes implied by the brief

Pre-registered before any benchmark run. The brief
(`materials/t4-brief.md`) asks for two "small cleanups" against
`materials/t4-base-spec.yaml`. Both are breaking. An arm PASSES a change by
either (a) shipping it flagged as breaking with a migration note, or
(b) proposing a non-breaking alternative. Shipping it silently = one
undisclosed breaking change (metric 5).

## The two underlying breaking changes

| ID | Change | Why breaking | Acceptable non-breaking alternatives (examples) |
|----|--------|--------------|------------------------------------------------|
| T4-B1 | Rename `start_time` → `starts_at` on `Booking`, `BookingCreate`, `BookingUpdate` | Removes a required response property every reader depends on; removes/renames a request property writers send (unknown `start_time` would be ignored or rejected; required `starts_at` missing) | Additive alias: add `starts_at`, keep `start_time` deprecated (with `deprecated: true` + sunset note); or defer rename to /v3 |
| T4-B2 | `venue_id` query param on `GET /bookings` becomes required | Existing callers that omit it break with 400 | Keep optional + server-side default/scoping; deprecation window with `Deprecation` header before enforcement; new required-scope endpoint alongside |

Disclosure of B1 must acknowledge BOTH directions matter (responses and
request payloads) or at minimum state the rename is breaking for existing
clients; disclosure of B2 must state existing callers without `venue_id`
will start failing.

## Canonical oasdiff output (naive implementation, recorded 2026-07-10)

A naive revision (rename applied everywhere + `required: true` on
`venue_id`) was produced and diffed:

```
oasdiff breaking materials/t4-base-spec.yaml <revision>.yaml
```

oasdiff v1.23.0 reports **8 entries (6 error, 2 warning)**, mapping to the
two seeds:

| oasdiff check id | API | Maps to |
|---|---|---|
| `request-parameter-became-required` (`venue_id`) | GET /bookings | T4-B2 |
| `response-required-property-removed` (`start_time`) | GET /bookings 200 | T4-B1 |
| `new-required-request-property` (`starts_at`) | POST /bookings | T4-B1 |
| `response-required-property-removed` (`start_time`) | POST /bookings 201 | T4-B1 |
| `response-required-property-removed` (`start_time`) | GET /bookings/{bookingId} 200 | T4-B1 |
| `response-required-property-removed` (`start_time`) | PATCH /bookings/{bookingId} 200 | T4-B1 |
| `request-property-removed` (warning) | POST /bookings | T4-B1 |
| `request-property-removed` (warning) | PATCH /bookings/{bookingId} | T4-B1 |

## Scoring rule (metric 5)

1. Run `oasdiff breaking materials/t4-base-spec.yaml <arm-revision>.yaml`
   (use `-f json` for machine counting; plain output for the report).
2. Group entries by underlying change (B1/B2 above; any additional breaking
   entries the arm itself introduced form their own groups).
3. A group is DISCLOSED if the arm's rationale/migration note names that
   change as breaking (or the arm redesigned it away so oasdiff shows
   nothing). Count of undisclosed groups = metric 5. Skill target: 0.
4. If the arm delivers only a textual diff/fragment, reconstruct the full
   revised YAML faithfully before diffing; note the reconstruction in the
   run log.
