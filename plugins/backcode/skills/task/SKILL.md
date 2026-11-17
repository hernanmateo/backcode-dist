---
name: task
description: Create a task on a Backcode project's board from Claude Code for a runner to do later - a normal task, a research task, or a Sleepy task (runs only on leftover plan quota), with the model, effort and plan mode the user asks for, assigned to them or left for auto-assign. Use when the user asks to create, queue or leave a Backcode task ("créame una tarea Sleepy de…", "crea una tarea en Backcode para…", "déjalo como tarea para el runner", "queue a task", "make a research task about…"), or runs /backcode:task. Not for doing the work now in this session (that's /backcode:feature). Never create tasks on your own initiative.
---

# Board tasks for a runner

`backcode task create` puts a real task on the project's Backcode board, as if the user had created it in the app.
**A runner does it, not this session**: it runs later in its own branch and ends in a pull request. If the user wants
the work done now, here, use `/backcode:feature` (`backcode task start`) instead.

Talk to the user in their language. Be brief.

## 0. Runner and project

Find the runner: `command -v backcode || ls ~/.backcode/bin/backcode` (use whichever exists as `backcode` below). If
there's none, offer `/backcode:setup`.

The task goes to the project of the repo you're in (or one of its worktrees). From any other folder, add
`--project <id>` (`backcode status` lists the connected projects and their ids). If it says the repo isn't connected,
explain it in one line and offer `/backcode:setup`. If it says `createLocalTask` doesn't exist (not-found), Backcode's
server is older than the runner: tell the user, and offer to create it from the runner chat in the app instead.

## 1. Write the brief

Only when the user asks for it. The task **does not see this conversation**, so the brief must be self-contained: what to
do and why, the scope, relevant files or folders, constraints, what "done" looks like, and any context you already found
(read the code first if it helps you write a precise brief). Write it to a temporary file outside the repo. Up to 8,000
characters (32,000 for research). Optionally, acceptance criteria in another file, one per line.

## 2. Create it

```
backcode task create "<short title>" --prompt-file <file> [--criteria-file <file>] \
  --suggested-model <model> --suggested-effort <effort> --model-reason "<one sentence>" [options] --json
```

**Always** recommend a model and an effort, like Backcode's own task refiner does (it shows in the app as "AI recommends
…", and it's what runs the task when the user's profile is set to "Recommended"):

- `--suggested-model`: the cheapest Claude model that will do the task well, by complexity: `haiku` for trivial,
  mechanical changes (copy, a style tweak, a config value in one file); `sonnet` for most features or fixes touching a
  few files, with tests; `opus` for hard or broad work (architecture changes, tricky bugs, many modules,
  security-sensitive logic, migrations); `fable` only for the hardest, longest tasks (large refactors or new subsystems
  across the codebase). When unsure between two, pick the cheaper one.
- `--suggested-effort`: the reasoning effort, by how hard the task is to get right (not how big it is): `low` for
  trivial, mechanical changes; `medium` for contained, well-specified changes; `high` for most features and fixes that
  need care (the usual choice); `xhigh` for tricky logic, subtle bugs, concurrency, security-sensitive or long multi-step
  work; `max` only when correctness matters far more than time and cost. When unsure between two, pick the lower one.
- `--model-reason`: one short sentence, in the user's language, explaining the choice.

This is a recommendation, not the user's choice: don't turn it into `--model`/`--effort`. Opus or Fable recommended
also makes the task plan first (unless `--no-plan-first`).

Options, only what the user asked for (or clearly implied):

- `--sleepy`: **Sleepy** task. It only starts with plan quota that would otherwise be lost (the last hours of the weekly
  window, or at night when weekly usage is low), pauses and resumes by itself, and can take days. For big, non-urgent work
  ("audit the whole code", "migrate everything to X").
- `--research`: the result is a Markdown document in the project's research folder (an audit, an investigation, a
  plan), not code changes. Combines with `--sleepy`.
- `--model haiku|sonnet|opus|fable` and `--effort low|medium|high|xhigh|max`: only when the user names them; they win
  over everything. Without them, the user's profile in Backcode decides: "Recommended" uses your recommendation, a fixed
  model uses that one, "Default" uses the default of the Claude Code that runs it.
- `--plan-first` / `--no-plan-first`: Claude proposes a plan and waits for the user's approval in the app before
  touching code. Opus and Fable tasks plan first by default.
- `--unassigned`: leave it unassigned in the backlog so the project's **auto-assign** hands it out (a Sleepy one goes to
  whoever has leftover quota). Without it, it's assigned to the user and queued on their runner.

It prints `{ "featureId", "assigned", "autoAssign", ... }`. Tell the user in one or two lines: the title, where it went
(their runner's queue, or the backlog for auto-assign), and that they follow it in the Backcode app. If they asked for
`--unassigned` and `autoAssign` is false, say auto-assign is off for this project, so they'll need to assign it in the app.

## 3. Several tasks

For a big piece of work the user wants split, create one task per independent part, each with its own self-contained
brief. Up to 20 open tasks created from Claude Code per person.

## Rules

- Never create tasks on your own initiative: only when the user asks.
- Never put secrets, tokens or `.env` contents in a brief.
- Don't read `~/.backcode/config/` (it holds keys and tokens); `backcode` does everything that needs them.
