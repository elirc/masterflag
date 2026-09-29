# Tooling and Build System

This file is judgment, not a manual — the manual is [01-codebase-cartography/04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md).

## Bundler: rspack, and why that choice makes sense

rspack is a Rust rewrite of webpack's contract. This codebase is old enough to have deep webpack-shaped config (loaders, handlebars template, SCSS pipeline — [frontend/rspack/](../../frontend/rspack/)); rspack buys build speed **without** a config rewrite, unlike a Vite migration. That is the transferable decision pattern: *when migrating tools, price the config-compatibility surface, not the benchmark chart*. Multiple prod configs exist ([package.json](../../frontend/package.json) `bundle` vs `bundledjango`) because the dashboard ships two ways: standalone Express server or baked into the Django image.

## Type-checking vs transpilation are split

Babel transpiles (fast, no type knowledge); `tsc` checks (`npm run typecheck`) and emits nothing. Consequences you should be able to recite: dev server never blocks on type errors; type errors only bite where the check runs (hooks/CI); Babel-TS has per-file semantics limits (e.g. `const enum`). This split is one of the most common real-world TS setups and a reliable interview topic.

## Test tooling as a system

- **Jest** for units next to source (`__tests__/`, e.g. [frontend/common/utils/\_\_tests\_\_/](../../frontend/common/utils/__tests__/)).
- **Playwright** E2E against a *real* API, with a bespoke retry runner ([e2e/run-with-retry.ts](../../frontend/e2e/run-with-retry.ts)), concurrency knobs, and artifacts (traces, DOM snapshots) per failure ([frontend/README.md](../../frontend/README.md)).
- **Visual regression** captured during E2E, compared out-of-band, **report-only in CI** — flake containment as policy.
- Backend: pytest + xdist, fixtures-first culture, 100% diff coverage.

The system-level insight: each layer has an explicit *flake budget*. Units must be deterministic; E2E gets retries; visuals can't fail CI at all. When you design CI, you're allocating trust, not just running commands.

## Dependency management

- Frontend: `package-lock.json`, npm. Check the `postinstall` → `npm run env` hook ([package.json](../../frontend/package.json)) — installing *configures* the app for an environment; surprising but deliberate.
- API: `uv` with a lockfile; **only maintainers can re-lock** because private indexes are involved ([api/README.md](../../api/README.md), [Makefile#L23-L28](../../api/Makefile#L23-L28)) — a real-world constraint OSS contributors must respect: never touch `uv.lock` in a PR.
- Private modules (SAML, RBAC) are cloned into site-packages at install ([Makefile#L41-L49](../../api/Makefile#L41-L49)) — the open-core seam made visible.

## Local ergonomics

`make install-hooks` (husky) at repo root; frontend lint autofix `npm run lint:fix`; env switching via `bin/env.js`. When something "doesn't run," your debugging order is: which env file got copied → which config the script selected → what the hook rejected.

🎤 **Interview angle:** (1) "Walk me through your frontend build" — bundler, transpile/check split, env selection, two deploy targets; (2) "How do you manage E2E flakiness?" — retries + artifacts + report-only visuals, plus the cost (a masked real regression rides the retry); (3) "Monorepo dependency risks?" — lockfile discipline and the private-package seam.

**Drill:** trace `ENV=local npm run dev` from package.json through [bin/env.js](../../frontend/bin/env.js) to the rspack config it launches; write the three files it touches. Basic: found the script chain. Solid: explained the project.js copy. Strong: explained how `globalThis.projectOverrides` lets ops change config without a rebuild, and one risk of that mechanism.
