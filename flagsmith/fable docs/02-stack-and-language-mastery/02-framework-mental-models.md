# Framework Mental Models

## React: render is a function call, commit is a side effect

Model: components are functions producing descriptions of UI; React may call them often and must be able to throw the result away (render phase), then applies diffs to the DOM (commit) and runs effects after. Everything else — hooks rules, purity, memoization — falls out of this.

Anchored in this repo:

- **State placement & derivation.** [FeaturesPage.tsx#L82-L86](../../frontend/web/components/pages/features/FeaturesPage.tsx#L82-L86) derives `effectiveFilters` with `useMemo` instead of storing it — derived data is computed, not synchronized. The page keeps *server* state in RTK Query and only view concerns (filters, page) in local/router state. That separation is the single most transferable frontend lesson here.
- **Effects are for external systems, not data flow.** The three effects at [FeaturesPage.tsx#L99-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L99-L134) all talk to an external system (the legacy Flux store). None compute data. Good hygiene even in awkward circumstances.
- **Memoized callbacks as row renderers.** [renderFeatureRow](../../frontend/web/components/pages/features/FeaturesPage.tsx#L248-L291) is `useCallback`-wrapped with a real dependency list feeding a virtualized list ([react-virtualized](../../frontend/package.json)) — memoization with a measurable purpose (row identity for list perf), not cargo cult.
- **Keys.** `key={projectFlag.id}` (L256) — entity id, not index, because rows reorder under filtering.
- **Stale-data UX.** `const isDataStale = !!data && !currentData` ([L95](../../frontend/web/components/pages/features/FeaturesPage.tsx#L95)) — RTK Query keeps last data while refetching; the page surfaces it as a loading hint. Know the `data` vs `currentData` distinction; it's a favourite senior-screen question.

Sharp edges here: no Server Components, no Suspense-for-data — this is a classic SPA. React 19 with `react-router` v5 and some class-era libraries; expect mixed idioms.

## The data-fetching layer: RTK Query

Model: a normalized **cache keyed by endpoint+args**, with declarative invalidation. `providesTags` labels what a query holds; `invalidatesTags` on a mutation marks labels dirty; dirty queries with mounted subscribers refetch. That's the entire magic.

- Base service: [service.ts#L63-L69](../../frontend/common/service.ts#L63-L69) — one `createApi`, endpoints injected per domain file (`injectEndpoints`, [useFeatureState.ts#L19-L69](../../frontend/common/services/useFeatureState.ts#L19-L69)) — code-splitting-friendly and keeps one cache.
- Auth: `prepareHeaders` ([service.ts#L23-L40](../../frontend/common/service.ts#L23-L40)) attaches the token except on auth endpoints — cross-cutting concern solved once.
- Escape hatch: `queryFn` ([useFeatureState.ts#L30-L51](../../frontend/common/services/useFeatureState.ts#L30-L51)) for composite requests — powerful, and where the N+1 fan-out lives. Custom `queryFn`s bypass the simple declarative path; treat them as review hot-spots.
- Imperative dispatch wrappers ([useFeatureState.ts#L71-L93](../../frontend/common/services/useFeatureState.ts#L71-L93)) let non-hook code (legacy stores) call endpoints — a migration affordance.

Failure modes: missing `invalidatesTags` → stale UI with no error; too-broad tags → refetch storms; forgetting `.unwrap()` → silently ignored mutation failures.

## The legacy layer: Flux stores

[frontend/common/stores/](../../frontend/common/stores/) holds a 2015-vintage Flux implementation (dispatcher + EventEmitter stores; `feature-list-store.ts` is 1,061 lines). The migration is *in progress and visible*: [FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134) copies RTK data **into** `FeatureListStore` for legacy consumers (CreateFlag modal) and refetches RTK when Flux emits `saved`/`removed`. Every TODO there names the exit condition ("Remove when CreateFlag is migrated").

Transferable lesson: strangler-fig migrations need (a) a bridge, (b) labelled exit criteria, (c) one direction of truth per interaction. Judge this bridge against (c): reads flow RTK→Flux, writes flow Flux→RTK-refetch — acceptable, but only because it's temporary.

## Django/DRF survival guide for the JS engineer

Map it to what you know (Express/Nest analogies):

| Django/DRF | Closest JS idea | Example here |
| --- | --- | --- |
| `urls.py` + routers | Express router / Nest controllers' routes | [api/features/urls.py](../../api/features/urls.py) |
| ViewSet / APIView | Controller class | [SDKFeatureStates](../../api/features/views.py#L1004) |
| Serializer | zod/DTO + mapper in both directions | [api/features/serializers.py](../../api/features/serializers.py) |
| Permission class | Route guard/middleware | [features/permissions.py](../../api/features/permissions.py) |
| Model + manager | ORM entity + repository | [features/models.py](../../api/features/models.py) |
| `services.py` | Domain service layer | [features_service.py](../../api/features/features_service.py) |
| signals / lifecycle hooks | Event emitter on entity lifecycle | [audit/models.py#L131-L168](../../api/audit/models.py#L131-L168) |
| `tasks.py` + processor | BullMQ worker | [environments/tasks.py](../../api/environments/tasks.py) |
| pytest fixtures | test factories/DI | [api/tests/conftest.py](../../api/tests/conftest.py) |

Reading protocol for any DRF endpoint: **url → view (auth_classes, permission_classes, serializer_class) → serializer (validation) → model/service (logic) → tasks (side effects)**. That order answers 90% of "where does X happen?"

🎤 **Interview angle** (cards in [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md)):
1. "Server state vs client state" → RTK Query owns server state; the page keeps only view state.
2. "How does tag invalidation work?" → toggle flow tags, verbatim.
3. "Tell me about a legacy migration" → the Flux bridge is a ready-made STAR story ([08/06](../08-interview-prep/06-behavioral-star-stories.md)).
4. "How do you approach an unfamiliar backend stack?" → the reading protocol above.

**Drill:** pick [useEnvironment.ts](../../frontend/common/services/useEnvironment.ts) (not covered in this curriculum), and write down: endpoints, tags provided/invalidated, and which components would refetch after `updateEnvironment`. Basic: listed endpoints. Solid: tag graph correct. Strong: found a consumer via grep and predicted its refetch behaviour.
