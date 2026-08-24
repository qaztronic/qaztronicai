---
name: verbose-session-logging
description: Maintain a verbose, incremental activity log in logs/ for every session in this repo
metadata:
  type: feedback
---

Keep a running log of everything done in this repo, verbose enough that a human could manually recreate
the work from it alone — not a summary, but the actual commands run, files written (with content or a
precise diff), and decisions made and why.

- Log lives in `logs/`, one file per calendar day (`logs/YYYY-MM-DD.md`), sessions on the same day append
  to the same file with a new dated/numbered section rather than creating a new file.
- `logs/` is entirely gitignored ([[repo-structure]] — added as a whole-folder entry in `.gitignore`, unlike
  `wip/` which keeps its own `.gitignore` tracked). Nothing in it is ever committed.
- Write log entries **as you go** — after each meaningful step or commit, not batched at the end of the
  session. If the session ends abruptly, the log up to that point should still be accurate and complete.
- Each entry should include: what was asked, what was decided (and the reasoning/tradeoff if a choice was
  made), the exact commands run in order, and the exact file contents written or changed.

**Why:** user wants a durable, human-readable record of every change made in this repo, detailed enough to
reproduce manually — separate from git history, which only shows the end state, not the reasoning or the
exact sequence of exploratory steps.

**How to apply:** in this repo (qaztronicai), write a `logs/` entry for any nontrivial action — file
scaffolding, permission changes, installing/removing things, committing. Trivial or purely conversational
turns (answering a question, no file/state change) don't need an entry. This is standing behavior for this
repo, not a one-time request — don't wait for the user to ask again each session.
