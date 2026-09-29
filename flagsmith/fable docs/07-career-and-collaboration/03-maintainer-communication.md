# Maintainer Communication

Maintainers optimize for *review cost*. Every template below front-loads what they need to say yes/no fast. Channels for this project: GitHub issues/discussions and the Discord linked in [README.md](../../README.md); read [CONTRIBUTING.md](../../CONTRIBUTING.md) before first contact.

## Asking for help without outsourcing thinking

```
I'm trying to <goal>. I expected <X> because <anchor/doc I read>, but got <Y>.
I've checked: <the 2–3 things you ruled out, with how>.
My current hypothesis is <Z>. Am I missing something about <specific mechanism>?
```

The ruled-out list is the respect signal. Contrast: "the toggle doesn't work, help?" vs "toggle 200s and the audit row exists, but the SDK serves stale past `CACHE_FLAGS_SECONDS`; task processor logs show the rebuild ran; is the HTTP `cache_page` layer ([features/views.py#L1019-L1025](../../api/features/views.py#L1019-L1025)) expected to lag here?" The second gets answered in minutes — and *often answers itself while you write it*.

## Reporting a bug

Minimal repro or it didn't happen: exact versions (compose image tag / commit), the smallest step list, expected vs actual, logs/response bodies trimmed to the relevant lines, and — the differentiator — *where you think it lives* with an anchor, labelled as a guess. If you can attach a failing test written in house style ([05/02](../05-quality-engineering/02-writing-tests-here.md)), your bug report becomes a gift.

## Proposing a feature

Issue first, RFC-lite: problem (whose pain, evidence), sketch (5 sentences max), blast radius, and the question "is this direction acceptable before I invest?" Never open a 2,000-line surprise PR — in this repo, watch for enterprise/OSS boundary questions (some surfaces are commercial; a maintainer may redirect you, and that's a *useful* fast no).

## Disagreeing with review

1. Restate their concern better than they said it (proves you heard).
2. Add the *new information* your position rests on (benchmark, anchor, failing case) — no new information means concede.
3. Offer the decision back: "If you still prefer X, I'll do X — flagging the tradeoff for the record."

Maintainers are load-bearing volunteers; you win by making agreement cheap, not by being right loudly. If you concede, concede cleanly — "good point, done" builds more capital than three paragraphs of face-saving.

## Responding to "no"

A rejected PR still pays: ask one question ("was it direction or timing?"), extract the reusable lesson, and keep the fork — several rejected-upstream ideas remain perfect portfolio/interview artifacts ([06/03](../06-contribution-practice/03-senior-build-projects.md) is designed for exactly that).

🎤 **Interview angle:** "Tell me about working with people you'd never met on code you didn't own." OSS contribution *is* the behavioural round: asynchronous, written, high-context communication with strangers under review. Keep links to 2–3 of your best threads — they're evidence, not just memories.

**Drill:** write (don't send) the help-request for debugging scenario 1 ([05/03](../05-quality-engineering/03-systematic-debugging.md)) as if you were stuck at the narrowing step. Basic: template followed. Solid: ruled-out list has real observations. Strong: writing it revealed the next check and you no longer need to send it — note what that felt like; it's rubber-ducking formalized.
