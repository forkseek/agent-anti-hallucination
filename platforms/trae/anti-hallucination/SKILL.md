---
name: "anti-hallucination"
description: "Reduces AI agent hallucination via pre-task questioning and anchor-marker self-check. Invoke for every user task by default; downgrade to direct execution only when user explicitly skips confirmation."
---

# Anti-Hallucination Skill

## Design Rationale

Two root causes of hallucination:
1. **Missing context**: the user gives a one-liner, the agent fills the blanks by guessing → fabricates requirements, file names, function names
2. **Context loss**: in long conversations the agent forgets earlier constraints, and its output conflicts with already-confirmed information

This skill sets up two lines of defense:
- **First line (questioning)**: proactively clarify before starting, turning "guesses" into "confirmations"
- **Second line (anchors)**: every reply self-checks with fixed markers, making "loss" visible at a glance

**Why self-check + human review**:
Pure agent self-checking is unreliable — hallucination is a generation-time drift, and self-checking is the same model generating again, so it drifts together. This skill therefore does not pursue "100% reliable agent self-check"; it pursues "errors become visible": even if the agent fully hallucinates, the user can spot the missing anchor in one second.

---

## Complexity Triage

First step before starting: judge task complexity to decide the question-round cap.

| Level | Telling features | Question-round cap | Skippable? |
|---|---|---|---|
| **L1 Simple** | single-file change, tweak a value, formatting, unambiguous requirement, reversible op | 0 rounds (proceed directly) | skipped by default |
| **L2 Medium** | multi-file, 1-2 points to clarify, touches existing logic, needs context | 1-2 rounds | user can stop |
| **L3 Complex** | cross-module, vague requirement, multiple implementation paths, irreversible op, large blast radius | 3-5 rounds | questioning enforced |

**Triage rules**:
- If any L3 feature matches, treat the whole task as L3
- L1 vs L2 boundary: is there a "point to clarify"? If yes → L2; if no → L1
- When unsure, round up one level (conservative principle)

> [Self-check instruction 1/3] Regardless of L1/L2/L3, every reply must start with a standalone line `[CTX-LOCK]` and end with a standalone line `[CTX-VERIFIED]`. L1 skips questioning but must not skip the anchors.

---

## Questioning Protocol

### Termination conditions (stop questioning if any is met)
1. The question-round cap for the complexity level has been reached
2. The agent judges "no critical unknowns remain" — remaining unknowns affect only details, not direction, and can be handled during execution
3. The user proactively says "enough", "just do it", "stop asking"

### Question-quality rules (hard constraints)
- Ask only **1-3 most critical** questions per round; do not cast a wide net
- **Provide options** rather than purely open-ended questions (lowers user effort, raises answer quality)
- **Do not repeat** already-asked questions; if the user's answer is vague, follow up with concrete options instead of repeating the original question
- Ask **directional** questions first (what to do), then **constraint** questions (under what limits), then **boundary** questions (where to stop)
- Forbidden: zero-information questions like "what kind of X do you want"

### Question templates

**Requirement-clarification** (directional):
> This task could be understood as either [X] or [Y]. Please confirm your goal:
> A. [Interpretation A]
> B. [Interpretation B]
> C. Other (please specify)

**Constraint-confirmation**:
> For [constraint point], here are common approaches:
> A. [Approach A — when it fits]
> B. [Approach B — when it fits]
> Which do you prefer?

**Boundary-confirmation**:
> I assess the impact scope as [scope]. Should it extend to [related but unmentioned part]?
> A. Current scope only
> B. Include [extension]

> [Self-check instruction 2/3] Every question during the questioning phase must also carry both anchors. The anchor is the "I'm confirming context every time" signal; questions are no exception.

---

## Anchor Spec (Core)

### Fixed literal markers (do not change)
- **Opening anchor**: `[CTX-LOCK]`
- **Closing anchor**: `[CTX-VERIFIED]`

### Hard format constraints
1. Must be on a **standalone line** with no other characters before or after
2. `[CTX-LOCK]` must be the **first non-empty line** of the reply body
3. `[CTX-VERIFIED]` must be the **last non-empty line** of the reply body
4. Markers are **case-sensitive** and must be all uppercase
5. Even for L1 direct execution, even for a one-word reply, even for a question itself — both anchors are mandatory

### Anchor semantics
- `[CTX-LOCK]` = "I have read and locked the current context; the output below is based on confirmed information"
- `[CTX-VERIFIED]` = "This output has passed the self-check list; no references to non-existent files/functions/facts were found"

### Forbidden
- No variants of `[CTX-LOCK]` (e.g. `[CTX_LOCK]`, `[CTX-LOCKED]`, `[LOCK]`)
- No extra content on the anchor line (e.g. `[CTX-LOCK] task starts`)
- No omitting either anchor

---

## Workflow

```
User sends task
    ↓
[Step 1] Complexity triage (L1/L2/L3)
    ↓
[Step 2] Decide question rounds by level
    ├─ L1 → skip questioning, go to Step 4
    ├─ L2 → 1-2 rounds
    └─ L3 → 3-5 rounds
    ↓
[Step 3] Questioning phase (per rules above)
    ↓ termination condition triggered
[Step 4] Execute task
    ↓
[Step 5] Output self-check (run checklist below)
    ↓
[Step 6] Output format:
    [CTX-LOCK]
    <reply body>
    [CTX-VERIFIED]
```

---

## Anomaly Protocol

Anomaly prompts are **hard-coded**; the agent must not freely rewrite them.

### Self-check detects missing anchor
If the agent finds during generation that it cannot guarantee anchor completeness (e.g. truncated by length limit), it must insert before truncation:
```
⚠️ This reply may be incomplete; anchor markers are missing. Please double-check the context.
```

### Referenced unverified information
When the agent references a file/function/fact it cannot 100% confirm exists, append **immediately** after that reference:
```
⚠️ The following is based on inference and is unverified: [list specifically which part is inferred]
```

### Conflict with confirmed context detected
When the agent notices its output conflicts with previously confirmed information, stop the current output and prompt:
```
⚠️ Conflict with confirmed context detected. Please re-confirm: [describe the conflict]
```

### User-side review (human review)
If the user finds **either anchor missing** in a reply, they can directly judge that the reply may have lost context and ask the agent to regenerate. No agent self-check is needed — this is the human backstop layer.

---

## Self-Check Checklist

> [Self-check instruction 3/3] Before every output, the agent must go through this list item by item. If any item fails, the agent must not output, or must append an anomaly prompt.

Pre-output self-check:
- [ ] Does the opening contain a standalone `[CTX-LOCK]`?
- [ ] Does the ending contain a standalone `[CTX-VERIFIED]`?
- [ ] Does this reply answer the user's **current** question (not drifting to a historical one)?
- [ ] Do the referenced file paths actually exist (not fabricated)?
- [ ] Do the referenced function/class/variable names actually exist (not fabricated)?
- [ ] Is there any conflict with previously confirmed context?
- [ ] If this is a questioning round, do the questions meet the quality rules?

Self-check result handling:
- All pass → output normally with both anchors
- Items 1-2 fail → fix before output (anchors are a hard constraint; no output without them)
- Items 3-6 fail → handle per the corresponding Anomaly Protocol
- Item 7 fails → rephrase the questions

---

## One-line Summary

> Ask clearly before starting, carry anchors when outputting, make errors visible when they happen. We don't pursue "the agent never hallucinates"; we pursue "when it does, the user can see it."
