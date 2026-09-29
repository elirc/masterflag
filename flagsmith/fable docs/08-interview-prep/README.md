# 08 — Interview Prep

Target: **mid-level fullstack JS loops.** Over 60% of the question cards here anchor to real Flagsmith code, because the winning move in every round is the same: *answer with a concrete example, a tradeoff, and a failure mode — never a definition.* That's the golden rule; everything else is logistics.

## How mid-level fullstack loops are typically structured

1. **Recruiter/tech screen** (30–45m): rapid JS/TS/React fundamentals + "walk me through a project." Your project is this repo — the [system map](../01-codebase-cartography/01-system-map.md) delivered in 3 minutes.
2. **Technical deep-dive** (60m): one area probed until you run out — cards in [01](01-js-ts-node-deep-dive.md)/[02](02-frontend-framework-questions.md)/[03](03-api-and-data-modeling-questions.md).
3. **Practical coding** (60–90m): build/extend a small feature. The repo's house patterns (RTK Query service + typed contract + test) are your muscle memory.
4. **System design** (45–60m): [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — you will literally design a feature-flag-ish system.
5. **Debugging / code review round** (45m, increasingly common): [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md).
6. **Behavioural** (45m): [06-behavioral-star-stories.md](06-behavioral-star-stories.md).

Two weeks out? → [07-two-week-cram-plan.md](07-two-week-cram-plan.md).

## Using this repo as your portfolio of talking points

You studied (and ideally contributed to) a production OSS platform: multi-tenant authz, three-layer caching, an async propagation pipeline, a live legacy migration, strict typing on both sides. Every card below ends with the anchor to *re-read the morning of*. When an interviewer asks anything generic, your reflex is: **claim → repo example → tradeoff → failure mode → (if senior signal needed) what you'd do differently.**

Calibration reminder from the [mindset ladder](../README.md#the-mindset-ladder): junior = makes it work; mid = right pattern, named tradeoffs; senior = commitments, costs, risk reduction. Interviewers grade the ladder, not the vocabulary.
