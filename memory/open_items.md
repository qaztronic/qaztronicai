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

- **SHELVED (2026-08-25): Phase 1 (web-delegation MCP server) deferred entirely — not Gemini, not Ollama,
  paused at the search-backend decision point.** Full sequence: Gemini approach retired (confirmed root
  cause: Free-tier project, no billing, Search grounding likely not entitled) → user explicitly deleted the
  built code (`wip/gemini-web-mcp/`, and its `claude mcp remove gemini-web` registration) → redo attempted
  for a local Ollama model instead → 7 candidate models researched and ranked (Qwen3 8B best fit; full
  ranking preserved in `wip/gemini-phase1-implementation-plan.md`'s STATUS section) → hit a real
  architectural gap (Ollama models have no built-in web search, unlike Gemini — something has to actually
  perform searches: self-hosted SearXNG vs. a paid API vs. URL-fetch-only) → user chose to shelve the whole
  phase at that decision point rather than choose, with "save memory, write logs" as the explicit next step.
  **Why it matters:** this is a pause, not a rejection — the Ollama model research took real effort and is
  time-sensitive; don't redo it from scratch if this resumes, but do re-verify current model names/rankings
  given the model landscape moves fast.
  **How to apply:** if the user brings this up again, start from `wip/gemini-phase1-implementation-plan.md`'s
  STATUS section (has the full timeline and the ranked model list) rather than re-researching. The open
  decision to resume at is the search backend. Also still genuinely unresolved: whether the broader parent
  doc `wip/browser-agent-implementation-plan.md` (references Gemini in other phases — provider abstraction,
  vision fallback) should be touched — not assumed either way.
