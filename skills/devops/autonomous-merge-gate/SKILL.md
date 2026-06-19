---
name: autonomous-merge-gate
description: Use when a reviewer/verifier profile must judge an implementer's pushed work in an autonomous Kanban pipeline (no human watching each card). Verify the real artifact — a fresh clone, pytest exit code, and git diff — never the worker's completion text. Gate the merge to main behind human approval for critical changes.
version: 1.0.0
author: Hermes Agent (ported from doitnow.park EngOS merge-gate, 2026-06-19)
license: MIT
platforms: [linux, macos, windows]
environments: [kanban]
metadata:
  hermes:
    tags: [kanban, multi-agent, verification, merge-gate, pytest, autonomy, false-reject]
    related_skills: [kanban-orchestrator, kanban-worker, requesting-code-review, test-driven-development]
---

# Autonomous Merge Gate — Trust the Artifact, Not the Narration

## Overview

In an autonomous pipeline (`implementer → reviewer`, no human watching each card), the
reviewer's single hardest job is deciding whether the implementer *actually* did the work.
The trap: a worker LLM ends its turn with narration like "awaiting commit approval" or
"changes ready for review" — text that *sounds* incomplete even when the code was pushed
and passes. A reviewer that judges this **text** will false-reject perfectly good work, requeue
it, and burn the next worker's budget on a problem that does not exist.

**Core principle: judge the artifact, never the claim.** The artifact is a fresh clone of the
branch the worker pushed, the exit code of `pytest` run against it, and the `git diff` versus
`main`. The worker's completion summary is *advisory prose* — it never gates pass/fail.

This is the autonomous-pipeline sibling of `requesting-code-review`. That skill verifies
*your own* working-tree changes before you commit. This skill verifies *another worker's
already-pushed branch* with no shared context — and adds the merge-to-main decision that an
unattended pipeline has to make on its own.

## When to Use

- You are a reviewer/verifier profile and an implementer card just reached `done` with a
  pushed feature branch (e.g. `feature/<task-id>`).
- A Kanban reviewer card has `parents=[implementer_card]` and must block or complete based on
  whether the implementation is real and green.
- Any unattended loop where "the worker said it's done" must become "the tests say it's done"
  before code reaches `main`.

**Don't use for:**
- Verifying your *own* uncommitted changes → use `requesting-code-review` (works on the local
  working tree / `git diff --cached`).
- Non-repo tasks (research, a written answer, a doc) — there is no branch to clone. Fall back
  to judging the worker's output text directly; this gate `skip`s.

## The Iron Law

```
PASS/FAIL COMES FROM A FRESH CLONE + pytest EXIT CODE — NEVER FROM THE WORKER'S TEXT
```

If the worker's summary says "done" but the branch does not exist, or pytest is red, it is a
FAIL. If the worker's summary says "awaiting approval" but the branch exists and pytest is
green, it is a PASS. **The branch and its tests are the ground truth.**

## The Pipeline — five ordered stages

Run these in order. Automerge is last and strictly after human approval, so a critical change
can never reach `main` before a person has acked it.

### 1. Resolve the branch (or skip)

The worker leaves a push marker in its output when it actually pushed code. No marker → this
is not a repo task → return `skip` and let the caller judge the output text.

```bash
# Convention: the implementer pushes feature/<task-id> and says so in its completion event.
BRANCH="feature/${TASK_ID}"
git ls-remote --heads "$REPO_URL" "$BRANCH" | grep -q "$BRANCH" || echo "SKIP: no pushed branch"
```

### 2. Verify in a fresh clone (read-only — do NOT merge yet)

A throwaway clone guarantees you are looking at exactly what was pushed, with zero leftover
state from the worker's session.

```bash
WS=$(mktemp -d)
git clone --depth 50 "$REPO_URL" "$WS/repo" && cd "$WS/repo"
git checkout "$BRANCH" || { git fetch origin "$BRANCH" && git checkout -B "$BRANCH" "origin/$BRANCH"; }

# pytest exit codes are the contract:
python -m pytest -q; rc=$?
#   0  -> pass
#   5  -> no tests collected: DO NOT block (the change had no tests; that is not a failure here)
#   else -> fail (requeue with the tail of output as feedback)

git diff --stat "origin/main...HEAD"
git diff --unified=3 "origin/main...HEAD"   # this diff is the reviewer's evidence
```

Map the result to a status:

| status | meaning | caller action |
|--------|---------|---------------|
| `pass` | pytest 0, or exit 5 (no tests) | proceed to stage 3 |
| `fail` | pytest non-zero (and not 5) | requeue with failing tail; FAIL after retries |
| `error` | clone/auth/pytest unavailable | **do not block** — fall through (availability first) |
| `skip` | no pushed branch (non-repo task) | judge worker output text instead |

Clean up the temp clone in a `finally` — always, even on exception.

### 3. Hand the reviewer the artifact, not the worker's text

