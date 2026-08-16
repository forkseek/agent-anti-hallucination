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
| `SKILL.md` | Skill main definition, Chinese (general reference spec) |
| `SKILL.en.md` | English version of the Skill definition |
| `platforms/trae/` | Trae platform build (Trae frontmatter spec) |
| `platforms/claude/` | Claude platform build (Agent Skills open standard) |
| `platforms/generic/` | Generic system-prompt build (any agent) |
| `LICENSE` | MIT license |

## Installation

This Skill has been adapted for three platforms. Pick yours and follow the instructions.

### Trae

```bash
# Copy into the Trae skills directory
cp -r platforms/trae/anti-hallucination ~/.trae-cn/skills/
```

Verify: send a task in Trae and check that replies start with `[CTX-LOCK]` and end with `[CTX-VERIFIED]`.

### Claude (Claude Code / Claude.ai)

```bash
# Claude Code: copy into .claude/skills
cp -r platforms/claude/anti-hallucination ~/.claude/skills/
```

Claude.ai users: paste the content of `platforms/claude/anti-hallucination/SKILL.md` into a Project's custom instructions.

Verify: type "what skills do you have" — `anti-hallucination` should appear; after a task, check for anchor markers.

### Generic system prompt (ChatGPT / Gemini / local LLMs, etc.)

Paste the full content of `platforms/generic/system-prompt.md` into your agent's system prompt / custom instructions / first message.

Verify: send any task and check that replies carry both `[CTX-LOCK]` and `[CTX-VERIFIED]`.

## Tech stack

- **Language**: Markdown (100%)
- **Executable code**: none
- **Dependencies**: none

This Skill is a prompt/instruction definition with no programming-language code. GitHub Linguist will classify the repo as Markdown.

## Known limitations

- **Model compatibility varies**: different models obey "force a fixed marker at the start/end" to different degrees; the anchor mechanism may be ignored by small models or heavily RLHF-aligned models
- **No large-scale effect validation**: the anti-hallucination effect has not been quantitatively tested at scale; validate it in your own use case before relying on it
- **Self-check is not absolutely reliable**: agent self-checking can hallucinate too; human review (watching for missing anchors) is the final backstop

## License

[MIT](./LICENSE)

## Author

Aikun ([@forkseek](https://github.com/forkseek))
