---
name: docs-organization
description: docs/ is organized into per-topic subfolders, mirroring .claude/skills/ for skill docs
metadata:
  type: feedback
---

`docs/` is never flat — every subject area gets its own top-level subfolder, and within `docs/skills/`
specifically, each skill gets its own folder mirroring `.claude/skills/<name>/` 1:1
([[repo-structure]]). Current layout:

```
docs/
  skills/
    SKILLS-GUIDE.md              (covers all 6 skills, stays at this level — doesn't belong to one)
    <skill-name>/methods.md      (one per skill, filename dropped its redundant skill-name prefix
                                   once nested — fact-falsifier/methods.md, not fact-falsifier/
                                   fact-falsifier-methods.md)
  browser-agent/
    implementation-plan.md
```

**Why:** user explicitly asked for docs/ to be organized into subfolders keeping related docs together,
after it had accumulated 8 files flat. A flat by-type layout (guides/, methods/, plans/) was considered and
rejected — it mixes unrelated subject areas together as docs/ grows past one topic. Per-skill folders were
chosen over a flat docs/skills/ bucket because they scale without a second reorg once any one skill
accumulates more than one doc (changelog, design notes, etc.).

**How to apply:** when adding new documentation to this repo, place it under an existing topic folder if
one fits, or create a new top-level `docs/<topic>/` folder for a genuinely new subject area — never add a
file directly under bare `docs/`. When a skill in `.claude/skills/<name>/` gains supporting docs, they go
in `docs/skills/<name>/`, matching the skill's folder name exactly.
