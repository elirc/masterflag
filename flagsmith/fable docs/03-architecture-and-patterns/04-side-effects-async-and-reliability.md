# Side Effects, Async, and Reliability

A **side effect** is anything a request changes beyond its own response: emails, webhooks, caches, external APIs, queued jobs. The reliability questions are always the same four: *When does it run? What if it fails? What if it runs twice? How do we know?*

## The machinery: a Postgres-backed task processor

Async work uses `@register_task_handler()` decorators ([webhooks/tasks.py](../../api/webhooks/tasks.py), [environments/tasks.py#L26-L31](../../api/environments/tasks.py#L26-L31)) from the shared `flagsmith-common` package; `.delay(args=…)` enqueues a row; a separate container consumes it ([docker-compose.yml#L81](../../docker-compose.yml#L81)). Features visible at call sites: **priorities** (`TaskPriority.HIGHEST` for environment updates), **scheduled execution** (`delay_until=feature_state.live_from`, [audit/tasks.py#L56-L59](../../api/audit/tasks.py#L56-L59)), and args-by-id (entities reloaded in the worker — serialization discipline).

Why a DB-backed queue instead of Redis/SQS: one fewer moving part for self-hosters, transactional adjacency to the data, and ordering/inspection via SQL. Costs: queue traffic competes with OLTP; throughput ceiling; polling latency. This tradeoff is a complete interview answer by itself.

## The side-effect map

| Side effect | Trigger | Code | Reliability posture |
| --- | --- | --- | --- |
| Environment document rebuild + SSE | AuditLog created with `environment_document_updated` | [audit/models.py#L141-L168](../../api/audit/models.py#L141-L168) → [environments/tasks.py#L31-L44](../../api/environments/tasks.py#L31-L44) | HIGHEST priority; idempotent (rebuild = recompute full document) |
| Environment/org webhooks | flag changed / audit created | [webhooks/webhooks.py#L63-L80](../../api/webhooks/webhooks.py#L63-L80), signed via `sign_payload` ([L17-L18](../../api/webhooks/webhooks.py#L17-L18)) | `backoff` retries (`WEBHOOK_BACKOFF_RETRIES`), failure email after N tries; **at-least-once** ⇒ receivers must dedupe |
| Third-party integrations (Datadog, Slack, Grafana…) | post_save signal on AuditLog | [audit/signals.py](../../api/audit/signals.py) | Fan-out of independent partial-failure surfaces |
| Edge/Dynamo sync (SaaS) | env/identity writes | [identities/views.py#L198-L206](../../api/environments/identities/views.py#L198-L206) `forward_identity_request.delay` | Fire-and-forget |
| Scheduled flag go-live | ChangeRequest with future `live_from` | [audit/tasks.py#L21-L60](../../api/audit/tasks.py#L21-L60) | Task re-enqueues itself if rescheduled — self-healing check-then-act |
| GitHub comments on flag delete | model lifecycle hook | [features/models.py#L143-L160](../../api/features/models.py#L143-L160) | Hook fires on save path — a side effect in a risky place (see below) |

## The concepts, defined where they bite

- **Idempotency** — running twice has the effect of once. Document rebuild: idempotent by construction (recompute-and-overwrite). Webhook *delivery*: not idempotent — retries can double-deliver, so payloads carry identity and receivers must dedupe. When you add a task, classify it first.
- **At-least-once vs at-most-once** — retries choose the first; the second loses work on crash. Everything here is at-least-once.
- **Outbox pattern** (not implemented here; name it as the alternative): write the "event to publish" in the same transaction as the data, publish from a relay. The AuditLog chain is outbox-*shaped* — the audit row is the committed fact; tasks fan out from it. The gap to a true outbox: if `.delay()` enqueues outside the commit, a crash between commit and enqueue loses the fan-out (**investigate** — task-processor internals are in the external package; verify enqueue transactionality before relying on it).
- **Backpressure & timeouts** — webhook calls use bounded retries with backoff rather than infinite queues; failure surfaces to humans via email ([call_webhook_with_failure_mail_after_retries](../../api/webhooks/tasks.py#L23-L25)).
- **Compensation** — no sagas here; flows are one-way fan-out from a committed fact, which is why they can be simple.

## Side effects in risky places — the anti-pattern to spot

Model lifecycle hooks that call external services (e.g. `create_github_comment` `AFTER_SAVE` on Feature, [models.py#L143-L160](../../api/features/models.py#L143-L160)) couple a DB write to third-party behaviour. It's mitigated (the hook enqueues a task) but the *trigger placement* means bulk operations, migrations, and test fixtures all walk through it. Prefer explicit service-level orchestration. Say it kindly in review: "does this need to fire on every save path, or only the user-facing delete?"

## Failure visibility

"How would I know it broke?" per effect: task rows persist state (queryable), webhooks email on final failure, Prometheus metrics + structlog events exist repo-wide ([api/README.md](../../api/README.md) metrics/logs sections), Sentry is wired in both API and FE ([pyproject.toml](../../api/pyproject.toml), [package.json](../../frontend/package.json)). Deep dive in [05-quality-engineering/06](../05-quality-engineering/06-observability-and-operations.md).

🎤 **Interview angle:** "Design webhook delivery" (signing, retries+backoff, at-least-once, dedupe key, failure notification, kill switch `DISABLE_WEBHOOKS` [webhooks.py#L78-L79](../../api/webhooks/webhooks.py#L78-L79)) and "How do you avoid slowing the write path as integrations grow?" (committed-fact + async fan-out). Both map to cards in [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) and [04](../08-interview-prep/04-system-design-from-this-repo.md).

**Drill:** classify each row of the side-effect map as idempotent / at-least-once-safe / needs-receiver-dedupe, and write the one-line justification. Basic: 3 rows right. Solid: all rows with justifications. Strong: identified the enqueue-transactionality question and how you'd verify it (read the task-processor source, or write a crash-injection test).
