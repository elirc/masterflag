# Language & Runtime Model (JS/TS)

## Event loop, tasks, microtasks

Mental model: one call stack; promise callbacks run as **microtasks** that drain completely before the next macrotask (timers, I/O) or render. Why it matters in production: a long microtask chain blocks paint just as surely as a `while` loop; "await in a loop" turns parallelizable I/O into serial latency.

Where this repo exercises it:
- [useFeatureState.ts#L42-L44](../../frontend/common/services/useFeatureState.ts#L42-L44): `await Promise.all(data.results.map(addFeatureSegmentsToFeatureStates))` — deliberately **parallel** fan-out. Contrast with what a `for…of await` would do: N sequential round-trips.
- [useToggleFeatureWithToast.ts#L36-L59](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L36-L59): sequential `await` because the second action (toast) genuinely depends on the first finishing.

Failure modes: unhandled rejection when one of a `Promise.all` batch fails (all-or-nothing — is that what the UI wants?); `await` inside `Array.map` returning an array of promises nobody awaits.

**Drill:** in [useFeatureState.ts](../../frontend/common/services/useFeatureState.ts), what happens to the whole query if *one* `getFeatureSegment` call rejects? Trace it; then propose the `Promise.allSettled` alternative and its UX tradeoff. Basic: "the query errors." Solid: traced through RTK Query's `queryFn` error contract. Strong: articulated partial-failure UX options and picked one with a reason.

## Serial vs parallel work, and where it hides

The repo's own risk register entry #1 is a latency bug born from client-side fan-out. The transferable rule: **count round-trips per user action**. One list render here costs `1 + (rows with numeric feature_segment)` requests.

## Closures and stale state

A closure captures *variables*, not values-at-render. In React that becomes the stale-closure bug: an effect or callback reads an old value because its dependency array pinned an old closure.

Where this repo defends against it: [FeaturesPage.tsx#L123-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L123-L134) — the Flux-bridge effect closes over `refetch` and correctly lists it in deps; the event handlers are re-registered when it changes, and the cleanup unsubscribes the *old* closure. Read the cleanup function until that sentence is obvious.

Failure mode to recognize anywhere: subscribing in an effect with `[]` deps while the handler reads changing props — works in the demo, goes stale in production.

## Modules and import discipline

ESM-style imports everywhere, bundled by rspack; path aliases (`common/`, `components/`) instead of relative imports — a house rule ([frontend/CLAUDE.md](../../frontend/CLAUDE.md)). Why aliases: moves are cheap, and module identity stays stable for Jest mocks. Cost: tooling must agree ([tsconfig.json](../../frontend/tsconfig.json) + rspack + Jest config all repeat the mapping).

## Error handling idioms

- RTK Query mutations **don't throw** unless you `.unwrap()` — verified usage at [useToggleFeatureWithToast.ts#L48,L58](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L48).
- Swallowed-error smell exists in the wild here: `catch (e) {}` around token reading in [service.ts#L33-L38](../../frontend/common/service.ts#L33-L38) — defensible (missing storage shouldn't kill requests) but note it silences *all* errors, not just the expected one. That's review-comment material, and exactly the shape [04-code-reading-gym/03-fake-code-contrasts.md](../04-code-reading-gym/03-fake-code-contrasts.md) drills.

## Node vs browser vs the third runtime in the room

`frontend/common/` must run under Jest (Node) and the browser; `frontend/api/` is an Express server; and the *actual* backend is Python. When you see `node-fetch`, `body-parser`, `express` in [package.json](../../frontend/package.json), that's the thin SSR/static server, not the API.

🎤 **Interview angle** (cards in [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md)):
1. "Explain the event loop" → answer with the `Promise.all` fan-out anchor, not textbook prose.
2. "What's a stale closure?" → the FeaturesPage subscription effect is your worked example.
3. "Sequential vs parallel awaits?" → segment fan-out vs toggle flow.
4. "When do promise errors get lost?" → `.unwrap()` and the empty catch.
