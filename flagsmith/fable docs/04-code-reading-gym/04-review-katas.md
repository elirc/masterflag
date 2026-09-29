# Review Katas

Nine fake PRs. For each: read the intent and diff summary, list findings graded **Blocking / Important / Optional**, then compare with the expected findings. Write your comments as you would post them — kind, specific, with a suggested path. The referenced files tell you what the diff "resembles" so you can check house style.

## Kata 1: "Add copy-flag-to-environment button"

Author intent: one-click copy of a feature state to another environment.
Fake diff summary: new FE button calls a new endpoint `POST /features/{id}/copy/`; DRF view fetches target environment by id from the body, clones the FeatureState, returns 200. No permission class beyond `IsAuthenticated`; no test for cross-project targets.
Files this resembles: [features/views.py](../../api/features/views.py), [features/permissions.py](../../api/features/permissions.py).
Expected — **Blocking:** target environment not permission-checked (cross-tenant write!); missing `UPDATE_FEATURE_STATE` on target. **Important:** no v2-versioning branch — direct clone will raise in v2 environments ([require_direct_state_write](../../api/features/versioning/versioning_service.py#L20-L37)); missing tests incl. cross-org rejection. **Optional:** endpoint naming vs existing `clone` conventions.
Good comment example:
> The copy writes into `target_env` but we only check the *source* project's permission. A user with access to project A could write into project B by id. Can we scope the target lookup like `get_feature_state_by_uuid` does (features/views.py#L995-L999) and add a cross-org 404 test?

## Kata 2: "Speed up features page"

Intent: memoize everything on FeaturesPage.
Fake diff: wraps every callback in `useCallback` with `[]` deps; wraps `projectFlags` in `useMemo` with `[data]`; removes the Flux-bridge `refetch` effect "because it causes extra renders."
Resembles: [FeaturesPage.tsx#L121-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L121-L134).
Expected — **Blocking:** removing the bridge effect breaks refresh-after-create while CreateFlag still writes via Flux. **Blocking:** `[]` deps create stale closures over `filters`/`page`. **Important:** memoization without a measured render problem; virtualized list already bounds row work. **Optional:** propose profiling first (React DevTools Profiler).

## Kata 3: "Return 400 for missing identifier" (SDK endpoint)

Intent: correctness fix on [identities/views.py#L170-L173](../../api/environments/identities/views.py#L170-L173).
Fake diff: changes the 200+detail to 400.
Expected — **Blocking:** breaking change to a deployed SDK fleet; the TODO exists *because* of compat. Needs: SDK-behaviour survey, versioning/deprecation plan, or content negotiation. **Important:** if pursued, must update OpenAPI + docs + all first-party SDK expectations. **Optional:** add a metric counting missing-identifier requests to size the blast radius first — turn an argument into data.

## Kata 4: "Add `created_by` to webhook payload"

Intent: enrich environment webhook payloads with the acting user's email.
Fake diff: adds `user.email` into the payload built in [webhooks/webhooks.py](../../api/webhooks/webhooks.py); reads `request.user` inside the task.
Expected — **Blocking:** tasks run in a worker with no request; `request.user` doesn't exist there — pass the id at enqueue time (see args-by-id convention, [environments/tasks.py](../../api/environments/tasks.py)). **Important:** payload shape is a public contract — additive is OK but document + test the serializer; PII (email) in webhooks may need config gating. **Optional:** structured log event for the change.

## Kata 5: "Refactor: move flag resolution into SQL"

Intent: replace the Python `__gt__` pass with `DISTINCT ON` for performance.
Fake diff: rewrites [get_environment_flags_dict](../../api/features/versioning/versioning_service.py#L82-L114) as raw SQL; deletes `__gt__`.
Expected — **Blocking:** `__gt__` has other call sites ([identities/models.py#L115-L120](../../api/environments/identities/models.py#L115-L120)) — deleting it breaks identity resolution. **Blocking:** no benchmark demonstrating the problem. **Important:** raw SQL bypasses soft-delete managers and v1/v2 branching; portability (CockroachDB is a supported backend per [pyproject.toml](../../api/pyproject.toml)). **Optional:** if perf is real, propose keeping both behind a flag with an equivalence test. Senior lesson: performance PRs earn merge with *measurements and equivalence proofs*, not elegance.

## Kata 6: "Add feature description tooltip" (the good PR)

Intent: show `feature.description` on hover in the features list.
Fake diff: small component change, uses existing Tooltip, reads an already-fetched field, adds one Jest test.
Expected — **Blocking:** none. **Important:** none. **Optional:** accessibility (is the tooltip keyboard-reachable?), truncation for long descriptions.
The lesson: say so! "LGTM, one optional a11y thought" is a complete review. Manufacturing findings on clean PRs is a junior tell.

## Kata 7: "Cache identity flags in Redis for 60s"

Intent: cut DB load on `/api/v1/identities/`.
Fake diff: wraps `get_all_feature_states` in a cache keyed on `identity.id`.
Expected — **Blocking:** key ignores traits sent with the request — transient trait evaluation returns another user's cached result… actually worse: stale *for the same user* after trait changes; and transient identities have no id (key collision on `None`). **Important:** invalidation story absent (trait writes, flag changes, segment edits all invalidate); interaction with existing `GET_IDENTITIES_ENDPOINT_CACHE_SECONDS` HTTP cache ([identities/views.py#L158-L163](../../api/environments/identities/views.py#L158-L163)) — double caching. **Optional:** metric for hit rate.

## Kata 8: "Tidy: remove unused `version` field"

Intent: delete the deprecated `FeatureState.version` ([models.py#L522-L523](../../api/features/models.py#L522-L523)).
Fake diff: drops the column + migration.
Expected — **Blocking:** v1 comparison logic still reads it ([models.py#L581-L597](../../api/features/models.py#L581-L597)); "deprecated" ≠ "unused." **Important:** migration on the largest table needs lock analysis; historical tables reference it. **Optional:** propose the actual deprecation plan (v2 completion first). Lesson: `grep` before you delete, and a TODO is not a work order.

## Kata 9: "Add loading spinner to Permission component"

Intent: show spinner while permissions load instead of hiding content.
Fake diff: [Permission.tsx](../../frontend/common/providers/Permission.tsx) renders `<Spinner/>` when `isLoading`.
Expected — **Blocking:** none functionally, but — **Important:** this component wraps *every row* in lists ([FeaturesPage.tsx#L255](../../frontend/web/components/pages/features/FeaturesPage.tsx#L255)); a spinner per row on first paint is UX regression at scale; render-prop consumers already receive `isLoading` to decide contextually. **Optional:** skeletons at the list level (the page already has [FeatureRowSkeleton](../../frontend/web/components/pages/features/FeaturesPage.tsx#L250-L252)).
Lesson: a component's *call-site cardinality* is part of its design context.

---

**Grading yourself.** Per kata — Basic: found the top Blocking issue. Solid: findings correctly *graded* (severity inflation is a real reviewer failure) and each comment names a path forward. Strong: you caught the second-order issues (other call sites, contract status, cache-key semantics, call-site cardinality) and kata 6's "approve it" answer. Language check: every Blocking comment you wrote contains a question mark or an offered alternative — firm on substance, soft on delivery ([07/01](../07-career-and-collaboration/01-code-review-mindset.md)).
