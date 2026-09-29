# Boundaries and Layers

A **boundary** is where responsibility and trust change hands; a layer is a boundary stacked horizontally. The test of a boundary is what it *refuses* to know.

## The declared layering (backend)

Per [api/README.md](../../api/README.md), each Django app aims for: `urls → views (HTTP) → serializers (validation/shape) → services (business logic) → models (persistence) → tasks (async side effects)`, with `mappers/dataclasses/types` as pure-data helpers.

What each layer owns / must not own:

| Layer | Owns | Must not own | Good example | Leak example |
| --- | --- | --- | --- | --- |
| views | HTTP: status codes, caching headers, request→args | domain rules | [SDKFeatureStates.get](../../api/features/views.py#L1026-L1073) delegates to the service | `_additional_filters` ([views.py#L1075-L1085](../../api/features/views.py#L1075-L1085)) embeds the *authorization-ish* rule "server-only flags hidden from client keys" in the view — works, but the rule is invisible to other callers |
| serializers | shape + field validation | writes with side effects | — | DRF culture allows `create()` overrides doing real logic; when you see one, ask where its tests are |
| services | domain logic, invariants | HTTP objects | [versioning_service.update_flag](../../api/features/versioning/versioning_service.py#L132-L138) takes entities + a dataclass `FlagChangeSet`, returns entities — cleanly transport-agnostic | |
| models | persistence, small invariant helpers | orchestration | `FeatureState.__gt__` ([models.py#L529](../../api/features/models.py#L529)) — invariant *belongs* to the entity | `Environment` ([models.py#L80-L620](../../api/environments/models.py#L80)) also builds documents, manages caches, knows Dynamo — a god-model accreting cross-layer duties |
| tasks | async orchestration | request/user context beyond ids | [environments/tasks.py](../../api/environments/tasks.py) passes ids, reloads entities — correct, because task args are serialized | |

The **service layer here is aspirational, not universal** — plenty of logic still sits in views/models from earlier eras. Reading the gap between a codebase's stated architecture (their README) and its median file is a senior skill; do it respectfully.

## Frontend layering

`web/components/pages` (route-level) → `components` (reusable) → `common/services` (RTK Query, the only place HTTP is allowed — "NO FETCH" rule in [frontend/CLAUDE.md](../../frontend/CLAUDE.md)) → `common/types` (contract). Legacy `common/stores` sits beside this as a parallel, deprecated layer with an explicit bridge ([FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134)).

Boundary strength worth copying: *all* HTTP behind services means auth headers, error envelopes, and caching policy have exactly one home ([service.ts#L23-L57](../../frontend/common/service.ts#L23-L57)).

## The load-bearing cross-cutting boundary: sync vs async

The request path may only do: validate → authorize → write → record history. Everything else (documents, SSE, webhooks, integrations) happens post-commit via AuditLog hooks + tasks ([audit/models.py#L141-L168](../../api/audit/models.py#L141-L168)). This is the repo's best boundary: it keeps write latency flat as integrations multiply. Its cost: **eventual consistency** — every consumer of derived state (SDK caches, realtime clients) must tolerate a propagation window.

## Boundary leaks to learn from

1. **Versioning branch in the client.** Frontend hooks branch on `use_v2_feature_versioning` ([useToggleFeatureWithToast.ts#L37](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L37)) — a server migration detail became client API surface. Every SDK/UI must now know both write shapes.
2. **Wire-shape enrichment in the FE service.** [useFeatureState.ts#L7-L18](../../frontend/common/services/useFeatureState.ts#L7-L18) patches `feature_segment: number` into an object client-side — the API's normalized shape leaks into browser round-trips (risk-register #1).
3. **Permission knowledge split four ways** (map, two permission methods, view queryset filtering — see [key flow 5](../01-codebase-cartography/05-key-flows.md#flow-5-authorization--from-permission-to-a-drf-permission-class-security-boundary)).

🎤 **Interview angle:** "Describe a well-layered backend" — use the declared layering plus ONE leak and its cost; naming a leak is what separates mid from junior. "Where should business logic live?" — services, and *why*: transport-agnostic = testable without HTTP, composable from tasks and views alike.

**Drill:** pick [api/segments/](../../api/segments/) and classify each module into the layer table, flagging anything that sits in the wrong layer. Basic: table filled. Solid: one leak candidate with evidence. Strong: a migration note for the leak — where the logic should move, what tests pin it during the move.
