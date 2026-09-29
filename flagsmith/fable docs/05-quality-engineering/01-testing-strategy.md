# Testing Strategy

## The layers this repo actually has

| Layer | Where | Runs | What it's for | What it must NOT be used for |
| --- | --- | --- | --- | --- |
| API unit | [api/tests/unit/](../../api/tests/unit/) (mirrors app structure) | `make test`, CI | Module/class/function behaviour incl. permissions, serializers, tasks — *with* DB access via fixtures | Full HTTP round-trips (that's integration) |
| API integration | [api/tests/integration/](../../api/tests/integration/) | same | Black-box endpoint tests: request in, response out ([api/README.md](../../api/README.md)) | Poking internals — if you import the module under test, you're in the wrong directory |
| FE unit (Jest) | `__tests__/` beside source, e.g. [common/utils/\_\_tests\_\_/](../../frontend/common/utils/__tests__/) | `npm run test:unit`, CI | Pure logic, hooks, small components | Anything needing a real API |
| E2E (Playwright) | [frontend/e2e/tests/](../../frontend/e2e/tests/) | `npm run test` vs real API | User journeys incl. permissions ([environment-permission-test.pw.ts](../../frontend/e2e/tests/environment-permission-test.pw.ts)), versioning, change requests | Edge cases enumerable at lower layers |
| Visual regression | captured in E2E, compared separately | report-only in CI | Style drift detection | Gating (deliberately) |

House rules that shape everything ([api/README.md](../../api/README.md)): **100% diff coverage**, no class-based tests, fixtures over setup methods, Given/When/Then structure, and the naming contract `test_{subject}__{condition}__{expected}` — e.g. `test_get_version__valid_file_contents__returns_version_number`. That name template is a *thinking* tool: if you can't fill the three slots, you don't know what you're testing yet.

## Fixtures: the vocabulary

[api/tests/conftest.py](../../api/tests/conftest.py) provides the tenancy ladder as composable fixtures — `organisation` (L315), `staff_user`/`staff_client` (L294/L308), `admin_client` (L267), plus per-app conftests deeper in the tree. Requesting `environment` transitively builds project and organisation. Two disciplines to copy anywhere:

- **Builders compose**: a fixture asks for the fixtures it needs; tests ask only for what they assert on.
- **Deny-by-default clients**: `staff_client` is a user with *no* permissions — permission tests grant exactly one and assert everything else fails. Design your tests from the deny side.

Also note [restrict_http_requests](../../api/tests/conftest.py#L233) — outbound HTTP is blocked in tests by monkeypatch, forcing explicit mocks (`post_request_mock`, L169). Steal this: accidental network calls are the #1 flake source.

## What belongs at which layer — the decision rule

Ask: *what's the cheapest layer at which this failure is observable?* Permission matrix → API unit (fast, exhaustive). Serializer shape → API unit. "Toggle updates the row and audit log" → API integration. "User can actually toggle from the UI" → one E2E happy path, not twenty variants. UI conditional rendering → Jest. The pyramid here is enforced economically: E2E gets retries and 20-way concurrency because it's expensive ([frontend/README.md](../../frontend/README.md)); unit layers are expected to be deterministic and fast.

## Time, randomness, isolation

- Time: scheduled-flag logic (`live_from`, `is_live` [models.py#L638-L652](../../api/features/models.py#L638-L652)) — freeze time in tests (the repo uses pytest fixtures/mocking for this; grep `freeze` under api/tests before inventing your own).
- Randomness: multivariate allocation is hash-based, not random — deterministic by design; test with fixed identity keys.
- Isolation: pytest-xdist runs tests in parallel — anything touching shared global state (caches!) needs care; clearing Django caches between tests is a common conftest chore.
- Flake prevention: block network (above), avoid sleeps, let Playwright auto-wait, and quarantine-by-retry only at the E2E layer.

🎤 **Interview angle:** "Describe your testing strategy" — answer with layers *by economic purpose* (cheapest observable failure), one concrete fixture-composition example, and the deny-by-default permission testing idea. That's a mid-to-senior answer; a junior lists tools.

**Drill:** for the change "add `is_archived` filter to the features list endpoint," write the test *names only* (naming template!) at each layer you'd touch, and justify any layer you skip. Basic: 3 sensible names. Solid: correct layer choices + one permission case. Strong: you skipped E2E with a defensible reason (filter logic observable at API layer; UI filter already covered by an existing pattern test).
