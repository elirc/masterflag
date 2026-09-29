# System Design From This Repo

"Design a feature-flag platform" — a genuinely common prompt, and you've read one. Walk the whiteboard in this order; at each step: what Flagsmith actually chose (anchored), the simpler/stronger alternative, and how junior/mid/senior answers differ. Companion critique: [03/06](../03-architecture-and-patterns/06-architecture-critique.md).

## Step 1 — Requirements (3 minutes, out loud)

Functional: define flags per project; per-environment on/off + values; target by user (identity) and by rule-based cohort (segment); dashboard for humans; API for SDKs; audit everything.
Non-functional: **reads ≫ writes** (every client app polls flags; humans toggle rarely); read latency budget in tens of ms; propagation delay tolerable (seconds); multi-tenant isolation is non-negotiable; self-hostable.
Junior skips this step. Mid states the read/write asymmetry. Senior *derives the whole architecture from it*: "reads dominate ⇒ cache aggressively and denormalize; writes are rare ⇒ they can afford expensive fan-out."

## Step 2 — API sketch

Two planes, two credentials (what Flagsmith chose — [authentication.py](../../api/environments/authentication.py)):
- **Data plane** (SDKs): `GET /flags`, `GET/POST /identities` — environment-key header, no user, unthrottled-but-cached, unpaginated (bounded set — [views.py#L1010-L1011](../../api/features/views.py#L1010-L1011)).
- **Control plane** (dashboard): CRUD on features/environments/segments under token auth + RBAC.
Alternative: one plane with scoped API keys — simpler, but couples fleet-frozen contracts to fast-moving admin contracts. Senior point: *split planes so contract stability pressure lands only where it must.* Mention client vs server key split (`ser.` prefix → server-only flags, [views.py#L1082-L1083](../../api/features/views.py#L1082-L1083)).

## Step 3 — Data model

Reproduce the spine (drill it from [03/02](../03-architecture-and-patterns/02-data-model-and-persistence.md)): Org → Project → Environment; Feature per project; **FeatureState per environment with nullable identity/segment discriminators**; Segment as rule tree; priority between overlapping segments (ordered FeatureSegment).
Resolution invariant: identity > segment(priority) > default — in Flagsmith, one comparison function ([models.py#L529-L601](../../api/features/models.py#L529-L601)) used by every read path.
Alternatives to name: JSON-blob per environment (simpler; loses queryability/audit granularity), three override tables (cleaner constraints; loses single-query evaluation). Junior draws tables; mid explains the discriminator choice; senior also places versioning (immutable published snapshots — [versioning_service.py#L141-L198](../../api/features/versioning/versioning_service.py#L141-L198)) and says what it buys (rollback = pointer move).

## Step 4 — The read path (where you win or lose the interview)

Layered caching exactly as [key flow 1](../01-codebase-cartography/05-key-flows.md): credential→entity cache with negative caching → response cache varying on credential+origin → replica reads → one-query fetch + in-app priority resolution.
Freshness signal: `updated_at` header so clients can decide ([views.py#L1069-L1072](../../api/features/views.py#L1069-L1072)).
Scale-up story (say it as a ladder): single Postgres → replicas → denormalized *environment documents* pushed to edge storage (what Flagsmith does for SaaS: DynamoDB + edge API, [write_environment_documents](../../api/environments/models.py#L304-L330)) → SDKs doing **local evaluation** from the document (compute moves to the client; the API becomes a document CDN).
Junior: "add Redis." Mid: names layers and TTLs and the staleness window. Senior: names the *invalidation event* driving each layer and the poisoning defense (origin in cache key).

## Step 5 — The write path and propagation

Write = validate → authorize → persist + history → respond. Everything else async off the committed fact: audit row → bump `updated_at` → rebuild documents → notify (SSE/streaming) → webhooks with signing/retries → integrations ([04-side-effects](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).
Queue choice: Postgres-backed here (self-host simplicity) vs Redis/SQS (throughput) — argue from the deployment constraint, not fashion.
Junior: does it synchronously. Mid: queue + retries + idempotency. Senior: outbox gap, ordering non-guarantees, propagation SLO and its metric, and the "audit row funds the architecture" observation.

## Step 6 — Tradeoffs recap (close with this)

1. Eventual consistency chosen for read scale; window bounded by TTLs; surfaced via header.
2. Evaluation in app code, not SQL — testability over DB cleverness; bounded rows make it safe.
3. One state table with discriminators — single-query reads at the cost of procedural uniqueness.
4. Two planes, two credentials — contract stability where the fleet is; velocity where the humans are.
5. Boring queue, boring DB — deployability is a feature for self-hosted OSS.

## Variation prompts (rehearse each for 10 minutes)

1. **"Add real-time flag updates."** Flagsmith's shape: SSE notify + client refetch ([sse/](../../api/sse/), [environments/tasks.py#L40-L44](../../api/environments/tasks.py#L40-L44)). Discuss: push-the-notification vs push-the-data; connection fan-out costs; fallback to polling; the identity-override freshness gap (risk #4) as a known hard part.
2. **"10× the read traffic."** Ladder from step 4; when Postgres replicas stop being the answer; document push + edge; cache stampede protection (request coalescing); measure before each rung.
3. **"Add A/B experiment analysis."** Multivariate options exist ([features/multivariate/](../../api/features/multivariate/)); deterministic hash-based bucketing (same identity → same variant, no storage); the *analytics* half is an events pipeline problem — keep it out of the flag read path.
4. **"Make one flag change require approval."** Change requests: state with `live_from` + approvals gate publication ([workflows](../../api/features/workflows/), FK at [models.py#L506-L511](../../api/features/models.py#L506-L511)); scheduled go-live via self-rescheduling tasks ([audit/tasks.py#L52-L60](../../api/audit/tasks.py#L52-L60)).
5. **"Now it must be multi-region."** Read path regionalizes cleanly (documents are already denormalized artifacts); write path stays single-primary with async replication; talk about what the `updated_at` freshness contract means cross-region.

## Delivery notes

Draw boxes for *deployables* (API, worker, DB, cache, edge), not concepts. Anchor every claim you can to "in the flag platform I've studied, this is how it actually works" — it converts theory questions into experience questions. And when you don't know, price it: "I'd need to measure X before choosing" is a senior sentence.

**Mock protocol:** 40 minutes, out loud, phone recording, whiteboard/excalidraw. Grade against: stated the read/write asymmetry in the first 3 minutes? every cache had an invalidation story? closed with tradeoffs unprompted? Cram-plan places two of these ([07](07-two-week-cram-plan.md)).
