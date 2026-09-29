# Behavioural STAR Stories

Eight worksheets. Sources: the tickets/projects in [06-contribution-practice/](../06-contribution-practice/) plus the study itself. **Fill the Action/Result cells with what you actually did** — the scaffolding below sets up the situation and the senior-signal details to make sure you capture. A story from a practice ticket in a fork is legitimate: "in an open-source codebase I work in" is true and verifiable.

Rehearsal check applies to all: under 2 minutes? concrete numbers/artifacts? ends with impact + lesson?

## Story 1: Learning a large unfamiliar codebase fast
Prompts: "how do you ramp up?" / "biggest codebase you've worked in"
Source: this curriculum's method. Situation: 3,000-file polyglot monorepo (Django+React), zero prior context. Task: productive contribution within N days. Action: entry-points→models→flows→boundaries method; wrote the system map; verified with query counts and tests rather than assumptions. Result: first merged/completed change + the map artifact. Evidence: [reading order](../01-codebase-cartography/02-file-reading-order.md), your notes. Senior signals: labelled legacy before touching it; distinguished public contracts from internals. Resume bullet: *"Mapped and contributed to a 3k-file OSS feature-flag platform (Django/React/TS) within two weeks."*

## Story 2: Technical tradeoff under constraint
Prompts: "a time you chose between two valid approaches"
Source: ticket M1 / kata 7 (server embed vs client fan-out) or M5 (the argued "no"). Capture: the measurement that decided it (request counts), who paid each cost (SDK users vs API payload), and the version-skew wrinkle. Senior signals: you priced the *rejected* option; rollback plan existed. Resume bullet: *"Eliminated a client-side N+1 (50→1 requests/page) via a backward-compatible API change."*

## Story 3: Debugging something genuinely hard
Prompts: "hardest bug" / "walk me through your debugging process"
Source: any scenario from [05/03](../05-quality-engineering/03-systematic-debugging.md) you performed for real (do one!). Capture: the narrowing tree, the observation that cracked it, the regression test. Senior signals: multi-layer cache reasoning; "the fix included a metric so it can't hide again." Resume bullet: *"Root-caused stale-config delivery across three cache layers; added propagation-lag instrumentation."*

## Story 4: Disagreeing with feedback / conflict
Prompts: "disagreement with a senior" / "pushback story"
Source: review sim F (SQL rewrite) or a real review thread. Capture: restated their case first; brought *new information* (second call site, equivalence gap); offered the decision back. Senior signals: substance/rank separation; the relationship survived — you cite a later collaboration. Resume bullet: rarely needed; keep as spoken story.

## Story 5: Working with ambiguity
Prompts: "requirements weren't clear"
Source: ticket M3 (copy-flag: segment overrides copied or not?) or M7 (what's even stored today?). Capture: you enumerated the interpretations, priced each, proposed a default with an escape hatch, and *shipped a decision document*, not a question. Senior signals: "a written 'no' / decision note is a deliverable." Resume bullet: *"Drove ambiguous cross-environment copy semantics to a documented, reviewable decision."*

## Story 6: A mistake and what changed
Prompts: "tell me about a failure"
Source: honest material from your practice — e.g. a test you wrote that pinned implementation instead of behaviour and shattered on refactor, or a PR rejected for scope. Capture: the mistake, the cost, the *system* you changed (checklist, template), not just "I'm more careful now." Senior signals: blameless framing; the artifact that prevents recurrence. Keep it real — interviewers smell synthetic failures.

## Story 7: Legacy migration / incremental modernization
Prompts: "improving old code without stopping the world"
Source: project 1 (Flux→RTK) even at partial completion. Capture: bridge design, TODO exit conditions, staged deletion, E2E as the net. Senior signals: "one direction of truth per interaction"; separate revertible commits; soak period before deleting the bridge. Resume bullet: *"Executed a staged Flux→RTK Query migration behind a feature flag, deleting a 1,000-line legacy store."*

## Story 8: Security thinking
Prompts: "a time you caught a security issue"
Source: review sim E (cross-tenant write in the copy-flag PR) or ticket 1/5 tests. Capture: how you spotted it (checklist habit: "whose data can this id reach?"), the 404-vs-403 reasoning, the regression test. Senior signals: systemic fix (checklist/recipe adoption) beyond the single finding. Resume bullet: *"Added tenant-isolation regression coverage and review checklist for a multi-tenant OSS API."*

---

## Prompt → story map (memorize)

| Interviewer says | Lead story | Backup |
| --- | --- | --- |
| conflict / disagreement | 4 | 6 |
| ambiguity | 5 | 2 |
| failure / mistake | 6 | 4 |
| technical tradeoff | 2 | 7 |
| leadership / initiative | 7 | 1 |
| debugging / firefighting | 3 | 8 |
| learning fast | 1 | 7 |
| security / quality | 8 | 3 |

One story must never answer two questions in the same loop — that's why there are eight.
