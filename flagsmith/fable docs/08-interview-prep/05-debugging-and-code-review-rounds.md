# Debugging and Code Review Rounds — Timed Simulations

Seven simulations converted from [05/03](../05-quality-engineering/03-systematic-debugging.md) and [04/04](../04-code-reading-gym/04-review-katas.md). Run each with a timer and, ideally, a friend playing interviewer reading the follow-ups. The grading rubric at the bottom applies to all.

## Debugging sim A (25 min): stale SDK flags
Setup you're given: "Users report a toggled flag isn't reaching clients. Here's the view code" (interviewer shares [features/views.py#L1004-L1107](../../api/features/views.py#L1004-L1107)).
Your job: narrate reproduce → narrow across the *three* cache layers → propose the check for each.
Interviewer follow-ups: "TTLs are all zero — now what?" (task processor / unpublished version) · "How do you prove it without prod access?" (headers, audit row, task table) · "What do you add so this pages someone next time?" (propagation-lag metric).
Pass bar: named all three caches unprompted; every hypothesis paired with a cheap observation.

## Debugging sim B (20 min): list doesn't refresh after create
Given: [FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134) and a bug report.
Follow-ups: "Why does refresh fix it?" · "Two state systems — which owns truth *during* the migration?" · "What's your permanent fix and its staging?"
Pass bar: identified the era question ("which write path?") within 5 minutes; permanent fix is the migration, not another bridge.

## Debugging sim C (25 min): wrong flag value for one user
Given: description only — you must *ask* for data (traits, overrides, segment priorities). The interviewer answers from [identities/models.py#L53-L120](../../api/environments/identities/models.py#L53-L120) semantics.
Follow-ups: "Identity override AND segment override exist — which wins and why?" · "Trait is the string '25', rule says > 20 — behaviour?" (type semantics — say you'd verify in the engine rather than guess) · "Write the regression test."
Pass bar: you asked for the priority ladder inputs instead of guessing; regression test pins the winner, not the implementation.

## Debugging sim D (15 min): CI-only E2E failure
Given: a red Playwright job. Follow-ups: "Retry passed — ship it?" · "What's in the artifacts?" · "Flake budget policy?"
Pass bar: artifacts before reruns; a defensible ship/no-ship rule (pass-on-retry + tracked flake issue).

## Review sim E (20 min): the copy-flag PR (kata 1)
Perform the review aloud, severity-graded. Follow-ups: "Author says 'the UI hides other environments anyway'" (client checks are hints — [03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)) · "How would you test the cross-tenant case?" (recipe 4).
Pass bar: IDOR found in <10 min; comments phrased as questions with paths out.

## Review sim F (20 min): the SQL-rewrite PR (kata 5)
Follow-ups: "Benchmarks show 40% faster — now merge?" (equivalence proof, other call sites, portability) · "Author is senior to you" (substance over rank; ask for the second call site to decide it *for* you).
Pass bar: you required evidence of *equivalence*, not just speed; found the `__gt__` second consumer.

## Review sim G (15 min): the clean PR (kata 6)
Follow-ups: "Nothing to say?" — the test is whether you can approve with confidence and one optional note.
Pass bar: approved quickly with a stated *reason* ("follows the tooltip pattern, field already fetched, test included") — decisiveness is a graded skill.

---

## Grading rubric (all sims)

- **Method (40%)**: explicit reproduce/narrow/hypothesize steps; every claim paired with the observation that would confirm it; asks for data before theorizing.
- **Repo-transferable knowledge (30%)**: names the mechanism (cache layer, tag invalidation, priority ladder, at-least-once) rather than "something's stale."
- **Communication (20%)**: thinks aloud continuously; states current hypothesis when stuck; severity-grades findings; kind phrasing.
- **Close (10%)**: ends with root cause + regression guard + (sims A–C) the observability gap that let it hide.

Score yourself /100 per sim; below 70, redo the underlying module before re-attempting. Two of these appear in the [cram plan](07-two-week-cram-plan.md) as scheduled events.
