# Fake-Code Contrasts

Ten pairs. The **bad** version is the instinct; the **better** version is the shape this repo actually uses (anchor given). Every snippet here is fake and labelled; only the anchors are real. For each: name the failure mode out loud before reading the "why."

## 1. Missing permission scope on a detail fetch (IDOR)

```python
# Illustrative fake code: not from this repo
def get_feature_state(request, uuid):
    fs = FeatureState.objects.get(uuid=uuid)   # anyone's row!
    return Response(serialize(fs))
```

```python
# Illustrative fake code: not from this repo (mirrors the real pattern)
qs = FeatureState.objects.filter(feature__project__in=request.user.get_permitted_projects(VIEW_PROJECT))
fs = get_object_or_404(qs, uuid=uuid)          # unauthorized == 404
```

Real: [features/views.py#L992-L1001](../../api/features/views.py#L992-L1001). Why: scope the queryset *before* fetching; can't forget the check afterwards, and existence doesn't leak.

## 2. Serial awaits where work is independent

```ts
// Illustrative fake code: not from this repo
for (const row of data.results) {
  row.feature_segment = await fetchSegment(row.feature_segment) // N sequential round-trips
}
```

```ts
// Illustrative fake code: not from this repo (mirrors the real pattern)
const results = await Promise.all(data.results.map(enrichRow))
```

Real: [useFeatureState.ts#L42-L44](../../frontend/common/services/useFeatureState.ts#L42-L44). Why: latency = max, not sum. And the *senior* follow-up: the better fix is no fan-out at all — embed server-side (risk-register #1).

## 3. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
type FeatureState = { feature_segment: number | null }  // wire id leaks everywhere
// components then fetch the segment themselves, each differently
```

```ts
// Illustrative fake code: not from this repo (mirrors the real pattern)
type Wire = Omit<FeatureState, 'feature_segment'> & { feature_segment: number | null }
// transform ONCE at the service boundary; app code sees the enriched object
```

Real: [useFeatureState.ts#L34-L38](../../frontend/common/services/useFeatureState.ts#L34-L38). Why: one boundary owns the translation; the type names the difference honestly.

## 4. Mutation without cache invalidation

```ts
// Illustrative fake code: not from this repo
updateFeatureState: builder.mutation({ query: putConfig })   // UI now silently stale
```

```ts
// Illustrative fake code: not from this repo (mirrors the real pattern)
updateFeatureState: builder.mutation({
  query: putConfig,
  invalidatesTags: [{ id: 'LIST', type: 'FeatureList' }, { id: 'LIST', type: 'FeatureState' }],
})
```

Real: [useFeatureState.ts#L57-L61](../../frontend/common/services/useFeatureState.ts#L57-L61). Why: staleness is a *silent* failure; declare the dependency where the mutation lives.

## 5. Swallowed mutation errors (`.unwrap()` missing)

```ts
// Illustrative fake code: not from this repo
try {
  await updateFeatureState(body)      // RTKQ mutations resolve even on error!
  toast('Saved')                      // lies on failure
} catch (e) { toast('Failed') }       // dead code
```

```ts
// Illustrative fake code: not from this repo (mirrors the real pattern)
await updateFeatureState(body).unwrap()   // now the catch is reachable
```

Real: [useToggleFeatureWithToast.ts#L48,L58](../../frontend/web/components/pages/features/hooks/useToggleFeatureWithToast.ts#L48). Why: know your library's error contract; a success toast on failure is worse than no handling.

## 6. Side effect on the request path

```python
# Illustrative fake code: not from this repo
def update_flag_view(request):
    fs = save_feature_state(request)
    requests.post(webhook_url, json=payload, timeout=30)   # user waits on a third party
    rebuild_environment_document(fs.environment_id)        # and on this
    return Response(...)
```

```python
# Illustrative fake code: not from this repo (mirrors the real pattern)
def update_flag_view(request):
    fs = save_feature_state(request)     # history/audit written with the data
    return Response(...)                 # AuditLog hook enqueues the rest
```

Real: [audit/models.py#L141-L168](../../api/audit/models.py#L141-L168) + [environments/tasks.py#L31-L44](../../api/environments/tasks.py#L31-L44). Why: request latency shouldn't scale with integration count; failures shouldn't 500 the user's save.

## 7. N+1 in the ORM

```python
# Illustrative fake code: not from this repo
for fs in FeatureState.objects.filter(environment=env):
    print(fs.feature.name, fs.feature_state_value.value)   # 2 queries per row
```

```python
# Illustrative fake code: not from this repo (mirrors the real pattern)
qs = FeatureState.objects.filter(environment=env).select_related("feature", "feature_state_value")
```

Real: [versioning_service.py#L542-L556](../../api/features/versioning/versioning_service.py#L542-L556). Why: know your relations before iterating; the repo's evaluation path select_relates *five* relations for exactly this reason.

## 8. Stale-closure subscription

```tsx
// Illustrative fake code: not from this repo
useEffect(() => {
  Store.on('saved', () => refetchRef)   // registered once; reads first-render refetch
}, [])                                  // never re-registered, never unsubscribed
```

```tsx
// Illustrative fake code: not from this repo (mirrors the real pattern)
useEffect(() => {
  const handler = () => refetch()
  Store.on('saved', handler)
  return () => Store.off('saved', handler)   // cleanup unsubscribes THIS closure
}, [refetch])
```

Real: [FeaturesPage.tsx#L123-L134](../../frontend/web/components/pages/features/FeaturesPage.tsx#L123-L134). Why: subscriptions capture closures; deps + cleanup keep the captured world current and leak-free.

## 9. Trusting the client's permission check

```tsx
// Illustrative fake code: not from this repo
{permission && <DeleteButton onClick={api.deleteFeature} />}
// backend: no permission class — "the button was hidden, so it's fine"
```

```python
# Illustrative fake code: not from this repo (mirrors the real pattern)
class FeaturePermissions(IsAuthenticated):
    def has_object_permission(self, request, view, obj):
        return request.user.has_project_permission(DELETE_FEATURE, obj.project)
```

Real: UI gate [FeaturesPage.tsx#L255-L263](../../frontend/web/components/pages/features/FeaturesPage.tsx#L255-L263) **and** server truth [features/permissions.py#L70-L85](../../api/features/permissions.py#L70-L85). Why: the client is an untrusted renderer of *hints*.

## 10. Casually changing a public contract

```python
# Illustrative fake code: not from this repo
# "cleanup": return 400 for missing identifier, rename 'flags' → 'feature_states'
return Response({"error": "identifier required"}, status=400)
```

```python
# Illustrative fake code: not from this repo (mirrors the real pattern)
# TODO: add 400 status - will this break the clients?
return Response({"detail": "Missing identifier"})   # frozen until a versioning plan exists
```

Real: [identities/views.py#L170-L173](../../api/environments/identities/views.py#L170-L173). Why: a deployed SDK fleet makes response shape a **contract**; "fixing" it is a breaking change needing versioning, deprecation windows, and comms — not a drive-by. The bad version is *better code* and *worse engineering*.

---

**Rubric per pair.** Basic: named the failure mode. Solid: pointed at the real anchor and explained the mechanism (why it fails, not just that it does). Strong: for pairs 2, 6, 10 you also articulated when the "bad" version is actually acceptable (tiny N; a truly synchronous requirement; a pre-1.0 API) — pattern judgment includes knowing the exceptions.
