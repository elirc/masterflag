# 03 — Architecture and Patterns

The judgment module. Six files:

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — who owns what, and where the leaks are
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — entities, migrations, transactions, safe schema change
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — every gate between a request and a write
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — the task processor, webhooks, retries, idempotency
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 14 pattern cards for recognition training
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, and what to change first (doubles as system-design interview prep)

Vocabulary you'll use throughout (each defined in context where it first appears): *invariant* — a condition the system promises to keep true; *boundary* — where responsibility (and trust) changes hands; *contract* — an agreed shape across a boundary; *idempotency* — same operation twice = same result once; *blast radius* — how much breaks if this goes wrong. These are interview words; use them only where they earn their place.
