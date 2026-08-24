# qaztronicai — instructions for any Claude Code session

This repo is the **canonical, durable storage location** for this user's Claude Code memories, skills, and
hooks — not just for work done in this directory, but the intended hub across all of their projects.
Structure (see `README.md` for the full rationale):

- `memory/` — the sole copy of long-term memory (the global auto-memory path for this repo is symlinked here)
- `.claude/skills/` — project skills
- `.claude/settings.json` / `.claude/settings.local.json` — hooks and permission config
- `docs/` — per-topic subfolders only, never files directly under bare `docs/`
- `wip/` — gitignored intake for anything new or unreviewed
- `logs/` — gitignored, verbose as-you-go session log (see `memory/verbose_session_logging.md`)

## Consolidating another project's memory/skills/hooks into this repo

**Do this only when a human explicitly asks the current session to consolidate, migrate, or onboard into
qaztronicai.** Reading this file is not itself a trigger — never do this just because a session happened to
load these instructions (e.g. by being pointed at this repo for an unrelated reason).

When explicitly asked, the session runs this from **its own current project** (not from inside qaztronicai)
and treats *that* project's own memories/skills/hooks as the source to relocate:

1. **Identify source artifacts** in the current project:
   - Its project-local auto-memory directory (`~/.claude/projects/<sanitized-cwd>/memory/`)
   - Its project-local `.claude/skills/`
   - The `hooks` key (if any) in its project-local `.claude/settings.json` and `.claude/settings.local.json`

2. **Stage everything found**, unmodified, into a new namespaced folder under this repo:
   `wip/incoming/<source-project-dirname>-<YYYY-MM-DD>/{memory,skills,hooks}/` — create only the
   subfolders that actually have content. Copy files in as-is (no reformatting, no merging, no renaming).

3. **Write a `MANIFEST.md`** in that staging folder recording: the source project's path, the date, exactly
   what was found in each category, and an explicit note that this is **awaiting human review — do not
   auto-merge into qaztronicai's own `memory/`, `.claude/skills/`, or `.claude/settings.json`.**

4. **Show the human a clear summary** of exactly what was found and what is about to be deleted from the
   source project, and get explicit confirmation in that session before deleting anything — even though the
   consolidation itself was explicitly requested, deletion is a separate, harder-to-reverse step and needs
   its own go-ahead.

5. **Only after confirmation, delete the originals from the source project** (this is a true move, not a
   copy, per explicit instruction):
   - Delete the project-local memory directory entirely.
   - Delete the project-local `.claude/skills/` directory entirely.
   - For hooks: remove only the `hooks` key from `.claude/settings.json` / `.claude/settings.local.json` —
     never delete the whole settings file, since other config (permissions, env, etc.) may live there and
     is out of scope for this consolidation.

6. **Never touch qaztronicai's own `memory/`, `.claude/skills/`, or `.claude/settings.json` during this
   process** — the destination is always the new `wip/incoming/...` staging folder, nothing else. Folding
   staged content into the canonical structure is a separate, human-directed step done later.

7. **"Forget" going forward**: once the move is done, stop treating the deleted local memory/skills/hooks as
   available or authoritative for the rest of that session — they're gone from the source project by design,
   staged in qaztronicai for a human to integrate on their own schedule.
