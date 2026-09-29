# Frontend Framework Questions — Question Cards

Eleven cards on React as this repo uses it.

## Q1: "Render vs commit — why should I care?"
Anchor: any FeaturesPage render path. Junior: lifecycle recital. Mid: render must be pure and discardable (StrictMode double-render as the enforcement); effects run post-commit — which is why the Flux bridge lives in effects, not render ([FeaturesPage.tsx#L99-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L99-L134)). Senior: what breaks purity in real apps (store writes during render, `Date.now()` in memo inputs) and how you'd detect it.
Follow-up: what does React 19 change about your mental model? (Compiler-era memoization expectations — be honest about what you haven't used.)

## Q2: "Server state vs client state — where do you draw the line?"
Anchor: RTK Query owns server entities; the page owns filters/pagination in the URL ([useFeatureFilters](../../frontend/web/components/pages/features/hooks/useFeatureFilters.ts)); view mode in a small hook ([useViewMode](../../frontend/common/useViewMode.ts)). Junior: "Redux for everything." Mid: cache-with-lifecycle vs owned-state distinction; URL-as-state for shareability. Senior: the migration cost when you get it wrong — this repo's Flux stores are exactly "server state treated as client state," and the bridge is the bill.

## Q3: "How does RTK Query decide when to refetch?"
Anchor: tags at [useFeatureState.ts#L20-L29, L57-L61](../../frontend/common/services/useFeatureState.ts#L20-L61); `refetchOnFocus/Reconnect` at [service.ts#L58-L59](../../frontend/common/service.ts#L58-L59). Junior: "it just caches." Mid: subscription lifetimes, tag invalidation graph, focus refetch. Senior: failure modes — missing tags (silent stale), broad tags (storms), and `keepUnusedDataFor` tuning; contrast with React Query/SWR vocabulary.

## Q4: "Optimistic updates — would you add them to this toggle?"
Anchor: [useToggleFeatureWithToast.ts](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts) (currently pessimistic + toast). Junior: "yes, feels faster." Mid: onQueryStarted + patchResult rollback sketch; why a *flag platform* toggle might stay pessimistic (the truth matters more than 300ms). Senior: invariants under failure (never show enabled-that-isn't), server-authoritative UI for audit-heavy domains, idempotent retries.

## Q5: "Why is there a `cloneDeep` in this page, and what does it tell you?"
Anchor: [FeaturesPage.tsx#L112-L116](../../frontend/web/components/pages/features/FeaturesPage.tsx#L112-L116). Junior: "copies data." Mid: RTK cache must stay immutable; legacy Flux consumers mutate — the clone is a firewall between paradigms. Senior: reads it as migration archaeology: cost per fetch, the TODO's exit condition, and how you'd stage the deletion (project 1 in [06/03](../06-contribution-practice/03-senior-build-projects.md)).

## Q6: "When is `useMemo`/`useCallback` worth it?"
Anchor: [renderFeatureRow](../../frontend/web/components/pages/features/FeaturesPage.tsx#L248-L291) feeding a virtualized list vs the memoized `paging` trio (L167-L175). Junior: wraps everything. Mid: referential stability for identity-sensitive consumers (list rows, deps arrays) — measure first otherwise. Senior: the *cost* of memoization (deps bookkeeping bugs — review kata 2), and what changes with the React compiler.

## Q7: "Keys — beyond 'don't use index'."
Anchor: `key={projectFlag.id}` ([FeaturesPage.tsx#L256](../../frontend/web/components/pages/features/FeaturesPage.tsx#L256)); skeleton rows keyed `skeleton-${i}` (L251) — index keys are *fine* there. Junior: recites the rule. Mid: keys are identity for state preservation; index keys break under reorder/filter; static placeholder lists are the legitimate exception. Senior: keys as a reconciliation control tool (deliberate key change = remount/reset).

## Q8: "How do you gate UI by permissions without security theatre?"
Anchor: [Permission.tsx](../../frontend/common/providers/Permission.tsx) + server truth ([features/permissions.py](../../api/features/permissions.py)). Junior: hides buttons. Mid: render-prop over a cached permission query; UI is UX, server is law; discriminated-union props catch level/permission mismatches at compile time. Senior: the drift problem (duplicated permission strings FE/BE) and a codegen/SSOT fix; disabled-with-tooltip vs hidden as product policy.

## Q9: "Accessibility — what would you audit first on this features table?"
Anchor: [PanelSearch usage](../../frontend/web/components/pages/features/FeaturesPage.tsx#L305-L327) (conceptual — audit for real). Junior: "add aria-labels." Mid: keyboard path for toggle switches (rc-switch — check focus/space handling), search input labelling, toast announcements (aria-live), focus management in the side-modal. Senior: an audit *method* — keyboard-only pass, screen-reader pass on one flow, then systemic fixes in shared components not pages.

## Q10: "Tell me about testing React beyond snapshots."
Anchor: table-driven utils ([format.test.ts](../../frontend/common/utils/__tests__/format.test.ts)); the E2E layer with retries/artifacts ([frontend/README.md](../../frontend/README.md)). Junior: snapshot everything. Mid: behaviour-first — user-visible assertions, hook tests for URL round-trips (ticket 14), E2E only for journeys. Senior: the flake economy (retries/report-only visuals) and what each layer is *allowed* to cost.

## Q11: "This page is slow. Go."
Anchor: [performance-thinking](../05-quality-engineering/04-performance-thinking.md) hotspot list. Junior: `useMemo`. Mid: Profiler → domain (render vs network vs server); names this repo's client N+1 as a real found-in-the-wild example with the fix. Senior: budgets and regression guards (request-count assertions, bundle budgets) so it *stays* fixed.

---

**Rehearsal:** cards 2, 3, 5 are the highest-yield — they turn "do you know React" into "have you operated a real React codebase," which is the entire mid-level question.
