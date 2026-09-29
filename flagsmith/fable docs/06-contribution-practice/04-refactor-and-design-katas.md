# Refactor and Design Katas

Senior exercises. Most produce *documents and diffs in a fork*, not upstream PRs — the skill under training is judgment under constraint. Self-grading criteria per kata.

## Kata 1: Untangle a boundary leak
Move the server-key-only filtering decision ([views.py#L1075-L1085](../../api/features/views.py#L1075-L1085)) conceptually into a policy object/service so any future flags-serving path (edge, documents) applies it uniformly. Deliverable: diff + a paragraph on what breaks if a second read path forgets this filter today.
Grade — Solid: the filter has one home and both call sites use it. Strong: you found whether the environment-document path has an equivalent rule (read [util/mappers/](../../api/util/mappers/)) and reconciled or explained the difference.

## Kata 2: Design the true outbox
Spec the change that would make the audit→task chain crash-proof (enqueue inside the commit or a relay polling AuditLog). Deliverable: 1-page design incl. failure matrix (crash before/after commit × before/after enqueue), and what `flagsmith-common` would need.
Grade — Solid: matrix complete. Strong: you quantified the added DB load of a poller and proposed the cheapest acceptable variant (hint: AuditLog is already durable — the relay may only need a cursor).

## Kata 3: Split the god model
Carve `Environment` ([environments/models.py#L80-L620](../../api/environments/models.py#L80)) into responsibilities on paper: identity/config, caching, document build, dynamo sync, metrics. Deliverable: target module map + the *order* of extractions with test protection for each step.
Grade — Solid: extraction order respects dependency direction. Strong: step 1 is something actually shippable this week (e.g. document-building → `environments/services.py`) with named tests pinning it.

## Kata 4: Remove duplication without creating coupling
`update_flag` v1/v2 ([versioning_service.py#L132-L245](../../api/features/versioning/versioning_service.py#L132-L245)) share value-update and priority logic. Identify what to unify and — harder — what to *leave duplicated* because the eras will diverge and v1 will die. Deliverable: annotated diff proposal.
Grade — Solid: unified the stable parts (`_update_feature_state_value` is already shared — what else?). Strong: your "leave duplicated" list has reasons tied to v1's deletion plan.

## Kata 5: Improve type safety at one boundary
Pick the FE service→component boundary for feature states and eliminate every `any`/assertion between [useFeatureState.ts](../../frontend/common/services/useFeatureState.ts) and [FeatureRow](../../frontend/web/components/feature-summary/FeatureRow.tsx) consumers. Deliverable: diff, typecheck green.
Grade — Solid: no new `any`, no behaviour change. Strong: one *illegal state made unrepresentable* (e.g. skeleton vs data union following [FeaturesPage.tsx#L43-L49](../../frontend/web/components/pages/features/FeaturesPage.tsx#L43-L49)).

## Kata 6: Design a migration (schema)
Write the migration plan for the declarative uniqueness constraint the Feature model TODO wants ([models.py#L138-L141](../../api/features/models.py#L138-L141)): from raw-SQL-index reality to `UniqueConstraint`, on a huge table, zero downtime.
Grade — Solid: additive → validate → swap sequence with lock analysis per step. Strong: you checked what the existing migrations 0005/0050 actually created (read them!) and your plan handles soft-deleted rows.

## Kata 7: Reduce an N+1 (for real)
Implement ticket M1 (server-side segment embed) in a fork end-to-end. Grade — Solid: request count provably drops (mock-count test). Strong: you also measured payload growth and set the tipping point where embedding loses.

## Kata 8: Write the RFC
Full RFC for senior project 3 (webhook ledger) using the [07/02 template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). Grade — Solid: alternatives section has a real contender killed by a real reason. Strong: a maintainer could say yes/no from your doc alone without asking a clarifying question.

## Kata 9: Review the flawed PR under time pressure
Take review kata 5 (SQL rewrite) from [04/04](../04-code-reading-gym/04-review-katas.md), 15-minute timer, write the full review. Grade — Solid: blocking findings + graded severities in time. Strong: your review *offered the author a path* (equivalence test + benchmark harness) rather than a wall.

---

The meta-skill across all nine: **sequencing under risk** — every deliverable above is judged more on ordering, rollback, and what-you-left-alone than on the end state. That is also precisely what separates senior system-design answers from mid-level ones ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)).
