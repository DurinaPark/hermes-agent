---
name: deterministic-routing
description: Use when an orchestrator/dispatcher assigns work to workers in an autonomous pipeline. Route by declared capability with deterministic rules — never by an LLM's guess at an assignee name. Verify the assignee exists before creating the card, prefer healthy/high-success workers, and FAIL LOUDLY when no worker matches instead of dropping the card silently.
version: 1.0.0
author: Hermes Agent (ported from doitnow.park EngOS capability router, 2026-06-19)
license: MIT
platforms: [linux, macos, windows]
environments: [kanban]
metadata:
  hermes:
    tags: [kanban, routing, dispatch, multi-agent, deterministic, capability, health-aware]
    related_skills: [kanban-orchestrator, kanban-worker, autonomous-merge-gate]
---

# Deterministic Routing — Match Capability, Don't Guess a Name

## Overview

When an orchestrator fans work out to workers, the assignment decision is a fork in the road:
either a rule decides it, or the orchestrator LLM picks an assignee name from memory. The
second path fails the same way every probabilistic decision fails — the model names a worker
that doesn't exist (a typo, a renamed profile, a plausible-but-absent specialist), and the
dispatcher **silently drops the card**. It doesn't autocorrect, doesn't suggest, doesn't fall
back. The card sits in `ready` forever and no one is told.

This skill makes routing a **deterministic, capability-driven rule**, the same way
`autonomous-merge-gate` makes verification trust artifacts over narration. The orchestrator
still decomposes the work; but *which worker* gets a card is decided by matching the card's
declared capabilities against the live roster — and an unmatched card **fails loudly**, never
silently.

## When to Use

- You are an orchestrator/dispatcher profile creating cards for other workers.
- Any unattended fan-out where "assign to the right specialist" must not depend on the model
  remembering the exact profile name.
- You want a routing decision that is reproducible: same card + same roster → same assignee.

**Don't use for:** a single-profile setup (nothing to route — one worker does everything), or a
one-shot reasoning task you should just answer directly.

## The Iron Law

```
ROUTING IS A RULE OVER THE LIVE ROSTER — NOT THE MODEL'S RECOLLECTION OF A NAME.
A CARD THAT MATCHES NO WORKER FAILS LOUDLY. IT IS NEVER CREATED-AND-FORGOTTEN.
```

## The Routing Procedure

### 1. Discover the live roster first (never assume)

Before assigning anything, get the *actual* profiles/capabilities on this machine. The roster is
data, not memory.

```bash
hermes profile list           # the profiles configured here
# or sanity-check one name (empty list = unknown, NOT an error):
# kanban_list(assignee="<name>")
```

Cache it for the rest of the conversation. If a card needs a capability the roster can't cover,
**ask the user** which profile to use or create — do not invent a name.

### 2. Declare each card's required capabilities

Tag the card with *what it needs* (e.g. `code`, `python`, `web`, `t32`), not *who* you think
does it. Capability is stable; names drift.

### 3. Match deterministically — intersection of healthy workers

For a card needing capabilities `C = [c1, c2, …]`, the candidate set is the **intersection** of
workers healthy for each `ci`:

```
candidates = ∩  healthy(ci)   for ci in C
```

- A worker must satisfy **every** required capability (intersection, not union).
- "Healthy" excludes any worker marked unhealthy / offline.
- This is pure set logic — no LLM call, no ranking model. Same inputs → same candidate set.

### 4. Empty match → FAIL LOUDLY (the whole point)

If `candidates` is empty, do **not** create the card against a guessed name and move on. Mark it
FAILED with an explicit reason and notify the operator:

```
[Alert] no healthy worker for capabilities [code, t32] — card not dispatched.
```

A silent `ready` card that never runs is the failure mode this skill exists to kill. Loud
failure is correct behavior, not an error to suppress.

### 5. Pick among candidates by health, then readiness

Order the candidate set by success rate (desc) and assign the first worker that is
infrastructure-ready:

- `ready` → assign, done.
- `waking` (worker reachable but spinning up) → **defer**, retry shortly. Not a failure.
- all candidates unreachable → fail as in step 4.

Health-aware tie-breaking is still deterministic: a stable sort over a stable success-rate
snapshot gives the same pick every time.

### 6. Idempotent assignment

If a card already has a routed worker, do not re-route it — re-routing churns assignments and
breaks the "same card → same worker" property. Route once; on a re-visit, only advance an
already-assigned card.

## Capability vs. Name — quick contrast

| | LLM picks a name (avoid) | Capability rule (this skill) |
|---|---|---|
| Decision basis | model's recollection | declared capability ∩ live roster |
| Unknown/typo name | silently dropped | impossible — names come from the roster |
| No suitable worker | card stuck in `ready` | FAILED + operator alert |
| Reproducible | no | yes (same card+roster → same worker) |
| Load / health | ignored | success-rate-ordered, health-filtered |

## Common Pitfalls

1. **Assigning by remembered name.** The model "knows" a `researcher` profile that was renamed
   last week → silent drop. Always match against the roster discovered in step 1.
2. **Union instead of intersection.** A card needing `python` AND `t32` must go to a worker with
   both, not either. Intersect.
3. **Swallowing the no-match.** An empty candidate set is a loud FAILED + alert, never a
   created-and-forgotten card.
4. **Treating `waking` as failure.** A worker spinning up is a `defer`/retry, not a dead end —
   only fail when every candidate is unreachable.
5. **Re-routing an assigned card.** Breaks determinism and churns the board. Route once;
   idempotent thereafter.
6. **Ranking with a model.** "Which worker is best?" is a sort over success rate, not an LLM
   judgment call. Keep it mechanical.

## Verification Checklist

- [ ] Roster discovered from the live system, not assumed.
- [ ] Each card declares required capabilities (not a hard-coded name).
- [ ] Candidate set = intersection of healthy workers across all required capabilities.
- [ ] Empty candidate set → FAILED + operator alert (no silent `ready` card).
- [ ] Among candidates: success-rate-ordered, first infra-ready wins; `waking` defers.
- [ ] Assignment is idempotent (no re-routing of already-routed cards).

## One-Shot Recipe — route a card needing [code, python]

```
1. hermes profile list                       # roster = {docker-worker:[code,python], researcher:[web]}
2. card.required = [code, python]
3. candidates = healthy(code) ∩ healthy(python) = {docker-worker}
4. non-empty -> order by success_rate -> docker-worker ready? assign : (waking? defer : fail)
5. empty -> kanban FAILED + "[Alert] no worker for [code,python]"
```

Pairs with `kanban-orchestrator` (decomposition: what the cards are) — this skill governs *which
worker* each card goes to, deterministically.
