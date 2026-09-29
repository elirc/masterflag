# JS / TS / Node Deep Dive — Question Cards

Thirteen cards. Format per card: what it tests → anchor → junior/mid/senior answer contrast → follow-ups → drill. Say answers **aloud**; reading is not rehearsal.

## Q1: "Walk me through what happens when you `await` inside `Array.map`."
Testing: promise mechanics, common footgun. Anchor: [useFeatureState.ts#L42-L44](../../frontend/common/services/useFeatureState.ts#L42-L44).
Junior: "it waits for each" (wrong). Mid: map returns promises immediately; you need `Promise.all`; this repo does exactly that to parallelize segment fetches. Senior: adds failure semantics — `all` is all-or-nothing; `allSettled` for partial tolerance; and questions whether the fan-out should exist at all (server-side embed).
Follow-ups: error handling in `Promise.all`? ordering guarantees? Drill: 90 seconds aloud with the anchor.

## Q2: "Explain the event loop — tasks vs microtasks."
Testing: runtime model. Anchor: conceptual — no direct anchor; ground it with "promise chains in our RTK Query layer resolve before any timer fires."
Junior: queue diagram recital. Mid: microtasks drain fully between macrotasks; consequences — starvation, `await` = suspension point, UI paint timing. Senior: when it matters in practice: long microtask chains block paint like sync code; batching state updates; why "async ≠ concurrent."
Follow-ups: `setTimeout(fn,0)` vs `queueMicrotask`? Node vs browser differences?

## Q3: "What's a stale closure and how have you actually hit one?"
Testing: closures + React reality. Anchor: [FeaturesPage.tsx#L123-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L123-L134).
Junior: definition. Mid: the subscription-effect example — handler closes over `refetch`; deps + cleanup re-register per change; without them the handler calls a dead reference. Senior: taxonomy — deps lie, refs as escape hatch, why ESLint exhaustive-deps is a *soundness* tool, and event-emitter bridges as the classic breeding ground.
Follow-ups: fix with `useRef`? when is disabling the lint rule legitimate?

## Q4: "How do promise errors get silently lost?"
Testing: error-contract literacy. Anchor: [useToggleFeatureWithToast.ts#L48](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L48) (`.unwrap()`), [service.ts#L33-L38](../../frontend/common/service.ts#L33-L38) (empty catch).
Junior: "use try/catch." Mid: libraries define their own contracts — RTK Query mutations *resolve* with an error field unless unwrapped; a success toast on failure is the resulting bug. Senior: unhandled-rejection monitoring, the empty-catch smell vs deliberate suppression, narrow catches.
Follow-ups: how would you lint for missing `.unwrap()`?

## Q5: "Type an API response so the compiler catches drift."
Testing: TS at boundaries. Anchor: [responses.ts#L1090](../../frontend/common/types/responses.ts#L1090) `Res` registry; wire-vs-app shape at [useFeatureState.ts#L34-L38](../../frontend/common/services/useFeatureState.ts#L34-L38).
Junior: writes an interface. Mid: registry pattern + indexed access `Res['featureStates']`; honest wire types via `Omit`+intersection. Senior: hand-written types can't catch drift — codegen from OpenAPI, CI diff, and the migration path for 60 services.
Follow-ups: `unknown` at the boundary? runtime validation (zod) vs compile-time only?

## Q6: "Generics beyond `identity<T>` — a real example."
Testing: applied generics. Anchor: [PagedResponse\<T\>](../../frontend/common/types/responses.ts#L11-L16).
Junior: syntax demo. Mid: parametrize the *envelope* (pagination) once; `EdgePagedResponse<T>` extends it. Senior: constraint design (`T extends { id: number }` when helpers need it), variance intuition, when to stop (two type params max before refactor).

## Q7: "`unknown` vs `any` — and how do you contain `any` in a big codebase?"
Testing: type-safety strategy. Anchor: `store: any` frontier at [useFeatureState.ts#L71-L93](../../frontend/common/services/useFeatureState.ts#L71-L93); house rule "improve nearby" ([frontend/README.md](../../frontend/README.md)).
Junior: definitions. Mid: `unknown` forces narrowing; `any` is contagious through inference; this repo quarantines `any` to the legacy layer and keeps services strict. Senior: ratchets — `--strict` on new dirs, lint budgets, typed boundaries around untyped cores.

## Q8: "Type guards and discriminated unions — when did they save you?"
Testing: narrowing in practice. Anchors: [isSkeletonItem](../../frontend/web/components/pages/features/FeaturesPage.tsx#L45-L49); level/permission union in [Permission.tsx#L27-L48](../../frontend/common/providers/Permission.tsx#L27-L48).
Junior: syntax. Mid: the Permission props make `level:'project'` + environment-permission a *compile error* — illegal states unrepresentable. Senior: designing unions so the discriminant is cheap and exhaustive (`never` checks in switches), and where the pattern breaks (server data without discriminants).

## Q9: "How does module resolution work in your frontend, and what breaks it?"
Testing: build-system literacy. Anchor: path aliases (`common/`, `components/`) per [frontend/CLAUDE.md](../../frontend/CLAUDE.md); mapping duplicated across tsconfig/rspack/Jest.
Junior: "imports just work." Mid: aliases must agree across three tools; drift = "works in dev, fails in test." Senior: module identity implications (singletons, mocking), why the repo bans relative imports (refactor cost, mock stability).

## Q10: "Node vs browser: what does *this* repo's JS run on?"
Testing: runtime boundaries. Anchor: Express static server ([package.json `start`](../../frontend/package.json)) vs browser SPA vs Jest's Node env; the *API* being Python.
Junior: assumes Node API server. Mid: three JS runtimes, none of which is the backend; `common/` must be isomorphic enough for Jest. Senior: what leaks across (globals, `window` guards, polyfills present in deps) and how you'd catch runtime-specific bugs (env-specific CI lanes).

## Q11: "Debounce a search input — then tell me what's wrong with your answer."
Testing: closures/timing + self-critique. Anchor: [useDebounce.ts](../../frontend/common/useDebounce.ts) / [useDebouncedSearch.ts](../../frontend/common/useDebouncedSearch.ts) — read both before the interview.
Junior: setTimeout version. Mid: hook version with cleanup; stale-closure risk; cancel on unmount. Senior: race with in-flight requests (RTK Query dedupes by arg — the *newer* problem is showing stale results, see `currentData` at [FeaturesPage.tsx#L95](../../frontend/web/components/pages/features/FeaturesPage.tsx#L95)), leading vs trailing edges, and testing with fake timers.

## Q12: "How do you handle errors across an async pipeline (UI → API → worker)?"
Testing: systems thinking in JS terms. Anchor: toggle flow + [webhooks retry](../../api/webhooks/webhooks.py#L63-L80).
Junior: try/catch everywhere. Mid: each boundary has a contract — mutation unwrap + toast; API 4xx/5xx envelopes; worker retries with backoff + terminal notification. Senior: correlate via ids (audit log), idempotency on retries, and "errors the user can act on vs errors ops must act on."

## Q13: "What would you look at first in an unfamiliar JS codebase?"
Testing: method. Anchor: your own [reading order](../01-codebase-cartography/02-file-reading-order.md).
Junior: "the README." Mid: entry points → store/service assembly → one page end-to-end → types → tests; names the two-generation state layer discovery here as the payoff. Senior: adds "find the newest code and the oldest code — the diff between them is the team's direction," and legacy-labelling before touching anything.

---

**Rehearsal protocol:** three cards per session, aloud, 90s each, then re-read the anchors. Record yourself once — the "um, so basically" count is your progress metric.
