# Mid-Level Feature Tickets

Ten cross-layer tickets. **Rule: write a half-page design note before coding** (data shape → API contract → UI → tests → risk/rollback). Each ticket lists the design questions your note must answer. These are practice-sized; upstreaming requires maintainer buy-in via an issue first.

## Ticket M1: Server-side embed of `feature_segment` in feature-state list
Layers: DRF serializer + FE service. The fix for risk-register #1.
Design note must answer: additive field or `include=` param? How does the FE fall back on old API versions (self-hosted skew!)? Anchors: [useFeatureState.ts#L30-L51](../../frontend/common/services/useFeatureState.ts#L30-L51), [features/serializers.py](../../api/features/serializers.py).
Tests: serializer unit; FE mock asserting request count drops. Risk: payload growth on large pages. Rollback: param-gated, FE fallback retained one release.
Interview story: "I removed a client-side N+1 by moving composition server-side, with version-skew handling."

## Ticket M2: `is_archived` filter across API + UI features list
Layers: queryset filter + query param + FE filter UI ([useFeatureFilters.ts](../../frontend/web/components/pages/features/hooks/useFeatureFilters.ts)).
Design note: filter semantics with existing tag/search params; index needed? (EXPLAIN on big table). Tests: API filter unit + FE URL round-trip. Risk: low. Rollback: param ignored server-side = harmless.
Interview story: "end-to-end filter feature: schema-aware, URL-as-state, indexed."

## Ticket M3: Copy-flag-to-environment (the review-kata PR, done right)
Layers: new endpoint + permissions + versioning-aware write + modal UI.
Design note: **both-sided permission checks** (source read, target `UPDATE_FEATURE_STATE`); v1 vs v2 write path via `update_flag` service ([versioning_service.py#L132](../../api/features/versioning/versioning_service.py#L132)); segment overrides copied or not (product decision — pick and defend). Tests: cross-tenant 404, v2 version creation, E2E happy path. Risk: silent value overwrite in target → require confirmation UI. Rollback: endpoint removal is clean (new surface).
Interview story: the best one here — "I designed a cross-environment write with tenant isolation and versioning semantics."

## Ticket M4: Propagation-lag surfacing (critique improvement #5)
Layers: metric + API field + small UI badge.
Design note: definition of lag (audit `created_date` vs document build time); where stored; cardinality of the metric. Anchors: [environments/tasks.py#L31-L44](../../api/environments/tasks.py#L31-L44), [audit/models.py#L141-L168](../../api/audit/models.py#L141-L168). Tests: unit on computation; log/metric capture. Risk: none user-facing. Rollback: hide badge.
Interview story: "I made eventual consistency observable."

## Ticket M5: Rate-limit unauthenticated SDK endpoints per key (design-first!)
Layers: DRF throttling + settings + docs.
Design note: why `throttle_classes = []` today ([views.py#L1011](../../api/features/views.py#L1011)) — SaaS scale means throttling belongs at the edge; is an *optional* self-hosted throttle worth the config surface? This ticket may correctly conclude **"don't build it"** — a written no is a mid-level deliverable. Risk: throttling legit SDK traffic = outage-grade.
Interview story: "I evaluated a plausible feature and argued against it with evidence."

## Ticket M6: Bulk-toggle selected flags in UI
Layers: FE multi-select + N mutations (or new bulk endpoint — decide!).
Design note: N PUTs vs bulk endpoint (partial failure semantics! versioning: one EFV per feature); optimistic UI or spinner; permission per row. Tests: partial-failure handling. Risk: half-applied bulk ops confuse users → per-row result UI. Rollback: feature-flag the button (pattern 16).
Interview story: "partial failure design for bulk operations."

## Ticket M7: Webhook delivery history UI (read-only)
Layers: persistence check (does delivery result persist? investigate task models) + endpoint + page.
Design note: what's queryable today vs needs new storage; retention. Anchors: [webhooks/](../../api/webhooks/), task tables. Tests: serializer + permission (org-scoped!). Risk: unbounded table growth → pagination + retention note.
Interview story: "I exposed async-job outcomes to end users."

## Ticket M8: Segment-override priority drag-and-drop guardrails
Layers: FE dnd (dnd-kit present in [package.json](../../frontend/package.json)) + `feature_segments` reorder API ([feature_segments/serializers.py#L53](../../api/features/feature_segments/serializers.py#L53) is atomic).
Design note: concurrent reorder conflicts (last-write-wins? version check?); large lists. Tests: reorder unit + race simulation (two swaps). Risk: silent priority corruption — the atomic serializer is your friend; add an audit assertion.
Interview story: "ordered-data concurrency semantics."

## Ticket M9: Type-safe `Res`/`Req` for one service via OpenAPI codegen (pilot)
Layers: tooling + one service migration (critique improvement #3, scoped).
Design note: generator choice; where generated types live; diff-check in CI mirroring the backend's docs check. Tests: typecheck is the test. Risk: generator output ergonomics — pilot on ONE service (`useWebhooks`?) before proposing widely.
Interview story: "I piloted contract codegen and wrote the adoption plan."

## Ticket M10: Identity-override freshness header fix (upstream TODO)
Layers: API only, but contract-sensitive ([identities/views.py#L208-L212](../../api/environments/identities/views.py#L208-L212)).
Design note: what timestamp correctly covers identity overrides (max of env.updated_at and identity's override updates); cost of computing it per request; SDK impact analysis. Tests: header value under each change type. Risk: extra query on the hottest path — measure. Rollback: header computation behind a setting.
Interview story: "I closed a documented consistency gap on a hot path without regressing latency."

---

**Grading your design notes** — Basic: all five sections present. Solid: named the partial-failure/conflict case and chose explicitly. Strong: included the *rejected alternative* with the reason, and the note would survive a maintainer's "why not X?" reply.
