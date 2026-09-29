# Data Model and Persistence

## The entity spine

```
Organisation ─1:N─ Project ─1:N─ Environment
                     │                 │
                     ├─1:N─ Feature    ├─1:N─ FeatureState ──1:1── FeatureStateValue
                     │        │        │        │  │
                     │        └────────┼────────┘  ├─ FK identity   (nullable)  ← identity override
                     │                 │           ├─ FK feature_segment (nullable) ← segment override
                     └─1:N─ Segment ───┼───────────┘        │
                              (rules/conditions tree)   FeatureSegment (feature × segment × environment, ordered)
Environment ─1:N─ Identity ─1:N─ Trait
Environment×Feature ─1:N─ EnvironmentFeatureVersion (v2: immutable published snapshots of FeatureStates)
```

Anchors: Feature [models.py#L92](../../api/features/models.py#L92); FeatureSegment [#L248](../../api/features/models.py#L248); FeatureState [#L461](../../api/features/models.py#L461) (nullable FKs at L477–L498); FeatureStateValue [#L1128](../../api/features/models.py#L1128); Segment tree [segments/models.py#L80,L201,L251](../../api/segments/models.py#L80).

The design trick to internalize: **FeatureState is one table playing three roles** (environment default / segment override / identity override), discriminated by which nullable FK is set — the `type` property makes it explicit ([models.py#L623-L636](../../api/features/models.py#L623-L636)) and logs an error on the impossible both-set state. Alternative design: three tables. Tradeoff: one table = single evaluation query + uniform versioning/audit; cost = nullable-FK discipline and a comparison operator that must understand all three roles.

## Consistency expectations and transactions

- **Invariant** (a condition the system promises to keep true): at most one *live* FeatureState per (feature, environment, segment?, identity?) key. It's enforced *procedurally* — uniqueness SQL in old migrations ([Meta note](../../api/features/models.py#L138-L141)), plus the "latest live version wins" resolution ([versioning_service.py#L104-L114](../../api/features/versioning/versioning_service.py#L104-L114)) — not by one declarative constraint. Know the difference when you write.
- Transactions are used surgically, not everywhere: `@transaction.atomic` on multi-row writes like segment cloning ([segments/models.py#L151-L183](../../api/segments/models.py#L151)), segment-override version creation ([versioning/serializers.py#L167](../../api/features/versioning/serializers.py#L167)), feature-segment reorder ([feature_segments/serializers.py#L53](../../api/features/feature_segments/serializers.py#L53)). Default Django autocommit covers single-row writes.
- **Soft deletes** (`SoftDeleteExportableModel` on Feature/FeatureState/Segment) — rows linger with `deleted_at`; every raw query must remember to exclude them (managers do it for you; raw SQL won't).
- **History tables**: django-simple-history writes a parallel `Historical*` row per change — that's the audit/event spine's raw material and a real storage/write-amplification cost.
- Cascades: `Feature.project` is `on_delete=DO_NOTHING` with cascades handled *outside* the ORM ([models.py#L108-L110](../../api/features/models.py#L108-L110), PR #3360) — deletion of big graphs is done asynchronously/manually to avoid multi-minute transactions. Senior fact: cascade choice is a latency and locking decision, not a modelling nicety.

## Versioning v2: immutability as a schema strategy

v2 never mutates live rows: a toggle creates a new `EnvironmentFeatureVersion`, copies states into it as drafts, mutates the draft, then `publish()`es ([versioning_service.py#L141-L198](../../api/features/versioning/versioning_service.py#L141-L198)). Reads pick the latest published version per feature ([get_current_live_environment_feature_version](../../api/features/versioning/versioning_service.py#L117-L129)). Benefits: instant rollback (re-point), clean audit, scheduled go-live. Costs: row multiplication and the *dual-write-path era* while v1 still exists — every flag-writing feature is implemented twice ([update_flag v1 vs v2](../../api/features/versioning/versioning_service.py#L132-L245)).

## How to safely change the schema here

1. Model change → `make docker-up django-make-migrations` (hand-named; squash when possible — [api/README.md](../../api/README.md)).
2. CI blocks PRs with missing migrations ([api-pull-request.yml](../../.github/workflows/api-pull-request.yml)).
3. Additive first (nullable column / new table), backfill via task, then constrain — because compose runs `migrate-db` *before* new code serves traffic ([docker-compose.yml#L55](../../docker-compose.yml#L55)), and old code must survive the new schema during the deploy window.
4. Migrations that touch big tables (FeatureState, Historical*) need lock-awareness; check existing migrations for the house style (74 in features alone).
5. Rollback story: Django migrations are reversible only if you write them so; for data migrations, write the reverse or document irreversibility.

🎤 **Interview angle:** "Design the schema for feature flags with per-user and per-segment overrides" is a *literal* interview question — reproduce the FeatureState discriminated-role table and defend it against the three-table alternative. "How do you deploy schema changes with zero downtime?" — the additive/backfill/constrain sequence above.

**Drill:** write the SQL (or Django queryset) that would detect a violated invariant: two live default FeatureStates for the same feature+environment in a v1 environment. Basic: query sketch. Solid: correct handling of `live_from`/version semantics. Strong: proposed the declarative constraint that would prevent it and why it can't be added trivially (historic rows, v2 drafts).
