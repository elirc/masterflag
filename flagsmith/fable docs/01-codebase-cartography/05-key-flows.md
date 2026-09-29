# Key Flows

Six end-to-end traces. The rest of the curriculum rotates examples across these — learn them and you own the repo's load-bearing paths. Each uses the same template; do the drill before moving on.

---

## Flow 1: An SDK fetches environment flags (`GET /api/v1/flags/`)

Why this flow matters: it's the product's hot path — every client app in the world hits it, so it carries the repo's most serious caching and read-scaling decisions.

Open these files first:
- [api/environments/authentication.py#L14-L49](../../api/environments/authentication.py#L14-L49) — the entire auth boundary
- [api/features/views.py#L1004-L1107](../../api/features/views.py#L1004-L1107) — the view
- [api/features/versioning/versioning_service.py#L51-L114](../../api/features/versioning/versioning_service.py#L51-L114) — evaluation

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | DRF auth | [authentication.py#L24-L41](../../api/environments/authentication.py#L24-L41) | `X-Environment-Key` → `Environment` from cache; request tagged `originated_from` CLIENT/SERVER by `ser.` prefix | header string → `Environment` | Stale cache serves a disabled org until TTL (kill switch at L33) |
| 2 | Cache | [models.py#L268-L302](../../api/environments/models.py#L268-L302) | Miss → union query over embedded + FK api keys; unknown keys negative-cached (L273, L292) | api_key → Environment | Negative cache protects DB from key-guessing storms |
| 3 | View | [views.py#L1019-L1025](../../api/features/views.py#L1019-L1025) | `cache_page` HTTP cache, `vary_on_headers` on the env key | — | Two cache layers below already — know all three TTLs |
| 4 | View | [views.py#L1075-L1085](../../api/features/views.py#L1075-L1085) | `_additional_filters`: no segment/identity rows; hide-disabled option; **server-only flags stripped for client keys** | `Q` objects | This is an authorization decision disguised as a filter |
| 5 | Service | [versioning_service.py#L82-L114](../../api/features/versioning/versioning_service.py#L82-L114) | One query for all live candidate states (replica, `select_related` at L542-556), then Python keeps the max per feature via `__gt__` | QuerySet → `dict[key, FeatureState]` | O(n) rows scanned per request when uncached |
| 6 | Model | [features/models.py#L529-L601](../../api/features/models.py#L529-L601) | Priority ladder: identity > segment (by `FeatureSegment` priority) > default; v2 compares `EnvironmentFeatureVersion`, v1 compares `live_from` | comparison | Central invariant; C901-complex |
| 7 | Response | [views.py#L1069-L1073](../../api/features/views.py#L1069-L1073) | Serialized list + `x-flagsmith-document-updated-at` header | JSON array | Header drives SDK freshness decisions |

Validation and authorization: no user — the environment key **is** the credential ([authentication.py#L24-L34](../../api/environments/authentication.py#L24-L34)); server-key-only flag filtering at [views.py#L1082-L1083](../../api/features/views.py#L1082-L1083); cache key isolates client vs server responses ([views.py#L1093-L1094](../../api/features/views.py#L1093-L1094)) — a cache-poisoning defense worth quoting.

Persistence and side effects: pure read; optionally from a replica (`from_replica=True`, [versioning_service.py#L538-L540](../../api/features/versioning/versioning_service.py#L538-L540)).

Tests that cover it: [api/tests/unit/features/test_unit_features_views.py](../../api/tests/unit/features/test_unit_features_views.py), SDK schema tests in [api/tests/integration/sdk/](../../api/tests/integration/sdk/).

What juniors usually miss: the three distinct caches (environment cache, flags cache, HTTP cache) with independent TTLs and invalidation stories.
What seniors notice: priority resolution deliberately lives in Python, not SQL — it trades DB cleverness for testable domain logic, and the `__gt__` overload makes the invariant portable to every caller.

🎤 Interview angle: "Design a read-heavy config API" — answer with this exact layering: credential→entity cache, negative caching, response cache varying on credential, replica reads, and one honest tradeoff (staleness window).

Drill: with all caches disabled in your head, count the SQL queries for one request. Then explain what each cache eliminates. Self-grade — Basic: named the caches. Solid: got the query shape (one big fetch + select_related). Strong: explained the client-vs-server cache-key isolation and what bug it prevents.

---

## Flow 2: A dashboard user toggles a flag (UI write path)

Why this flow matters: the canonical cross-layer write — React event → RTK Query mutation → DRF permissions → versioning branch — and it exposes the repo's living legacy migration.

Open these files first:
- [FeaturesPage.tsx#L150-L165](../../frontend/web/components/pages/features/FeaturesPage.tsx#L150-L165) — toggle callback
- [useToggleFeatureWithToast.ts#L23-L78](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L23-L78) — the branch
- [useFeatureState.ts#L53-L67](../../frontend/common/services/useFeatureState.ts#L53-L67) — the mutation

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | [FeaturesPage.tsx#L255-L263](../../frontend/web/components/pages/features/FeaturesPage.tsx#L255-L263) | Row wrapped in `<Permission level='environment'>` — UX gate only | render prop `{ permission }` | Client checks are advisory; server must re-check |
| 2 | Hook | [useToggleFeatureWithToast.ts#L37-L59](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L37-L59) | v2 env → `createAndSetFeatureVersion`; else PUT featurestate | `FeatureState` | Two write paths to keep behaviourally identical |
| 3 | Service | [useFeatureState.ts#L53-L67](../../frontend/common/services/useFeatureState.ts#L53-L67) | Mutation + `invalidatesTags` → dependent queries refetch | `Req['updateFeatureState']` | Wrong tags = stale UI, no error |
| 4 | API authz | [features/permissions.py#L133-L151](../../api/features/permissions.py#L133-L151) | Object permission: `UPDATE_FEATURE_STATE` or `MANAGE_SEGMENT_OVERRIDES` (+tag scoping) | request.user + obj | The real boundary |
| 5 | Write | v1: FeatureState PUT; v2: [versioning_service.py#L141-L198](../../api/features/versioning/versioning_service.py#L141-L198) | v2 creates a **new version**, mutates its draft states, publishes | new EFV row + cloned FeatureStates | Direct writes blocked in v2 ([L20-L37](../../api/features/versioning/versioning_service.py#L20-L37)) |
| 6 | History | django-simple-history on save | Historical record → AuditLog (Flow 4 takes over) | history rows | — |
| 7 | UI refresh | [FeaturesPage.tsx#L121-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L121-L134) | Legacy Flux `saved` events also force RTK `refetch()` — the bridge | — | Double-source-of-truth until CreateFlag migrates |

Validation and authorization: serializer validation on body; object-level permission at step 4; note the UI permission (step 1) and API permission (step 4) are **separately maintained** — drift between them is a real bug class.

Persistence and side effects: FeatureState/EFV writes; then the full async fan-out of Flow 4.

Tests that cover it: [test_unit_features_permissions.py](../../api/tests/unit/features/test_unit_features_permissions.py), [versioning tests](../../api/tests/unit/features/versioning/test_unit_versioning_versioning_service.py), E2E [flag-tests.pw.ts](../../frontend/e2e/tests/flag-tests.pw.ts) and [versioning-tests.pw.ts](../../frontend/e2e/tests/versioning-tests.pw.ts).

What juniors usually miss: `.unwrap()` on the mutation — without it RTK Query mutations don't throw, and the catch block would be dead code.
What seniors notice: the v1/v2 branch lives in a *frontend hook* ([useToggleFeatureWithToast.ts#L37](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L37)) — the API leaks its versioning migration to every client. An alternative: one endpoint that branches server-side.

🎤 Interview angle: "How do you keep UI state consistent after a mutation?" — tag invalidation vs manual refetch vs optimistic update; this repo shows the first two coexisting (and why that's transitional, not ideal).

Drill: predict which queries refetch after a toggle by reading tags in [useFeatureState.ts#L20-L29,L57-L61](../../frontend/common/services/useFeatureState.ts#L20-L61). Basic: found the tags. Solid: mapped tags to `providesTags` consumers. Strong: explained why `Environment/METRICS` is invalidated too.

---

## Flow 3: An SDK identifies a user and gets personalized flags (`GET/POST /api/v1/identities/`)

Why this flow matters: it's where persistence, segmentation, and evaluation meet — the closest thing to "business logic" in a flag platform, and the source of its hardest performance questions.

Open these files first:
- [identities/views.py#L148-L235](../../api/environments/identities/views.py#L148-L235) — the view, with its honest TODOs
- [identities/models.py#L53-L130](../../api/environments/identities/models.py#L53-L130) — resolution

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | View | [views.py#L167-L173](../../api/environments/identities/views.py#L167-L173) | Missing identifier → **200 with a detail message** (TODO admits it should be 400) | — | Contract debt frozen by deployed SDK fleet |
| 2 | View | [views.py#L175-L186](../../api/environments/identities/views.py#L175-L186) | `transient` → unsaved Identity; else `get_or_create_for_sdk` | `Identity` | get_or_create under concurrency |
| 3 | View | [views.py#L188-L194](../../api/environments/identities/views.py#L188-L194) | Existing identities re-read **from replica**; new ones stay on primary | — | Read-your-writes handled explicitly — quote this |
| 4 | Model | [identities/models.py#L71-L92](../../api/environments/identities/models.py#L71-L92) | Segments evaluated (flag-engine), then one OR of three Q-objects: identity ∪ segment ∪ default | `Q` union | Segment rule count affects latency |
| 5 | Model | [identities/models.py#L96-L120](../../api/environments/identities/models.py#L96-L120) | Same service query as Flow 1, then per-feature max via `__gt__` | list → dict | Shared invariant, single implementation — good design |
| 6 | Response | [views.py#L208-L212](../../api/environments/identities/views.py#L208-L212) | `updated_at` header — TODO: identity overrides don't bump it | JSON traits+flags | Realtime clients can miss override changes |

Validation and authorization: environment-key auth (same boundary as Flow 1); trait persistence can be disallowed per environment ([Environment.trait_persistence_allowed](../../api/environments/models.py#L506)).

Persistence and side effects: identity + traits written on POST; optional fire-and-forget forwarding to the edge API ([views.py#L198-L206](../../api/environments/identities/views.py#L198-L206)).

Tests that cover it: [test_unit_identities_views.py](../../api/tests/unit/environments/identities/test_unit_identities_views.py).

What juniors usually miss: transient identities exist precisely so you can evaluate flags for a user you never store — privacy and storage tradeoff in one query param.
What seniors notice: the replica dance at step 3 is a textbook read-your-writes consistency fix, and the TODOs are *good* engineering: known debt, documented, with the compat reason.

🎤 Interview angle: "How would you personalize config per user without storing every user?" and "How do you handle replication lag?" — both answered concretely here.

Drill: write the three Q-objects from memory, then check. Basic: two of three. Solid: all three + the `overrides_only=True` segment filter. Strong: explained why transient identities skip the identity-override clause ([models.py#L76-L80](../../api/environments/identities/models.py#L76-L80)).

---

## Flow 4: A flag change propagates — audit, documents, realtime, webhooks (async backbone)

Why this flow matters: this is the repo's answer to "how do you keep derived state consistent without slowing down writes?" — the most senior-flavoured material in the codebase.

Open these files first:
- [audit/models.py#L131-L168](../../api/audit/models.py#L131-L168) — the lifecycle hook that fans out
- [environments/tasks.py#L26-L44](../../api/environments/tasks.py#L26-L44) — the rebuild task
- [audit/signals.py](../../api/audit/signals.py) — integrations fan-out

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Model save | django-simple-history on FeatureState et al. | Historical row written in the request transaction | history table | — |
| 2 | Audit | [audit/models.py#L141-L168](../../api/audit/models.py#L141-L168) | `AFTER_CREATE` hook: bump `environment.updated_at` per env (individually, "to avoid deadlock" L162), enqueue `process_environment_update` | AuditLog row | Hook ordering via `priority=HIGHEST_PRIORITY` |
| 3 | Queue | `.delay(args=(self.id,))` | Task row written to Postgres (task_processor from `flagsmith-common`) | task row | If enqueue is outside the tx → lost-update window (**investigate**) |
| 4 | Worker | [environments/tasks.py#L31-L44](../../api/environments/tasks.py#L31-L44) | Rebuild environment documents (Dynamo/cache), send SSE message | env document JSON | Processor down = staleness accumulates silently |
| 5 | Integrations | [audit/signals.py](../../api/audit/signals.py) | post_save on AuditLog → Datadog/Grafana/Slack/… + org webhooks | per-integration payloads | Each receiver is a partial-failure surface |
| 6 | Webhooks | [webhooks/webhooks.py#L63-L80](../../api/webhooks/webhooks.py#L63-L80) + [tasks.py](../../api/webhooks/tasks.py) | Signed payloads (`sign_payload`), `backoff` retries, failure email after N tries | signed JSON | Retries without receiver idempotency = duplicate deliveries |

Validation and authorization: none — by design. Everything here is post-commit machinery; authorization happened at the API edge.

Persistence and side effects: this **is** the side-effect flow: audit rows, task rows, document writes, SSE, HTTP calls out.

Tests that cover it: [test_unit_audit_models.py](../../api/tests/unit/audit/test_unit_audit_models.py), [test_unit_audit_signals.py](../../api/tests/unit/audit/test_unit_audit_signals.py), [test_unit_webhooks.py](../../api/tests/unit/webhooks/test_unit_webhooks.py), integration [featurestate/test_webhooks.py](../../api/tests/integration/features/featurestate/test_webhooks.py).

What juniors usually miss: the AuditLog isn't just compliance — it's the *trigger* for cache invalidation. Delete "boring" audit code and realtime breaks.
What seniors notice: at-least-once delivery everywhere; consumers must be idempotent; scheduled change requests reschedule themselves by re-enqueueing with `delay_until` ([audit/tasks.py#L52-L60](../../api/audit/tasks.py#L52-L60)) — a neat self-healing pattern.

🎤 Interview angle: "A write must update a cache, notify clients, and call webhooks — design it." Answer with this chain and name the guarantees: transactional history, async fan-out, retries with backoff, signing, and the idempotency requirement you'd impose on receivers.

Drill: list what breaks (and what doesn't) if the task processor is stopped for an hour. Basic: docs go stale. Solid: + SSE silence, webhook delay, but reads keep serving. Strong: + recovery behaviour when it restarts and the ordering questions that raises.

---

## Flow 5: Authorization — from `<Permission>` to a DRF permission class (security boundary)

Why this flow matters: multi-tenant authorization is the #1 thing reviewers and interviewers probe on any B2B SaaS; this repo has a full worked example.

Open these files first:
- [frontend/common/providers/Permission.tsx](../../frontend/common/providers/Permission.tsx) — UI gate
- [api/features/permissions.py#L28-L151](../../api/features/permissions.py#L28-L151) — server truth
- [api/features/views.py#L992-L1001](../../api/features/views.py#L992-L1001) — IDOR defense in miniature

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | [FeaturesPage.tsx#L330-L336](../../frontend/web/components/pages/features/FeaturesPage.tsx#L330-L336) | `<Permission level='project' permission={CREATE_FEATURE}>` hides affordances | boolean | UX only |
| 2 | API map | [permissions.py#L28-L39](../../api/features/permissions.py#L28-L39) | Action → required permission (`destroy`→`DELETE_FEATURE`…) | dict | New actions must be added or they fall through |
| 3 | List/create | [permissions.py#L43-L68](../../api/features/permissions.py#L43-L68) | Non-detail: resolve project from kwargs/body, check project permission | — | Fallthrough returns `view.detail` — read it twice |
| 4 | Object | [permissions.py#L70-L85](../../api/features/permissions.py#L70-L85) | Detail: object permission incl. **tag-scoped** permissions | obj + tag_ids | Tag scoping adds a second axis |
| 5 | Cross-tenant read | [views.py#L995-L999](../../api/features/views.py#L995-L999) | UUID lookup filtered to `request.user.get_permitted_projects(VIEW_PROJECT)` → 404, not 403 | queryset filter | The IDOR pattern: scope the queryset, don't check after fetch |

Validation and authorization: authn = token/cookie (dashboard) vs environment key (SDK); authz = permission constants evaluated against org/project/environment membership (+ RBAC in the private package).

Persistence and side effects: none — pure gatekeeping.

Tests that cover it: [test_unit_features_permissions.py](../../api/tests/unit/features/test_unit_features_permissions.py), E2E permission suites ([environment-permission-test.pw.ts](../../frontend/e2e/tests/environment-permission-test.pw.ts), [project-permission-test.pw.ts](../../frontend/e2e/tests/project-permission-test.pw.ts)).

What juniors usually miss: returning 404 for resources you can't see (existence-leak prevention) — and that hiding a button is not authorization.
What seniors notice: permission logic is spread across map + two methods + view-level filtering; a new endpoint author must know all four places. That's a boundary-cohesion critique to raise politely (see [03-architecture-and-patterns/06](../03-architecture-and-patterns/06-architecture-critique.md)).

🎤 Interview angle: "How do you prevent IDOR in a multi-tenant API?" Point at the queryset-scoping pattern verbatim.

Drill: write the pytest that proves a user in org A cannot fetch a feature state UUID from org B. Basic: test sketch. Solid: correct fixtures from [conftest.py](../../api/tests/conftest.py). Strong: asserted 404 (not 403) and said why.

---

## Flow 6: A pull request earns a merge (testing/CI flow)

Why this flow matters: "what does CI run?" is the fastest tell of engineering culture, and you'll be asked how you ship safely.

Open these files first:
- [.github/workflows/api-pull-request.yml](../../.github/workflows/api-pull-request.yml)
- [.github/workflows/frontend-pull-request.yml](../../.github/workflows/frontend-pull-request.yml)
- [api/tests/conftest.py](../../api/tests/conftest.py)

Trace:

| Step | Owner | What happens | Risk |
| --- | --- | --- | --- |
| 1 | API CI | Missing-migrations check (`make django-make-migrations` diff) | Catches model/migration drift |
| 2 | API CI | `make typecheck` (mypy strict) + generated-docs diff check | Contract drift caught mechanically |
| 3 | API CI | `make test` (pytest+xdist) + Codecov; **100% diff coverage** policy | Untested lines block merge |
| 4 | FE CI | `npm run test:unit` | Note: typecheck/lint enforced via hooks & other workflows — check before relying |
| 5 | Platform CI | Docker build/test/publish + Trivy scan ([platform-docker-*](../../.github/workflows/)) | Supply-chain gate |
| 6 | Local | husky hooks via `make install-hooks` ([AGENTS.md](../../AGENTS.md)) | Fast feedback before CI |

Test-layer map: pytest **unit** ([api/tests/unit/](../../api/tests/unit/)) and **integration** (endpoint-level black box, [api/tests/integration/](../../api/tests/integration/)); Jest units next to source; Playwright E2E ([frontend/e2e/tests/](../../frontend/e2e/tests/)) against a real API with retry orchestration and visual regression that **never fails CI** (report-only — a deliberate flake-containment decision, [frontend/README.md](../../frontend/README.md)).

What juniors usually miss: the fixture pyramid in conftest (org → project → environment → feature) is *the* API test vocabulary; fighting it means you're doing it wrong.
What seniors notice: visual diffs as PR comments instead of gates, E2E retries as a first-class script ([run-with-retry.ts](../../frontend/e2e/run-with-retry.ts)) — flake is managed as a product, not wished away.

🎤 Interview angle: "How do you keep E2E suites from blocking the team?" — retries + quarantine + report-only visual diffs, with the tradeoff (masked real regressions) named.

Drill: run one Jest file and one pytest file by name (commands in [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md) — pytest path needs the API dev DB). Basic: both ran. Solid: explained what each layer would catch that the other can't. Strong: named the flake-containment strategy and its cost.