When you dispatch the independent reviewer subagent (fresh context, as in
`requesting-code-review` Step 5), the review *input* is the evidence bundle, not the worker's
self-report:

```
[EXECUTION EVIDENCE — what the worker actually pushed to the branch. Not a plan, not a claim.]

## pytest result (gate-measured)
<tail of pytest output, or "passed (no tests collected)">

## git diff (vs origin/main)
<the unified diff>
```

The reviewer judges *this*. Never paste the worker's "awaiting approval / done" narration into
the reviewer prompt — that is the exact input that caused false-rejects.

### 4. Human gate (L3) for critical changes — BEFORE any merge

If the card's priority is `critical` (system-core / irreversible), request human approval and
**stop here** — do not merge. Emit the request once (idempotent; the loop revisits the card),
ping the operator, and leave the card in its verifying/blocked state until a human acks.

```
L3 approval required — operator must ack before merge to main.
```

This ordering is the safety property: review and tests can run unattended, but a critical
change waits for a person. Putting automerge before this gate would let a critical change land
on `main` without human ack — never do that.

### 5. Automerge (only after stages 2–4 pass)

Only if: gate enabled **and** automerge enabled **and** (not critical OR human approved).
Merge `--no-ff` and push. A conflict or push failure is reported as "not merged" — it does
**not** crash the pipeline; a human resolves it.

```bash
git checkout main && git reset --hard origin/main
git merge --no-ff -m "merge $BRANCH (autonomous gate)" "$BRANCH"
git push origin main
```

If automerge is off, leave the green feature branch for a human to merge. Verification and
merge are decoupled on purpose — you can run the gate for weeks with automerge off to build
confidence before letting it touch `main`.

## Retry policy

`fail` is recoverable. Requeue the implementer card with the failing pytest tail as feedback,
up to **2 reworks**. After that, mark FAILED and notify the operator with the failure summary —
do not loop forever. (Mirror this count with the reviewer's own reject budget so a card can't
ping-pong between the two gates indefinitely.)

## Graceful degradation — the gate must never freeze the system

A verification gate that halts the whole pipeline when *it* breaks is worse than no gate. Every
gate-inability is `error`, which **falls through** rather than blocking:

- No repo token / clone fails / auth fails → `error`, flow continues.
- `pytest` not installed in the runner → `error`, flow continues (and fix the image).
- diff extraction fails but pytest passed → still `pass` (judge on pytest alone).

Availability first. The merge gate is a safety net, not a tripwire.

## Common Pitfalls

1. **Judging the worker's completion text.** "Awaiting commit approval" is narration, not a
   result. If you reject on it, you false-reject green work. Judge the branch + pytest. This is
   the single failure mode this skill exists to prevent.
2. **Running the reviewer before the tests.** If a text-only reviewer rejects first, pytest
   never runs and you lose your ground truth. Verify (clone + pytest + diff) *first*, then
   review the evidence.
3. **Automerge before the human gate.** A critical change must not reach `main` before a person
   acks it. Stages 4 then 5, never the reverse.
4. **Treating exit 5 as failure.** `pytest` exit 5 means "no tests collected" — the change
   simply had no tests. That is not a red build; do not block on it.
5. **Letting the gate's own breakage block the pipeline.** Missing token or pytest = `error` =
   fall through, not halt.
6. **Verifying in the worker's dirty workspace.** Always a fresh clone — leftover state lies
   about what was actually pushed.
7. **Infinite requeue.** Cap reworks (2) and then FAIL loudly with operator notification.

## Verification Checklist

- [ ] Branch resolved from a real push marker (or `skip` for non-repo work).
- [ ] pytest ran in a **fresh** clone; exit code, not text, decided pass/fail.
- [ ] exit 5 treated as pass (no tests), non-zero (≠5) as fail.
- [ ] Reviewer received the diff + pytest evidence — not the worker's narration.
- [ ] Critical change paused for human L3 approval before any merge.
- [ ] Automerge ran only after tests + review + (approval if critical) all passed.
- [ ] Any gate inability surfaced as `error` and did **not** block the pipeline.
- [ ] Temp clone cleaned up.

## One-Shot Recipe — reviewer card on an implementer's branch

```
Goal: verify card T123 (implementer pushed feature/T123), priority=normal, automerge=on.

1. git ls-remote --heads $REPO feature/T123   # exists? else skip
2. clone fresh -> checkout feature/T123 -> python -m pytest -q  (rc)
3. rc==0 or rc==5 -> capture `git diff origin/main...HEAD`; rc∈{else} -> requeue w/ tail (<=2x)
4. dispatch reviewer with [pytest result + diff] as the ONLY evidence
5. priority normal -> skip L3; merge --no-ff -> push origin main
6. kanban_complete(T123, summary="verified: pytest green, merged to main")
```

The implementer can end its turn saying anything at all. Stage 2 is what decides the truth.
