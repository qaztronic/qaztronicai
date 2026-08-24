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

- **`gemini_web_search` blocked by an account-level quota, `gemini_web_fetch` works fine.** Built during
  wip/gemini-web-mcp Phase 1 implementation (2026-08-24). The `google_search` grounding tool on the
  supplied Gemini API key returns `429 RESOURCE_EXHAUSTED` on every attempt (confirmed 2x, ~20s apart, not
  a transient per-minute limit) while `url_context` (used by `gemini_web_fetch`) and plain
  `generate_content` (no tools) both work fine on the same key. This isolates the problem to the Search
  grounding tool's own quota specifically — commonly requires a billing-enabled Google Cloud/AI Studio
  project even when the base API key otherwise works.
  **Why it matters:** blocks Phase 1's Definition of Done, which needs both tools working together in one
  turn (see `wip/gemini-phase1-implementation-plan.md` step 6).
  **How to apply:** needs the user's action (enable billing on the key's project, check
  https://ai.dev/rate-limit for the specific quota, or supply a different key) — not fixable by more code.
  Don't keep retrying against this quota; check with the user for an updated key/billing status first.
