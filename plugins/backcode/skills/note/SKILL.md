---
name: note
description: Create notes in a Backcode project's Notes tab from Claude Code - one note, or a batch (e.g. migrating notes from a spreadsheet, Notion, a doc or a README). Use when the user asks to save, write down or keep something as a Backcode note ("apúntalo como nota", "guárdalo en las notas", "save this as a note"), to import or migrate notes into Backcode, or runs /backcode:note. Never create notes on your own initiative.
---

# Notes in a Backcode project

The runner (`backcode note create`) creates real notes in the project's **Notes** tab, under the user's name, as if they
had written them in the app. Use it instead of driving the Backcode web app with a browser: it's faster and it doesn't break.

Talk to the user in their language. Be brief.

## 0. Runner and project

Find the runner: `command -v backcode || ls ~/.backcode/bin/backcode` (use whichever exists as `backcode` below). If
there's none, offer `/backcode:setup`.

The notes go to the project of the repo you're in (or one of its worktrees). From any other folder, add `--project <id>`
(`backcode status` lists the connected projects and their ids). If it says the repo isn't connected, explain it in one
line and offer `/backcode:setup`.

## 1. One note

Only when the user asks for it. Write the text to a temporary file (outside the repo) and run:

```
backcode note create --title "<short title>" --body-file <file> --json
```

Write it so it reads well on its own in the app, without this conversation: what it is, the conclusion and the context
that matters (commands, links, file paths). Markdown is fine. Title up to 200 characters, text up to 20,000.

## 2. A batch (import or migration)

1. Read the source (an exported CSV, a spreadsheet via a connector, Notion, files…) and build a JSON file outside the repo:

   ```json
   [
     { "title": "…", "text": "…", "id": "row-12" }
   ]
   ```

   `id` is the note's id in its source (the row, the page id…). **Always set it in a migration**: running the same file
   again then skips the notes that already exist instead of duplicating them, so a migration that stops halfway can just
   be run again. Keep the source's content: don't summarize or rewrite it unless the user asks. If a source item has no
   title, use its first line (up to 200 characters).
2. Check it first, without creating anything: `backcode note create --file <file> --dry-run`. Tell the user how many notes
   it will create, with a few titles as examples, and ask before creating them (it's their team's Notes tab).
3. Create them: `backcode note create --file <file> --json`. It prints how many were `created`, how many already existed
   (`exists`) and how many `failed`. Any number of notes works: the runner sends them in batches of 100.
4. If it fails, read the message and fix the cause. Before creating anything it validates the whole file and looks for
   secrets: an invalid note (empty title, too long…) or a secret (an API key, a token…) stops the whole run, and nothing
   is created. Never put secrets in notes.

## Rules

- Only notes the user asked for. If something might be worth keeping (a diagnosis, a decision), you can offer it in one
  line; don't create it without a yes.
- Don't read `~/.backcode/config/` (it holds keys and tokens); `backcode` does everything that needs them.
- Notes are for knowledge. Work to do is a task (`/backcode:feature`).
