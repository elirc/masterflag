# Writing Tests Here — Recipes

Seven recipes using this repo's actual helpers. Study the real style first: [test_unit_features_permissions.py#L25-L42](../../api/tests/unit/features/test_unit_features_permissions.py#L25-L42) (Given/When/Then, fixtures-as-parameters, naming template) and the table-driven Jest style in [format.test.ts](../../frontend/common/utils/__tests__/format.test.ts).

Commands (from [command-cheatsheet](../09-reference/command-cheatsheet.md)): API — `make test opts='-k <expr> -n0'`; FE — `npm run test:unit -- --testPathPatterns=<name>`; E2E — `E2E_RETRIES=0 SKIP_BUNDLE=1 npm run test -- tests/<file>.pw.ts`.

## Recipe 1 — API happy path (integration style)

```python
# Illustrative fake code: not from this repo — follows house conventions
def test_update_feature_state__valid_payload__returns_200_and_updates(
    admin_client: APIClient, environment: Environment, feature_state: FeatureState,
) -> None:
    # Given
    url = reverse("api-v1:environments:environment-featurestates-detail",
                  args=[environment.api_key, feature_state.id])
    # When
    response = admin_client.patch(url, data={"enabled": True}, format="json")
    # Then
    assert response.status_code == 200
    feature_state.refresh_from_db()
    assert feature_state.enabled is True
```

Notes: `admin_client` from [conftest.py#L267](../../api/tests/conftest.py#L267); assert the *database*, not just the response — the response can lie.

## Recipe 2 — validation failure

Same shape, payload `{"enabled": "banana"}` → assert 400 **and** the error body names the field. Name: `test_update_feature_state__invalid_enabled_type__returns_400`. The test protects the *error contract*, which clients parse.

## Recipe 3 — permission failure (the one juniors skip)

```python
# Illustrative fake code: not from this repo — follows house conventions
def test_update_feature_state__user_without_permission__returns_403(
    staff_client: APIClient, environment: Environment, feature_state: FeatureState,
) -> None:
```

`staff_client` ([conftest.py#L308](../../api/tests/conftest.py#L308)) has **no** permissions — deny-by-default. Grant nothing; expect 403/404. Then a sibling test grants exactly `UPDATE_FEATURE_STATE` (see `UserProjectPermission.objects.create` usage in [test_unit_features_permissions.py#L57](../../api/tests/unit/features/test_unit_features_permissions.py#L57)) and expects success — the permission matrix, two tests at a time.

## Recipe 4 — cross-tenant rejection

Build a *second* organisation/project/environment inline (fixtures give you the first), then hit org-A's client against org-B's resource ids; assert 404 (not 403 — existence must not leak, [key flow 5](../01-codebase-cartography/05-key-flows.md)). Name: `test_get_feature_state_by_uuid__different_organisation__returns_404`.

## Recipe 5 — async side effect

Call the *undecorated* function or trigger the enqueue and assert on the mock: outbound HTTP is blocked by [restrict_http_requests](../../api/tests/conftest.py#L233), so webhook tests assert against `post_request_mock` ([conftest.py#L169](../../api/tests/conftest.py#L169)) — e.g. "toggling a flag calls the environment webhook with a signed payload". Real examples: [tests/integration/features/featurestate/test_webhooks.py](../../api/tests/integration/features/featurestate/test_webhooks.py). Also verify structured logs with the `log` fixture (pytest-structlog) per [api/README.md](../../api/README.md).

## Recipe 6 — Jest: pure util (table-driven)

Copy the `it.each` table style from [format.test.ts#L5-L17](../../frontend/common/utils/__tests__/format.test.ts#L5-L17). Rule: one behaviour per `describe`, edge cases as table rows, no snapshots for logic.

## Recipe 7 — E2E journey (Playwright)

Only for journeys crossing FE+API. Follow an existing spec ([flag-tests.pw.ts](../../frontend/e2e/tests/flag-tests.pw.ts)) — the suite bootstraps auth via `E2E_TEST_AUTH_TOKEN` ([frontend/README.md](../../frontend/README.md)). House move: tag `@oss` vs `@enterprise` so the right CI lane picks it up. Keep assertions on user-visible outcomes (row shows "On"), never on network internals.

## Anti-recipes (what not to write here)

- A unit test that re-tests DRF itself ("serializer serializes").
- An E2E for every permission combination (that's Recipe 3's job, ×N cheaply).
- Mock-everything unit tests asserting call counts on your own internals — they pin implementation, not behaviour, and 100% diff coverage tempts exactly this. Coverage is the *floor*, not the goal.

**Drill:** implement Recipes 3+4 for `get_feature_state_by_uuid` ([views.py#L992-L1001](../../api/features/views.py#L992-L1001)) for real, run them, and delete them (or keep — they're a genuine contribution; see [06/01](../06-contribution-practice/01-good-first-tickets.md)). Basic: they pass. Solid: correct naming + Given/When/Then + typed params (mypy runs on tests!). Strong: your cross-tenant test failed first for an interesting reason and you can explain it.
