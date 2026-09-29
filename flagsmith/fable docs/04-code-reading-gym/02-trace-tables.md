# Trace Tables

Fill in every cell yourself before comparing to the key flows. Columns: Step / File:line / Value shape / Owner / Transformation / Risk. Partial keys are provided for self-checking; the learning is in the *value shape* column — force yourself to write concrete shapes (`"ser.8x…"`, `Q(identity=None)`, `{ id: 42, enabled: true }`).

## Trace 1 (UI → API): user toggles "dark_mode" in a **v2-versioned** environment

Start: click in [FeatureRow](../../frontend/web/components/feature-summary/FeatureRow.tsx) → [FeaturesPage.tsx#L150-L165](../../frontend/web/components/pages/features/FeaturesPage.tsx#L150-L165) → [useToggleFeatureWithToast.ts#L37-L50](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L37-L50) → `createAndSetFeatureVersion` ([useFeatureVersion.ts](../../frontend/common/services/useFeatureVersion.ts)) → HTTP → DRF view → [versioning_service update path](../../api/features/versioning/versioning_service.py#L141-L198) → publish → response → tag invalidation refetch.

Minimum rows: 9. Self-check: your table must contain (a) the exact mutation payload shape (find it in the service file — includes `featureStates` array), (b) the permission class that fires, (c) the row(s) inserted (`EnvironmentFeatureVersion` + cloned `FeatureState`s), (d) which RTK tags invalidate.

## Trace 2 (persistence): what rows exist after that toggle?

Columns become: Table / Row created-or-updated / Key fields / Written by (file:line). Cover: `feature_versioning_environmentfeatureversion`, `features_featurestate` (draft copies), `features_featurestatevalue`, `*_historical*` rows, `audit_auditlog`, task-queue row(s), `environments_environment.updated_at`. Anchors to verify against: [versioning_service.py#L146-L196](../../api/features/versioning/versioning_service.py#L146-L196), [audit/models.py#L158-L168](../../api/audit/models.py#L158-L168).

Risk column prompt: which of these writes are in the same transaction? (You may have to say "unknown — verify" for the task enqueue; that's the honest cell.)

## Trace 3 (auth): SDK request with a **server** key for a server-only flag

Start: `curl -H "X-Environment-Key: ser.abc…" /api/v1/flags/`. Rows: header extraction ([authentication.py#L25](../../api/environments/authentication.py#L25)) → prefix check → cache/DB resolve ([models.py#L268-L302](../../api/environments/models.py#L268-L302)) → `originated_from = SERVER` ([authentication.py#L37-L41](../../api/environments/authentication.py#L37-L41)) → `_additional_filters` outcome ([views.py#L1075-L1085](../../api/features/views.py#L1075-L1085)) → cache key ([views.py#L1094](../../api/features/views.py#L1094)) → response.

Then rerun the trace with a **client** key and highlight exactly the two cells that change (filter + cache key). That diff *is* the security property.

## Trace 4 (error path): identity request for an environment whose org has `stop_serving_flags=True`

Rows: request → cache hit returns environment → kill-switch check ([authentication.py#L33-L34](../../api/environments/authentication.py#L33-L34)) → `AuthenticationFailed` → DRF exception handler → 401 JSON. Risk cells: what if the environment was cached *before* the org flipped the switch? Where does the staleness end? (TTL — [models.py#L296-L300](../../api/environments/models.py#L296-L300).) This trace teaches that **an error path has a data flow too**.

## Trace 5 (async): from AuditLog row to an SSE message reaching a dashboard

Rows: AuditLog `AFTER_CREATE` hook fires-or-not (condition!) → per-environment `updated_at` bump → `process_environment_update.delay` → worker picks task (priority HIGHEST) → `write_environment_documents` ([environments/models.py#L304-L330](../../api/environments/models.py#L304-L330)) → `send_environment_update_message_for_environment` ([environments/tasks.py#L40-L44](../../api/environments/tasks.py#L40-L44)) → SSE consumer. Owner column alternates request-process / worker-process — mark the process boundary with a heavy line; everything after it can lag or fail independently.

---

**Rubric.** Basic: rows in the right order with real file:lines. Solid: value shapes are concrete (could be pasted into a test), owner column correct at every process/package boundary. Strong: every Risk cell is either a real named risk or "none — because X"; traces 3 and 4 each end with the one-sentence security/consistency property they demonstrate.
