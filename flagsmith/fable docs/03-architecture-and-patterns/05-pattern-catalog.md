# Pattern Catalog

Sixteen cards. The goal is **recognition** — seeing the shape in any codebase and knowing when it's the wrong shape. Format is compressed; every card still answers: problem, shape, real anchors, why it works, failure modes, when to avoid, interview angle, drill.

---

## Pattern 1: Credential→entity cache with negative caching

**Problem:** hot-path auth can't afford a DB hit per request, and invalid keys can be weaponized.
**Shape:** `cache.get(key) → miss: DB lookup → hit: cache entity; unknown keys cached as "bad"`.
**Real:** [environments/models.py#L268-L302](../../api/environments/models.py#L268-L302) (`is_bad_key`/`set_bad_key` at [L489-L503](../../api/environments/models.py#L489-L503)). Second example: flags response cache [features/views.py#L1087-L1107](../../api/features/views.py#L1087-L1107).
**Why it works:** SDK keys change rarely; TTL bounds staleness; negative cache turns key-guessing storms into cache hits.
**Failure modes:** revocation lags TTL (org kill-switch delay — risk #8); cached entity goes stale relative to DB.
**Avoid when:** credentials must revoke instantly (then: cache with explicit invalidation, or check a revocation list).
🎤 "How do you rate-limit/protect auth lookups?" **Drill:** find what invalidates this cache on environment update ([clear_environment_cache](../../api/environments/models.py#L173-L178)) and when it *doesn't* fire.

## Pattern 2: Domain invariant as operator overload

**Problem:** "which of these records wins?" needed identically in N call sites.
**Shape:** define `__gt__` on the entity; callers just use `>`.
**Real:** [features/models.py#L529-L601](../../api/features/models.py#L529-L601); consumed at [versioning_service.py#L111](../../api/features/versioning/versioning_service.py#L111) and [identities/models.py#L115-L120](../../api/environments/identities/models.py#L115-L120).
**Why it works:** single implementation of the priority ladder; comparison is unit-testable in isolation.
**Failure modes:** operator overloads hide complexity (this one raises on cross-environment comparison — surprising for `>`); partial orderings masquerading as total ones.
**Avoid when:** the comparison needs context (already strained here: v1 vs v2 branches inside).
🎤 "Where should business rules live?" **Drill:** list the exceptions `__gt__` can raise and what caller bug each would reveal.

## Pattern 3: Fetch-all-then-resolve-in-app

**Problem:** "latest live winner per feature" is gnarly SQL (greatest-n-per-group) but trivial in a dict pass.
**Shape:** one broad indexed query → single O(n) pass building `dict[key] = max(...)`.
**Real:** [versioning_service.py#L82-L114](../../api/features/versioning/versioning_service.py#L82-L114) with `select_related` batching at [L542-L556](../../api/features/versioning/versioning_service.py#L542-L556).
**Why it works:** avoids window-function SQL portability issues; keeps the invariant in tested Python; row count per environment is bounded in practice.
**Failure modes:** unbounded environments (huge feature × segment counts) blow memory/latency; hidden N+1 if select_related is forgotten.
**Avoid when:** you can't bound the candidate set — then push to SQL (`DISTINCT ON`/window).
🎤 "SQL vs application logic?" **Drill:** write the equivalent `DISTINCT ON` query and name two things it does worse here.

## Pattern 4: Committed-fact fan-out (outbox-shaped eventing)

**Problem:** one write must drive caches, realtime, webhooks, integrations — without bloating the request.
**Shape:** write a durable fact row (AuditLog) in/near the transaction; lifecycle hook enqueues; workers fan out.
**Real:** [audit/models.py#L141-L168](../../api/audit/models.py#L141-L168) → [environments/tasks.py#L31-L44](../../api/environments/tasks.py#L31-L44); integrations via [audit/signals.py](../../api/audit/signals.py).
**Why it works:** the fact is durable and replayable; consumers are independent; write latency stays flat.
**Failure modes:** enqueue outside the commit → lost fan-out on crash (gap vs a true outbox); consumers assume ordering that isn't guaranteed.
**Avoid when:** you need strong read-after-write on the derived state.
🎤 "Cache invalidation at scale." **Drill:** diagram this chain from memory with failure points marked.

## Pattern 5: Task-handler registration decorator

**Problem:** async execution should not change how a function reads.
**Shape:** `handler = register_task_handler()(plain_function)`; call `.delay(args)` to enqueue.
**Real:** [webhooks/tasks.py](../../api/webhooks/tasks.py) wraps service functions; [environments/tasks.py#L26-L31](../../api/environments/tasks.py#L26-L31) with priorities.
**Why it works:** business logic stays importable/testable synchronously; the queue is an implementation detail; ids-not-objects as args force serialization discipline.
**Failure modes:** closure over non-serializable state; forgetting that the worker runs *later* against *changed* data (hence reload-by-id).
**Avoid when:** the caller needs the result — this is fire-and-forget.
🎤 "How do you test async code?" (call the undecorated function synchronously). **Drill:** find one task that reloads its entity and one that would break if the entity was deleted meanwhile (see [audit/tasks.py#L38-L46](../../api/audit/tasks.py#L38-L46) handling exactly that).

## Pattern 6: Immutable version publish (copy-on-write)

**Problem:** edits to live config must be atomic, auditable, schedulable, and revertible.
**Shape:** never mutate live rows: create version → copy states as draft → mutate draft → publish.
**Real:** [versioning_service.py#L141-L198](../../api/features/versioning/versioning_service.py#L141-L198); guard blocking direct writes: [#L20-L37](../../api/features/versioning/versioning_service.py#L20-L37).
**Why it works:** "live" becomes a pointer move; history is free; concurrent editors conflict at publish, not mid-edit.
**Failure modes:** row multiplication; every consumer must resolve "current version" correctly; dual-era code while old mutable path survives.
**Avoid when:** data is huge per version or writes dominate reads.
🎤 "Design config rollback." **Drill:** trace what `publish()` must set for [get_current_live_environment_feature_version](../../api/features/versioning/versioning_service.py#L117-L129) to pick the new version.

## Pattern 7: Discriminated single-table entity

**Problem:** three related concepts (default/segment/identity flag state) share 90% of shape and must be queried together.
**Shape:** one table, nullable discriminator FKs, a `type` property, and uniqueness per role.
**Real:** [features/models.py#L477-L498](../../api/features/models.py#L477-L498) + `type` at [#L623-L636](../../api/features/models.py#L623-L636).
**Why it works:** single evaluation query (Flow 1/3); uniform audit/versioning.
**Failure modes:** illegal states representable (both FKs set — logged as error, not prevented); every query must remember role filters.
**Avoid when:** the variants diverge in shape or lifecycle — then split tables.
🎤 Classic "polymorphism in the database" question. **Drill:** write the check constraint that would forbid identity+segment both set; why might the team not add it? (historic data, migration cost.)

## Pattern 8: Queryset scoping against IDOR

**Problem:** detail endpoints must not leak other tenants' objects.
**Shape:** filter the queryset by the caller's permitted scope *before* `get_object_or_404`.
**Real:** [features/views.py#L992-L1001](../../api/features/views.py#L992-L1001). Second example: identity flags scoped by `environment=self.environment` inside [identities/models.py#L74](../../api/environments/identities/models.py#L74).
**Why it works:** unauthorized == nonexistent (404), no existence leak; impossible to "forget the check after fetch."
**Failure modes:** someone adds a sibling endpoint using `objects.get(uuid=…)` raw.
**Avoid when:** never — this is the default.
🎤 "Prevent IDOR." **Drill:** grep `get_object_or_404` in [api/features/](../../api/features/) and classify each call's scoping.

## Pattern 9: Action→permission map

**Problem:** per-action authorization in class-based views degenerates into if-trees.
**Shape:** a dict from view action to permission constant; methods look up and delegate.
**Real:** [features/permissions.py#L28-L39](../../api/features/permissions.py#L28-L39).
**Why it works:** the whole policy is legible in one screen; adding an action forces a policy decision.
**Failure modes:** unmapped actions fall through to defaults (read L64-65's `return view.detail` carefully); map drifts from custom `@action`s.
**Avoid when:** permissions depend on payload, not action (then it moves to object/domain checks — as segment-override vs default does at [L139-L141](../../api/features/permissions.py#L139-L141)).
🎤 "How do you keep authz auditable?" **Drill:** add a hypothetical `clone` action — which permission, and where else must you touch?

## Pattern 10: Declarative cache invalidation by tags (RTK Query)

**Problem:** after a mutation, which queries are stale?
**Shape:** queries `providesTags`; mutations `invalidatesTags`; the store refetches intersections.
**Real:** [useFeatureState.ts#L20-L29, #L57-L61](../../frontend/common/services/useFeatureState.ts#L20-L61).
**Why it works:** invalidation intent lives next to the endpoint, not scattered through components.
**Failure modes:** missing tag = silent staleness; over-broad tag = refetch storm; custom `queryFn`s bypassing the discipline.
**Avoid when:** truly real-time data (use streams) or cross-tab consistency needs.
🎤 "Cache invalidation on the client." **Drill:** map the full tag graph for `FeatureState`/`FeatureList` across services (grep `'FeatureList'`).

## Pattern 11: Single API slice, endpoints injected per domain

**Problem:** one HTTP client config, many feature areas, code-splitting.
**Shape:** `createApi` once ([service.ts#L63-L69](../../frontend/common/service.ts#L63-L69)); each domain file `injectEndpoints` ([useFeatureState.ts#L19-L24](../../frontend/common/services/useFeatureState.ts#L19-L24)).
**Why it works:** auth/base-url/retry policy defined once; tag namespace shared; bundles stay lean.
**Failure modes:** tag-name collisions across 60+ service files; duplicate endpoint names.
**Avoid when:** genuinely separate backends with different auth (second `createApi`).
🎤 "Structure a large frontend's data layer." **Drill:** count services in [common/services/](../../frontend/common/services/) and find one tag used by more than one file.

## Pattern 12: Strangler-fig bridge with labelled exit

**Problem:** replace Flux with RTK Query without a big-bang rewrite.
**Shape:** new system is source of truth; a bridge feeds legacy consumers; events flow back as refetch triggers; every bridge line carries a TODO naming the exit condition.
**Real:** [FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134).
**Why it works:** ships value during migration; the TODOs make the debt discoverable and finishable.
**Failure modes:** bridges outliving their authors; double-write races; new code accidentally depending on the legacy side.
**Avoid when:** the legacy surface is small enough to cut over atomically.
🎤 THE migration story question. **Drill:** list what must be true to delete each bridge effect (grep who reads `FeatureListStore`).

## Pattern 13: Typed permission gate as render prop

**Problem:** UI must show/hide by permission without sprinkling fetch logic.
**Shape:** `<Permission level permission id>{({permission}) => …}</Permission>` over a cached query; **discriminated-union props** tie `level` to the right permission enum.
**Real:** [Permission.tsx#L27-L48](../../frontend/common/providers/Permission.tsx#L27-L48) (union), used at [FeaturesPage.tsx#L255-L263](../../frontend/web/components/pages/features/FeaturesPage.tsx#L255-L263).
**Why it works:** the type system rejects `level='project'` with an environment permission at compile time.
**Failure modes:** treating it as security (it's UX); waterfalls of permission queries per row (watch the RTK cache save you — verify).
**Avoid when:** hiding is wrong UX (prefer disabled + tooltip — the component supports `showTooltip`).
🎤 "Make illegal states unrepresentable" with a production example. **Drill:** break it deliberately (wrong enum for level) and read the compiler error.

## Pattern 14: Read-your-writes replica routing

**Problem:** replicas scale reads but lag writes; a just-created row may not exist there.
**Shape:** route to replica only when the entity is known to pre-date the request.
**Real:** [identities/views.py#L188-L194](../../api/environments/identities/views.py#L188-L194) — new identities stay on primary; existing ones re-read from replica. Also `from_replica=True` on the flags path ([views.py#L1043-L1047](../../api/features/views.py#L1043-L1047)).
**Why it works:** encodes the consistency decision at the exact point of knowledge.
**Failure modes:** the *next* request for the new identity may still hit a lagging replica; every new call site must remember the rule.
**Avoid when:** you can pin sessions or use causal consistency primitives instead.
🎤 "Replication lag" with a concrete fix. **Drill:** find another `using_database_replica` call site and judge whether it's lag-safe.

## Pattern 15: Self-rescheduling scheduled task

**Problem:** "do X at time T" where T can move after scheduling.
**Shape:** task fires, checks the *current* T; if moved, re-enqueues itself with `delay_until=T` and exits.
**Real:** [audit/tasks.py#L38-L60](../../api/audit/tasks.py#L38-L60) (also handles the entity being deleted meanwhile).
**Why it works:** no scheduler mutation needed; the check-then-act happens at fire time against fresh data.
**Failure modes:** clock skew; duplicate enqueues (needs idempotent effect); infinite reschedule loops if T keeps moving.
**Avoid when:** high-precision scheduling (this is queue-granularity).
🎤 "Design scheduled/delayed jobs." **Drill:** what happens if the change request is rescheduled *twice* before first fire? Trace it.

## Pattern 16: Dogfooding flags behind an abstraction

**Problem:** the flag platform needs flags for its own rollouts — without circular runtime dependency.
**Shape:** OpenFeature SDK + local-evaluation provider + synced cache; org-scoped evaluation context.
**Real:** [integrations/flagsmith/client.py#L31](../../api/integrations/flagsmith/client.py#L31) `get_openfeature_client`; usage guidance in [api/README.md](../../api/README.md).
**Why it works:** vendor-neutral interface; local evaluation means no network dependency on itself.
**Failure modes:** stale local cache masking a kill switch; flags checked in hot paths without defaults.
**Avoid when:** a config file would do — flags are for *runtime* variation.
🎤 "How would you roll out a risky backend change?" **Drill:** find one `get_boolean_value` call site (grep) and identify its default and blast radius if the flag service is unreachable.
