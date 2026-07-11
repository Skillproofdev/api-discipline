# api-discipline benchmark — task prompts (pre-registered)

Frozen before any run. 7 tasks x 2 arms (base / skill) = 14 runs, one fresh
agent per run, same model and settings for both arms of a task. Do not edit
prompts after the first run; if a prompt is broken, void all runs and bump
the version header below.

Prompt-set version: 1.0 (2026-07-10)

## Arm protocol

- **base arm**: the agent receives the task prompt below, verbatim, and
  nothing else.
- **skill arm**: the agent receives, before the identical task prompt, only
  this preamble line:
  > Read /Users/afin/Desktop/SkillProof-GitHub/api-discipline/SKILL.md and
  > apply it to the following task.
  Skill-run cost totals include the skill read.
- Each run writes into its own directory
  `bench/runs/<date>-<task>-<arm>/` (created by the harness). File names are
  fixed per task below. Anything delivered outside the named files is scored
  from the agent's final message.
- Arms may run validators/tools themselves (that is part of what is being
  measured). Arms must NOT open the forbidden paths at the bottom of this
  file — the harness states this to the arm as a single rule:
  > You may read only the input files named in the task and files you
  > yourself create in your run directory.

## T1 — greenfield: task tracker

Files: input `bench/materials/t1-brief.md` (copied into the run directory as
`brief.md`). Output: `spec.yaml` (+ optional `notes.md`).

Prompt:

```
You are the API designer for this project. Read brief.md in your working
directory — it is a product brief for "Trackle", a team task tracker.
Design the HTTP API it asks for.

Deliver:
- spec.yaml — the API contract as OpenAPI 3.1 YAML
- notes.md — any short notes you want the reviewing engineers to read

Work autonomously; do not ask clarifying questions. When information is
missing, make a reasonable decision and record it in notes.md.
```

## T2 — greenfield: payments + webhooks

Files: input `bench/materials/t2-brief.md` as `brief.md`. Output:
`spec.yaml` (+ optional `notes.md`).

Prompt:

```
You are the API designer for this project. Read brief.md in your working
directory — it is a product brief for "Ledgerline", a merchant payments
API. Design the HTTP API it asks for. Engineering's hard requirements in
the brief are non-negotiable.

Deliver:
- spec.yaml — the API contract as OpenAPI 3.1 YAML
- notes.md — any short notes you want the reviewing engineers to read

Work autonomously; do not ask clarifying questions. When information is
missing, make a reasonable decision and record it in notes.md.
```

## T3 — extend a clean spec

Files: inputs `bench/materials/t3-brief.md` as `brief.md` and
`bench/materials/t3-base-spec.yaml` as `current-spec.yaml`. Output:
`revised-spec.yaml` (full updated spec) + `notes.md`.

Prompt:

```
You maintain the Stockroom API. current-spec.yaml in your working directory
is the production contract; brief.md is a change request for three new
capabilities. Extend the API.

Deliver:
- revised-spec.yaml — the complete updated OpenAPI 3.1 contract
- notes.md — whatever the reviewing engineers need to know about the change

Work autonomously; do not ask clarifying questions. When information is
missing, make a reasonable decision and record it in notes.md.
```

## T4 — extend with an implied breaking change

Files: inputs `bench/materials/t4-brief.md` as `brief.md` and
`bench/materials/t4-base-spec.yaml` as `current-spec.yaml`. Output:
`revised-spec.yaml` (full updated spec) + `notes.md`.

Prompt:

```
You maintain the RoomBoard Bookings API. current-spec.yaml in your working
directory is the production contract; brief.md is a change request from the
platform review. Apply the requested changes.

Deliver:
- revised-spec.yaml — the complete updated OpenAPI 3.1 contract
- notes.md — whatever the reviewing engineers need to know about the change

Work autonomously; do not ask clarifying questions. When information is
missing, make a reasonable decision and record it in notes.md.
```

(Ground truth: both requested changes are breaking — see
`ground-truth/t4-breaking-changes.md`. Pass = flagged with migration note,
or a non-breaking alternative proposed. The prompt deliberately does not
hint at this.)

## T5 — review flawed spec #1

Files: input `bench/materials/t5-flawed-spec.yaml` as `spec-under-review.yaml`.
Output: `review.md`.

Prompt:

```
An acquired team is handing over their "Crmly" API.
spec-under-review.yaml in your working directory is their OpenAPI contract.
Before we adopt it, review the API design and report every defect you find.

Deliver:
- review.md — your findings, in whatever structure you think a good API
  design review should have

Work autonomously; do not ask clarifying questions.
```

(Ground truth: 12 seeded violations — see `ground-truth/t5-answer-key.md`.)

## T6 — review flawed spec #2 (semantics-weighted)

Files: input `bench/materials/t6-flawed-spec.yaml` as `spec-under-review.yaml`.
Output: `review.md`.

Prompt:

```
A contractor delivered the "Pressroom" CMS API design.
spec-under-review.yaml in your working directory is their OpenAPI contract.
Before we build against it, review the API design and report every defect
you find.

Deliver:
- review.md — your findings, in whatever structure you think a good API
  design review should have

Work autonomously; do not ask clarifying questions.
```

(Ground truth: 10 seeded violations — see `ground-truth/t6-answer-key.md`.)

## T7 — conventions question

Files: no inputs. Output: `answer.md`.

Prompt:

```
Our team is about to publish our first public REST API. How should we
version and paginate it? Write your recommendation to answer.md.
```

(Measured qualitatively per the research doc: does the answer commit to
declared-once conventions — one versioning strategy, one pagination pair,
stated as the decision with reasons — versus a generic options tour.
Scored by the verifier as commit/no-commit plus a note; no numeric metric.)

## Paths arms must NOT open

Absolute roots; everything beneath them is off-limits to arm agents. Both
arms — a run that opens any of these is void and rerun once with a fresh
agent; a second violation voids the task pair.

- /Users/afin/Desktop/SkillProof-GitHub/api-discipline/bench/ground-truth/
- /Users/afin/Desktop/SkillProof-GitHub/api-discipline/bench/scoring.md
- /Users/afin/Desktop/SkillProof-GitHub/api-discipline/bench/tasks.md
- /Users/afin/Desktop/SkillProof-GitHub/api-discipline/bench/materials/
  (arms get their inputs COPIED into the run directory; the materials
  directory itself, which sits next to the answer keys and other tasks'
  fixtures, stays closed)
- /Users/afin/Desktop/SkillProof-GitHub/api-discipline/bench/runs/ except
  the arm's own run directory
- /Users/afin/Desktop/Skill Agregator/ (the whole research workspace,
  including docs/api-skill-research.md)
- base arm additionally:
  /Users/afin/Desktop/SkillProof-GitHub/api-discipline/SKILL.md and
  /Users/afin/Desktop/SkillProof-GitHub/api-discipline/README.md
  (skill arm reads SKILL.md only, per protocol)
