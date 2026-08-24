---
name: open-items
description: Outstanding to-dos in qaztronicai not yet resolved
metadata:
  type: project
---

- **Unrestricted Bash permission grant lost.** User explicitly chose "leave it open" (unrestricted `Bash`
  in `.claude/settings.local.json`, see [[settings-watcher-limitation]]) but the harness silently rewrote
  that file mid-session to hold only narrow, per-command allow rules instead (confirmed again as of
  2026-08-24: file now has 9 exact-command entries, no bare `"Bash"` rule). User was told about this once;
  not yet fixed.
  **Why it matters:** the user's actual stated preference (no Bash prompts at all, ever) isn't currently
  in effect — every new command still needs individual approval.
  **How to apply:** next session, check `.claude/settings.local.json` for a bare `"Bash"` allow rule. If
  it's still missing, ask the user whether to restore it (rewrite the file back to the broad grant) or
  whether they've since decided the narrow per-command list is actually fine.
