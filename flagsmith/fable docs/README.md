# Flagsmith Upskill Curriculum

A training lab built on the real Flagsmith codebase for one specific learner: a **junior fullstack JS engineer** (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **Transferable skill** — why the pattern exists, its failure modes, and how to talk about it in an interview.

## What Flagsmith is

Flagsmith is an open-source feature-flag, remote-config, and A/B-testing platform. Client and server SDKs fetch flags from an API; humans manage flags through a React dashboard. The repo is a **polyglot monorepo**: a Django/DRF API in [api/](../api/), a React 19 + TypeScript dashboard in [frontend/](../frontend/), and a Docusaurus docs site in [docs/](../docs/). Flags resolve through a priority ladder — identity override beats segment override beats environment default — and every change is audited, versioned, and pushed asynchronously to caches, DynamoDB (SaaS edge), SSE streams, webhooks, and third-party integrations. That makes it an unusually good teaching repo: one product, but real caching, async task processing, multi-tenancy, permissions, and versioning problems everywhere you look.

**A note on the stack:** the backend is Python/Django, not Node. That is a feature of this curriculum, not a bug. Mid-level fullstack interviews test whether you understand *API design, data modelling, authorization, and async reliability* — concepts that live above the language. You will read Django here the way you'd read any unfamiliar backend on the job: by following routes, models, and tests. The frontend and its build/test tooling are pure React/TS, squarely in your lane.

## The tracks

| Module | What it gives you |
| --- | --- |
| [00-fast-track.md](00-fast-track.md) | One weekend: run it, trace two flows, make one safe change |
| [01-codebase-cartography/](01-codebase-cartography/) | Maps: system shape, reading order, glossary, tooling, key flows |
| [02-stack-and-language-mastery/](02-stack-and-language-mastery/) | JS/TS/React mental models anchored to real files; Django survival guide |
| [03-architecture-and-patterns/](03-architecture-and-patterns/) | Layers, data model, authz, async reliability, pattern catalog, critique |
| [04-code-reading-gym/](04-code-reading-gym/) | Annotation drills, trace tables, fake-code contrasts, review katas |
| [05-quality-engineering/](05-quality-engineering/) | Testing strategy, debugging method, performance, security, observability |
| [06-contribution-practice/](06-contribution-practice/) | Tickets and projects sized junior → senior, each with rollback thinking |
| [07-career-and-collaboration/](07-career-and-collaboration/) | Review mindset, PR/RFC writing, maintainer communication |
| [08-interview-prep/](08-interview-prep/) | 45+ question cards, system-design walkthrough, STAR stories, cram plan |
| [09-reference/](09-reference/) | Command cheatsheet, risk register, rubrics, verification log |

## Recommended paths

- **Brand-new junior** — 00 → 01 (all) → 02 → 04 (drills as you read) → 05.01–02 → 06.01. Eight weeks at ~6 hrs/week.
- **Junior with React/TS familiarity** — 00 → 01.05 (key flows) → 03 → 04 → 06.01–02. Skim 02, but do its drills.
- **Mid-level new to the repo** — 00 → 01.01/01.05 → 03.05–06 → 06.02–03 → 05.03–06.
- **Senior doing architecture review** — 01.01 → 03.06 (critique) → 09 (risk register) → 06.04 (katas).
- **Interview in two weeks** — go straight to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md). It pulls from everything else on a day-by-day schedule.

## Conventions

- **Anchors**: links written as `[path](…/api/file.py#L10-L20)` point at the actual code. Line numbers were verified against the working tree on 2026-07-09 (see [09-reference/verification-log.md](09-reference/verification-log.md)); they will drift as the repo evolves — trust the file, search for the symbol.
- **Fake code**: every snippet not from the repo starts with `// Illustrative fake code: not from this repo` (or the `#` Python equivalent). Everything else is real.
- **Drills**: exercises come with self-grading rubrics — *Basic* (you found the file), *Solid* (you traced input → output and named the validation boundary), *Strong* (you named the invariant, a risk, a test strategy, and an alternative design).
- **Verified vs inferred**: commands marked __verified__ were run while building this curriculum; __inferred__ means read from scripts/CI config but not executed here.
- **Interview angle**: sections flagged with 🎤 tell you how the material shows up in interviews and what a mid-level answer sounds like.

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What are the tradeoffs?"
- **Senior asks:** "What does this commit us to, who pays the cost, and how do we reduce risk?"

Interviews for mid-level roles test exactly the second and third question. A definition answers nothing; a concrete example with a tradeoff and a failure mode answers everything. This repo is your source of concrete examples.
