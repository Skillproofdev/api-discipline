# api-discipline benchmark — scoring protocol (pre-registered)

Frozen before any run, together with `tasks.md` and `ground-truth/`.
Scorers (human or agent) never see which arm produced an output until all
metrics are recorded (run directories are scored under blinded copies named
`output-A`/`output-B` per task).

## 0. Validator availability — VERIFIED on this machine, 2026-07-10

Environment: macOS (Darwin 23.5.0), node v24.7.0, npm 11.5.1, Homebrew at
/opt/homebrew.

| Tool | Status | Verified invocation | Verified against |
|------|--------|---------------------|------------------|
| Redocly CLI 2.38.0 | WORKS via npx (cached after first `--yes` install) | `npx --yes @redocly/cli lint <spec>.yaml --format=json` | t3-base (0 err/0 warn), t4-base (0/0), t5-flawed (exactly 1 error = seeded broken $ref), t6-flawed (0 err) |
| Spectral CLI 6.16.1 | WORKS via npx; REQUIRES a ruleset file — `--ruleset spectral:oas` inline does NOT exist as CLI syntax; use the checked-in `bench/.spectral.yaml` (`extends: ["spectral:oas"]`) | `npx --yes @stoplight/spectral-cli lint <spec>.yaml --ruleset bench/.spectral.yaml --format json` | t3-base (0 errors, 9 style warnings), t4-base (0 errors), t5-flawed (exactly 1 error: `invalid-ref` = seeded) |
| oasdiff 1.23.0 | WORKS — installed 2026-07-10 via `brew install oasdiff`, binary at `/opt/homebrew/bin/oasdiff` | `/opt/homebrew/bin/oasdiff breaking <base>.yaml <revised>.yaml` (add `-f json` for machine counting) | t4-base vs naive revision → 8 entries (6 error, 2 warning), matching `ground-truth/t4-breaking-changes.md` |

Notes and honest fallbacks:

- Both npx tools are now in the local npx cache, so runs do not need network.
  If the cache is ever purged and npx cannot install (offline), do NOT
  substitute a weaker check — the benchmark is blocked until a validator
  runs. Fallback order: `npm install --prefix bench/.tools @redocly/cli`
  then `bench/.tools/node_modules/.bin/redocly`; same for spectral. If
  neither validator can execute, metric 1 is N/A for the whole run set and
  results are not publishable (metric 1 is a pass-criteria metric).
- Version pinning: score all 14 runs with the SAME versions recorded above.
  If a version bump is unavoidable mid-benchmark, void and restart scoring.
- Redocly exit code is 1 whenever any problem (even a warning) exists —
  count errors from the JSON (`totals.errors`), never from the exit code.
- Arm agents may invoke validators themselves during their runs; that is
  measured behavior, not scoring.

## 1. Metric 1 — OpenAPI validator errors (T1-T4 outputs)

Primary count (redocly):

```
npx --yes @redocly/cli lint <output>.yaml --format=json > out.json
python3 -c "import json; print(json.load(open('out.json'))['totals']['errors'])"
```

Cross-check (spectral, severity 0 = error):

```
npx --yes @stoplight/spectral-cli lint <output>.yaml \
  --ruleset bench/.spectral.yaml --format json > sp.json
python3 -c "import json; print(sum(1 for p in json.load(open('sp.json')) if p['severity']==0))"
```

- Score = redocly error count; spectral count reported alongside. If the
  two disagree on zero-vs-nonzero, the itemized problems are published and
  the redocly number stands.
- A file that fails to parse as YAML/OpenAPI at all (either tool crashes or
  reports an unparseable document) scores **+20**.
- No spec file delivered where the task requires one: +20.
- T3/T4: the metric is run on the full revised spec. The base specs are
  verified 0-error, so every error is attributable to the arm's edit.
- Lower is better. Skill target: 0.

## 2. Metric 2 — seeded-violation catch rate (T5, T6)

- Keys: `ground-truth/t5-answer-key.md` (12 seeds),
  `ground-truth/t6-answer-key.md` (10 seeds). Catch definition and per-seed
  criteria live in the keys and are frozen.
- An independent verifier agent (fresh context; sees ONLY the review file
  and the answer key, not the arm identity or this repo) marks each seed
  caught / not caught, quoting the review line that earns each catch.
- Rate = caught / seeded, per run. Also recorded: false-positive findings
  (defects claimed that do not exist in the fixture), reported separately.

## 3. Metric 3 — consistency violations (T1-T4 outputs)

- Checklist: `ground-truth/consistency-checklist.md` (10 fixed points).
- Pass A (scripted screen, items marked mechanical in the checklist):
  extract property keys, non-2xx responses (schema ref + media type),
  collection query params, id/timestamp schemas, operationIds from the YAML
  and flag candidate hits. The screen script is written at scoring time and
  published with the results; it only proposes candidates.
- Pass B (verifier agent, fresh context): confirms/rejects each candidate
  with a concrete location and evaluates the judgment items (#2, #3).
  A point counts only with a named location. Script-vs-verifier
  disagreements are logged and published.
- Score = number of checklist items violated at least once (0-10 per spec).
  For T3/T4, divergence from the base spec's conventions counts (see
  checklist notes). Lower is better.

## 4. Metric 4 — HTTP-semantics errors (T1-T4 outputs + T6 review)

- Checklist: `ground-truth/http-semantics-checklist.md` (9 fixed items,
  with pre-registered anchor points for T2/T4/T6).
- Same two-pass procedure as metric 3 (mechanical screen for items
  1, 2, 3, 6; verifier for the rest). Score = occurrence count (not capped
  per item). Lower is better.

## 5. Metric 5 — silent breaking changes (T4)

- Ground truth and grouping procedure:
  `ground-truth/t4-breaking-changes.md` (2 underlying breaking changes;
  canonical oasdiff mapping recorded there).
- Command: `/opt/homebrew/bin/oasdiff breaking
  materials/t4-base-spec.yaml <arm revised spec> -f json`
- Score = number of breaking-change GROUPS present in the diff that the
  arm's notes/rationale never disclose as breaking (groups per the ground
  truth doc; arm-introduced extra breaking changes form additional groups).
  An arm that redesigns a change away non-breakingly (oasdiff silent) and
  says so scores 0 for that group. Skill target: 0 undisclosed.
- oasdiff `warning`-level entries count toward a group's presence but a
  group consisting ONLY of warnings still requires disclosure (a removed
  request property breaks writers).

## T7 (no numeric metric)

Verifier judgment, pre-registered question: does the answer COMMIT to one
versioning strategy and one pagination pair, stated as the team's
convention with reasons — or present an options tour without a decision?
Recorded as commit / no-commit + a two-line justification; published
verbatim.

## Cost accounting

Per run, record: wall-clock time, total tokens (in/out) if the harness
exposes them, and number of tool calls. Skill-arm totals INCLUDE reading
SKILL.md. Published alongside quality metrics.

## Publishing pass criteria (from the research doc, frozen)

Publish "skill wins" only if the skill arm wins metrics 1-3 with no
regression >1 point on metrics 4-5. All losses are published verbatim
(research-discipline precedent). Otherwise publish results as-is with a
"no win claimed" verdict.
