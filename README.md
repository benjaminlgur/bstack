# bstack

Claude Code skills.

| Skill | What it does |
| --- | --- |
| [`/cxr`](skills/cxr/SKILL.md) | Codex review mode: every batch of code Claude writes for the rest of the session is reviewed by GPT Sol (xhigh), fixed, and re-reviewed until clean. |
| [`/acxr`](skills/acxr/SKILL.md) | Same as `/cxr`, but reviews with GPT Astra 6.2 (high) instead of Sol. |
| [`/council`](skills/council/SKILL.md) | Convenes a two-member council (Claude Opus 5.5 and GPT Sol 6.1, both xhigh) to review the work, then fixes and re-reviews until both are satisfied. |

Both skills work best in T3 Code (they delegate through its `delegate_task` tool) and fall back to the `codex` / `claude` CLIs elsewhere.

## Install

Copy the skill folders into your Claude Code skills directory:

```bash
git clone https://github.com/benjaminlgur/bstack.git
cp -r bstack/skills/* ~/.claude/skills/
```
