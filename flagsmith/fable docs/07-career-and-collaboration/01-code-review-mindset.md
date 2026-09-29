# Code Review Mindset

## The five layers, in order

1. **Does it work?** — happy path, obvious errors. (Table stakes; if you stop here you're a linter.)
2. **Is it correct?** — edge cases, concurrency, permissions, contract adherence. *Where IDOR and stale-cache bugs live.*
3. **Will it stay correct?** — tests pinning the invariant (not the implementation), migrations reversible, types honest.
4. **Does it fit?** — house patterns: services layer, fixtures, naming templates, RTK-not-fetch, tag invalidation. Divergence is a cost even when the code is fine.
5. **Is it kind to future maintainers?** — names, comments explaining *why* (the deadlock loop comment at [audit/models.py#L162](../../api/audit/models.py#L162) is the exemplar), TODOs with exit conditions (the Flux bridge style, [FeaturesPage.tsx#L97-L98](../../frontend/web/components/pages/features/FeaturesPage.tsx#L97-L98)).

Review in that order and *say which layer a comment belongs to* — it calibrates severity automatically (layer 1–2 findings block; 4–5 rarely do).

## Repo-specific checklist

**API PRs:** permission class + action map updated? queryset scoped (grep `objects.get`)? migration present & named? mypy clean, no naked `type: ignore`? test naming template? Given/When/Then? diff coverage 100%? structured log for the "moment"? task args are ids? contract change → OpenAPI/docs regenerated (CI checks anyway)?
**FE PRs:** no raw `fetch`? endpoint via `injectEndpoints` with tags *both* directions? types in `Res`/`Req`? no relative imports? `.unwrap()` on awaited mutations? new inline unions extracted? lint/typecheck/unit green? Flux store touched — why, and is the exit TODO updated?
**Both:** does the PR description say how it was tested? is anything user-visible behind a flag?

## Comment craft — real-shaped examples

Blocking, with a path out:
> **[correctness/blocking]** The target environment comes from the request body but isn't permission-checked — a member of project A can write into project B (see the scoping pattern at `features/views.py#L995-L999`). Could we resolve the target through `get_permitted_projects` and add a cross-org 404 test (recipe 4 in the test guide)?

Important, hedged honestly:
> **[fit/important]** This adds a second fetch path outside RTK Query; the house rule is services-only HTTP (frontend/CLAUDE.md). If there's a constraint I'm missing, happy to hear it — otherwise moving it into `common/services/` keeps the auth/header logic centralized.

Optional, clearly optional:
> **[nit/optional]** `it.each` table style (see format.test.ts) would compress these five cases — fine either way.

Anti-patterns: severity inflation ("blocking" on nits burns trust), drive-by scope expansion ("while you're here…"), rhetorical questions that are actually orders, and reviewing the *author* ("you always…") instead of the diff.

## Receiving review

Respond to every comment (fix / push back with a reason / file follow-up). Batch pushes. If two rounds haven't converged, move to a call/issue and *summarize the resolution in the thread* — the thread is documentation.

🎤 **Interview angle:** "Tell me about a difficult review." A strong answer names the layer of disagreement (usually 4: fit), how you separated substance from preference, and the artifact that resolved it (benchmark, test, house rule). Practice with review kata 5's SQL rewrite — it's a natural "correct code, wrong change" story.

**Drill:** review one real open Flagsmith PR on GitHub (read-only, don't post). Write your comments locally with layer tags, then compare with the maintainers' actual review when it lands. Basic: your blocking set ⊆ theirs. Solid: you caught everything they blocked on. Strong: you predicted a comment *and* one place they'd accept-with-nit.
