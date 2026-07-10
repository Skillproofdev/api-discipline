---
name: api-discipline
description: Use when the user asks to design an API, add or extend an endpoint, review an OpenAPI/Swagger spec, or asks REST convention questions (naming, versioning, pagination, status codes, PUT vs PATCH). Enforces design discipline on any REST/HTTP API work: consistent resource naming, an error model on every endpoint, conventions declared once and reused, correct HTTP semantics, breaking-change checks on edits, and validator-clean OpenAPI 3.1 output with a cross-endpoint consistency audit before delivery. Do not trigger for GraphQL-only work, client SDK/code generation from an existing spec, or API security testing.
---

# API Discipline

An API that works and an API that stays consistent under change are different products. These
rules force the second one — on greenfield designs, single-endpoint additions, and spec reviews
alike. They are hard rules, not style suggestions: each states what to do, what to reject, and
how the output is checked before it ships.

## Rule 1 — Resources are plural nouns; paths carry no verbs

- Collections are plural nouns: `/users`, `/orders`, `/orders/{orderId}/items`. Never
  `/user`, never `/getUser`, never `/users/create`.
- Actions that don't map to CRUD become sub-resources or state fields, not verbs:
  `POST /orders/{id}/cancellations` or `PATCH /orders/{id}` with `status: cancelled` —
  not `POST /cancelOrder`.
- Pick ONE casing for paths (kebab-case recommended) and ONE for properties
  (snake_case or camelCase), state the choice, apply it everywhere. Mixed casing anywhere
  in the spec is a defect, not a nit.
- Path parameters are named after their resource: `{orderId}`, not `{id}` twice in one path.

## Rule 2 — Every endpoint declares its error model

No endpoint ships with only its happy path.

- Define ONE shared error schema — RFC 9457 Problem Details (`application/problem+json`)
  unless the project already standardized on another — as a component, and `$ref` it from
  every documented 4xx/5xx. Never invent per-endpoint error shapes.
- Every operation documents the errors it can actually return: at minimum 400/422 where input
  is validated, 401/403 where auth exists, 404 for `{id}` lookups, 409 where conflicts are
  possible, 429 where rate limits apply. A validation failure is 4xx — never document 500
  for a client mistake.
- Error responses that exist in prose but not in the spec don't exist. Put them in the spec.

## Rule 3 — Conventions are declared once and reused

Pagination, filtering, and sorting are spec-level decisions, not per-endpoint improvisation.

- Define pagination parameters (e.g. `cursor` + `limit`, or `page` + `per_page` — pick one
  pair) as shared components; every collection endpoint `$ref`s them. Two endpoints with
  `limit` and `page_size` for the same concept is a defect.
- Every collection endpoint is paginated. An unpaginated list is a production incident
  on a delay timer.
- Filtering and sorting follow one declared pattern (e.g. `?status=active&sort=-created_at`)
  reused everywhere. The response envelope for lists is one shared shape (`data` +
  pagination object), not per-endpoint variations.
- When designing from scratch, emit the conventions table (Rule 9) BEFORE the first endpoint —
  endpoints then conform to it, not the other way around.

## Rule 4 — HTTP semantics are law, not vibes

- `GET` is safe and idempotent, never has a request body, never mutates.
- `POST` creates or triggers non-idempotent work → `201` + `Location` header on create
  (`202` for async jobs). If the caller must be able to retry safely, say how
  (idempotency key) — don't pretend POST is idempotent.
- `PUT` replaces the full resource and is idempotent. Using PUT for a partial update is a
  defect; that's `PATCH`.
- `PATCH` applies a partial update. Document the patch semantics (merge-patch vs JSON Patch).
- `DELETE` is idempotent → `204` with no body (or `200` with a job/receipt body — but the
  same choice everywhere).
- Status codes match meaning: `200` read/update, `201` created, `204` no content, `409`
  conflict, `412` failed precondition, `422` semantic validation failure. Never `200` with
  an error payload.

## Rule 5 — Versioning is stated, never implied

- Every spec states its versioning strategy explicitly (URL prefix `/v1/`, header, or media
  type — with the reason) and its deprecation policy, even if the answer is "v1, breaking
  changes get v2, deprecated things get a `Deprecation` header and 6 months".
