# Type System and Contracts

A **contract** is a shape two parties agree on across a boundary. In this repo the API↔frontend contract is *hand-maintained TypeScript*, and that decision drives everything in this file.

## The contract registry pattern

- [responses.ts#L1090](../../frontend/common/types/responses.ts#L1090) `export type Res = { … }` and [requests.ts#L143](../../frontend/common/types/requests.ts#L143) `export type Req = { … }` — two giant lookup types, one key per endpoint. Services then declare `builder.query<Res['featureStates'], Req['getFeatureStates']>` ([useFeatureState.ts#L25-L28](../../frontend/common/services/useFeatureState.ts#L25-L28)).
- Why it's good: one place to look for any endpoint's shape; indexed-access types (`Res['featureState']`) keep service signatures terse; adding an endpoint forces you to name its contract.
- Why it's risky: **nothing machine-checks these against the Django serializers.** The backend generates OpenAPI (drf-spectacular, CI-diffed), but the frontend types are written by hand. Contract drift compiles fine and fails at runtime. A senior contribution here: generate `Res`/`Req` from [sdk/openapi.yaml](../../sdk/openapi.yaml) — see [06-contribution-practice/03](../06-contribution-practice/03-senior-build-projects.md).

## Generics that earn their keep

- [responses.ts#L11-L16](../../frontend/common/types/responses.ts#L11-L16) `PagedResponse<T>` — the DRF pagination envelope typed once, reused everywhere; `EdgePagedResponse<T>` (L3) extends it. This is the generics interview answer: *parametrize the envelope, not each payload*.
- Transformation types in service `queryFn`s: [useFeatureState.ts#L34-L38](../../frontend/common/services/useFeatureState.ts#L34-L38) types the wire shape as `Omit<FeatureState, 'feature_segment'> & { feature_segment: number | null }` before enriching it to the app shape — an honest encoding of "the API returns an id; the app wants the object." When wire shape ≠ app shape, *say so in the types*.

## Narrowing, guards, and the `any` frontier

- User-defined type guard: [FeaturesPage.tsx#L45-L49](../../frontend/web/components/pages/features/FeaturesPage.tsx#L45-L49) `isSkeletonItem(item): item is SkeletonItem` — the honest way to mix heterogeneous list items.
- Non-null assertions: `routeContext.projectId!` ([FeaturesPage.tsx#L64-L65](../../frontend/web/components/pages/features/FeaturesPage.tsx#L64-L65)) — a bet that routing guarantees presence. Each `!` is an unchecked invariant; know what enforces it (here: route nesting) and what happens when it's wrong (runtime undefined, not a compile error).
- `any` frontier: legacy stores and older utils are loosely typed (`store: any` in [useFeatureState.ts#L72](../../frontend/common/services/useFeatureState.ts#L72)); typed islands (services, types/) are kept strict. House policy is *improve types when working nearby* ([frontend/README.md](../../frontend/README.md)). `unknown` vs `any` in one line: `unknown` makes you prove the shape before use; `any` switches the checker off and is contagious through inference.
- `satisfies` is essentially absent here (grep finds none in `common/`) — fine: it matters when you need a value checked against a type *without widening*; mention it as a tool, don't force it.

## The other half of the contract: mypy-strict Django

The backend is fully type-checked in strict mode with typed serializers/pydantic dataclasses ([api/README.md](../../api/README.md)); e.g. `get_environment_flags_list(...) -> list[FeatureState]` ([versioning_service.py#L51-L60](../../api/features/versioning/versioning_service.py#L51-L60)). So both sides are internally typed — the untyped gap is precisely the HTTP boundary. Being able to say that sentence is a mid-level marker.

## Failure modes checklist

- Hand-written response type diverges from serializer → runtime `undefined`, no compile error.
- `Omit`+intersection transformations drift from the actual enrichment code.
- `!` assertions on route params break when a component is reused under a different route.
- Enum-ish string unions duplicated FE/BE (e.g. feature types, permission names) — grep both sides before renaming anything.

🎤 **Interview angle** (cards in [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) & [03](../08-interview-prep/03-api-and-data-modeling-questions.md)):
1. "How do you type API responses?" → the `Res`/`Req` registry + its drift risk + codegen as the fix.
2. "Generics example that isn't `identity<T>`?" → `PagedResponse<T>`.
3. "`unknown` vs `any`?" → the `store: any` frontier and containment strategy.
4. "How do you keep FE/BE contracts honest?" → OpenAPI diff-check on the backend; missing codegen on the frontend; what you'd add.

**Drill:** find `FeatureState` in [responses.ts](../../frontend/common/types/responses.ts) and diff its fields against the DRF serializer used by `SDKFeatureStates`. List any mismatches (there may be none — the exercise is the method). Basic: found both. Solid: field-by-field diff. Strong: proposed where a codegen step would slot into CI and what it would break first.
