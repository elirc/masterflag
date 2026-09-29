# Two-Week Cram Plan

Assumes ~2.5 hrs/weekday, 4 hrs/weekend day, interview at the end of week 2. Everything references material you can do without upstreaming anything. If you have less time, do the **bold** items only.

## Week 1 — load the repo into your head

**Day 1** — **[00-fast-track](../00-fast-track.md) Saturday-morning + flow #1**; get the stack running (Docker). Evening: cards [01/Q1–Q4](01-js-ts-node-deep-dive.md) aloud.
**Day 2** — **Flow #2 (toggle) + [key flows](../01-codebase-cartography/05-key-flows.md) 4 & 5**. Cards 01/Q5–Q9. Write your 3-minute repo pitch (system map aloud) and record it.
**Day 3** — [Data model](../03-architecture-and-patterns/02-data-model-and-persistence.md) until you can draw the ERD blind. Cards [03/Q1–Q4](03-api-and-data-modeling-questions.md). **Drill: whiteboard the FeatureState table from memory.**
**Day 4** — [Caching + async](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md). Cards 03/Q8–Q9. Trace table 5 ([04/02](../04-code-reading-gym/02-trace-tables.md)).
**Day 5** — **Do one real ticket locally: ticket 1 or 2 from [06/01](../06-contribution-practice/01-good-first-tickets.md), tests green.** This becomes STAR evidence. Cards [02/Q2–Q5](02-frontend-framework-questions.md).
**Day 6** — **Mock system design #1** (40 min, recorded): [04-system-design](04-system-design-from-this-repo.md) main walkthrough. Self-grade against its rubric. Patch the two weakest steps by rereading anchors.
**Day 7 — checkpoint.** Can you: draw the ERD? name the three caches + invalidation for each? deliver the 3-min pitch without notes? explain the priority ladder? If any "no": that's Day 8's first hour. Fill all 8 STAR worksheets ([06](06-behavioral-star-stories.md)) with real content tonight.

## Week 2 — pressure and polish

**Day 8** — **Timed debugging sim A + B** ([05](05-debugging-and-code-review-rounds.md)), scored. Review misses against [05/03](../05-quality-engineering/03-systematic-debugging.md). Cards 02/Q6–Q11.
**Day 9** — **Review sims E + F** (+G for decisiveness). Reread [security checklist](../05-quality-engineering/05-security-checklist.md) — IDOR must be reflexive. Cards 03/Q3, Q6, Q10–Q12.
**Day 10** — **Mock system design #2: variation prompts 1 & 2** (real-time, 10× reads). Then [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) once — its structure is your "critique something you know" answer.
**Day 11** — STAR rehearsal day: all 8 stories aloud, 2-min timer, using the prompt→story map. Record two; cut filler. Cards 01/Q10–Q13.
**Day 12** — Debugging sim C (the ask-for-data one) + practical-coding warm-up: build a tiny RTK Query service + typed contract + Jest test from scratch in a sandbox, muscle-memory speed.
**Day 13** — Light pass: reread [key flows](../01-codebase-cartography/05-key-flows.md) "what seniors notice" lines and every card's *anchor* (not the card). One full loop simulation if you have a partner: pitch → deep-dive (pick your weakest file) → design variation 4 → two STAR prompts.
**Day 14 — interview eve.** No new material. Reread: [system map](../01-codebase-cartography/01-system-map.md), [critique](../03-architecture-and-patterns/06-architecture-critique.md), your STAR sheets. Prepare *your* questions for them (team's flag/rollout practice is a natural, informed one). Sleep.

## Day-of one-pager (write it yourself, from memory, as the final drill)

The read/write asymmetry sentence · ERD sketch · three caches + invalidation events · priority ladder · committed-fact fan-out chain · IDOR pattern · your 8 story titles · the golden rule: **example, tradeoff, failure mode — never a definition.**

## Self-assessment checkpoints

Day 7 gate above; Day 14 gate: system-design self-score ≥ "mid" on every step of [04](04-system-design-from-this-repo.md)'s junior/mid/senior contrasts, and all sims ≥ 70. Below the gate → prioritize the miss over new material; cramming breadth beats nothing, but depth on flows 1/2/4 beats breadth everywhere.
