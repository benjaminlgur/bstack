---
name: council
description: Convenes a two-member council (Claude Opus 5.5 and GPT Sol 6.1, both at xhigh reasoning) on the user's request, usually a review or feedback, then fixes what you agree with and re-reviews until both members are satisfied. Invoke ONLY when the user types `/council`. Never invoke it for a general review request or on your own initiative.
argument-hint: "[what the council should review or weigh in on]"
disable-model-invocation: true
---

# Council

Request, if any: $ARGUMENTS

If the request is empty, the council takes up whatever the user's message or the current task is about.

Before an audit, address outstanding PR comments, fix known issues, and run
relevant checks. Give the council the updated implementation; do not defer known
fixes until reviews return. If the user wants to discard the implementation
and explore new ideas, skip fixes to code being replaced.

## Members

The council has exactly two members. Start both in parallel for every round:

| Member | Model | Reasoning effort |
| --- | --- | --- |
| Claude Opus 5.5 | `claude-opus-5-5` | `xhigh` |
| GPT Sol 6.1 | `gpt-6.1-sol` | `xhigh` |

If a member is unavailable, or only at a different model or effort, report it
instead of substituting, and do not run a one-member council.

## Starting the members

**In T3 Code** (the `delegate_task` tool exists): call `orchestrator_capabilities`
once, then call `delegate_task` once per member in the same turn, with
`role: "review"` for a review:

- Opus: `target: {providerInstanceId: "claudeAgent", model: "claude-opus-5-5", options: {effort: "xhigh"}}`
- Sol: `target: {providerInstanceId: "codex", model: "gpt-6.1-sol", options: {reasoningEffort: "xhigh"}}`

If `orchestrator_capabilities` lists different ids or option names, use what it
lists; if it lacks either model or `xhigh` for it, report that. Each new round is
a new `delegate_task`, not a message to an earlier delegate.

**In Claude Code outside T3:** run both members as CLI processes, each in its
own Bash call with `run_in_background: true` (an xhigh run can outlast the
10-minute foreground limit), both from the project root:

```bash
codex exec -m gpt-6.1-sol -c model_reasoning_effort=xhigh -s read-only --skip-git-repo-check -C <project-root> -o <sol-out-file> "Read the council brief at <brief-file> and follow it."
claude -p "Read the council brief at <brief-file> and follow it." --model claude-opus-5-5 --effort xhigh --permission-mode plan --add-dir <brief-dir> > <opus-out-file>
```

- Use `claude -p` for Opus, not the Agent tool: the Agent tool's `opus` alias
  runs on this session's own model whenever that is any Opus, so it cannot pin
  Opus 5.5.
- Keep the prompt directly after `-p`. `--add-dir` takes several values and
  swallows a prompt placed after it.
- When you are notified that both exited, read the two output files.
  `<sol-out-file>` holds only Sol's final answer. If a run failed, show the user
  its error (not logged in, model unavailable, usage limit) instead of switching
  models.

Put the brief and output files outside the project (your scratchpad directory,
else the system temp directory) and use new names each round, so they never show
up in a member's diff.

## Briefing the members

Members start with none of this conversation. Give each the same brief: the
user's request; the context they need (project path, what changed and why, how to
see it, such as uncommitted changes, a base branch, a commit, or file paths; the
checks you ran and their results; known trade-offs); and what to return.

For review or feedback, tell them to stay read-only: no edits, commits, pushes,
or PR comments. The CLI runs enforce this (`-s read-only` for Sol,
`--permission-mode plan` for Opus); T3 delegates rely on the instruction.

For a code review, ask each member for a score out of 10 along with its findings.

## Rounds

Collect both responses and summarize the useful findings, disagreements, and
recommended next steps.

Count each full review pass as one round. A fresh review after fixes starts a
new round.

Treat findings as claims to check, not orders. If you agree with a
recommendation, make the change; if you decline one, say why. After making
changes, rerun the relevant checks and start a fresh round, giving the members
your reasons for anything you declined. Continue until both members are
satisfied with the outcome.

Do not let repeated reviews result in scope creep. It is your job to fight to
keep scope limited.

## Reporting

If the work ends in a pull request, put the results in the PR description.
Put this section before the model and harness line at the end:

```markdown
## Council review

Rounds: 2

| Delegate | Round 1 | Round 2 |
| --- | --- | --- |
| Claude Opus 5.5 | 7/10 | 9/10 |
| GPT Sol 6.1 | 8/10 | 10/10 |
```

If a delegate gives no score, write `no score`. Do not invent one.
If the PR is open, edit the description. Do not post a comment for this.
If there is no PR, put the same results in the final reply only.
