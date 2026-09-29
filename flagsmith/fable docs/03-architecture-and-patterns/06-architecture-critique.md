# Architecture Critique

An honest assessment — the file to reread the night before a system-design interview (cross-linked from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md)). Confirmed observations are anchored; everything speculative says so.

## Strongest design choices

1. **The committed-fact fan-out** ([audit/models.py#L141-L168](../../api/audit/models.py#L141-L168) → tasks). Write latency stays flat while consumers multiply; the audit requirement and the invalidation trigger are the *same row*, so the compliance feature funds the architecture. This is the repo's best idea.
2. **Priority resolution as tested domain code** (`__gt__` + fetch-then-resolve, patterns 2–3). The single most business-critical rule has a single implementation shared by all read paths ([flags](../../api/features/views.py#L1043), [identities](../../api/environments/identities/models.py#L96-L120)).
3. **Immutable versioning (v2)** — rollback as pointer-move, drafts, scheduling ([versioning_service.py#L141-L198](../../api/features/versioning/versioning_service.py#L141-L198)); with an explicit guard making the old mutable path illegal in v2 environments ([#L20-L37](../../api/features/versioning/versioning_service.py#L20-L37)).
4. **Layered read-path caching with credential isolation** — env cache + flags cache + HTTP cache, cache keys split by request origin ([views.py#L1093-L1094](../../api/features/views.py#L1093-L1094)), negative caching for bad keys.
5. **Quality machinery as policy**: mypy strict + 100% diff coverage + missing-migration CI + OpenAPI diff — correctness is enforced mechanically, not by heroics ([api-pull-request.yml](../../.github/workflows/api-pull-request.yml)).
6. **Deployment simplicity for self-hosters** — Postgres-backed queue, one compose file, no broker ([docker-compose.yml](../../docker-compose.yml)).

## Risks and tradeoffs (the other side of each coin)

1. **Eventual consistency everywhere post-write.** SDK caches/documents/realtime lag by design; TTL knobs decide the staleness window. There's no dashboard "your change is fully propagated" signal (**hypothesis** — not found; verify before claiming).
2. **Dual-era problem, twice.** Backend: versioning v1 vs v2 forks every write path ([update_flag](../../api/features/versioning/versioning_service.py#L132-L138)); frontend: Flux vs RTK Query with a live bridge ([FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134)). Migrations are managed (guards, TODOs) but every feature costs ~2× while both live. The v1/v2 branch leaking into client hooks ([useToggleFeatureWithToast.ts#L37](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L37)) is the sharpest edge.
3. **God-model gravity.** `Environment` accretes caching, document building, Dynamo, metrics querysets ([environments/models.py#L80-L620](../../api/environments/models.py#L80)); the declared `services.py` layer is newer than much of the code.
4. **Hand-maintained FE contract** (`Res`/`Req`) with no codegen against the OpenAPI the backend already produces — drift compiles ([03-type-system-and-contracts](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
5. **Authorization spread** across map/methods/view-filters — correct today, but every new endpoint re-derives the recipe ([key flow 5](../01-codebase-cartography/05-key-flows.md)).
6. **Client-side N+1** in the feature-state service ([useFeatureState.ts#L42-L44](../../frontend/common/services/useFeatureState.ts#L42-L44)) — risk-register #1.
7. **Frozen contract debt on SDK endpoints** — 200-for-missing-identifier, header gaps for identity overrides ([identities/views.py#L170-L173, #L208-L212](../../api/environments/identities/views.py#L170-L212)) — the price of a deployed SDK fleet; changing it needs API versioning strategy, not a patch.

## Prioritized improvements — "owning this for 3 months"

| # | Change | Why first | Migration path | Test strategy |
| --- | --- | --- | --- | --- |
| 1 | Finish the CreateFlag→RTK migration and delete the Flux bridge | Highest bug-surface-to-effort ratio; exit criteria already written in TODOs | Migrate CreateFlag's reads/writes to services; delete bridge effects; then delete dead store methods | E2E flag suites ([flag-tests.pw.ts](../../frontend/e2e/tests/flag-tests.pw.ts)) as the safety net; add unit tests on new service endpoints |
| 2 | Server-side embed/batch for feature-segment data (kill client N+1) | User-visible latency; small API change | Add serializer field or `include=feature_segment` param; FE consumes it behind a fallback | API: serializer test; FE: mock-count requests before/after |
| 3 | Generate `Res`/`Req` from OpenAPI | Removes a whole drift class | Codegen script + CI diff (mirror backend's docs-diff pattern); adopt file-by-file | Typecheck is the test; start with one service |
| 4 | Consolidate an endpoint-authoring recipe (cookbook or base class) for scoping+permissions | Prevents the next IDOR, cheaply | Document → lint rule or shared mixin | Permission test template ([05-quality-engineering/02](../05-quality-engineering/02-writing-tests-here.md)) |
| 5 | Propagation visibility: expose "environment document updated_at vs last change" | Turns silent staleness into an observable | Metric + dashboard field; no schema change | Unit on the comparison; manual staging validation |

Each is deliberately **small-blast-radius**: additive, behind existing test nets, reversible by revert.

## What I would *not* change

The Postgres task queue (right for the self-host constraint), fetch-then-resolve evaluation (bounded and tested), the single-table FeatureState (the alternative is worse in this domain), soft deletes (audit product requirement).

🎤 **Interview transfer:** this file *is* the answer to "critique a system you know well." Structure to reuse: 3 strengths with mechanisms → 3 risks with evidence → top-2 changes with migration paths → what you'd leave alone (that last section is what makes it senior — restraint, cost awareness).

**Drill:** pick improvement #2 and write its one-page RFC using the template in [07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). Grade: Basic — problem+proposal. Solid — rollout+rollback. Strong — named who pays (SDK teams? self-hosters?) and an explicit non-goal.
