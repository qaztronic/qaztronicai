---
name: open-items
description: Outstanding to-dos in qaztronicai not yet resolved
metadata:
  type: project
---

- **Unrestricted Bash permission grant — restored twice now, watch for a third silent rewrite.** User chose
  "leave it open" (unrestricted `Bash` in `.claude/settings.local.json`). The harness has now silently
  rewritten that file mid-session at least once, replacing the broad `"Bash"` rule with narrow per-command
  entries — restored on 2026-08-24 back to `Read`/`Edit`/`Write` scoped to `/home/qaz/qaztronicai/**` plus
  bare `"Bash"`.
  **Why it matters:** this has already regressed once without any user action causing it — likely an
  automatic behavior of the permission-approval flow itself (each approved prompt gets appended as its own
  narrow rule), not a one-off bug.
  **How to apply:** if Bash commands start prompting again despite this grant, check
  `.claude/settings.local.json` for a bare `"Bash"` allow rule first before assuming something else broke;
  restore it the same way (overwrite with the broad grant) without waiting to be asked, since the user has
  now stated this preference twice.
