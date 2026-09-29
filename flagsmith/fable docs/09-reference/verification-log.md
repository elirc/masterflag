# Verification Log

What was actually inspected while building this curriculum, on what basis claims are made, and what remains unverified. Date: **2026-07-09**, working tree at `flagsmith` repo (branch `main`, version 2.68.0 per [api/pyproject.toml](../../api/pyproject.toml)). Platform: Windows 11; **no services were run** — Docker/DB/dev servers were not started, so all run/serve/test commands are marked __inferred__ (read from Makefiles, package.json, READMEs, CI workflows) rather than __verified__.

## Files read in full or in targeted sections

| Area | Files (line ranges read) | Notes |
| --- | --- | --- |
| Repo meta | README.md, AGENTS.md, CLAUDE.md, api/README.md, frontend/README.md, frontend/CLAUDE.md, docker-compose.yml (service list), api/Makefile (head), api/pyproject.toml (deps) | House rules, commands, deps |
| SDK auth | api/environments/authentication.py (all 50 lines) | EnvironmentKeyAuthentication verified |
| SDK flags | api/features/views.py L990–1109 | SDKFeatureStates, caching, filters |
| Evaluation | api/features/versioning/versioning_service.py (all 572 lines) | flags list/dict, update_flag v1/v2 |
| Models | api/features/models.py L92–160 (Feature), L461–680 (FeatureState incl. `__gt__`, `type`, `is_live`, `clone`); api/environments/models.py L255–330 (get_from_cache, write_environment_documents) + class/def outline of whole file | |
| Identities | api/environments/identities/views.py L148–235 (SDKIdentities.get); api/environments/identities/models.py L40–120 (get_all_feature_states) | |
| Permissions | api/features/permissions.py L1–155; outline to L221 | ACTION_PERMISSIONS_MAP verified |
| Audit/async | api/audit/models.py L120–168; api/audit/tasks.py L1–60; api/audit/signals.py L1–60; api/environments/tasks.py L1–80; api/webhooks/webhooks.py L1–80; api/webhooks/tasks.py (all) | Task chain verified |
| Segments | api/segments/models.py (class/def outline only) | Rules/conditions not read line-by-line |
| API tests | api/tests/ directory shape; api/tests/conftest.py fixture outline (L139–393) | Fixture bodies not all read |
| Frontend core | frontend/common/service.ts L1–70; frontend/common/services/useFeatureState.ts (all); frontend/common/stores + services directory listings; frontend/common (dir listing) | |
| Frontend features | frontend/web/components/pages/features/FeaturesPage.tsx (all 392 lines); hooks/useToggleFeatureWithToast.ts (all ~82 lines) | Flux bridge verified |
| CI | .github/workflows directory list; api-pull-request.yml + frontend-pull-request.yml (step names) | |
| Counts | `rg --files`: 3091 files total; frontend 1011; api 1701; feature-list-store.ts = 1061 lines | __verified__ via rg/wc |

## Commands run (all __verified__)

- `rg --files` variants, `grep -n`, `sed -n`, `wc -l`, `ls` across the tree — file inventory and targeted reads. All succeeded.
- No build, test, lint, migration, or server command was executed.

## Uncertainties and areas not covered line-by-line

- **flag-engine internals** — `flagsmith-flag-engine` is an external package ([api/pyproject.toml](../../api/pyproject.toml)); segment-evaluation logic claims are conceptual, anchored only to its call sites.
- **task_processor internals** — lives in the external `flagsmith-common` package; claims about at-least-once semantics/priorities are based on call sites (`register_task_handler`, `TaskPriority`, `delay_until`) and docs, labelled accordingly.
- **DynamoDB / Edge API paths** — read call sites only ([api/environments/tasks.py](../../api/environments/tasks.py), `forward_identity_request` in identities views); not traced end to end.
- **Private packages** — RBAC and SAML are cloned from private-ish repos at install time ([api/Makefile#L44-L49](../../api/Makefile#L44-L49)); permission claims are for the OSS core only.
- **Legacy Flux stores** — [frontend/common/stores/feature-list-store.ts](../../frontend/common/stores/feature-list-store.ts) (1061 lines) was sized and sampled, not fully read.
- **Enterprise/SaaS-only surfaces** (sales_dashboard, platform_hub, billing) — deliberately out of curriculum scope.
- Anything labelled "investigate" or "possible risk" in the curriculum is a hypothesis, not a confirmed bug.

## Anchor drift warning

All `#L<n>` anchors were confirmed against the working tree on 2026-07-09 by re-opening the file immediately before writing. Upstream Flagsmith moves fast; if an anchor misses, search the file for the named symbol.
