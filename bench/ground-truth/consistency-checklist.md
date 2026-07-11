# 10-point cross-endpoint consistency checklist (metric 3)

Fixed before any benchmark run. Applied to every produced spec (T1-T4
outputs; for T3/T4 the arm's ADDED/CHANGED surface is checked against the
base spec's conventions). Each item violated at least once anywhere in the
spec = 1 point. Score 0-10 per spec; lower is better.

Scoring is two-pass: a scripted pass where mechanical (below), then an
independent verifier agent confirms every point with a concrete location.
A point requires a named location; "feels inconsistent" scores nothing.

| # | Check | Violated when | Mechanically checkable? |
|---|-------|---------------|-------------------------|
| 1 | Property-name casing uniform | Any two schema property names use different casing conventions (e.g. `firstName` and `last_name`) anywhere in the spec | Yes — extract all property keys, classify camel/snake/kebab, >1 class = hit |
| 2 | Path segments are plural nouns | Any collection path segment is singular (`/order`) while the spec's convention (or REST convention) is plural; sub-resource collections included | Partly — needs noun judgment; verifier decides |
| 3 | No verbs in paths | Any path segment is a verb or verb-phrase (`/getUser`, `/create`, `/search` as RPC) — state-transition sub-resources modeled as nouns (`/cancellations`) do NOT count | Partly — verb list heuristic + verifier |
| 4 | All 4xx/5xx `$ref` one shared error schema, same content type | Any documented 4xx/5xx uses an inline/ad-hoc schema, a second error schema, or a divergent content type | Yes — walk all non-2xx responses, compare schema `$ref` targets and content types |
| 5 | Pagination param names + types identical across all collections | Two collection endpoints use different names/types for the same pagination concept (`limit` vs `page_size`), or any collection list is unpaginated | Yes — collect list-endpoint query params |
| 6 | ID format uniform | Mixed id types/formats across resources (uuid vs int64) without a stated reason | Yes — collect `id`/`*_id` property schemas |
| 7 | Timestamp format uniform | Any timestamp not `type: string, format: date-time`, or mixed representations (epoch ints, plain strings) | Yes — collect `*_at`/`*_time`/date-ish properties |
| 8 | operationId pattern uniform | operationIds mix naming patterns (`listUsers` vs `FetchAuthor` vs `get_user`) or are missing on some operations | Yes — regex-classify all operationIds |
| 9 | Error content-type / envelope uniform | Error responses served under different media types (`application/problem+json` here, `application/json` there) or different envelope shapes | Yes — same walk as #4 |
| 10 | Sort/filter param naming uniform | Two endpoints expose the same sort/filter concept under different conventions (`sort=-created_at` vs `order_by`+`direction`) | Yes — collect sort/filter-named query params |

Notes:
- #4 and #9 overlap by construction (shared-schema vs content-type); a single
  divergent response can score both only if it violates both dimensions
  (different schema AND different media type).
- For T3/T4 (extensions), a point is also scored when the added surface
  diverges from the base spec's established convention (e.g. adds `page`
  pagination to the cursor-based Stockroom API), even if the addition is
  internally consistent.
- The scripted pass lives in `bench/scoring.md` (section "Metric 3"); where
  script and verifier disagree, the verifier's located finding wins and the
  disagreement is logged.
