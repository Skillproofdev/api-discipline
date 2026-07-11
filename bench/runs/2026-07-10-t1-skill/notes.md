# Notes for reviewing engineers — Trackle API v1

## Conventions (declared once, applied everywhere)

- **Path casing**: kebab-case (no multi-word path segments needed yet; convention
  stated for future endpoints). **Property casing**: snake_case.
- **Pagination**: `cursor` + `limit` (opaque cursor, default `limit=20`, max 100).
  Chosen over `page`/`per_page` because task lists run to 10,000+ rows per
  project and members lists can be in the hundreds — offset pagination would
  degrade and drift under concurrent writes; cursor pagination stays stable and
  cheap at any depth.
- **List envelope**: every collection returns `{ data: [...], pagination: {
  next_cursor, has_more } }`. Identical shape on every list endpoint, per the
  web client team's explicit ask for uniform responses.
- **Error schema**: RFC 9457 Problem Details (`application/problem+json`) on
  every documented 4xx/5xx, one shared `Error` component.
- **Versioning**: URL prefix `/v1`. Breaking changes ship as `/v2`; deprecated
  operations get a `Deprecation` header and a minimum 6-month sunset window
  before removal (third parties integrate against this, so no silent breaks).
- **Auth**: bearer token via the gateway (`bearerAuth` security scheme).
  Unauthenticated calls get `401`; authenticated-but-not-permitted calls
  (e.g. a `viewer` trying to write) get `403`.
- **IDs**: all `uuid`. **Timestamps**: all `date-time`, UTC.
- **operationId pattern**: `listX` / `getX` / `createX` / `updateX` / `deleteX`.

## Design decisions worth flagging

- **Assign/unassign and complete/reopen are not separate endpoints.** Both are
  just fields already on the task (`assignee_id`, `status`), so they're
  exposed via `PATCH /projects/{projectId}/tasks/{taskId}` with JSON Merge
  Patch semantics (RFC 7396) rather than inventing
  `POST /tasks/{id}/assign` or `POST /tasks/{id}/complete` verb endpoints.
  This keeps the surface smaller and avoids two ways to change the same
  field. If the team later needs an audit trail per assignment/completion
  event (who/when), that's a case for revisiting this as a sub-resource
  (e.g. `POST /tasks/{id}/assignments`) — flagging it now since the brief
  doesn't ask for history, just current state.
- **`assignee_id` must reference a project member** — enforced as a semantic
  (422), not structural, validation: the shape is a valid UUID, the business
  rule is what can fail.
- **Removing the last owner of a project is `409`**, not `422`: it's a
  conflict with current state (would leave the project ownerless), not a
  malformed request.
- **Comments are edit/delete by author only.** The spec documents `403` for
  non-authors attempting `PATCH`/`DELETE`; enforcement is server-side, not
  visible in the schema.
- **`PATCH` uses `application/merge-patch+json`**, not JSON Patch — simpler
  for the client team, adequate since no operation needs array-element-level
  patching. Documented in each PATCH operation's description.
- Members and tasks are always scoped under `/projects/{projectId}/...` since
  a task/member cannot exist outside a project — this also keeps
  authorization checks (project membership) natural at the routing layer.

## Validation

Ran `npx @redocly/cli lint spec.yaml`: **0 errors, 1 warning** (recommends
`license.url` or `license.identifier` in `info.license` — cosmetic, left as
a placeholder since the real license/URL isn't specified in the brief).

## Consistency audit (Rule 8, self-check)

1. Property casing uniform (snake_case) — yes.
2. All collection paths plural nouns (`/projects`, `/members`, `/tasks`,
   `/comments`) — yes.
3. Zero verbs in paths — yes.
4. All 4xx/5xx `$ref` the shared `Error` schema via `application/problem+json`
   — yes.
5. Pagination param names/types identical on every collection (`cursor`
   string, `limit` integer 1-100 default 20) — yes.
6. ID formats uniform (all `uuid`) — yes.
7. Timestamps uniform (`format: date-time`, UTC) — yes.
8. `operationId` pattern uniform — yes.
9. Same action → same status code everywhere (all creates `201` + `Location`,
   all deletes `204`, all updates `200`) — yes.
10. Sort/filter param conventions uniform (`sort` takes a key, `-` prefix for
    descending) — yes, used on `projects` and `tasks` list endpoints (the
    only two with an explicit sort requirement in the brief).
