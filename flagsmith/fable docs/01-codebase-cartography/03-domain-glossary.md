# Domain Glossary

The product nouns, with where each lives. Interviewers at any feature-flag/SaaS company use this exact vocabulary — multi-tenancy hierarchies and override ladders are industry-standard shapes.

## Tenancy hierarchy (top → bottom)

| Term | Meaning | Code |
| --- | --- | --- |
| **Organisation** | The tenant. Billing, members, org-level webhooks, the `stop_serving_flags` kill switch | [api/organisations/models.py](../../api/organisations/models.py); kill-switch check at [authentication.py#L33-L34](../../api/environments/authentication.py#L33-L34) |
| **Project** | Groups features and segments; owns environments | [api/projects/models.py](../../api/projects/models.py) |
| **Environment** | A deploy target (dev/staging/prod) inside a project. Owns feature *states* and the SDK api_key | [api/environments/models.py#L80](../../api/environments/models.py#L80) |

Invariant to internalize: **a Feature is defined once per project; its on/off/value state exists per environment.** That split is the whole data model.

## Flag domain

| Term | Meaning | Code |
| --- | --- | --- |
| **Feature** | The flag definition (name, type, default). Project-scoped; name unique per project (enforced in raw-SQL migrations — see note at [models.py#L138-L141](../../api/features/models.py#L138-L141)) | [api/features/models.py#L92](../../api/features/models.py#L92) |
| **FeatureState** | *The* central entity: enabled/value for a feature in an environment, optionally narrowed to a segment or an identity | [api/features/models.py#L461](../../api/features/models.py#L461) |
| **FeatureStateValue** | The typed value (string/int/bool) hanging off a FeatureState | [api/features/models.py#L1128](../../api/features/models.py#L1128) |
| **Multivariate (MV) options** | Weighted value variants for A/B tests | [api/features/multivariate/](../../api/features/multivariate/) |
| **Identity** | An end user of *your* app, per environment; identified by `identifier` string | [api/environments/identities/models.py](../../api/environments/identities/models.py) |
| **Trait** | Key/value data on an identity; input to segment rules | [api/environments/identities/traits/](../../api/environments/identities/traits/) |
| **Segment** | A rule tree (`SegmentRule` ALL/ANY/NONE + `Condition` operators) matching identities by traits | [api/segments/models.py#L80,L201,L251](../../api/segments/models.py#L80) |
| **Segment override / FeatureSegment** | "For identities in segment X, this feature behaves differently" — with a **priority** ordering between overlapping segments | [api/features/models.py#L248](../../api/features/models.py#L248) |
| **Identity override** | Per-single-user flag pin; beats everything | resolution in [identities/models.py#L53-L120](../../api/environments/identities/models.py#L53-L120) |

## Change management

| Term | Meaning | Code |
| --- | --- | --- |
| **Versioning v2 / EnvironmentFeatureVersion (EFV)** | Immutable published snapshots of a feature's states per environment; toggles create-and-publish a new version | [api/features/versioning/models.py](../../api/features/versioning/models.py); write path [versioning_service.py#L141-L198](../../api/features/versioning/versioning_service.py#L141-L198) |
| **Change Request** | Approval workflow before a state goes live; can be scheduled (`live_from`) | [api/features/workflows/](../../api/features/workflows/); FK at [models.py#L506](../../api/features/models.py#L506) |
| **Audit log** | Who changed what, derived from django-simple-history records; also the trigger for cache/document rebuilds | [api/audit/models.py](../../api/audit/models.py) |
| **Environment document** | Denormalized JSON of an entire environment for server-side SDK local evaluation and the edge API | [Environment.write_environment_documents](../../api/environments/models.py#L304-L330) |

## Access control

| Term | Meaning | Code |
| --- | --- | --- |
| **FFAdminUser** | Dashboard user (not an Identity!) | [api/users/models.py](../../api/users/models.py) |
| **Environment key vs server key** | `X-Environment-Key` header; server keys prefixed `ser.` unlock server-only flags | [api/environments/api_keys.py](../../api/environments/api_keys.py); origin logic [authentication.py#L37-L41](../../api/environments/authentication.py#L37-L41) |
| **Master API key** | Org-level programmatic admin key | [api/api_keys/](../../api/api_keys/) |
| **VIEW_PROJECT / CREATE_FEATURE / UPDATE_FEATURE_STATE / MANAGE_SEGMENT_OVERRIDES** | Permission constants checked by DRF permission classes | usage map at [features/permissions.py#L28-L39](../../api/features/permissions.py#L28-L39) |

## Confusing near-synonyms — get these straight

- **User vs Identity**: `FFAdminUser` operates the dashboard; `Identity` is your app's end user being targeted. Different tables, different auth.
- **Feature vs FeatureState vs FeatureStateValue**: definition vs per-environment state vs typed value. "Flag" colloquially means all three.
- **FeatureSegment vs Segment**: `Segment` is the reusable rule tree; `FeatureSegment` is the join row applying it to one feature in one environment with a priority.
- **version (int, deprecated) vs EnvironmentFeatureVersion**: legacy per-row counter ([models.py#L522-L523](../../api/features/models.py#L522-L523)) vs the v2 immutable-snapshot model. Code branches on `environment.use_v2_feature_versioning` in both backend and frontend.
- **environment (Django app) vs environment document vs environment key**: entity vs denormalized export vs credential.

**Drill:** close this page and write the entity-relationship chain from Organisation to FeatureStateValue, marking cardinalities and where segment/identity overrides attach. Basic: chain right. Solid: cardinalities + the per-project/per-environment split. Strong: you also placed EFV and ChangeRequest.
