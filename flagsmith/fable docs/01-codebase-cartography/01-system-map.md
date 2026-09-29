# System Map

## Shape: a polyglot monorepo, one product, four deployables

```
flagsmith/
├── api/          Django 5 + DRF monolith (Python 3.11+, uv)      ← business logic lives here
├── frontend/     React 19 + TS dashboard (rspack, RTK Query)     ← admin UI, served by Express or bundled into API image
├── docs/         Docusaurus site (docs.flagsmith.com)
├── docker-compose.yml        postgres + migrate + api + task-processor
├── infrastructure/aws/       ECS task definitions (SaaS ops, read-only for you)
└── .github/workflows/        per-surface CI (api-*, frontend-*, docs-*, platform-*)
```

Runtime picture (self-hosted OSS path):

```
 SDKs (15+ languages)          Browser (dashboard user)
      │  X-Environment-Key           │  Token / cookie auth
      ▼                              ▼
 ┌─────────────────────────────────────────────┐
 │  Django API (api/)                          │
 │  /api/v1/flags, /identities  ← SDK surface  │
 │  /api/v1/environments, ...   ← admin surface│
 └───────┬───────────────┬─────────────────────┘
         │ writes        │ enqueue (task_processor tables in Postgres)
         ▼               ▼
     PostgreSQL     Task processor (separate container)
     (+ replicas)     → environment documents, webhooks, integrations,
                        SSE "flags changed" messages, DynamoDB sync (SaaS edge)
```

The four services in [docker-compose.yml#L40-L81](../../docker-compose.yml#L40-L81): `postgres`, one-shot `migrate-db`, `flagsmith` (API, which also serves the built dashboard), and `flagsmith-task-processor`. **Boundary worth naming in interviews:** the task processor is a separate deployable that shares the database — async work is queued in Postgres tables, not an external broker. Simple ops, at-least-once semantics, and the queue competes with OLTP for the same database.

## Ownership map

| Concern | Lives in | Notes |
| --- | --- | --- |
| UI pages | [frontend/web/components/pages/](../../frontend/web/components/pages/) | Page components; feature dir example: [features/](../../frontend/web/components/pages/features/) |
| UI data layer | [frontend/common/services/](../../frontend/common/services/) (RTK Query) + legacy Flux in [frontend/common/stores/](../../frontend/common/stores/) | Two generations coexist; bridge visible at [FeaturesPage.tsx#L97-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L134) |
| API contracts (FE view) | [frontend/common/types/requests.ts](../../frontend/common/types/requests.ts), [responses.ts](../../frontend/common/types/responses.ts) | Hand-maintained, not generated — a contract-drift risk to know about |
| HTTP surface | Each Django app's `views.py`/`urls.py`, rooted at [api/app/urls.py](../../api/app/urls.py) | |
| Domain logic | Django apps: [features/](../../api/features/), [environments/](../../api/environments/), [segments/](../../api/segments/), [projects/](../../api/projects/), [organisations/](../../api/organisations/), [users/](../../api/users/) | House layering adds `services.py`, `tasks.py`, `mappers.py`, `dataclasses.py` per [api/README.md](../../api/README.md) |
| Flag evaluation | [api/features/versioning/versioning_service.py](../../api/features/versioning/versioning_service.py) + `FeatureState.__gt__` ([api/features/models.py#L529-L601](../../api/features/models.py#L529-L601)); segment matching via external `flagsmith-flag-engine` | The crown jewels |
| Persistence | Models + per-app `migrations/`; Postgres primary, optional replicas ([using_database_replica](../../api/features/versioning/versioning_service.py#L538-L540)) | |
| Async workers | Per-app `tasks.py` with `@register_task_handler` (e.g. [api/environments/tasks.py](../../api/environments/tasks.py)) | Runner comes from the external `flagsmith-common` package |
| Auth/security | [api/custom_auth/](../../api/custom_auth/) (users), [api/environments/authentication.py](../../api/environments/authentication.py) (SDK keys), per-app `permissions.py`, [api/api_keys/](../../api/api_keys/) | |
| Audit trail | [api/audit/](../../api/audit/) + django-simple-history records on models | Doubles as the event spine for cache/document rebuilds |
| Tests | [api/tests/unit/](../../api/tests/unit/), [api/tests/integration/](../../api/tests/integration/); FE `__tests__/` dirs + [frontend/e2e/](../../frontend/e2e/) (Playwright) | |
| Public interfaces | REST API (OpenAPI at [sdk/openapi.yaml](../../sdk/openapi.yaml)), SSE, webhooks ([api/webhooks/](../../api/webhooks/)) | Contract changes here have SDK-fleet blast radius |

## Public vs private

**Public (contract, expensive to change):** `/api/v1/flags/`, `/api/v1/identities/`, environment documents, webhook payload shapes ([Webhook.generate_webhook_feature_state_data](../../api/environments/models.py#L632)), the OpenAPI spec. **Private (refactor freely):** everything else — internal services, serializers for the dashboard, frontend stores.

🎤 **Interview angle:** "Walk me through a codebase you know" wants exactly this artifact: deployables, ownership boundaries, one public contract, one piece of labelled legacy. Practice delivering this page aloud in 3 minutes.

**Drill:** without looking, draw the runtime diagram and mark where a flag toggle enters and every place its effect must propagate. Self-grade — Basic: API + DB. Solid: + task processor and audit log. Strong: + SSE/realtime, environment-document cache, webhooks, and the flags cache TTL caveat.
