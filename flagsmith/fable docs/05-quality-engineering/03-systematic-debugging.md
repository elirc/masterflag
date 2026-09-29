# Systematic Debugging

The method: **reproduce → narrow → hypothesize → test cheaply → fix the root cause → add regression coverage.** Amateurs skip steps 1 and 6; the interview version of every scenario below is "narrate your narrowing," not "guess the answer."

Tools this repo gives you: browser DevTools + Redux DevTools (RTK Query cache inspector), Playwright traces/DOM snapshots per failed E2E ([frontend/README.md](../../frontend/README.md)), Django shell (`make shell` style via manage.py), SQL logging (Django `settings.DEBUG` query log), task-queue tables (inspect via SQL), structlog events, Sentry, CI logs.

---

## Scenario 1: "I toggled a flag but the SDK still returns the old value"

Reproduction: toggle in dashboard; `curl -H "X-Environment-Key: <key>" localhost:8000/api/v1/flags/` shows stale.
First question: is the *write* wrong or the *read* stale? Check the dashboard/API detail endpoint — if it shows the new value, the write is fine; you're in cache/propagation land.
Narrowing path: 1) Check `x-flagsmith-document-updated-at` response header vs your change time ([views.py#L1069-L1072](../../api/features/views.py#L1069-L1072)). 2) Is `CACHE_FLAGS_SECONDS` / `GET_FLAGS_ENDPOINT_CACHE_SECONDS` non-zero? ([views.py#L1019-L1025, #L1057](../../api/features/views.py#L1019-L1057)) → wait TTL, retest. 3) Still stale past TTL → is the task processor running? (`docker compose ps`; task rows pending?) 4) v2 environment? Check the version actually *published* ([versioning_service.py#L193-L196](../../api/features/versioning/versioning_service.py#L193-L196)).
Useful probes: response headers; task table row states; `Environment.updated_at` in DB.
Likely root causes: cache TTL (working as designed), task processor down, unpublished draft version.
Regression test: integration test asserting header bumps after toggle.
Senior lesson: "stale" has *layers* — enumerate them before touching code. Interview narration: name each cache and cross it off with an observation, not a vibe.

## Scenario 2: "Features page shows a new flag only after manual refresh"

Reproduction: create a flag via the modal; list doesn't update.
First question: which write path did the modal use — RTK Query or legacy Flux?
Narrowing: 1) Redux DevTools: did any `service` mutation fire? If not → Flux path ([CreateFlagModal](../../frontend/web/components/pages/features/FeaturesPage.tsx#L187-L198) is legacy). 2) Did `FeatureListStore` emit `saved`? Breakpoint in the bridge ([FeaturesPage.tsx#L123-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L123-L134)). 3) Is the bridge effect mounted (component under test actually FeaturesPage, or a page missing the bridge)? 4) If RTK path: were `invalidatesTags` present on the mutation?
Likely root causes: missing tag invalidation on a newer mutation, or a page that lacks the Flux bridge listener.
Regression: Jest test on the service's tag wiring; E2E create-flag assertion already exists — extend it.
Senior lesson: during migrations, *the first debugging question is "which era is this code path from?"*

## Scenario 3: "Webhook receiver got the same event twice"

Reproduction: partner reports duplicates at their endpoint.
First question: duplicate *delivery* (retry) or duplicate *trigger* (two audit events)?
Narrowing: 1) Compare payload contents/signature — identical body ⇒ retry; differing timestamps ⇒ two triggers. 2) Receiver returned non-2xx or slow? `backoff` retries on failure ([webhooks/webhooks.py#L63-L80](../../api/webhooks/webhooks.py#L63-L80)). 3) Two triggers: was the change made via a path that writes two audit records (e.g. state + value)? Check [audit/signals.py](../../api/audit/signals.py) receivers.
Likely root causes: receiver timeout → at-least-once redelivery (expected!); or double audit rows.
Regression: none for case 1 — instead *document* dedupe requirement; case 2 → unit test on audit-record cardinality per change.
Senior lesson: some "bugs" are contracts. The fix is receiver idempotency, and the interview answer says "at-least-once" unprompted.

## Scenario 4: "E2E flag test fails in CI, passes locally"

Reproduction: CI red on [flag-tests.pw.ts](../../frontend/e2e/tests/flag-tests.pw.ts); local green.
First question: flake or environment difference? Check the retry report — did it pass on retry? ([run-with-retry.ts](../../frontend/e2e/run-with-retry.ts) orchestrates; `E2E_REPEAT` measures flakiness.)
Narrowing: 1) Pull CI artifacts: `error-context.md` DOM snapshot + trace.zip ([frontend/README.md](../../frontend/README.md)). 2) Trace shows what the app displayed — assertion raced a refetch? 3) Reproduce locally with CI-ish conditions: `E2E_CONCURRENCY=20` (contention) or `E2E_REPEAT=5`. 4) If deterministic in CI only: token/env config (`E2E_TEST_AUTH_TOKEN` mismatch) or bundle staleness (`SKIP_BUNDLE` misuse).
Likely root causes: race between Flux-bridge refetch and assertion; concurrency contention on shared API state.
Regression: fix the wait (assert on the post-refetch state), not the timeout number.
Senior lesson: E2E debugging is *artifact* reading, not rerunning and praying.

## Scenario 5: "Segment-targeted flag serves the wrong value to one user"

Reproduction: identity X should match segment "beta" but gets the default.
First question: is the identity *in* the segment, or is the override *losing on priority*?
Narrowing: 1) Dashboard: check identity's traits and segment membership. 2) If in segment: does another override (identity-level, or higher-priority segment) win? Recall the ladder ([models.py#L546-L566](../../api/features/models.py#L546-L566)) — identity override beats segment; between segments, `FeatureSegment` priority decides. 3) Check `FeatureSegment` priorities in DB for that feature/environment. 4) If not in segment: trait types (string "25" vs int 25 in conditions) — inspect rule conditions ([segments/models.py#L251](../../api/segments/models.py#L251)); evaluation happens in the external flag-engine, so reproduce with a unit test against `get_all_feature_states` ([identities/models.py#L53](../../api/environments/identities/models.py#L53)).
Likely root causes: forgotten identity override; segment priority ordering; trait type mismatch.
Regression: unit test pinning the expected winner for that identity/trait fixture.
Senior lesson: when logic spans your code and a library, build the *smallest harness that includes the library* — don't stub the part you suspect.

---

🎤 Interview simulations of scenarios 1, 2, and 5 with timers and follow-ups: [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
