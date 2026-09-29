# Security Checklist

Mapped to real code, ending with the pre-merge pass. Vocabulary: **authorization** (may you?), **IDOR** (acting on someone else's object id), **SSRF** (server fetching attacker-chosen URLs), **XSS** (attacker script in victim's browser), **CSRF** (victim's browser making authenticated requests it didn't intend).

| Concern | Where this repo handles it | What to check when touching |
| --- | --- | --- |
| Authorization | Permission classes per app ([features/permissions.py](../../api/features/permissions.py)); hierarchy membership; tag scoping | Every new action mapped ([ACTION_PERMISSIONS_MAP](../../api/features/permissions.py#L28-L39)); object *and* list paths |
| IDOR / tenant isolation | Queryset scoping ([views.py#L992-L1001](../../api/features/views.py#L992-L1001)); 404-for-unowned | Any `objects.get(...)` on request-supplied ids without a scope filter |
| Input validation | DRF serializers; value length caps (`FEATURE_VALUE_LIMIT`, [models.py#L112-L114](../../api/features/models.py#L112-L114)); `isdigit` guards ([permissions.py#L102](../../api/features/permissions.py#L102)) | Raw `request.data.get` reaching queries; regex via `google-re2` (ReDoS-resistant — [pyproject.toml](../../api/pyproject.toml)) for user-supplied patterns |
| SSRF | Webhook URLs are user-configured outbound calls ([webhooks/webhooks.py](../../api/webhooks/webhooks.py)) — inherently SSRF-adjacent | New "fetch this URL" features: scheme/host validation, no internal-network access, timeouts. **Investigate** before assuming existing validation |
| XSS | React escapes by default; `dompurify` present for markdown/HTML paths ([package.json](../../frontend/package.json)) | Any `dangerouslySetInnerHTML`; markdown renderers ([react-markdown](../../frontend/package.json)); flag *values* are user content rendered in dashboards |
| CSRF | Token-in-header auth is CSRF-resistant; cookie mode exists (`cookieAuthEnabled`, [service.ts#L22](../../frontend/common/service.ts#L22)) → Django CSRF applies | If enabling cookie auth: SameSite, CSRF middleware on state-changing endpoints |
| Injection | ORM parameterization everywhere; raw SQL rare (uniqueness migrations) | Any `.raw()`/`extra()`/f-string SQL in new code |
| Secrets | env vars; signing key for webhooks (`sign_payload`, [webhooks.py#L17-L18](../../api/webhooks/webhooks.py#L17-L18)); no secrets in repo | Logs/structlog events leaking keys or PII (house rule: IDs not emails — [api/README.md](../../api/README.md)) |
| Dependency risk | Trivy scans in CI ([platform-docker-trivy-scan.yml](../../.github/workflows/platform-docker-trivy-scan.yml)); lockfiles; private-index discipline | Never edit `uv.lock`; new FE deps need justification (bundle + supply chain) |
| Webhooks (outbound integrity) | HMAC signature header (`FLAGSMITH_SIGNATURE_HEADER`) so receivers can verify origin | New event types must sign; docs for receivers to verify |
| Rate limiting / brute force | django-axes on auth ([pyproject.toml](../../api/pyproject.toml)); throttling scopes on some views; SDK endpoints deliberately unthrottled (`throttle_classes = []`, [views.py#L1011](../../api/features/views.py#L1011)) + negative key cache ([models.py#L273](../../api/environments/models.py#L273)) | New public endpoints: which throttle scope? Onboarding has its own ([onboarding/throttling.py](../../api/onboarding/throttling.py)) |
| Cookies / sessions | Token default; cookie mode `credentials: 'include'` ([service.ts#L21-L22](../../frontend/common/service.ts#L21-L22)) | Cookie flags (HttpOnly/Secure/SameSite) if touching auth |
| Uploads / imports | Import/export subsystems ([features/import_export/](../../api/features/import_export/), LaunchDarkly importer) parse user files | Size limits, content-type checks, no eval of imported values |
| Cache poisoning | Response cache varies on env key + request origin in the key ([views.py#L1093-L1094](../../api/features/views.py#L1093-L1094)) | Any new `cache_page`: what's in the key? What headers vary? |

## The pre-merge security pass (run on every PR you write here)

1. **Who can call this?** Name the authn scheme and permission constant. If you can't, stop.
2. **Whose data can it reach?** Every id/uuid from the request must pass through a scoped queryset. Grep your diff for `objects.get`.
3. **What does it do with strings?** Length caps, type validation, no reflected HTML, no f-string SQL.
4. **Does it call out?** Then: timeout, retry policy, SSRF surface, signed payload, kill switch.
5. **What appears in logs/metrics?** IDs yes; emails/keys/values no.
6. **What's cached, keyed on what?** Could two principals share a key?
7. **404 or 403?** Unowned = 404, consistently.
8. **Tests:** at minimum Recipe 3 (permission denied) + Recipe 4 (cross-tenant) from [02-writing-tests-here.md](02-writing-tests-here.md).

🎤 **Interview angle:** security rounds for mid-level fullstack are almost always IDOR + XSS + secrets hygiene. Your differentiator: the cache-key poisoning point and "404 not 403," both with anchors you've actually read.

**Drill:** run the 8-step pass against review kata 1 (the copy-flag PR) and kata 7 (Redis cache) in [04-code-reading-gym/04](../04-code-reading-gym/04-review-katas.md). Basic: caught the IDOR. Solid: all 8 steps produce a verdict. Strong: you found the step-6 violation in kata 7 without prompting (identity-id cache key shared across trait states).
