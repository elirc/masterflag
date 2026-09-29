# Annotation Drills

For each excerpt: open the anchor, then write **I**nputs, **O**utputs, **D**ependencies, **Inv**ariants, **S**ide effects, **F**ailure modes. Grade with the rubric at the bottom. Don't read the "what to notice" line until you've written yours.

## Drill 1 — [environments/authentication.py#L24-L49](../../api/environments/authentication.py#L24-L49) `EnvironmentKeyAuthentication.authenticate`

What to notice after annotating: output is `(None, None)` — authentication that produces *no user*; the real outputs are request attribute mutations (`request.environment`, `request.originated_from`). Side effect: structured log enrichment. Failure mode: three distinct `AuthenticationFailed` causes indistinguishable to the caller (deliberate — don't leak which).

## Drill 2 — [versioning_service.py#L82-L114](../../api/features/versioning/versioning_service.py#L82-L114) `get_environment_flags_dict`

Notice: `key_function` parameter changes the dedupe grain — callers can collapse per-feature or per-(feature,segment,identity); default at [#L564-L571](../../api/features/versioning/versioning_service.py#L564-L571). Invariant: for each key, result holds the max by `__gt__`. Failure: passing states from mixed environments → `ValueError` from `__gt__`.

## Drill 3 — [features/models.py#L529-L601](../../api/features/models.py#L529-L601) `FeatureState.__gt__`

Notice: raises on cross-environment/cross-feature/cross-identity comparison (guards against caller bugs); v1 path compares `live_from` before `version` with an issue link explaining why (L582-587). Write the decision tree as nested bullets — if yours has fewer than 6 leaves you missed branches.

## Drill 4 — [identities/models.py#L53-L120](../../api/environments/identities/models.py#L53-L120) `Identity.get_all_feature_states`

Notice: `traits` can be passed in (transient evaluation) or loaded; segments evaluated `overrides_only=True` — segments without overrides don't matter here (perf). Transient identities skip the identity-override clause because `self.id` is None (L76-80). Dependency: delegates to the same `get_environment_flags_list` as Flow 1 — one invariant, one home.

## Drill 5 — [audit/models.py#L141-L168](../../api/audit/models.py#L141-L168) `AuditLog.process_environment_update`

Notice: hook condition (`when="environment_document_updated", is_now=True`) — annotation must include *when this runs at all*. Updates environments **individually** with the comment "to avoid deadlock" (L162) — an invariant about lock ordering encoded as a loop. Side effects: DB updates + task enqueue.

## Drill 6 — [webhooks/webhooks.py#L63-L80](../../api/webhooks/webhooks.py#L63-L80) `call_environment_webhooks`

Notice: `settings.DISABLE_WEBHOOKS` early return (silent — failure-visibility question); retries parameterized from settings. Input is `environment_id` not an object — this function is designed to run in a worker (pattern 5).

## Drill 7 — [useFeatureState.ts#L23-L52](../../frontend/common/services/useFeatureState.ts#L23-L52) `getFeatureStates` queryFn

Notice: the wire type honestly encodes `feature_segment: number | null`; enrichment is parallel (`Promise.all`) and unconditional per row; failure of any single enrichment fails the whole query (all-or-nothing). Dependencies: reaches into another service (`getFeatureSegment`) and the raw store — a service-to-service coupling worth flagging.

## Drill 8 — [FeaturesPage.tsx#L99-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L99-L134) the three bridge effects

Notice: effect #2 `cloneDeep`s RTK data into the Flux store — why clone? (Flux consumers mutate; RTK cache must stay immutable. That single line encodes the whole reason the bridge is dangerous.) Effect #3's cleanup unsubscribes — annotate what leaks if it didn't.

## Drill 9 — [features/permissions.py#L88-L131](../../api/features/permissions.py#L88-L131) `FeatureStatePermissions.has_permission`

Notice: reads `environment` from body *or* query params; `isdigit()` guard means non-numeric input falls to `return False` (deny-by-default — good); the create-with-feature_segment branch requires the *stronger* `MANAGE_SEGMENT_OVERRIDES`. Failure mode: `request.data.get("feature")` for tag scoping trusts the body (risk-register #7 — investigate).

## Drill 10 — [audit/tasks.py#L35-L60](../../api/audit/tasks.py#L35-L60) `_create_feature_state_audit_log_for_change_request`

Notice: first lines handle *entity deleted before task ran* (at-least-once world); `is_scheduled` re-enqueue with `delay_until` (pattern 15); `RuntimeError` when the change request is missing — annotate why that's raise-worthy but deletion isn't (one is impossible-by-invariant, the other is a normal race).

---

## Self-grading rubric (per drill)

- **Basic**: correct inputs/outputs; found the side effects.
- **Solid**: named the invariant in one sentence; listed ≥2 realistic failure modes; identified every external dependency.
- **Strong**: your failure modes include one *systemic* one (race, retry, cache, migration-era) and you wrote the regression test you'd add. For drills 3, 5, 10 you also explained the *comment* in the code (deadlock loop, issue link, reschedule) — comments encode the expensive lessons.
