---
name: cxr
description: Turns on Codex review mode for the rest of the session. From then on, every batch of code you write is reviewed by GPT Sol (gpt-6.1-sol, xhigh) as a T3 review delegate briefed with the change's intent and scope; you make the smallest fix for each finding you agree with and re-review until clean, before you call the work done. Invoke ONLY when the user types `/cxr` or their message contains the literal token `cxr`. Never invoke it for a general "review this" request or on your own initiative.
argument-hint: "[task]"
allowed-tools: mcp__t3-code__orchestrator_capabilities, mcp__t3-code__delegate_task, mcp__t3-code__task_cancel, Bash(codex exec:*)
---

# Codex review mode

This turns on a standing rule for the rest of the session. It is not a one-time review. From now until the user turns it off, any code you write gets reviewed by GPT Sol before you treat it as done: Sol reviews, you fix what you agree with, and you repeat until a round turns up nothing worth acting on.

The review is there to make the change correct, not bigger. Make the smallest changes that answer real findings, watch for scope creep, and don't be afraid to push back.

Input, if any: $ARGUMENTS

If the input (or the rest of the user's message) describes a task, start on it now with the rule in force. If there is no task, the rule applies to whatever the user asks next. Code you already wrote earlier in this session that has not been reviewed counts as the first batch, so review it now. If there is nothing to review yet, say in one line that review mode is on.

## When to review

- Review each batch of code once it is written and you have run the checks you normally would, and before you report it as done or end your turn. A batch is a coherent piece of work (a feature, a fix, a refactor step), not each individual edit.
- On a long task, review at each milestone instead of once at the end. Later work then builds on reviewed code, and each review stays small.
- Follow-ups are covered too. If the user later asks for a tweak, a bug fix, or a whole new task, that code gets the same treatment without them asking again.
- Changes with no code in them (docs, comments, prose) don't need a review.
- Throwaway scripts don't need a review: code you write to inspect, debug, or measure something (a scratchpad script, a one-off query, an inline `python -c`) that you run, read, and discard. Its output is the check. Write them outside the repo so they don't show up in the diff under review.
- A one-off script that changes things that are hard to undo (bulk deletes or renames, a data backfill, writes to a database or remote service) does get reviewed, and before you run it: a review after it has run can't prevent anything. Scope its brief to that file.
- A script that stays (committed, scheduled, or one the user will rerun) is ordinary code, not a throwaway.
- The mode stays on until the user says to stop, for example "cxr off".

## Running a review

Each round is one review delegate in T3 Code.

1. Call `orchestrator_capabilities` once per thread, before the first review. Target provider `codex`, model `gpt-6.1-sol`, with the effort option (currently `reasoningEffort`) set to `xhigh`. If that model or effort is not listed, or a call errors (not logged in, usage limit), stop and show the user the error instead of switching models or reviewing the code yourself.
2. Start the round with `delegate_task`: `role: "review"`, `mode: "async"`, default `shared` isolation (a worktree would leave out your uncommitted changes), the target above, the brief below as `task`, and a new `clientRequestId` each round. Save the `taskId`.
3. Wait for the completion notice and follow the `next` the call returns. Don't poll, don't start a second reviewer, and don't edit the files under review while it runs, or Sol is reviewing a moving target. A timeout, error, or empty reply is not approval: `task_cancel` it and start a fresh round with the same brief.

Every round, including a re-review, is a new `delegate_task`. The child has no history, so everything it needs goes in the brief. Never continue an old reviewer with `t3_thread_send`.

The child runs with the thread's permissions, not a read-only sandbox, so the brief's "Review only" line is what keeps it from editing. If a reviewer changes files anyway, stop and tell the user before touching anything.

Outside T3 Code (no T3 tools listed, and a direct `orchestrator_capabilities` call fails), send the same brief to the Codex CLI. Write the brief to a file in your scratchpad, then run from the repo root with `run_in_background: true`:

```bash
codex exec -m gpt-6.1-sol -c model_reasoning_effort=xhigh -c sandbox_mode=read-only -o <out-file> - < <brief-file>
```

When notified that it exited, read `<out-file>`, which holds only the final reply. Put both files outside the repo with new names each round. Keep `sandbox_mode=read-only`: the user's Codex config defaults to full access.

### The brief

Before writing it, settle:

- **Diff:** the one command the reviewer reads, kept the same every round of a batch. Uncommitted work: `git diff HEAD -- <paths>` plus untracked files from `git status --short`. Work committed on a branch: `git diff $(git merge-base HEAD <base-branch>) -- <paths>`. A single commit: `git show <sha>`. Outside a git repo: list the files.
- **Paths:** only what this batch touched. Leave unrelated changes in the tree out of the diff; don't stash or discard them. If they can't be separated cleanly, ask the user.
- **Intent and out of scope:** 1-2 sentences from what the user asked for, not a summary of the diff. This is what lets the reviewer judge whether code is needed.
- **Checks:** the checks you ran and their results; note any that are missing or were already failing. Don't send code that fails a check this change broke.

Round 1 brief:

```
Review only. Do not edit files, commit, push, comment on PRs, or start subagents.
Repo: <abs path>
Intent: <what this change is for>
Out of scope: <what it deliberately does not do>
Checks: <commands and results>
Read: <diff command>; git status --short (read untracked files in scope).
Focus: <later batch only: what changed since the last clean review>
Read surrounding code. Find bugs, broken callers, and mismatches with the intent.
Skip style unless it hides a bug. Hold to the smallest change: flag code, tests,
or guards the intent does not need. Other paths, rare cases, small perf, and
extra compat are nits.
blocker = intended case wrong. should = real bug callers will hit, within intent. nit = the rest.
Reply:
VERDICT: APPROVE or CHANGES
C1 [blocker|should|nit] <file>:<line> <claim>. Fix: <suggestion>
```

Later rounds send the round 1 brief again, plus:

```
Ledger:
C1 fixed: <what changed>
C2 disagree: <evidence: file:line, test, or output>
C3 declined (nit): <one line>
Changed since last round: <files, or a diff command>
Check only open ids and changed code. Mark each old id resolved or
still open: <reason>. Number new findings after the last id.
```

## Handling findings

Treat each finding as a claim from a capable colleague: check it and its severity yourself, don't just obey it. A finding about another path, a rare case, or a small cost is a nit, whatever label it came with.

- Agree (blocker or should): make the smallest fix. Prefer deleting or narrowing code over adding a lock, cache, helper, option, or fallback. Fix the case that actually breaks. Add one focused test that fails without the fix, only if the intent needs it.
- Disagree: leave the code alone and record why, with evidence.
- Nit or out of scope: decline in one line and list it as a possible follow-up.
- It concerns code you did not write this session: leave it, and mention it to the user.

Keep a ledger per batch (id, severity, outcome) and note `git diff --shortstat` on the batch's diff before round 1. After fixing, rerun the checks that cover what you changed and compare the shortstat. If review grew the diff, cut before the next round. Don't agree with a finding just to end the loop.

Run another round when you fixed a blocker or should, since the fixes are new code too. A round that returned only nits needs no new round, and neither do declined nits or comment, doc, or format-only fixes. Don't reopen a finding you already settled unless the reviewer brings a new argument. If a finding is still disputed after two replies, show both sides in 2-3 lines and let the user decide.

A batch is done when the current diff has an APPROVE, or a round leaves nothing you would act on: no blocker or should open, only nits, repeats, and ones you reject. If round after round keeps surfacing new real bugs in the same area, the design is probably the problem, so step back and rework it rather than patching finding by finding.

## Reporting

When you report a batch as done, include a few lines on its review: how many rounds ran, the model and effort, what you fixed, what you declined and why, the diff size before and after review, and the state of the tests.
