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

- **RESOLVED BY ABANDONMENT (2026-08-25): Gemini removed from the plan entirely, not pursued further.**
  `gemini_web_search` was blocked by a confirmed root cause (Free-tier project, no billing, Search grounding
  likely not entitled at all — see the dashboard screenshot analysis in `wip/gemini-phase1-implementation-plan.md`,
  now marked `STATUS: RETIRED` at the top). Rather than enable billing, the user decided Gemini is "useless
  for our needs" and to drop it entirely — there's no replacement provider being substituted in, since
  native Claude Code `WebSearch`/`WebFetch` already covers the need this phase existed to serve.
  **Still open, not yet decided:** whether to delete `wip/gemini-web-mcp/` (the built server/rate-limiter/
  regression-suite code) or leave it in place as unused-but-working infrastructure. Flagged to the user, not
  yet answered as of this entry. Also open: whether the broader parent doc
  `wip/browser-agent-implementation-plan.md` (which references Gemini in several other phases — provider
  abstraction, vision fallback) should also be edited, or whether "remove gemini from the plan" was scoped
  to just the Phase 1 document. Not assumed either way — ask if it comes up again.
