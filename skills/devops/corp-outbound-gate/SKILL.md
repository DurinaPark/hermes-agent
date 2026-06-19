---
name: corp-outbound-gate
description: Use before sending ANY content from this EXTERNAL node toward the corporate side (claude-175 / 사내 OA / VDI). Run the text through the deterministic IP-gateway sanitizer (python3 -m relay.make_packet) first — it masks names/numbers/codes/claims by rule, not by LLM judgement. If it exits non-zero, the content is blocked: do NOT send it.
version: 1.0.0
author: Hermes Agent (doitnow.park IP gateway wiring, 2026-06-19)
license: MIT
platforms: [linux]
environments: [kanban]
metadata:
  hermes:
    tags: [ip-safety, masking, cognitive-firewall, outbound, corp, deterministic, one-way]
    related_skills: [autonomous-merge-gate, deterministic-routing]
---

# CORP Outbound Gate — Sanitize by Rule Before It Leaves

## Overview

This EXTERNAL node may build or analyze material that ends up flowing toward the corporate
side (the abstraction relay: `NAS Hermes → gate → claude-175 → 사내 git → VDI`). Anything
crossing that boundary must be stripped of internal identifiers — real names, part/code names,
yield/perf numbers, verbatim patent claims — **by a deterministic rule, never by your own
judgement.** An LLM deciding "is this safe to send?" leaks probabilistically; that is exactly
what the cognitive-firewall design (SPEC-02) forbids.

You do not do the masking. A code gate does. Your job is simply: **always route outbound-to-CORP
through the gate, and obey its verdict.**

## When to Use

- You are about to hand any text/package toward the corporate side (a card for claude-175, a
  packet for 사내 git, an answer that will be carried into CORP).
- Any time internal-derived content might leave this node.

**Don't use for:** content that stays EXTERNAL (personal/home tasks, public research) — there is
nothing to mask. This gate is specifically the EXTERNAL→CORP boundary.

## The Iron Law

```
NO INTERNAL-BOUND CONTENT LEAVES WITHOUT PASSING THE DETERMINISTIC GATE.
GATE EXIT ≠ 0  →  BLOCKED  →  DO NOT SEND. The model never overrides the rule.
```

## How to Run the Gate

The gate is `relay.make_packet`, mounted into this container at `/opt/home-system`
(`PYTHONPATH` already set). It reads the masking policy and emits an abstraction packet only if
the content is clean.

```bash
echo "<raw content>" | python3 -m relay.make_packet \
  --from vdi --to oa --domain <general-domain> \
  --request "<what you want done>" \
  --policy /opt/home-system/config/masking.yaml      # real policy (CORP); falls back to .example
```

Interpret the result strictly by **exit code**, not by reading the text:

| exit | meaning | action |
|------|---------|--------|
| `0` | clean — packet written to stdout | send the packet (and only the packet) |
| `2` | BLOCKED — a forbidden pattern survived sanitizing | **do not send.** Report which fields (stderr) and fix upstream |
| other | gate error (bad args / policy missing) | do not send; surface the error |

The packet on stdout already has names→role codes, numbers/part-codes→tokens, claims
generalized. Forward *that*, never your original draft.

## One-Way Ingestion (the reverse path)

When the corporate side sends results back (OA → VDI), do not let internal specifics
re-contaminate EXTERNAL context. Scan inbound text and refuse to ingest if it trips the policy:

```bash
python3 -c "import sys; from relay.make_packet import scan_inbound; \
v=scan_inbound(sys.stdin.read()); print(v); sys.exit(2 if v else 0)"
```

Non-empty result → the inbound text carries internal patterns; quarantine it, don't echo it into
EXTERNAL memory or downstream cards.

## Common Pitfalls

1. **Sending your draft instead of the packet.** Forward the gate's stdout, not the raw text you
   fed in. The raw text is unmasked.
2. **Reading the output to decide if it's "safe."** The decision is the exit code. `2` means
   stop, full stop — do not reason your way past it.
3. **Skipping the gate for "obviously harmless" content.** The rule is unconditional; harmlessness
   is the gate's call, not yours.
4. **Masking it yourself.** You are not the sanitizer. Run the gate; it is deterministic and
   auditable, you are not.
5. **Ignoring the reverse path.** Inbound corporate results get scanned too (one-way ingestion).

## Verification Checklist

- [ ] Outbound-to-CORP content was piped through `python3 -m relay.make_packet`.
- [ ] Decision taken from the **exit code** (0 send / 2 block / other error), not the text.
- [ ] Only the gate's emitted packet was forwarded — never the raw draft.
- [ ] On exit 2, content was withheld and the blocked fields reported.
- [ ] Inbound corporate replies were `scan_inbound`-checked before ingestion.

## Note (migration)

`relay.make_packet` + `shared_services.ip_gateway` currently live in the mounted home-system
tree (SSOT during the hermes-agent migration). End state: port them into a native hermes tool /
provider middleware so the gate runs automatically on outbound, without depending on the agent
remembering to call it. Until then, this skill enforces the call.
