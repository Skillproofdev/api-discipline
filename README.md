# api-discipline

**REST/OpenAPI design that validates and stays consistent — 9 enforceable rules, benchmarked.** Published API-design skills teach principles: plural nouns, version your API, paginate your lists. Then the agent adds one endpoint with `page_size` where every other list uses `limit`, documents a 500 for a validation error, and ships a spec with a broken `$ref` — because nothing forced a check. This skill is the enforcement layer: an error model on every endpoint, conventions declared once and reused, correct PUT/PATCH/POST semantics, a breaking-change diff on every edit, validator-clean OpenAPI 3.1 output, and a mandatory cross-endpoint consistency audit before anything ships.

Built and bench-tested by [SkillProof](https://skillproof.dev), the tested Claude skills directory.

## What it does differently

We surveyed 84 API-design/OpenAPI skills in our 16,682-skill index plus the standalone web ecosystem before writing this one. The biggest (37.6k★ repo) is a concepts textbook with zero enforcement; the best (10.5k★ repo) names a linter but stops there. Four things existed in **none** of them:

1. **A cross-endpoint consistency audit as a mandatory pass.** Ten defined checks — casing, pluralization, shared error schema, identical pagination params, uniform id/timestamp formats, operationId pattern, same-action-same-status-code — run across every endpoint before delivery. Every competitor has, at most, one "be consistent" bullet.
2. **Breaking-change discipline that fires on edits.** Every spec edit gets an enumerated breaking-change pass (removed ops, newly required fields, narrowed enums, type changes…), backed mechanically by `oasdiff breaking` when available. The tooling ecosystem is mature; no skill wires it in.
3. **HTTP semantics as rules, not trivia.** PUT replaces (idempotent), PATCH partials, POST creates with 201+Location, DELETE returns 204, GET never has a body — enforced, with a status-code table, not listed as "concepts to know".
4. **A review/extend output contract.** "Review this spec" returns findings keyed to the audit checklist with locations and fixes; "add an endpoint" returns a diff that inherits the existing spec's conventions plus a breaking-changes block. Competitors only define greenfield output.

Plus the rest: plural-noun/no-verb resource naming with one declared casing, an RFC 9457 error model `$ref`'d from every endpoint, pagination/filtering/sorting declared once as shared components, an explicit versioning statement, and validator-clean OpenAPI 3.1 (`npx @redocly/cli lint` — with a defined self-check fallback when no CLI can run).

## Install

```bash
git clone https://github.com/Skillproofdev/api-discipline ~/.claude/skills/api-discipline
# restart Claude Code — triggers on "design an API", "add an endpoint", "review this OpenAPI spec", REST convention questions
```

One command — the repo IS the skill.

## The benchmark (measured, not vibes)

Seven tasks — two greenfield designs (one seeded with idempotency traps), two spec extensions (one implying a breaking change), two reviews of flawed specs with 22 seeded violations between them, one conventions question. Each ran twice: one Claude Sonnet agent bare, one reading this SKILL.md first. Identical prompts; skill totals include the skill read. Specs scored mechanically with `redocly lint` and `spectral lint`, edits diffed with `oasdiff breaking`, consistency and HTTP-semantics violations counted against fixed published checklists, seeded-violation catches judged by independent verifier agents.

| Metric (lower is better unless noted) | base | skill |
|---|---|---|
| Validator errors, T1 greenfield (redocly) | 12 | **0** |
| Validator errors, T2 greenfield (redocly) | 6 | **0** |
| Validator errors, T3/T4 edits (redocly) | 0 / 0 | 0 / 0 |
| Seeded violations caught, T5 (of 12, higher better) | 12 | 12 |
| Seeded violations caught, T6 (of 10, higher better) | **10** | 9 |
| Consistency violations, T1–T4 (of 10 each) | 2 / 2 / 2 / 0 | **0 / 0 / 0 / 0** |
| HTTP-semantics errors, T1–T4 | 2 / 0 / 1 / 0 | **0 / 0 / 0 / 0** |
| Undisclosed breaking changes, T4 | 0 | 0 |
| Conventions question, T7 | commits | commits |

Measured 2026-07-10 with Redocly CLI 2.38.0, Spectral 6.16.1, and oasdiff 1.23.0. Full method and per-finding detail: [`bench/results/verdict.md`](bench/results/verdict.md); pre-registered ground truth in [`bench/ground-truth/`](bench/ground-truth/).

**What the numbers say.** The skill's decisive wins are validator-clean output (both bare runs shipped OpenAPI 3.0 `nullable:` into 3.1 documents — 12 and 6 structural errors; the skill runs validated clean) and the consistency + HTTP-semantics audits (0 violations across all four design tasks, versus verb-in-path endpoints, ad-hoc error schemas, and wrong create/delete status codes in the bare runs). Losses are published too: on the semantics-heavy review (T6) the bare run caught one seeded defect the skill's structured review missed — a `201` with no `Location` header — 10/10 vs 9/10. Both runs disclosed every T4 breaking change; the skill additionally shrank the breaking surface by keeping a deprecated field alias. The T4 result is not claimed as an independent win: the skill run was found to have seen a ground-truth hint, though oasdiff and the independent bare run corroborate the finding regardless.

## When it triggers

"Design an API for…", "add/extend an endpoint", "review this OpenAPI/Swagger spec", REST convention questions (naming, versioning, pagination, PUT vs PATCH, status codes). Explicitly excluded: GraphQL-only schema work, client SDK/type codegen from an existing spec, API security testing.

## Why trust this

We test other people's skills for a living ([public methodology](https://skillproof.dev/methodology)). Our own skills get the same treatment: controlled runs, mechanical scoring where possible, and the losses published next to the wins.

More skills from us: [skillproof-skills index](https://github.com/Skillproofdev/skillproof-skills) — [token-discipline](https://github.com/Skillproofdev/token-discipline) (−20% tokens on multi-step work), [research-discipline](https://github.com/Skillproofdev/research-discipline) (59% fewer wrong claims). Free tools for skill authors: [SKILL.md validator, token calculator](https://skillproof.dev/tools).

## License

MIT — use it, fork it, ship it.
