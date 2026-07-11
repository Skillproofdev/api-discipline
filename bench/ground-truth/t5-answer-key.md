# T5 answer key — seeded violations in `materials/t5-flawed-spec.yaml`

Pre-registered before any benchmark run. 12 seeds. A review CATCHES a seed
when it identifies (a) the location (path/method or schema — paraphrase ok)
and (b) the problem class (paraphrase ok). Naming the right endpoint with the
wrong diagnosis, or the right diagnosis with no location, is NOT a catch.
Judged by an independent verifier agent against this key.

| ID | Class | Location | Seeded violation | Catch requires |
|----|-------|----------|------------------|----------------|
| T5-V1 | naming | `GET /getUser` | Verb in path; lookup modeled as RPC-style verb endpoint instead of a resource (e.g. `GET /users?email=` or `GET /users/{id}`) | Flags `/getUser` as verb/non-resource path |
| T5-V2 | naming | `/order`, `/order/{orderId}` | Singular collection noun (`/order` vs `/users` plural elsewhere) | Flags singular `/order` (should be `/orders`) |
| T5-V3 | naming | `components/schemas/User` (also `UserCreate`, `UserUpdate`) | Mixed property casing in one schema: `firstName`, `createdAt` (camelCase) vs `last_name`, `phone_number` (snake_case) | Flags mixed camel/snake property naming in User schema(s) |
| T5-V4 | error-model | `GET /users/{userId}` | Only 200 documented — no 4xx at all; an `{id}` lookup without 404 (and no 401 despite global auth) | Flags missing error responses / missing 404 on this endpoint |
| T5-V5 | error-model | `POST /users` → `200` | Create returns 200; no 201 (and no Location) | Flags 200-on-create (should be 201) |
| T5-V6 | error-model | `POST /users` → `400` response schema | Ad-hoc inline error shape `{error_code, msg, field_errors}` while every other endpoint `$ref`s the shared `Error` schema `{code, message}` | Flags divergent/ad-hoc error shape on this response |
| T5-V7 | pagination | `GET /users/{userId}/notes` | Collection endpoint with no pagination parameters, returns bare unbounded array | Flags unpaginated collection |
| T5-V8 | pagination | `GET /order` vs `GET /users` | Divergent pagination param names for the same concept: `limit`+`offset` here vs `page`+`page_size` on `/users` | Flags inconsistent pagination params across collections |
| T5-V9 | semantics | `PUT /users/{userId}` | PUT documented as partial update ("Only the fields present in the request body are changed") — that is PATCH semantics | Flags PUT-for-partial-update (should be PATCH, or PUT must replace) |
| T5-V10 | semantics | `DELETE /order/{orderId}` → `200` + body | Returns 200 with `{deleted, order}` body while sibling `DELETE /users/{userId}` returns 204 no-content | Flags inconsistent DELETE status/body convention |
| T5-V11 | versioning | global (servers URL `https://api.crmly.dev`, info.description) | No versioning strategy anywhere: no `/v1` prefix, no header/media-type versioning, no policy stated | Flags absent API versioning strategy |
| T5-V12 | validity | `GET /order/{orderId}` → `200` schema `$ref` | `$ref: '#/components/schemas/OrderDetails'` — component does not exist (unresolvable) | Flags the broken/unresolvable `$ref` |

## Mechanical confirmation (recorded 2026-07-10)

- `npx --yes @redocly/cli lint materials/t5-flawed-spec.yaml --format=json`
  → 1 error: `no-unresolved-refs` at
  `#/paths/~1order~1{orderId}/get/responses/200/...` (= T5-V12);
  warning `operation-4xx-response` at `GET /users/{userId}` (= T5-V4).
- `npx --yes @stoplight/spectral-cli lint materials/t5-flawed-spec.yaml
  --ruleset .spectral.yaml --format json`
  → 1 error: `invalid-ref` `#/components/schemas/OrderDetails` (= T5-V12).
- All other seeds are semantic and invisible to validators — by design.

## Notes for the verifier

- Findings beyond these 12 are not penalized, but false-positive findings
  (claims of a defect that does not exist in the file) are counted and
  reported separately per run.
- T5-V5 and T5-V6 are on the same operation; award them independently.
- Do not award T5-V11 for a remark that `info.version: 1.0.0` exists — the
  seed is the missing API versioning strategy, not the document version.
