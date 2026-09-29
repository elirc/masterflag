# Observability and Operations

**Observability** = can you explain what the system did from its outputs? This repo's stack: Prometheus metrics (`metrics.py` modules, naming `flagsmith_{domain}_{entity}_{unit}`), structlog events flowing through OpenTelemetry (`entity.action` naming, IDs-not-PII), Sentry both sides, django-health-check, and Playwright artifacts for the FE ([api/README.md](../../api/README.md), [frontend/README.md](../../frontend/README.md)).

## The instrument panel by subsystem

| Subsystem | Signals that exist | Anchors |
| --- | --- | --- |
| SDK read path | request logs enriched with `environment_id` at auth time; env metrics querysets | [authentication.py#L42-L46](../../api/environments/authentication.py#L42-L46), [environments/metrics.py](../../api/environments/metrics.py) |
| Async pipeline | task rows (state queryable via SQL); priorities; structlog events per domain | [environments/tasks.py](../../api/environments/tasks.py) |
| Webhooks | retry/backoff logs; failure email after final retry | [webhooks/tasks.py#L23-L25](../../api/webhooks/tasks.py#L23-L25) |
| Dashboard | Sentry browser SDK; Amplitude product analytics; API baggage headers carrying session ids | [service.ts#L42-L51](../../frontend/common/service.ts#L42-L51) |
| Deploys | health checks (django-health-check), one-shot migrate container gating app start | [docker-compose.yml#L55-L67](../../docker-compose.yml#L55-L67) |

## "How would I know this broke?" — per key flow

- **Flow 1 (SDK flags):** 5xx rate + latency histograms on `/api/v1/flags/`; env-cache behaviour visible via DB load. Gap to notice: a *stale-but-200* response looks healthy — hence the `updated_at` header as a client-checkable freshness signal ([views.py#L1069-L1072](../../api/features/views.py#L1069-L1072)).
- **Flow 2 (toggle):** mutation 4xx/5xx in Sentry FE + API logs; audit row presence is the ground truth ("no audit row" = the write never committed).
- **Flow 4 (propagation):** task-queue depth and age are the leading indicators; webhook failure emails are the trailing one. The critique's improvement #5 (propagation-lag metric) exists because there's no single "documents are current" gauge (**hypothesis** — verify against the metrics docs before repeating).
- **Flow 6 (CI):** codecov + required checks; flaky-test rate via `E2E_REPEAT` runs.

## Error-handling posture worth copying

- Log events are **named for the action, not the emotion**: `import.failed`, not `error` ([api/README.md](../../api/README.md) logs section).
- Context binding once (`logger.bind(...)`) instead of repeating ids — correlatable events.
- FE errors: user-facing toast + console.error + (via Sentry) alerting — see [useToggleFeatureWithToast.ts#L61-L69](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L61-L69); note it shows a *generic* message and keeps the detail in the console — good PII/confusion hygiene, debatable for support.

## Deploy and rollback (self-hosted shape)

Compose ordering: postgres → `migrate-db` (one-shot) → app + processor ([docker-compose.yml#L40-L81](../../docker-compose.yml#L40-L81)). Rollback = previous image **plus** the migration question: forward-only migrations mean old code must tolerate the new schema (why [02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md) preaches additive-first). Feature-gated rollout via Flagsmith-on-Flagsmith ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)) is the cheaper rollback: flip the flag, no deploy.

🎤 **Interview angle:** "How do you know your feature works in production?" — answer as: leading indicator (queue depth / mutation error rate), trailing indicator (webhook failure email / support ticket), ground truth (audit row), and the rollback lever (flag flip vs redeploy). Naming a *gap* you'd instrument (propagation lag) reads senior.

**Drill:** for the copy-flag feature from review kata 1, write its observability plan: one counter, one histogram, one structured event (correctly named per house convention), and its rollback lever. Basic: three signals. Solid: correct naming conventions + label cardinality sanity. Strong: you specified the alert threshold and who gets paged.