- No public surface ships unversioned. "We'll add versioning later" means a breaking change
  later — see Rule 6.

## Rule 6 — Edits are diffed for breaking changes

When modifying an existing spec (or adding endpoints to one), you are editing a contract
other people already depend on.

- Before delivering, enumerate every breaking change in the edit: removed/renamed path,
  operation, or property; a property or parameter becoming required; a type or format change;
  a narrowed enum; a tightened maximum; a changed status code; a changed auth requirement.
- Each breaking change is either (a) redesigned away (additive alternative), or (b) shipped
  **flagged**, with a one-line migration note. Never silently.
- Where tooling is available, back the manual pass mechanically:
  `oasdiff breaking old.yaml new.yaml`. If you can't run it, say so and rely on the
  enumerated checklist above.
- New endpoints added to an existing spec inherit ITS conventions (casing, error schema,
  pagination params) even when they conflict with your defaults — consistency with the
  existing contract beats your preferences. Flag the divergence once; don't fork the style.

## Rule 7 — Output is valid OpenAPI 3.1, validator-clean

- Any spec or spec fragment you deliver must be syntactically valid OpenAPI 3.1: every `$ref`
  resolves, every response has a description, schemas are well-formed.
- Validate before delivering: `npx @redocly/cli lint spec.yaml` (or
  `npx @stoplight/spectral-cli lint spec.yaml`). Zero errors is the bar; remaining warnings
  are listed in the delivery, not hidden.
- If no validator can run in the environment, say so explicitly and run the self-check:
  all `$ref`s resolve · unique operationIds · every path parameter declared · every response
  has description + content type · no duplicate paths · `openapi: 3.1.x` header present.
- Never deliver a spec you know wouldn't parse "to save time". An invalid spec is not a draft,
  it's a defect.

## Rule 8 — Consistency audit before delivery

Rules 1–7 govern writing; this one catches drift. Before delivering ANY spec, diff, or review,
run this cross-endpoint pass and fix or flag every hit:

1. Property casing uniform across all schemas?
2. All collection paths plural nouns?
3. Zero verbs in paths?
4. All 4xx/5xx `$ref` the shared error schema, same content type?
5. Pagination param names and types identical on every collection?
6. ID formats uniform (all uuid, or all int64 — not mixed without reason)?
7. Timestamps uniform (`format: date-time`, UTC)?
8. operationId pattern uniform (`listUsers`/`getUser`/`createUser`…)?
9. Same action → same status code everywhere (all creates 201, all deletes 204)?
10. Sort/filter param conventions uniform?

For reviews, this checklist IS the review skeleton: report violations per item with
path + fix. Skipping the audit because "it's just one endpoint" is how specs rot — one
endpoint is exactly when drift starts.

## Rule 9 — Output contract

Every deliverable has the same shape:

- **The artifact**: full OpenAPI 3.1 YAML for greenfield; a diff (changed paths/schemas only,
  clearly marked) for edits; a findings list (checklist item → location → fix) for reviews.
- **One paragraph of rationale** — the decisions that would surprise a reviewer (why cursor
  pagination, why 409 here, why this isn't breaking). One paragraph, not an essay.
- **Conventions table** (greenfield only): casing, pagination pair, error schema, versioning,
  auth scheme, timestamp/id formats — declared before endpoints, per Rule 3.
- **Breaking-changes block** (edits only): the Rule 6 enumeration, or the line
  "Breaking changes: none".
- **Validation line**: the validator command run and its result, or the explicit fallback
  statement (Rule 7).

## Do not

- Trigger for GraphQL-only schema work, client SDK/type codegen from an existing spec, or
  API pentesting/fuzzing — different disciplines.
- Rewrite a working spec's existing conventions to match these rules during a small edit —
  Rule 6 outranks Rules 1–3 on existing contracts. Flag, don't fork.
- Pad reviews with style opinions beyond the checklist; a finding without a location and a fix
  is noise.
- Sacrifice correctness for ceremony: if the user asks for a quick sketch, the sketch still
  gets valid YAML and the audit — it skips the essay, not the checks.
