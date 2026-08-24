---
name: repo-structure
description: How qaztronicai repo is organized — memory symlink, wip intake, skills/hooks locations
metadata:
  type: project
---

qaztronicai is the launchpad repo: durable Claude Code state (memory, skills, hooks) is versioned here
instead of living only in global (~/.claude) config.

- `memory/` in this repo is the sole physical copy of long-term memory. The global auto-memory path
  `~/.claude/projects/-home-qaz-qaztronicai/memory` was replaced with a symlink pointing here — the harness
  reads/writes through the link transparently, so there is no separate global copy to keep in sync.
- `.claude/skills/` holds project skills; `.claude/settings.json` holds hooks and project config.
- `wip/` is intake for anything new — draft skills, scratch plans, half-formed ideas. Its `.gitignore`
  ignores everything except itself, so contents are disposable by default and untracked.

**Why:** user wants one durable home base that persists across projects/sessions, not scattered global
state, with a clear disposable staging area separate from the tracked, permanent structure.

**How to apply:** when work started in `wip/` matures (e.g. a `.skill` draft becomes real), move/promote it
out into `.claude/skills/` (or wherever it belongs) rather than leaving it gitignored — `wip/` is not meant
to hold anything permanent. When saving new memories in this project, they land in `memory/` here, not
anywhere else.
