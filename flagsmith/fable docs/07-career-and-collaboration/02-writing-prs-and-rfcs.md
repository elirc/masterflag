# Writing PRs and RFCs

## PR descriptions — the five sections

```
## What
One paragraph: the change, at the level a reviewer skims.

## Why
The problem/issue link. If a maintainer discussion happened, link it.

## How tested
Exact commands + what you observed. "make test opts='-k feature_state'" beats "tests pass".
For UI: before/after screenshot or the e2e file touched.

## Risks
What could break, who's affected (self-hosters? SDK fleet? SaaS?), how it rolls back.
"None" is acceptable ONLY with a reason ("tests-only change").

## Follow-ups
Deliberately deferred work, so scope cuts look intentional (they are).
```

House specifics: conventional-commit-style titles are used with release-please ([platform-release-please.yml](../../.github/workflows/platform-release-please.yml)) — look at recent merged PR titles (`feat:`, `fix:`, `docs:`) and copy the pattern; British English ([AGENTS.md](../../AGENTS.md)); never touch `uv.lock`; expect the docs-diff and migration checks to run against you ([api-pull-request.yml](../../.github/workflows/api-pull-request.yml)).

Commit messages: imperative, scoped, and the *why* in the body when non-obvious. A reviewer should be able to `git log --oneline` your branch and see a plan, not a diary ("wip", "fix", "fix2" — squash before review).

## When to write an RFC instead

Trigger any of: new table/endpoint surface, public-contract change, new dependency, cross-cutting pattern (codegen, outbox), anything whose *revert is not a revert* (data written, webhooks sent). For this repo, that's most of [06/02](../06-contribution-practice/02-mid-level-feature-tickets.md) M3+ and everything in [06/03](../06-contribution-practice/03-senior-build-projects.md).

## RFC template (tuned to this repo)

```
# RFC: <title>              Status: Draft | Discussed | Accepted | Rejected

## Problem
What hurts, for whom (self-hosted / SaaS / SDK users), with evidence (anchor, issue, metric).

## Proposal
The design. Include: data shape (+ migration sketch), API contract (+ OpenAPI impact),
sync vs async placement (request path or task processor?), permission model
(which constant, which scope), rollout (setting? flag? — pattern 16), rollback.

## Alternatives considered
At least one real contender and the reason it loses. "Do nothing" is always a row.

## Blast radius
Tables touched (size!), endpoints changed (SDK-fleet frozen contracts?),
self-hosted upgrade story, docs to update.

## Test & observability plan
Layer by layer (unit/integration/E2E) + the metric/log that proves it works in prod.

## Open questions
Numbered, answerable, each assigned to someone (even if "me, after reading X").
```

The two sections that get RFCs accepted: **Blast radius** (shows you know who pays) and **Alternatives** (shows the proposal survived contact with another idea). The section that gets them rejected when missing: **rollback**.

## Worked micro-example

For ticket M1 (server-side segment embed): Problem = client N+1 ([useFeatureState.ts#L42-L44](../../frontend/common/services/useFeatureState.ts#L42-L44), request counts). Proposal = additive serializer field behind `?include=feature_segment`. Alternatives = FE batch endpoint (loses: new surface for old data); do nothing (loses: measured latency). Blast radius = one serializer, response size +N×~200B, self-hosted skew handled by FE fallback. Rollback = param ignored. Tests = serializer unit + FE request-count. That's ten sentences — RFCs are judged on decision density, not length.

🎤 **Interview angle:** "How do you propose technical changes?" Walk the template and *why each section exists*. Bonus signal: "revert is not a revert" as your RFC trigger — it shows you think in state, not diffs.

**Drill:** kata 8 in [06/04](../06-contribution-practice/04-refactor-and-design-katas.md) — full RFC for the webhook ledger. Grade there.
