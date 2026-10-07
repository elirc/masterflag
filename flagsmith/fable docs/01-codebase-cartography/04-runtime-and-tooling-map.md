# Runtime and Tooling Map

## Runtimes and boundaries

| Surface | Runtime | Entry | Notes |
| --- | --- | --- | --- |
| Dashboard | Browser (React 19) | [frontend/web/main.js](../../frontend/web/main.js), routes in [frontend/web/routes.js](../../frontend/web/routes.js) (react-router v5) | SPA; all data via `/api/v1` |
| FE static server | Node/Express | [frontend/package.json](../../frontend/package.json) `start` → `node ./api/index` | Thin — serves bundle, no business logic |
| API | Python/Django 5 + DRF, gunicorn | [api/manage.py](../../api/manage.py), apps under [api/](../../api/) | Also serves the dashboard bundle in the all-in-one image |
| Task processor | Same Django codebase, different process | compose service [docker-compose.yml#L81](../../docker-compose.yml#L81) | Consumes tasks queued in Postgres |
| SSE / realtime | [api/sse/](../../api/sse/) | Pushes "environment updated" signals | |
| Edge (SaaS only) | DynamoDB-backed edge API | call sites e.g. [identities/views.py#L198-L206](../../api/environments/identities/views.py#L198-L206) | Out of OSS scope but shapes the code |

Boundary rule worth quoting: **browser code lives in `frontend/web/`, shareable logic in `frontend/common/`** (enforced by convention, [frontend/CLAUDE.md](../../frontend/CLAUDE.md) forbids relative imports and raw `fetch`).

## Frontend toolchain

| Tool | Config | What it does |
| --- | --- | --- |
| **rspack** (webpack-compatible, Rust) | [frontend/rspack/](../../frontend/rspack/), selected per script | Bundling + dev server; `bundledjango` variant builds into the Django image |
| Babel + TS | [frontend/tsconfig.json](../../frontend/tsconfig.json) | `npm run typecheck` runs `tsc` as a *checker only* — transpilation is Babel's job, so type errors never block the dev server. Classic setup to name in interviews |
| Jest | [frontend/jest.config.js](../../frontend/jest.config.js) | Unit tests in `__tests__/` dirs |
| Playwright | [frontend/playwright.config.ts](../../frontend/playwright.config.ts), tests in [frontend/e2e/tests/](../../frontend/e2e/tests/) | E2E vs a real API on :8000; retries orchestrated by [e2e/run-with-retry.ts](../../frontend/e2e/run-with-retry.ts) |
| ESLint + husky | repo hooks via `make install-hooks` | `npm run lint` |
| Env selection | `frontend/bin/env.js` (invoked by `npm run env`, [frontend/package.json#L25](../../frontend/package.json#L25), but not tracked in this snapshot — behavior described from upstream docs, treat as inferred) copies `env/project_<ENV>.js` → `common/project.js`; runtime overrides via `globalThis.projectOverrides` | Build-time env with deploy-time escape hatch — a pattern worth stealing |

## API toolchain

| Tool | Config | What it does |
| --- | --- | --- |
| **uv** | [api/pyproject.toml](../../api/pyproject.toml), `uv.lock` | Dependency management (npm-lockfile equivalent) |
| pytest (+xdist) | [api/tests/](../../api/tests/), fixtures in [api/tests/conftest.py](../../api/tests/conftest.py) | `make test`; **100% diff coverage required** |
| mypy strict | `make typecheck` | Whole codebase incl. tests; `# type: ignore` needs a reason |
| pre-commit | `make lint` | Formatters + linters as git hooks |
| Django migrations | per-app `migrations/` | `make docker-up django-make-migrations`; names are hand-written |
| drf-spectacular | OpenAPI generation; CI diffs generated docs ([api-pull-request.yml](../../.github/workflows/api-pull-request.yml)) | Contract kept honest by CI |

## Environment variables (high level, no secrets)

- **API**: `DATABASE_URL`, cache tuning (`ENVIRONMENT_CACHE_SECONDS`, `CACHE_FLAGS_SECONDS`, `GET_FLAGS_ENDPOINT_CACHE_SECONDS` — used at [features/views.py#L1019-L1025](../../api/features/views.py#L1019-L1025)), `TASK_RUN_METHOD`, `DISABLE_WEBHOOKS`, integration credentials. Grep `settings.` before assuming a default.
- **Frontend**: `ENV` selects the project config; E2E knobs (`E2E_CONCURRENCY`, `E2E_RETRIES`, `SKIP_BUNDLE`) documented in [frontend/README.md](../../frontend/README.md).

🎤 **Interview angle:** three tooling questions this repo answers concretely — (1) "How do TS projects separate type-checking from transpilation, and what's the risk?" (tsc-as-linter; type errors don't stop builds); (2) "How do you run E2E against a real backend in CI?" (compose + token bootstrap + retries); (3) "What does a strict-typing + 100%-diff-coverage policy cost and buy?" (see [api/README.md](../../api/README.md) — buy: refactor confidence; cost: slower first PRs, fixture investment).

**Drill:** run `npm run typecheck` and `npm run lint` in `frontend/`, then find which rspack config `ENV=local npm run dev` uses and what differs from prod. Basic: commands ran. Solid: named the config file. Strong: explained the `project.js` copy mechanism and its deploy-time override design.
