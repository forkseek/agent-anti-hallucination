# Anti-Hallucination Skill

English | [简体中文](./README.md)

A general-purpose, instruction-only Skill for reducing AI agent hallucination. It uses a two-line-of-defense mechanism — **pre-task questioning to fill context gaps** + **anchor-marker self-check** — so that when hallucination happens, the signal is visible.

## What problem does this Skill solve

AI agent hallucination mainly comes from two sources:

1. **Missing-context hallucination** — the user gives a one-liner, the agent fills the blanks by guessing, fabricating requirements, file names, and function names
2. **Context-loss hallucination** — in long conversations the agent forgets earlier constraints, and its output conflicts with already-confirmed information

This Skill sets up one line of defense for each type.

## Core mechanism

### First line of defense: pre-task questioning to fill context gaps

Tasks are triaged by complexity (L1/L2/L3) to decide the question-round cap, turning "guesses" into "confirmations".

| Level | Telling features | Question-round cap |
|---|---|---|
| L1 Simple | single-file change, tweak a value, unambiguous requirement | 0 rounds (proceed directly) |
| L2 Medium | multi-file, 1-2 points to clarify | 1-2 rounds |
| L3 Complex | cross-module, vague requirement, irreversible op | 3-5 rounds |

### Second line of defense: anchor self-check + human review

Every reply must be wrapped in fixed markers, making "context loss" visible at a glance:

```
[CTX-LOCK]
<reply body>
[CTX-VERIFIED]
```

- `[CTX-LOCK]` opening anchor = "I have read and locked the current context"
- `[CTX-VERIFIED]` closing anchor = "This output has passed the self-check list"

**Why "self-check + human review" instead of pure agent self-check**: pure agent self-checking is unreliable — hallucination is a generation-time drift, and self-checking is the same model generating again, so it drifts together. This Skill therefore does not pursue "100% reliable agent self-check"; it pursues "errors become visible": even if the agent fully hallucinates, the user can spot the missing anchor in one second.

## Files

| File | Description |
|---|---|
| `SKILL.md` | Skill main definition (Chinese) — metadata, design rationale, triage criteria, questioning protocol, anchor spec, self-check list, anomaly protocol, examples |
| `SKILL.en.md` | English version of the Skill definition |
| `README.md` | This file (Chinese) |
| `README.en.md` | This file |
| `LICENSE` | MIT license |

## Usage

This Skill is a **general-purpose reference spec**, instruction-only, with no executable code. To deploy it on a specific agent platform, place `SKILL.md` (or `SKILL.en.md` for English) in that platform's skills directory.

Example (Trae):
```
~/.trae-cn/skills/anti-hallucination/SKILL.md
```

## Tech stack

- **Language**: Markdown (100%)
- **Executable code**: none
- **Dependencies**: none

This Skill is a prompt/instruction definition with no programming-language code. GitHub Linguist will classify the repo as Markdown.

## License

[MIT](./LICENSE)

## Author

Aikun ([@forkseek](https://github.com/forkseek))
