# Senior Build Projects

Six projects, 2 days–4 weeks. Each is realistic — something a maintainer *might* accept given an RFC — and each produces a portfolio-grade interview story. Do them in a fork; the deliverable is working code **plus** the design artifacts.

## Project 1: Finish the Flux→RTK migration for the feature-create flow (1–2 weeks)
Problem: [CreateFlag modal](../../frontend/web/components/modals/create-feature) still writes through the legacy Flux store, forcing the bridge at [FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134).
Value: deletes the repo's most bug-prone seam; unblocks store deletion.
Design checklist: inventory every `FeatureListStore` consumer (grep); map each Flux action to an existing or new RTK mutation; decide tag invalidation replacing `saved` events; migration order (reads first, writes second, bridge deletion last).
Likely files: create-feature modal tree, [feature-list-store.ts](../../frontend/common/stores/feature-list-store.ts), features page hooks, services.
Test plan: E2E flag suites green throughout; new Jest on mutation wiring; delete-bridge commit is separate and revertible.
Security/perf: none new / drops `cloneDeep` per fetch.
Rollout: ship behind `flagsmith` feature flag (pattern 16); bridge deletion only after a soak.
Open questions: does anything outside FeaturesPage read `FeatureListStore.model`? (Answer changes scope — find out FIRST.)
Stretch: delete the store file; write the "how we killed our Flux layer" post.
Interview story: real strangler-fig migration with staged rollback — senior catnip.

## Project 2: Contract codegen pipeline (OpenAPI → Res/Req) (2–3 weeks)
Problem: hand-written FE types drift ([03-type-system](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
Design checklist: generator (openapi-typescript?); output layout; incremental adoption map (60+ services); CI diff gate mirroring [api-pull-request.yml](../../.github/workflows/api-pull-request.yml)'s docs check; handling endpoints missing from the spec.
Test plan: typecheck as gate; one migrated service per PR.
Rollout/rollback: generated types are additive; each service migration is a small revertible PR.
Open questions: is [sdk/openapi.yaml](../../sdk/openapi.yaml) complete for *admin* endpoints, or SDK-only? (Scope-defining — check first.)
Stretch: publish the generator config as a reusable pattern.
Interview story: "I built contract enforcement between a Django API and a React app."

## Project 3: Webhook delivery ledger with replay (3–4 weeks)
Problem: deliveries are fire-and-retry with terminal email ([webhooks/](../../api/webhooks/)); operators can't audit or replay.
Design checklist: `WebhookDelivery` model (status, attempts, response code, payload hash — not full payload? PII/size decision); write path in the retry loop; org-scoped read API; replay endpoint (idempotency! signed re-send with same event id); retention policy + cleanup task.
Migration plan: additive table; zero changes to existing behaviour when feature is off.
Test/security plan: permission tests (org isolation), replay-idempotency test, retention task test; signature on replays.
Performance: index on (webhook_id, created_at); write amplification acceptable? measure.
Rollout: setting-gated; docs page.
Open questions: does the task processor already record enough to derive this? (Read `flagsmith-common` task models first.)
Stretch: dead-letter UI with bulk replay.
Interview story: "I designed an auditable, replayable delivery ledger" — a complete system-design answer you actually built.

## Project 4: Environment-document propagation SLO dashboard (2 weeks)
Problem: no first-class signal for write→propagation lag (critique #5, ticket M4 grown up).
Design checklist: define the measurement points (audit created → document written → SSE sent, [environments/tasks.py#L31-L44](../../api/environments/tasks.py#L31-L44)); Prometheus histograms per stage (naming convention!); Grafana dashboard JSON; alert thresholds.
Test plan: metric emission tests via prometheus-client registry inspection.
Rollout: pure additive observability.
Open questions: existing metrics overlap ([metrics docs](https://docs.flagsmith.com/deployment-self-hosting/observability/metrics) — check first).
Stretch: expose per-environment lag in the dashboard UI.
Interview story: "I defined and instrumented an SLO for an eventually-consistent pipeline."

## Project 5: Load-test rig + performance budget for the SDK read path (2 weeks)
Problem: the hot path's performance characteristics are folklore; [api/jmeter-tests/](../../api/jmeter-tests/) exists (age unknown — investigate) but isn't a CI budget.
Design checklist: k6/locust scenario for `/flags/` + `/identities/` with realistic feature/segment cardinalities; seed script; budget assertions (p95, queries/request); optional CI job (nightly, not per-PR).
Test plan: the rig *is* the test; validate against the three cache layers on/off.
Open questions: representative cardinalities (ask maintainers/issue data).
Stretch: publish findings; attach to ticket M10's "measure" requirement.
Interview story: "I built the load harness and set the performance budget for a config-delivery hot path."

## Project 6: Self-hosted backup/restore verification kit (2 days–1 week)
Problem: self-hosters' most common disaster is an unverified backup; compose ships no restore drill.
Design checklist: script pg_dump/restore against the compose stack; smoke assertions post-restore (env document rebuild fires? [write_environment_documents](../../api/environments/models.py#L304-L330)); docs page walkthrough.
Rollout: docs + `scripts/` addition only; zero product risk.
Open questions: maintainer appetite — propose in an issue first; this may fit docs better than code.
Stretch: scheduled verification container.
Interview story: "I wrote the disaster-recovery drill" — rare, memorable, operations-minded.

---

**Per-project deliverables checklist:** RFC (template in [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)) → issue/discussion upstream → staged PRs each independently revertible → demo recording → retro note (what you'd do differently). The retro note is the STAR story's "learning" beat — write it while it's fresh.
