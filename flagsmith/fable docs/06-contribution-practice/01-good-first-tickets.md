# Good First Tickets

Sixteen junior-sized tickets spread across the repo. Fields are compressed but complete; treat "Reject risk" as the maintainer's voice. Verify each is still unfixed upstream before investing (that check is Skill #1 of contributing). Tests-only and types-only tickets are deliberately over-represented — they're the classic low-blast-radius entry point.

---

## Ticket 1: Cross-tenant regression test for feature-state UUID lookup
Difficulty: Easy — 2h. Skills: pytest fixtures, IDOR thinking.
Story: as a maintainer, I want the tenant-scoping of `get_feature_state_by_uuid` pinned by a test so refactors can't silently drop it.
Why good: pure test addition; zero runtime risk; exercises the security fixture patterns.
Acceptance: ☐ test builds a second organisation ☐ asserts 404 not 403 ☐ naming template followed.
Read first: [views.py#L992-L1001](../../api/features/views.py#L992-L1001), [conftest.py#L294-L332](../../api/tests/conftest.py#L294-L332). Touches: `api/tests/unit/features/test_unit_features_views.py`.
Plan: recipe 4 in [05/02](../05-quality-engineering/02-writing-tests-here.md) verbatim.
Could go wrong: fixture soup — keep the second-org setup inline and minimal.
Reject risk: duplicate of an existing test — grep `get_feature_state_by_uuid` under tests first.
Interview story potential: "I added tenant-isolation regression coverage to a multi-tenant OSS platform" (security + testing).

## Ticket 2: Unit tests for `featureValuesEqual`
Difficulty: Easy — 2h. Skills: Jest, edge-case enumeration.
Story: as a developer, I want the value-comparison util covered so flag-diff UI can't regress silently.
Why good: [frontend/common/featureValuesEqual.ts](../../frontend/common/featureValuesEqual.ts) is small, pure, and (check!) likely undertested; table-driven tests fit the house style ([format.test.ts](../../frontend/common/utils/__tests__/format.test.ts)).
Acceptance: ☐ `it.each` table ☐ covers null/undefined/""/0/false/numeric-string cases ☐ `npm run test:unit -- --testPathPatterns=featureValuesEqual` green.
Could go wrong: you discover an actual inconsistency (e.g. `"1"` vs `1`) — that becomes a *separate* bug report, not a sneaky behaviour change in a test PR.
Reject risk: tests that pin accidental behaviour; state intent in the PR.
Interview story: "testing revealed an equality edge case in flag value comparison" (attention to contracts).

## Ticket 3: Type the imperative service helpers (`store: any`)
Difficulty: Easy–Medium — 3h. Skills: TS, RTK types.
Story: as a maintainer, I want `getFeatureStates(store: any, …)` typed so misuse fails at compile time.
Read first: [useFeatureState.ts#L71-L93](../../frontend/common/services/useFeatureState.ts#L71-L93), store types in [store.ts](../../frontend/common/store.ts). Grep sibling services for the same signature — fix the pattern in ONE file first.
Acceptance: ☐ no new `any` ☐ `npm run typecheck` green ☐ no runtime change.
Reject risk: a 60-file mechanical PR — maintainers prefer one exemplar + follow-ups.
Interview story: "incremental typing improvements in a large mixed-typed codebase" (pragmatism).

## Ticket 4: Named type for an inline union
Difficulty: Easy — 1h. Skills: TS, house conventions ([frontend/CLAUDE.md](../../frontend/CLAUDE.md) rule 6).
Story: extract one inline union (grep `: '` in components for candidates, e.g. toast/danger variants) into a named exported type.
Acceptance: ☐ single named type reused at all call sites of that union ☐ typecheck green.
Reject risk: churn without consumer benefit — pick a union used ≥3 places.
Interview story: minor, but stacks into "I internalize team conventions fast."

## Ticket 5: Unit test for server-key-only flag filtering
Difficulty: Medium — 4h. Skills: DRF, security-relevant testing.
Story: as a maintainer, I want `_additional_filters` behaviour (client keys never see `is_server_key_only` flags) pinned at the view level.
Read first: [views.py#L1075-L1085](../../api/features/views.py#L1075-L1085), [authentication.py#L37-L41](../../api/environments/authentication.py#L37-L41). Check existing coverage in [test_unit_features_views.py](../../api/tests/unit/features/test_unit_features_views.py) first.
Acceptance: ☐ parametrized client vs server key ☐ asserts presence/absence of the flag in response.
Could go wrong: cache decorators interfering — disable/vary settings via fixtures.
Interview story: "I tested an authorization filter disguised as a query filter" (great security vocabulary).

## Ticket 6: Structured log when webhooks are globally disabled
Difficulty: Easy–Medium — 3h. Skills: structlog conventions, tiny API change.
Story: as an operator, when `DISABLE_WEBHOOKS` silently swallows events ([webhooks.py#L78-L79](../../api/webhooks/webhooks.py#L78-L79)) I want one log line telling me why nothing fired (risk-register #9).
Acceptance: ☐ event named per convention (e.g. `webhooks.delivery.skipped`) ☐ includes environment/organisation id ☐ log-capture test (`pytest-structlog`) ☐ no behaviour change.
Reject risk: log spam — emit at debug/info and only when a webhook *would* have fired.
Interview story: "I turned a silent failure mode into an observable one" (ops maturity).

## Ticket 7: Negative-cache test for `Environment.get_from_cache`
Difficulty: Medium — 3h. Skills: caching semantics, Django test caches.
Story: pin that an unknown api_key results in `set_bad_key` and short-circuits the DB on the second call ([models.py#L268-L302](../../api/environments/models.py#L268-L302)).
Acceptance: ☐ asserts DB query count drops on second call (`django_assert_num_queries`) ☐ cache cleared between tests.
Could go wrong: cache backend differences in test settings — check how existing env-cache tests configure caches (grep `environment_cache` in tests).
Interview story: "I wrote query-count assertions around a negative cache" (performance testing).

## Ticket 8: Query-count regression test on the flags endpoint
Difficulty: Medium — 4h. Skills: N+1 detection, pytest.
Story: as a maintainer, I want `GET /api/v1/flags/` to keep constant query count regardless of feature count, guarding the `select_related` set ([versioning_service.py#L542-L556](../../api/features/versioning/versioning_service.py#L542-L556)).
Acceptance: ☐ two environments (5 vs 25 features) same query count ☐ caches disabled in test.
Reject risk: brittle exact-count assertions — assert equality between sizes, not a magic number.
Interview story: "I added an N+1 tripwire to a hot endpoint" — excellent performance interview evidence.

## Ticket 9: Fix `console.warn` toggle dead-end UX
Difficulty: Easy–Medium — 3h. Skills: React, error UX.
Story: when `environmentFlag` is undefined, toggling warns to console and silently no-ops for the user ([useToggleFeatureWithToast.ts#L30-L35](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L30-L35)); show the error toast instead (consistent with L61-69 path).
Acceptance: ☐ user-visible toast ☐ Jest test for the branch ☐ no change to the happy path.
Reject risk: product opinion — check with maintainers whether this state is reachable in practice; link the reproduction.
Interview story: "smallest possible UX-correctness fix, shipped with a test."

## Ticket 10: Storybook story for `FeatureRowSkeleton`
Difficulty: Easy — 2h. Skills: Storybook, component API reading.
Story: skeleton components ([FeatureRowSkeleton](../../frontend/web/components/feature-summary/FeatureRowSkeleton.tsx)) lack stories (verify: grep `.stories.` nearby); add one so designers can review loading states.
Acceptance: ☐ story renders in `npm run storybook` ☐ follows existing story file conventions (find one via glob `*.stories.*`).
Reject risk: low value if the team doesn't use Storybook actively — check recent story-file commit dates first (`git log`).
Interview story: minor; "I improved component documentation."

## Ticket 11: Test `isFreeEmailDomain` edge cases
Difficulty: Easy — 1–2h. Skills: Jest.
Story: extend [isFreeEmailDomain tests](../../frontend/common/utils/__tests__/isFreeEmailDomain.test.ts) with case-sensitivity, subdomain, and plus-addressing cases.
Acceptance: ☐ new table rows ☐ any *discovered* bug filed separately.
Interview story: minor; stacking evidence of test-first instincts.

## Ticket 12: Resolve one `# type: ignore` with a documented reason
Difficulty: Medium — 3–5h. Skills: mypy, Django typing.
Story: the API README explicitly welcomes resolving existing ignores ([api/README.md](../../api/README.md) typing section). Pick ONE in code you now understand (e.g. a `no-untyped-def` in [features/permissions.py](../../api/features/permissions.py)) and type it properly.
Acceptance: ☐ `make typecheck` green ☐ no runtime change ☐ adjacent ignores considered per house guidance.
Reject risk: type changes that ripple into dozens of files — pick a leaf function.
Interview story: "I removed type debt in a strict-mypy Django codebase" — surprisingly strong signal for a JS dev (range).

## Ticket 13: Document the three flag-read caches in the deployment docs
Difficulty: Easy–Medium — 3h. Skills: technical writing, docs site.
Story: as a self-hoster, I want one docs page explaining `ENVIRONMENT_CACHE_SECONDS`, `CACHE_FLAGS_SECONDS`, `GET_FLAGS_ENDPOINT_CACHE_SECONDS` and their staleness interplay ([views.py#L1019-L1058](../../api/features/views.py#L1019-L1058)) — verify current docs coverage under [docs/docs/deployment-self-hosting/](../../docs/docs/deployment-self-hosting/) first.
Acceptance: ☐ table of TTL, layer, invalidation ☐ `make lint` (prettier) green.
Reject risk: duplicating existing env-var reference — extend, don't fork.
Interview story: "I documented cache semantics operators kept tripping on" (communication).

## Ticket 14: Jest test for `useFeatureFilters` URL round-trip
Difficulty: Medium — 4h. Skills: hook testing, router mocking.
Story: filters live in the URL ([useFeatureFilters.ts](../../frontend/web/components/pages/features/hooks/useFeatureFilters.ts)); pin the parse/serialize round-trip so shared links keep working.
Acceptance: ☐ renderHook with router wrapper ☐ set filter → URL updated → re-parse equals original.
Could go wrong: react-router v5 test wrappers — copy an existing hook test setup if one exists (search `renderHook` in frontend).
Interview story: "testing URL-as-state" — a genuinely good frontend-interview topic.

## Ticket 15: Assert webhook payloads carry the signature header in tests
Difficulty: Medium — 3h. Skills: security testing, mocks.
Story: pin that environment webhook deliveries include `FLAGSMITH_SIGNATURE_HEADER` when a secret is configured ([webhooks.py#L17-L18](../../api/webhooks/webhooks.py#L17-L18)); check existing coverage in [test_unit_webhooks.py](../../api/tests/unit/webhooks/test_unit_webhooks.py) first — extend, don't duplicate.
Acceptance: ☐ asserts header presence + HMAC correctness against the body.
Interview story: "I tested webhook signing" — pairs with the webhook design interview card.

## Ticket 16: E2E: assert flag toggle survives a page reload
Difficulty: Medium — 4h. Skills: Playwright, E2E judgment.
Story: extend [flag-tests.pw.ts](../../frontend/e2e/tests/flag-tests.pw.ts) with a reload after toggle, asserting persisted state — catches invalidation/persistence bugs the current in-page assertions might miss (verify it's not already covered).
Acceptance: ☐ follows existing helpers/tags ☐ passes locally 3× with `E2E_REPEAT=2` ☐ no fixed sleeps.
Reject risk: E2E minutes are expensive — justify with the class of bug it catches (stale cache masking).
Interview story: "I extended an OSS E2E suite and dealt with flake discipline."

---

**Working protocol for all tickets:** branch → smallest diff → run the layer's checks locally ([cheatsheet](../09-reference/command-cheatsheet.md)) → PR description per [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) (what/why/how-tested/risk) → expect and welcome review pushback.
