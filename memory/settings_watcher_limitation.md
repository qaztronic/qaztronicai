---
name: settings-watcher-limitation
description: Files added under .claude/ mid-session (skills, settings.local.json) need a restart to take effect
metadata:
  type: reference
---

Claude Code's settings/skill watcher only picks up files and directories that existed under `.claude/`
when the session started. Anything added mid-session needs a session restart (not just `/hooks` dialog
open-and-dismiss) before it's actually live:

- `.claude/settings.local.json` created mid-session: permission rules inside it are not honored until
  restart, even after opening and dismissing `/hooks`. Bash calls kept prompting even though the file
  granted `"Bash"` unrestricted — confirmed during [[repo-structure]] setup.
- `.claude/skills/<name>/` created mid-session (e.g. via unzip, not through the harness's own skill
  install flow): `Skill(<name>)` fails with "Unknown skill" until restart. Confirmed when trying to invoke
  `structured-brainstorm` right after installing it in the same session.

**Why this matters:** don't waste time debugging "why isn't my permission/skill working" as a logic bug —
if it was added or edited *this session* under `.claude/`, first ask whether a restart is needed before
assuming the config itself is wrong.

**How to apply:** after writing/editing anything under `.claude/` (settings.local.json, settings.json,
skills/), tell the user a restart is required before it takes effect, rather than repeatedly retrying the
same call. Verify config correctness with static checks (jq, reading the file) instead of relying on a
live invocation to prove it's right.
