---
name: consolidation-protocol
description: CLAUDE.md defines how other Claude Code sessions migrate their memory/skills/hooks into this repo
metadata:
  type: project
---

`CLAUDE.md` at this repo's root (auto-loaded by any Claude Code session working here) now documents a
consolidation protocol: when a human explicitly asks a session running in some *other* project to
consolidate/migrate/onboard into qaztronicai, that session true-moves (copies, confirms, then deletes) its
own project-local memory dir, `.claude/skills/`, and the `hooks` key of its settings into a namespaced
`wip/incoming/<project>-<date>/` folder here, with a `MANIFEST.md` describing what landed. It never touches
qaztronicai's own `memory/`, `.claude/skills/`, or `.claude/settings.json` directly, and never triggers
automatically just from reading the file — only on explicit human ask in that other session.

**Why:** user wants qaztronicai to become the durable hub for memories/skills/hooks across *all* their
projects, not just this one, but without any session unilaterally merging into or deleting from the
canonical structure — see [[repo-structure]] and [[docs-organization]] for the same "human integrates
deliberately" principle applied to `wip/` generally. Confirmed via AskUserQuestion: deletion of source
originals is a true move (not copy-only), but only human-triggered per session, never automatic.

**How to apply:** if a `wip/incoming/` folder ever appears, treat its contents as unreviewed — don't
assume they're safe to fold into `memory/`, `.claude/skills/`, or `docs/` without the human explicitly
directing that integration.
