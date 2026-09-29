# File Reading Order

Thirty files, ordered so each one pays into the next. Per file: why it matters, what to look for, what to ignore. Junior path = 1–14. Mid path adds 15–24. Senior path adds 25–30.

## Junior path — orientation and the two core flows

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | [AGENTS.md](../../AGENTS.md) → [api/README.md](../../api/README.md) → [frontend/README.md](../../frontend/README.md) | House rules: 100% diff coverage, mypy strict, test naming (`test_x__cond__outcome`), layering conventions (`services.py`, `tasks.py`) | — |
| 2 | [docker-compose.yml](../../docker-compose.yml) | Four deployables; task processor is separate | Volume/env plumbing |
| 3 | [api/app/urls.py](../../api/app/urls.py) | URL roots — the API's table of contents | Conditional enterprise routes |
| 4 | [api/features/models.py](../../api/features/models.py) | `Feature` (L92), `FeatureSegment` (L248), `FeatureState` (L461) and the `__gt__` ladder (L529) | Import/export helpers |
| 5 | [api/environments/models.py](../../api/environments/models.py) | `Environment`, `get_from_cache` (L268), `write_environment_documents` (L305), `EnvironmentAPIKey` (L701) | Dynamo wrappers on first pass |
| 6 | [api/environments/authentication.py](../../api/environments/authentication.py) | The SDK auth boundary in 50 lines — read all of it | — |
| 7 | [api/features/views.py#L1004-L1120](../../api/features/views.py#L1004-L1120) | `SDKFeatureStates`: caching decorators, `_additional_filters`, replica reads | The 1000 lines of admin viewsets above it, for now |
| 8 | [api/features/versioning/versioning_service.py#L40-L115](../../api/features/versioning/versioning_service.py#L40-L115) | One query + Python max — the evaluation heart | update/delete helpers below |
| 9 | [frontend/common/service.ts](../../frontend/common/service.ts) | RTK Query base: token header, `refetchOnFocus` | Amplitude baggage |
| 10 | [frontend/common/store.ts](../../frontend/common/store.ts) | Store assembly, `StoreStateType` | — |
| 11 | [frontend/common/services/useFeatureState.ts](../../frontend/common/services/useFeatureState.ts) | An injected endpoint pair; `invalidatesTags`; the client-side segment fan-out (L42-44) | — |
| 12 | [frontend/web/components/pages/features/FeaturesPage.tsx](../../frontend/web/components/pages/features/FeaturesPage.tsx) | A modern page: hooks, `Permission` wrapper, and the labelled Flux bridge (L97–134) | Render minutiae |
| 13 | [frontend/common/types/responses.ts](../../frontend/common/types/responses.ts) | The FE's picture of API shapes; find `FeatureState`, `ProjectFlag` | The sheer size |
| 14 | [api/tests/conftest.py](../../api/tests/conftest.py) | Fixture vocabulary: `organisation`, `project`, `environment`, `admin_client`, `staff_user` | Chargebee/subscription fixtures |

## Mid path — authorization, async, segments, workflows

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 15 | [api/features/permissions.py](../../api/features/permissions.py) | `ACTION_PERMISSIONS_MAP`; list-vs-detail split; tag-scoped permissions | — |
| 16 | [frontend/common/providers/Permission.tsx](../../frontend/common/providers/Permission.tsx) | How the UI asks "can I?" — render-prop over a permissions endpoint | — |
| 17 | [api/audit/models.py](../../api/audit/models.py) | `AuditLog` lifecycle hooks; `process_environment_update` (L141–168) is the event spine | — |
| 18 | [api/environments/tasks.py](../../api/environments/tasks.py) | `@register_task_handler`, priorities, document rebuild + SSE fan-out | Dynamo deletes |
| 19 | [api/webhooks/webhooks.py](../../api/webhooks/webhooks.py) | Event types, signing, backoff/retries, `DISABLE_WEBHOOKS` | Email templating |
| 20 | [api/audit/signals.py](../../api/audit/signals.py) | AuditLog → integrations (Datadog/Slack/…) + org webhooks fan-out | Per-integration wrappers |
| 21 | [api/segments/models.py](../../api/segments/models.py) | `Segment` → `SegmentRule` → `Condition` tree; versioned via `version_of` self-FK ([clone](../../api/segments/models.py#L152)) | Manager plumbing |
| 22 | [api/environments/identities/models.py#L53-L130](../../api/environments/identities/models.py#L53-L130) | `get_all_feature_states`: the Q-object union + priority resolution | Dynamo identity path |
| 23 | [api/environments/identities/views.py#L148-L235](../../api/environments/identities/views.py#L148-L235) | Transient identities, replica reads, edge forwarding, honest TODOs | Deprecated sibling view |
| 24 | [frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts) | The v1-vs-v2 versioning branch a toggle must take | — |

## Senior path — versioning, change management, ops

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 25 | [api/features/versioning/models.py](../../api/features/versioning/models.py) | `EnvironmentFeatureVersion`: immutable published versions | — |
| 26 | [api/features/versioning/versioning_service.py#L132-L360](../../api/features/versioning/versioning_service.py#L132-L360) | `update_flag` writes: create-version → mutate draft → publish; the v1/v2 forked implementations | — |
| 27 | [api/features/workflows/](../../api/features/workflows/) + `ChangeRequest` FK on FeatureState ([models.py#L506-L511](../../api/features/models.py#L506-L511)) | Change requests: approvals + scheduled `live_from` | Enterprise glue |
| 28 | [frontend/common/stores/feature-list-store.ts](../../frontend/common/stores/feature-list-store.ts) | 1,061 lines of legacy Flux — read enough to argue the migration plan, no more | Most of it |
| 29 | [.github/workflows/api-pull-request.yml](../../.github/workflows/api-pull-request.yml) + [frontend-pull-request.yml](../../.github/workflows/frontend-pull-request.yml) | What merging actually requires | Deploy workflows |
| 30 | [api/features/feature_health/](../../api/features/feature_health/) or [api/sse/](../../api/sse/) | One newer subsystem read cold — practice for the job | — |

**Transferable method** (say this in interviews): entry points → data model → one read flow → one write flow → auth boundary → async boundary → tests. Order matters: models before views, flows before features, boundaries before internals.

**Pause-and-predict drill:** before opening file 8, predict from file 7 what the service function must return and why a dict. Before file 22, predict how identity flags combine the three Q-objects. Write predictions down; grade yourself against the code.
