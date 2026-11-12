---
name: feature
description: Do a new feature or fix the Backcode way, in a repo connected to Backcode - its own git worktree and branch from the up-to-date main branch, then a pull request registered as a task on the Backcode board (review, merge and deploy from the app). Use when the user asks to build, add, change or fix something in a Backcode project, when a Backcode session-start note says the project follows these guidelines, or when they run /backcode:feature.
---

# A feature or fix, the Backcode way

Backcode projects can ask that work done with Claude Code outside Backcode follows the same rules as its tasks: **one
feature or fix = one worktree + one branch + one PR**, which then shows up on the team's Backcode board, where it's
reviewed, merged and deployed. The runner (`backcode`) does the git plumbing; you do the work in between.

Talk to the user in their language. Be brief.

## 0. Is this repo connected?

Find the runner: `command -v backcode || ls ~/.backcode/bin/backcode` (use whichever exists as `backcode` below).
If there's none, or `backcode task start` says the repo isn't connected, explain it in one line and offer
`/backcode:setup`. Don't fall back to committing on the main branch.

If you are **already in a worktree on a `backcode/…` branch** (`git branch --show-current`), this is the continuation of
that work: run `backcode task resume --json` there first, then skip to step 2 and keep everything there. That's also
the case when the user hands you a task **the runner did** ("continue here on my computer a Backcode task…", a failed
or waiting task, its branch and worktree): work on that branch, never start a new task or open another PR. `task resume`
puts the task back to **In progress** under the user's name and stops the runner from picking it up again; it's
harmless to repeat. If it refuses because a runner is still running or has it queued, tell the user to stop it from
the app first.

## 1. Start: worktree and branch

1. One topic per branch. If the request mixes unrelated things, say so and do them one at a time (one PR each).
2. Run `backcode task start "<short title>" --json` from the repo (the title is required: it's the task's name on the
   board). It fetches the remote, creates a worktree from `<remote>/<base>` on a branch `backcode/<handle>/<id>`, copies
   the local `.env` files, puts the task on the Backcode board already **In progress** (marked as made in Claude Code),
   and prints `{ "path", "branch", "base", "featureId" }`.
   If the project has a **preview branch** (e.g. `dev`), the task starts from it and its PR targets it, like the
   runner's tasks. **Hotfix:** when the user asks to fix something that is already **in production** (a crash or bug in
   the live app, web or backend that can't wait for the next release), add `--hotfix`: the task starts from the main
   branch and its PR targets it, and the output includes `"hotfix": true`. Say so to the user, and tell them that auto
   mode doesn't merge hotfixes and that after merging and deploying it they should use **Sync** (Publish) to bring the
   fix into the preview branch. Without a preview branch every task already starts from the main branch: no `--hotfix`.
3. Work **only inside that `path`** from now on: `cd` into it for every command, and read/edit files with that absolute
   path. The original checkout stays as it was. If the project needs its dependencies installed, install them there.

## 2. Do the work

- Focused, minimal changes that follow the repo's conventions (its CLAUDE.md and existing code). No unrelated refactors.
- Run the project's tests, linter and type checker for what you touched, and fix what broke. Add tests when the repo
  has a test suite.
- Never commit secrets, credentials or `.env` files.
- **Research** (an audit, an investigation, a long-term plan the user asks you to write up): a Markdown document in the
  project's research folder, `docs/research/` unless `backcode.toml` sets another one in `[research] dir`. One file per
  topic (`YYYY-MM-DD-<slug>.md`) with a short frontmatter (`title`, `status: in-progress` or `done`, `commit` of the base
  you read), in its own task and PR with no code changes. Write it as you go, not only at the end. Once merged, the team
  reads it in the Backcode app (Notes › Research) and can turn it into tasks.
- Commit in the worktree with clear messages (`git add -A && git commit -m "…"` inside `path`). Committing there is fine:
  it's the task branch, not the main branch.

## 3. Open the PR (registered in Backcode)

1. Write the PR body to a temporary file (not inside the worktree), in the user's language, with these sections:
   **Summary**, **Changes**, **How it was tested**, **Risks / follow-ups**. If something must be done by hand after
   merging (deploy rules or functions, a migration, an env var…), list each step with why and the exact command. Never
   run deploys yourself.
   Backcode already detects Cloudflare Workers (wrangler config and D1 migrations), Supabase (migrations and edge
   functions) and Firebase (rules and functions). If the change needs any other deployment and the repo has no
   deploy for it, add one in this PR, calling the project's existing deploy command (lowercase name with dashes; a
   database migration ending in `-db` / `kind = "migration"`; a deploy that publishes an Expo OTA with `kind = "ota"` and `channel = "preview"` or `"production"`; no secrets): in a JavaScript repo, a
   `backcode:deploy:<name>` script in that part's `package.json`; in a repo without JavaScript, a `[deploy.<name>]`
   entry with `run = "…"` in a `backcode.toml`. So the team can deploy it from the app. Deploys run in a clean copy of
   the commit: if one needs git-ignored local state (a download cache, a previous step's output), list those paths in
   the entry's `shared = ["…"]` so the runner links them from the user's checkout instead of starting from scratch
   (they must exist in the checkout of the computer that deploys, or the deploy won't start: say so in the PR).
2. From the worktree: `backcode task pr --title "<imperative title>" --body-file <file> --json`.
   It checks there are no uncommitted changes and no secrets, pushes the branch, opens (or updates) the PR and moves the
   task to "PR ready" on the Backcode board, under the user's name. It prints `{ "prUrl" }`. From there it moves by
   itself: Merged when the PR is merged, Deployed when an OTA (from the app or the terminal) includes it, with push
   notifications to the team. If the project has auto mode on, `task pr` says so and the user's runner merges it on its
   own once CI is green: tell the user, and don't merge it yourself.
   Also for a task the runner did: it reuses the branch's PR if one is open and moves the task to "PR ready". Never
   open the PR with `gh pr create` or by hand: then Backcode doesn't know about it and it doesn't show up in the app.
3. If it fails, read the message, fix the cause (commit pending changes, remove a secret…) and run it again.
4. Tell the user the PR link and that it's on the Backcode board, where it's reviewed, merged and deployed.

## Later changes

To change the same PR (review comments, a follow-up): go back to that worktree, run `backcode task resume`, commit, and
run `backcode task pr` again with the same or a better title. It updates the same PR and task. A different topic is a
new `backcode task start`.

## Rules

- Never commit or push directly to the main branch for this work. If the project requires it, a direct commit or push
  there asks the user first; don't try to get around it.
- Every PR targets the main (base) branch, never another feature branch: `backcode task pr` already does it, so don't open
  PRs by hand with `gh pr create --base <other branch>`. Repos usually delete a PR's branch when it's merged, and GitHub
  then closes any PR stacked on it. If a change depends on another unmerged PR, put both in the same PR or wait until the
  first is merged and start the second from the updated main branch.
- Don't read `~/.backcode/config/` (it holds keys and tokens); `backcode` does everything that needs them.
- Don't delete the worktree when you finish: the user may need it for changes. The runner cleans old ones up.
