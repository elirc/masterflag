# Validation, Auth, and Permissions

Two words juniors blur and interviewers always separate: **authentication** (who are you?) vs **authorization** (what may you do?). This repo has two complete authn schemes and a layered authz system.

## Authentication surfaces

| Scheme | For | Code | Notes |
| --- | --- | --- | --- |
| Token / cookie (djoser + DRF) | Dashboard users | [api/custom_auth/](../../api/custom_auth/); FE attaches token at [service.ts#L23-L40](../../frontend/common/service.ts#L23-L40) | django-axes rate-limits login attempts ([pyproject.toml](../../api/pyproject.toml)) |
| Environment key header | SDKs | [environments/authentication.py#L14-L49](../../api/environments/authentication.py#L14-L49) | Returns `(None, None)` — **no user**; the request carries an `environment` instead. Key prefix `ser.` ⇒ SERVER origin |
| Environment API keys (FK) | Rotatable server keys | [EnvironmentAPIKey](../../api/environments/models.py#L701-L731) with `is_valid` (expiry/active) | Union lookup in [get_from_cache](../../api/environments/models.py#L286-L290) |
| Master API key | Org-level automation | [api/api_keys/](../../api/api_keys/) | Admin-scope credential; treat as break-glass |

## The validation ladder (request → write)

1. **Transport/DRF parsing** — content type, field presence.
2. **Serializer validation** — shapes and field rules per endpoint ([features/serializers.py](../../api/features/serializers.py)); query-param serializers exist too ([SDKFeatureStatesQuerySerializer](../../api/features/serializers.py#L788)).
3. **Permission classes** — the authz gate (below).
4. **Domain guards** — invariants enforced in services/models regardless of caller: e.g. direct feature-state writes are *rejected* in v2 environments ([require_direct_state_write](../../api/features/versioning/versioning_service.py#L20-L37)); value length capped by `FEATURE_VALUE_LIMIT` ([models.py#L112-L114](../../api/features/models.py#L112-L114)).
5. **Database constraints** — the last line (uniqueness SQL, FKs).

A mid-level answer knows *each rule's home*: shape rules in serializers, tenancy in permissions, invariants in domain, integrity in DB. A rule in the wrong layer (shape check in a view, tenancy in a serializer) is a review finding.

## Authorization: the layered model

Permission constants (`VIEW_PROJECT`, `CREATE_FEATURE`, `UPDATE_FEATURE_STATE`, `MANAGE_SEGMENT_OVERRIDES`, …) come from the shared `flagsmith-common` package and are evaluated at org/project/environment levels, with optional **tag scoping** (permissions restricted to features carrying certain tags):

- Action→permission map + project-level checks: [features/permissions.py#L28-L85](../../api/features/permissions.py#L28-L85)
- Feature-state checks incl. the segment-override split: [#L88-L151](../../api/features/permissions.py#L88-L151) — updating a *segment override* requires a different permission than updating the default state; object-level check picks by `feature_segment_id` (L139–141)
- Queryset scoping against IDOR: [get_feature_state_by_uuid](../../api/features/views.py#L992-L1001) filters by `request.user.get_permitted_projects(VIEW_PROJECT)` **before** `get_object_or_404` — unauthorized existence is indistinguishable from absence (404, not 403)

**IDOR** (Insecure Direct Object Reference), defined: attacker supplies someone else's id/uuid and the endpoint honours it because authorization checked only "logged in", not "owns this object". The defense pattern — *scope the queryset, then fetch* — appears above; every new detail endpoint must copy it.

**Tenant isolation** here is hierarchical membership: user → organisation → project → environment. There's no single `tenant_id` column; isolation is only as strong as each endpoint's permission class + queryset scoping. That's why the review checklist ([05-quality-engineering/05](../05-quality-engineering/05-security-checklist.md)) makes you grep both on every new endpoint.

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| UI `<Permission>` ([Permission.tsx](../../frontend/common/providers/Permission.tsx)) is UX, not security | That FE permission names and BE constants stay in sync (they're duplicated strings) |
| List endpoints returning only "mine" isn't automatic | `list` actions: where exactly is the queryset filtered? ([FeaturePermissions](../../api/features/permissions.py#L51-L53) defers list to the view — go read the view) |
| 403 vs 404 choice | Existence leaks; consistent 404-for-unowned |
| Create endpoints: body says `project: X` | Whether the permission check reads the *same* project the write will use ([permissions.py#L56](../../api/features/permissions.py#L56) reads kwargs *or* body — think about mismatch) |
| SDK endpoints have "no auth" | The environment key **is** the credential; what it must never unlock (admin data; server-only flags for client keys, [views.py#L1082-L1083](../../api/features/views.py#L1082-L1083)) |

🎤 **Interview angle:** "How do you authorize a multi-tenant API?" — hierarchy membership + per-action permission map + queryset scoping + 404-for-unowned, with the tag-scoping wrinkle as your depth signal. Cards in [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).

**Drill:** for `PUT /api/v1/environments/{key}/featurestates/{id}/`, write down every gate from the ladder that fires, in order, with file:line. Basic: 3 gates. Solid: all 5 layers placed. Strong: identified which gate would catch (a) oversized value, (b) cross-org id, (c) v2-environment direct write.
