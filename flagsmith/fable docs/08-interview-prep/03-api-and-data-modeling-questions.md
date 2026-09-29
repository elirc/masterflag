# API and Data Modeling Questions — Question Cards

Twelve cards. This is where a JS-only candidate usually loses the mid-level offer; you won't, because you've read a production API line by line.

## Q1: "Design the schema for feature flags with per-user and per-segment overrides."
Anchor: [FeatureState's three roles](../../api/features/models.py#L461-L498) + [glossary ERD](../01-codebase-cartography/03-domain-glossary.md). Junior: one table with a JSON column. Mid: definition (Feature) split from per-environment state (FeatureState); nullable discriminator FKs; uniqueness per role; priority ladder at read time. Senior: defends single-table vs three-table, names the illegal-state risk and the constraint you'd add, and the versioning extension (EFV).

## Q2: "Authn vs authz — show me both in a system you know."
Anchor: [authentication.py](../../api/environments/authentication.py) (SDK) + token auth (dashboard) + [permission classes](../../api/features/permissions.py). Mid: two credentials, two audiences, permission constants at three scopes. Senior: the *no-user authentication* design (credential resolves to an environment, request tagged with origin) and why that's cleaner than a service account.

## Q3: "How do you prevent IDOR?"
Anchor: [views.py#L992-L1001](../../api/features/views.py#L992-L1001). Mid: scope the queryset before fetch; 404 not 403. Senior: makes it systemic — the recipe must be un-forgettable (base classes/lint/review checklist), plus the existence-leak rationale. Follow-up: "when *would* you return 403?" (When existence is already public — e.g. you can see the project but not edit.)

## Q4: "Validation — where does each kind of rule live?"
Anchor: [validation ladder](../03-architecture-and-patterns/03-validation-auth-and-permissions.md). Mid: shape in serializers, tenancy in permissions, invariants in domain ([require_direct_state_write](../../api/features/versioning/versioning_service.py#L20-L37)), integrity in DB. Senior: what happens when a rule is in the wrong layer (bypassable via a second endpoint) with the v2-write-guard as the counter-example done right.

## Q5: "Pagination — design choices and failure modes."
Anchor: `PagedResponse<T>` envelope ([responses.ts#L11](../../frontend/common/types/responses.ts#L11)); SDK endpoints deliberately unpaginated (`pagination_class = None`, [views.py#L1010](../../api/features/views.py#L1010)) because the domain bounds the set. Mid: offset vs cursor tradeoffs; consistent envelopes. Senior: when *not* to paginate (bounded domain sets, SDK simplicity) — an actual design decision this repo made, and page-drift under concurrent writes for offset pagination.

## Q6: "How would you version an API used by a fleet of deployed SDKs?"
Anchor: the frozen contract debt ([identities/views.py#L170-L173](../../api/environments/identities/views.py#L170-L173)). Mid: additive-only changes, deprecation windows, version headers/paths. Senior: reality — the TODO that can't be fixed, measuring blast radius before change (metric on offending requests), and contract tests as the enforcement.

## Q7: "Transactions — when do you need one, and what did it cost you?"
Anchor: surgical `@transaction.atomic` on multi-row ops ([segment clone](../../api/segments/models.py#L151), [reorder](../../api/features/feature_segments/serializers.py#L53)); giant cascades moved OUT of the ORM ([models.py#L108-L110](../../api/features/models.py#L108-L110)). Mid: atomicity for multi-row invariants; autocommit otherwise. Senior: long transactions = lock time = latency for others; deadlock avoidance by consistent ordering (the [audit hook's per-row update loop](../../api/audit/models.py#L158-L166) with its comment).

## Q8: "Design caching for a read-heavy config endpoint."
Anchor: the three layers + key isolation ([key flow 1](../01-codebase-cartography/05-key-flows.md)). Mid: TTL cache with entity cache under it; invalidation via updated_at bump. Senior: negative caching for invalid credentials, cache-key poisoning defense (origin in key, [views.py#L1094](../../api/features/views.py#L1094)), and the honest staleness-window conversation.

## Q9: "A write must fan out to caches, webhooks, and integrations. Design it."
Anchor: [committed-fact chain](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md). Mid: queue + workers, retries with backoff, at-least-once, idempotent consumers. Senior: outbox gap analysis, priorities, self-rescheduling scheduled work ([audit/tasks.py#L52-L60](../../api/audit/tasks.py#L52-L60)), and observability (queue depth as the leading indicator).

## Q10: "Migrations on a huge table — walk me through your process."
Anchor: [safe-schema-change sequence](../03-architecture-and-patterns/02-data-model-and-persistence.md); CI migration check. Mid: additive → backfill → constrain; old code must run on new schema during deploy. Senior: lock analysis per DDL statement, reversibility policy, and the org process (hand-named migrations, squashing, CI gates) that makes it stick.

## Q11: "Why would replica reads return wrong data, and what do you do?"
Anchor: [read-your-writes routing](../../api/environments/identities/views.py#L188-L194). Mid: replication lag; route fresh entities to primary. Senior: the general taxonomy (monotonic reads, RYW), where each matters in this product (new identity vs flags list), and pinning strategies.

## Q12: "REST resource design: critique `POST /environments/{key}/featurestates/{id}` style nesting."
Anchor: [frontend request URLs](../../frontend/common/services/useFeatureState.ts#L62-L66) and DRF nested routers ([pyproject.toml](../../api/pyproject.toml) drf-nested-routers). Mid: nesting expresses ownership + scoping; ids inside a scope. Senior: nesting depth costs (URL churn on re-parenting), when to flatten with query params, and consistency-with-existing-API beating theoretical purity — cite that this repo values the existing pattern over REST aesthetics.

---

**Rehearsal:** Q1, Q3, Q8, Q9 are near-guaranteed in some form. For each, your first sentence should be the *design*, not the preamble: "Two tables — definition and per-environment state — plus nullable discriminators for overrides. Here's why…"
