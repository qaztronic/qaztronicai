# qaztronicai

Launchpad repo. Durable Claude Code state lives here, versioned, instead of scattered in global config.

- `memory/` — the sole copy of long-term memory. `~/.claude/projects/-home-qaz-qaztronicai/memory` is a
  symlink into this directory, so the auto-memory system reads and writes here directly. Nothing to sync.
- `.claude/skills/` — project skills.
- `.claude/settings.json` — hooks and other project config.
- `wip/` — intake for anything new: drafts, half-formed skills, scratch plans. Everything inside is
  gitignored except `.gitignore` itself, so it's disposable by default. Work that matures graduates out of
  `wip/` into `.claude/skills/`, `memory/`, or wherever it belongs — it doesn't stay here.
