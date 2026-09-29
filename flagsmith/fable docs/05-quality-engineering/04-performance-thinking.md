# Performance Thinking

Rule zero: **measure first**. Every fake-optimization kata in this curriculum dies on "where's the benchmark?" ([review kata 5](../04-code-reading-gym/04-review-katas.md)). This file maps the performance domains of *this* system and where each likely hotspot lives.

## The domains here

| Domain | Where it bites | How to measure |
| --- | --- | --- |
| DB queries (N+1, missing index, unbounded) | evaluation paths, admin list endpoints | Django query logging, `django-debug-toolbar`-style counting in tests (`django_assert_num_queries` pytest fixture), `EXPLAIN` |
| Server CPU | per-feature resolution loops, serialization of big lists | py-spy in staging; Prometheus histograms ([api/README.md](../../api/README.md) metrics) |
| Network round-trips (client) | RTK Query fan-outs, page-load waterfalls | DevTools Network; count requests per user action |
| Render cost | features list, virtualized rows | React DevTools Profiler |
| Bundle size | rspack outputs, heavyweight deps (moment, jquery, lodash are all present — [package.json](../../frontend/package.json)) | `rspack build` stats / bundle analyzer |
| Cache effectiveness | env cache, flags cache, HTTP cache | hit-rate metrics; header timestamps |
| Memory | fetch-all evaluation on giant environments | worker/process RSS; row counts |

## Likely hotspots, with anchors and the check for each

1. **Client-side N+1** — [useFeatureState.ts#L42-L44](../../frontend/common/services/useFeatureState.ts#L42-L44). Check: page with 50 segment-override rows → 50 extra GETs? Fix direction: server-side embed (critique improvement #2).
2. **Evaluation row volume** — [get_environment_flags_dict](../../api/features/versioning/versioning_service.py#L95-L114) loads *all* candidate states per request when caches are cold. Check: `django_assert_num_queries` on the flags endpoint (should be ~constant thanks to `select_related` at [L542-L556](../../api/features/versioning/versioning_service.py#L542-L556)); row count scales with features × overrides.
3. **Serial async on the identity path** — segment evaluation then flags query ([identities/models.py#L71-L110](../../api/environments/identities/models.py#L71-L110)); plus per-request `get_or_create` writes. Check: p95 on `/api/v1/identities/` vs `/flags/`.
4. **Expensive renders in the features list** — mitigated by virtualization + `useCallback` rows ([FeaturesPage.tsx#L248-L291](../../frontend/web/components/pages/features/FeaturesPage.tsx#L248-L291)); regression risk if someone passes a fresh object/array prop into every row. Check: Profiler "why did this render" on one row while typing in search.
5. **cloneDeep on every data arrival** — the Flux bridge deep-copies the whole page of features per fetch ([FeaturesPage.tsx#L112-L116](../../frontend/web/components/pages/features/FeaturesPage.tsx#L112-L116)). Cheap at 20 rows; measure before/after page-size changes. Dies with the bridge.
6. **Environment-document rebuilds** — full recompute per change ([write_environment_documents](../../api/environments/models.py#L304-L330) with heavy prefetches). Fine per-change; a bulk import triggering hundreds is the risk case. Check: task-queue depth during imports.
7. **Missing-index candidates** — any new filter you add (e.g. `is_archived` on big tables). Check: `EXPLAIN` before merging; migrations for indexes are cheap *now*, expensive later.

## How to find each classic, in this repo

- **N+1 (server):** run a list endpoint under `django_assert_num_queries(N)`; if N grows with result count, bisect with `select_related`/`prefetch_related` (worked example: the five relations at [versioning_service.py#L547-L553](../../api/features/versioning/versioning_service.py#L547-L553)).
- **N+1 (client):** Network tab filtered to XHR, one user action at a time.
- **Serial async:** await-in-loop grep + timeline gaps in DevTools.
- **Oversized bundle:** analyzer; usual suspects here: moment (locales!), lodash full import, jquery.
- **Unbounded queries:** endpoints without pagination — grep `pagination_class = None` (deliberate on SDK endpoints — bounded by domain size; verify that assumption holds for admin endpoints).

🎤 **Interview angle:** "The features page is slow — what do you do?" The winning shape: *reproduce with a number* (Profiler/Network) → *localize the domain* (render vs network vs server vs DB) → *cheapest fix at the right layer* → *regression guard* (query-count assertion, bundle-size budget). Never lead with `useMemo`.

**Drill:** cold-cache, how many HTTP requests and DB queries does one features-page load cost? Get real numbers (Network tab; API query logging). Basic: counted requests. Solid: separated RTK queries from legacy-store calls. Strong: identified the single biggest lever and the measurement that would prove it shipped.
