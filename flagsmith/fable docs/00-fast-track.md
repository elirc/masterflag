# Fast Track — One Weekend in Flagsmith

Goal: by Sunday night you can run the platform, trace two end-to-end flows aloud, and have made one safe change with a passing test.

## Saturday morning — run it

All commands below are __inferred__ from repo scripts and READMEs unless marked otherwise (this curriculum was written on Windows without Docker running; see [09-reference/verification-log.md](09-reference/verification-log.md)).

```bash
# Whole platform in Docker (from repo root) — easiest path
docker-compose -f docker-compose.yml up
# Watch the logs for the superuser password-reset link, then open http://localhost:8000
```

The compose file ([docker-compose.yml](../docker-compose.yml)) runs four services: `postgres`, `migrate-db` (one-shot migrations), `flagsmith` (API + bundled frontend), and `flagsmith-task-processor` — note that last one: **async work is a separate deployable**, which will matter in every architecture conversation later.

For frontend development against that API ([frontend/README.md](../frontend/README.md)):

```bash
cd frontend
npm install            # Node 22.x / npm 10.x
ENV=local npm run dev  # dev server against localhost:8000
npm run typecheck      # tsc, no emit
npm run test:unit      # Jest
```

For the API ([api/README.md](../api/README.md), requires Python 3.11–3.13, GNU Make, Docker):

```bash
cd api
make install
make docker-up django-migrate  # dev database
make serve                     # or: make serve-with-task-processor
make test                      # pytest; subset: make test opts='-k <expr>'
```

## Saturday afternoon — trace flow #1: an SDK asks for flags

This is the product's reason to exist. Read these in order:

1. [api/environments/authentication.py#L14-L49](../api/environments/authentication.py#L14-L49) — how an SDK authenticates: an `X-Environment-Key` header, resolved to an `Environment` through a cache, no user at all. Pause and predict: what happens if the key is garbage? (Answer: negative caching — [api/environments/models.py#L273-L274](../api/environments/models.py#L273-L274).)
2. [api/features/views.py#L1004-L1073](../api/features/views.py#L1004-L1073) — `SDKFeatureStates.get`: two caching layers (HTTP `cache_page` + a manual flags cache), a replica read, and filters that differ for client vs server keys.
3. [api/features/versioning/versioning_service.py#L51-L114](../api/features/versioning/versioning_service.py#L51-L114) — `get_environment_flags_list`: one query fetches all candidate feature states, then **Python picks the winner per feature** using `FeatureState.__gt__`.
4. [api/features/models.py#L529-L601](../api/features/models.py#L529-L601) — the `__gt__` priority ladder: identity > segment > environment default. This is the domain's core invariant expressed as an operator overload.

Say it aloud, interview style: *"An SDK sends an environment key; auth middleware resolves it from cache and tags the request with client/server origin; the view serves from cache or asks the versioning service, which loads all live candidate states in one query and resolves priority in Python; identity overrides beat segment overrides beat defaults."* If you can say that unprompted, you've earned dinner.

## Sunday morning — trace flow #2: a human toggles a flag

1. [frontend/web/components/pages/features/FeaturesPage.tsx#L150-L165](../frontend/web/components/pages/features/FeaturesPage.tsx#L150-L165) — the toggle callback.
2. [frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L23-L78](../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L23-L78) — branches on `environment.use_v2_feature_versioning`: versioned environments create-and-publish a version; legacy ones PUT the feature state directly.
3. [frontend/common/services/useFeatureState.ts#L53-L67](../frontend/common/services/useFeatureState.ts#L53-L67) — the RTK Query mutation and its `invalidatesTags` — this is how the UI refetches without manual wiring.
4. Server side: permission check in [api/features/permissions.py#L133-L151](../api/features/permissions.py#L133-L151), then the save produces a history record, an `AuditLog`, and — via [api/audit/models.py#L141-L168](../api/audit/models.py#L141-L168) — bumps `environment.updated_at` and queues `process_environment_update` ([api/environments/tasks.py#L31-L45](../api/environments/tasks.py#L31-L45)) to rebuild environment documents and notify SSE clients.

One write, five downstream effects, all decoupled through an audit record and a task queue. Remember this shape.

## Sunday afternoon — the first 10 files, one safe change

Open in order:

1. [AGENTS.md](../AGENTS.md) + [api/README.md](../api/README.md) + [frontend/README.md](../frontend/README.md) — the house rules
2. [docker-compose.yml](../docker-compose.yml) — deployable shape
3. [api/app/urls.py](../api/app/urls.py) — API surface roots
4. [api/features/models.py](../api/features/models.py) — Feature / FeatureState / FeatureSegment
5. [api/environments/models.py](../api/environments/models.py) — Environment, caching, document builds
6. [frontend/common/store.ts](../frontend/common/store.ts) + [frontend/common/service.ts](../frontend/common/service.ts) — Redux + RTK Query base
7. [frontend/common/types/responses.ts](../frontend/common/types/responses.ts) — the API contract as the frontend sees it
8. [frontend/web/components/pages/features/FeaturesPage.tsx](../frontend/web/components/pages/features/FeaturesPage.tsx) — modern page with a legacy bridge
9. [api/tests/conftest.py](../api/tests/conftest.py) — the fixture vocabulary all API tests speak
10. [frontend/e2e/tests/flag-tests.pw.ts](../frontend/e2e/tests/flag-tests.pw.ts) — what "working" means end to end

**One safe change:** pick a frontend unit-tested utility (e.g. anything under [frontend/common/utils/\_\_tests\_\_/](../frontend/common/utils/__tests__/)), add one test case for an uncovered edge (empty input, unicode, boundary number). Run `npm run test:unit -- --testPathPatterns=<file>`. You've now touched the codebase without touching product behaviour — the correct first move in any unfamiliar repo.

**Teach-back exercise:** explain flow #1 to a rubber duck as if the interviewer asked *"walk me through a codebase you've studied recently."* Two minutes, no notes. Rubric: named the auth boundary? named both caches? named the priority invariant? said one thing you'd improve?

## What the fast path skips

Segments and the flag engine, change requests/workflows, the DynamoDB edge path, integrations, RBAC internals, the legacy Flux stores, observability, and everything about contributing. That's what the other nine modules are for.
