# HTTP-semantics error checklist (metric 4)

Fixed before any benchmark run. Applied to T1-T4 produced specs and to the
T6 review output (does the review's own proposed fixes respect these rules).
Each occurrence counts 1 (not capped per class). Lower is better.

| # | Error | Detected when |
|---|-------|---------------|
| 1 | POST-create without 201 | A create operation documents 200 (or other) as its success code instead of 201 (202 acceptable only for an explicitly async job) |
| 2 | 201 without Location | Any 201 response lacking a `Location` header (exception: batch/bulk creates returning a result list, if stated) |
| 3 | DELETE with 200 + body | A DELETE returning 200 with a resource body when it is not returning a job/receipt — or inconsistent with sibling DELETEs in the same spec |
| 4 | PUT for partial update | PUT whose description/schema implies partial semantics (optional-everything body described as "only provided fields change") |
| 5 | PATCH described as replace | PATCH documented with full-replacement semantics |
| 6 | GET or DELETE with request body | Any `requestBody` on a GET or DELETE operation |
| 7 | Non-idempotent PUT | PUT documented as producing different results on repeat (e.g. appends, increments) |
| 8 | Missing 409/412 where the prompt implies concurrent edits | T1: task assign/complete flows and member role changes imply conflicts; T2: refund-over-remaining implies 409/422; no conflict/precondition response documented on the relevant operations |
| 9 | Retry-unsafe create where the prompt demands idempotency (T2 only) | Charge creation has no documented idempotency mechanism (idempotency key header/param or equivalent) making client retries safe, despite the brief's hard requirement 1. Also scored if webhook-config "submit twice" behavior (brief req. 2) is left undefined |

Scoring procedure:
1. Scripted screen where mechanical (#1, #2, #3, #6 are walkable from the
   YAML; see `bench/scoring.md`).
2. Independent verifier agent confirms each hit with operation + line and
   evaluates the judgment-based items (#4, #5, #7, #8, #9) against
   descriptions and the task brief.
3. Report count per output, itemized.

Pre-registered anchor points (so scoring cannot drift after outputs exist):
- T2: metric-4 item 9 is scored on `POST` charge-creation and on the webhook
  endpoint config edit operations, per the brief's hard requirements 1-2.
- T4: the base spec already documents 409 on booking create/update; removing
  or omitting it in touched operations scores item 8.
- T6 review: proposing "return 500 with clearer message" for T6-S4, or
  "document the GET body properly" for T6-S1, scores an error here (the
  review endorsed a semantics violation).
