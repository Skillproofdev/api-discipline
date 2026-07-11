# T6 answer key — seeded violations in `materials/t6-flawed-spec.yaml`

Pre-registered before any benchmark run. 10 seeds: 5 HTTP-semantics (S) +
5 consistency (C). Catch criteria identical to the T5 key: location +
problem class, judged by an independent verifier against this key.

## HTTP-semantics seeds

| ID | Location | Seeded violation | Catch requires |
|----|----------|------------------|----------------|
| T6-S1 | `GET /articles` (requestBody) | GET declares a request body ("complex filter expressions ... sent in the request body") — GET must not carry a body; belongs in query params or a POST search resource | Flags GET-with-request-body on `/articles` |
| T6-S2 | `POST /articles/{articleId}/content` | POST used for an explicitly idempotent full replace ("Replaces the entire content ... idempotent") — that is PUT | Flags POST-for-idempotent-replace (should be PUT) |
| T6-S3 | `POST /articles` → `201` | 201 Created without a `Location` header (sibling `POST /comments` has one) | Flags missing Location on the 201 |
| T6-S4 | `POST /comments` → `500` | 500 documented for a request-body validation failure ("missing body text, unknown article, banned words") — client mistakes are 4xx (400/422); no 4xx validation response exists on this operation | Flags 500-for-validation (should be 400/422) |
| T6-S5 | `PATCH /articles/{articleId}` | PATCH described as full replace ("Replaces the full article resource ... omitted fields are reset to defaults") — that is PUT semantics; PATCH is partial | Flags PATCH-described-as-replace |

## Consistency seeds

| ID | Checklist item | Location | Seeded violation | Catch requires |
|----|----------------|----------|------------------|----------------|
| T6-C1 | id format uniform | `Article.id` / `Author.id` (uuid) vs `Comment.id` (integer int64) | Mixed identifier formats across resources without stated reason | Flags mixed id formats (uuid vs integer) |
| T6-C2 | timestamp format uniform | `Article.created_at` / `Author.created_at` (`string date-time`) vs `Comment.created_at` (`integer`, "Unix epoch seconds") | Mixed timestamp representations | Flags epoch-int vs date-time inconsistency |
| T6-C3 | operationId pattern uniform | `GET /authors/{authorId}` → `operationId: FetchAuthor` | PascalCase + `Fetch` verb vs camelCase `list*/get*/create*` everywhere else | Flags divergent operationId `FetchAuthor` |
| T6-C4 | error content-type uniform | all 4xx/5xx under `/comments` (both operations) | Error responses use `application/json` while every other endpoint uses `application/problem+json` (same `Problem` schema) | Flags inconsistent error content type on `/comments` |
| T6-C5 | sort/filter param naming uniform | `GET /comments` params `order_by` + `direction` vs `GET /articles` param `sort` (`-field` syntax) | Two different sort conventions for the same concept | Flags divergent sort parameter convention |

## Mechanical confirmation (recorded 2026-07-10)

- `npx --yes @redocly/cli lint materials/t6-flawed-spec.yaml --format=json`
  → 0 errors, 1 warning (`info-license`, benign).
- `npx --yes @stoplight/spectral-cli lint materials/t6-flawed-spec.yaml
  --ruleset .spectral.yaml --format json` → 0 errors.
- All 10 seeds are invisible to both validators — by design; this task
  measures review judgment, not linting.

## Notes for the verifier

- Findings beyond these 10 are not penalized, but false-positive findings
  are counted and reported separately per run.
- T6-S3: do not award for a generic "add Location headers everywhere" remark
  that never points at `POST /articles`.
- T6-S4: award if the review flags either the 500-for-validation or the
  missing 4xx validation response on `POST /comments` — same underlying seed,
  one point maximum.
