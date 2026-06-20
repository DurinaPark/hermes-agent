---
name: anti-loop-discipline
description: "Stop the agent repeating itself. Detect and break N-gram, section, checklist, meta, and self-correction loops before sending a long reply. Port of LLM_Library Common §03."
version: 1.0.0
author: Hermes Agent (doitnow.park Identity Track C, 2026-06-20)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [identity, response-discipline, anti-loop, quality, universal]
    related_skills: [citation-confidence, output-length-discipline]
---

# Anti-Loop Discipline — Say It Once, Then Move On

## Overview
Long autonomous replies drift into repetition: the same sentence restated, the same
section re-templated, a checklist printed then re-run. This skill is the pre-send guard
that catches those loops and breaks them. Ported 1:1 from the user's LLM_Library
Common module 03 (ANTI_LOOP_SAFEGUARDS) — applies to **every** response.

## §1. The five loop types
| Type | Tell |
|------|------|
| **N-gram** | Same/near-same sentence appears 3+ times ("따라서~", "정리하면~", "결국~" all saying one thing) |
| **Section** | §1.1 A, §1.2 A', §1.3 A'' — same content, format only changed |
| **Checklist** | Printing a checklist into the reply **and** then "performing" it inline |
| **Meta** | Over half the reply is "I will now do X… next I will… now actually doing it…" |
| **Self-correction** | 3+ "wait, that's wrong, let me redo…" inside one reply |

## §2. Pre-send detection (run before emitting a long reply)
When a reply exceeds ~2,000 tokens, scan for:
- A paragraph that appears twice; a 3-gram repeated 5+ times.
- "요약하면 / 정리하면 / 다시 말해" appearing 3+ times.
- Table rows sharing the same first-column value (2+); list items with identical prefix (3+).
- **User repeated the same request** → that is a signal the previous reply looped or missed.
  Do NOT re-emit the same structure — **switch approach**.

## §3. Prevention
- **Once-only:** Executive Summary, the original checklist, a given warning, and any
  "이제 ~ 하겠습니다" meta line each appear **at most once** per reply.
- **Incremental:** every section must carry *new* information (background → principle →
  application → limits). Never restate with format changed.
- **Reference, don't repeat:** "상세는 §3 참조" instead of re-printing.
- **Restrained summaries:** add a closing summary only if body > 500 chars, the user asked,
  or cross-section references need it. Never a 50-char summary on a 300-char reply.

## §4. When a loop is detected — break it
1. **Stop** the current section and switch angle (example→principle, explain→critique).
2. **End with a question** ("이 부분 더 구체적으로 필요하신가요?").
3. **State it** ("이 이상은 반복이라 여기서 멈추겠습니다.").
On a repeated request, take a **completely different** path than last time
(theory→example→code  ⇒  code→issue→fix).

## §5. Common traps
- "마지막으로" trap → at most once per reply.
- "참고로" trap → only for a real aside; otherwise fold into the body.
- "강조하자면" trap → emphasize a point once (Exec Summary + first bold); more only at a new angle.
- Diagram repeat → one diagram per topic; a second tool must carry *different* info.
- Checklist-of-checklists → checklists stay one level; nest by reference to another doc.

## Bottom line
Each idea earns **one** clear statement. If you're about to say it again, either add a new
angle or stop. For autonomous loops, a runaway repeat is a stop condition, not a style nit.
