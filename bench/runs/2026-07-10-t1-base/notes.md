# Trackle API — design notes

## Decisions made where the brief left gaps

- **Response envelope.** Every success response is `{ "data": ... }`
  (single resource) or `{ "data": [...], "meta": {...} }` (collections). This
  directly answers the brief's request for predictable, uniform responses so
  the web client can write one parser. Errors always come back as
  `{ "error": { "code", "message", "details? } }` regardless of endpoint or
  status code.
- **Pagination.** Cursor-based (`pageCursor` + `pageSize`, opaque cursor
  string), not offset/limit. Task lists can exceed 10,000 rows per project
  and the brief calls this out explicitly — offset pagination degrades badly
  at that scale and is unstable under concurrent writes. All three paged
  collections (projects, members, tasks) share the same `pageCursor` /
  `pageSize` / `meta.nextCursor` / `meta.hasMore` shape.
- **Filtering/sorting on tasks.** Query params `status`, `assigneeId`,
  `priority`, `dueBefore`, and a single `sort` param taking a fixed enum of
  six values (three fields x two directions). Chose one combined `sort` enum
  over separate `sortBy`/`sortDir` params — smaller surface, no invalid
  combinations to validate. `assigneeId=none` is a sentinel for "unassigned"
  since assignee is nullable and query params can't express null directly.
- **Assign/unassign/complete/reopen as dedicated action endpoints**
  (`PUT/DELETE /tasks/{id}/assignee`, `POST /tasks/{id}/complete`,
  `POST /tasks/{id}/reopen`) rather than folding them into the general
  `PATCH /tasks/{id}`. The brief calls these out as named flows, and they
  carry business rules (assignee must be a project member; complete/reopen
  are the only way to reach/leave `done`) that don't belong in a generic
  field-patch endpoint. `PATCH` explicitly excludes `assigneeId` and the
  `done` status value to keep one way to do each of these transitions.
  Assign/complete/reopen are all specified as idempotent (repeat call ->
  200, same resulting state) since these are exactly the actions a flaky
  mobile client will retry.
- **Task IDs are global, not nested under project.** `GET/PATCH/DELETE
  /tasks/{taskId}` instead of `/projects/{projectId}/tasks/{taskId}`, because
  task UUIDs are already globally unique and a flat path is one segment
  shorter for the hottest endpoints in the API. Creation and listing stay
  nested (`/projects/{projectId}/tasks`) since those operations are
  inherently scoped to a project. Comments follow the same pattern: nested
  under `/tasks/{taskId}/comments` for list/create, flat
  `/comments/{commentId}` for edit/delete.
- **Comment `editedAt` is a separate field from `updatedAt`.** `updatedAt`
  changes on any write; `editedAt` is null until the author has edited the
  body, so the client can show "(edited)" without re-deriving it from
  create-vs-update timestamps.
- **Member removal doesn't cascade.** Removing a member unassigns their
  tasks (assignee -> null) but leaves their authored tasks/comments intact,
  since those are historical record, not ownership. This is called out on
  `DELETE /projects/{projectId}/members/{memberId}`.
- **Last-owner protection.** `PATCH` on member role rejects demoting the
  last remaining owner (409), so a project can never end up ownerless via
  the API. Not stated in the brief; added because owners are the only role
  that can invite/remove/change roles, so losing the last one would brick
  project administration.
- **Auth responses.** Every endpoint documents 401 (no/invalid token) and
  403 (authenticated, not permitted — e.g. non-owner calling member-admin
  endpoints, or a caller who isn't a member of the project at all). Where a
  project isn't visible to the caller, we return 404 rather than 403 to
  avoid confirming a project ID exists to someone who isn't a member of it;
  genuine permission errors *within* a project the caller is a member of
  (e.g. viewer trying to invite) return 403.
- **Versioning.** `/v1` in the server URL, per "third parties will integrate
  against this API, so we cannot casually change it after launch" — a v1
  path segment gives us a clean place to introduce v2 later without
  breaking existing integrations.

## Open items for reviewing engineers

- Rate limiting is not specified — the brief doesn't mention it, but
  third-party integrations usually need documented limits before launch.
- Idempotency keys for `POST` (create project/task/comment) aren't included.
  Worth a follow-up if third-party clients are expected to retry creates
  over flaky connections, since retrying a plain `POST` today can create
  duplicates.
- Webhooks/events aren't in scope here since the brief only asked for the
  request/response API, but "third-party integrations" often want to be
  notified of task changes rather than polling — flagging for a v1.1
  discussion, not blocking this deliverable.
