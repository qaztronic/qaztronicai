# Implementation Plan: Hybrid DOM-First + Vision-Fallback Browser Agent
### Playwright MCP + Gemini, with vision as a fallback layer

---

## 1. Goal & Architecture Summary

Build a browser automation agent where:
- **Gemini** is the reasoning/planning engine (decides what action to take next)
- **Playwright MCP server** is the execution layer (actually drives the browser)
- **Default mode: DOM/accessibility-tree snapshots** (fast, cheap, deterministic)
- **Fallback mode: screenshot + coordinate clicking** (only when the accessibility tree can't resolve an element — canvas, custom widgets, maps, charts, obscured DOM)

This mirrors Playwright MCP's own built-in design: it ships with two operating modes, and the maintainers explicitly recommend snapshot-first with vision as an escape hatch, not a default.

---

## 2. Brainstormed Methods (considered, then narrowed down)

| # | Method | Verdict |
|---|--------|---------|
| A | Pure vision-only agent (screenshot → coordinates every step) | Rejected as primary — too slow/expensive/nondeterministic for routine flows |
| B | Pure DOM/accessibility-only agent | Rejected as sole method — fails on canvas, maps, custom drag-and-drop, some SPA widgets |
| C | Playwright MCP in **snapshot mode** (default) | **Adopted — primary path** |
| D | Playwright MCP in **vision mode** (`--caps=vision` or `--vision`) | **Adopted — fallback path only** |
| E | Google's dedicated Gemini Computer Use model (screen-native, not general Gemini) | Considered as an alternative fallback engine instead of general Gemini + screenshots — worth a spike, see Phase 5 |
| F | WebMCP (page exposes its own agent tools) | Too early — Chrome Canary preview only as of early 2026, most target sites won't support it. Revisit later |
| G | Fully custom CDP-level control (bypass Playwright) | Rejected — reinvents what Playwright MCP already solved; only worth it at extreme scale |

**Decision:** Build on C, with D wired in as an automatic fallback, and E kept as a Phase 5 experiment.

---

## 3. Fact-Checked Technical Foundations

These are the specific, verifiable mechanics the plan depends on:

- Playwright MCP's **default behavior reads the accessibility tree** and returns structured data (roles, labels, refs) — no screenshots or vision model required for this path.
- **Vision mode is opt-in**, enabled via the `--vision` flag (legacy) or the newer `--caps=vision` capability flag, and unlocks coordinate-based tools like `browser_mouse_click_xy` and `browser_mouse_move_xy`.
- Microsoft's own guidance: *"For most web applications, the default snapshot-based approach is more reliable and token-efficient. Use vision mode only when the accessibility tree doesn't cover your use case."*
- Capabilities are mixable — you can run with `--caps=vision,pdf,devtools` to combine accessibility snapshots, coordinate clicking, PDF tools, and DevTools/network inspection in one server instance.
- A concrete fallback pattern already documented by Playwright: agent calls `browser_snapshot`; if an element (e.g., an icon button) has no accessible name, it calls `browser_take_screenshot`, visually locates the element, then calls `browser_mouse_click_xy` with estimated coordinates.
- Production security guidance for the MCP server: run headless by default, pin browser binary versions, restrict `allowedOrigins`, isolate `userDataDir`, and never expose the MCP port publicly without auth (reverse proxy + basic auth or Cloudflare Access).
- Vision mode carries a real cost: screenshot capture and processing are slower, and coordinate-based interaction breaks more easily on layout shifts, resizes, or reflow — reinforcing why it should stay a fallback, not the default.
- **Gemini's built-in Grounding with Google Search tool** (`google_search`) lets the model decide on its own when a query needs live web results, run the search, and return cited sources — no custom search plumbing needed for that use case alone.
- **Gemini's built-in URL context tool** (`url_context`) lets the model fetch and reason over the full content of a specific page (including PDFs, images, and several text formats) just by including the URL in the prompt — this is page-reading, not general scraping (no JS-heavy SPA rendering, no login walls, no multi-step crawling).
- The two combine: with both tools enabled, Gemini can search to discover relevant pages, then use URL context to deep-read each one — useful for research-style tasks, but still not a substitute for the Playwright-based agent when the task requires clicking through forms, logins, or JS-rendered flows.

---

## 4. Phased Build Plan

### Phase 0 — Environment setup
- Install Node.js 18+, Playwright browser binaries (pinned version), and `@playwright/mcp`.
- Start the MCP server in **snapshot mode only** first, headless:
  ```
  npx @playwright/mcp@latest --headless --isolated
  ```
- Get a Gemini API key; confirm function-calling / tool-use works against a trivial tool first (sanity check before wiring browser tools).

### Phase 1 — Core DOM-first loop
- Wire Gemini's function-calling to the MCP server's core tools: `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, `browser_press_key`, etc.
- Loop: send task + current accessibility snapshot → Gemini picks next action → execute via MCP → re-snapshot → repeat until goal-complete or max-steps.
- Add basic guardrails: max step count, per-step timeout, action allowlist (no arbitrary `eval`/navigation to disallowed domains via `allowedOrigins`).

### Phase 2 — Vision fallback trigger
- Detect fallback conditions programmatically, not just via model judgment:
  - Snapshot returns an element with no accessible name/role for something the task clearly needs (e.g., an icon-only button).
  - N consecutive failed action attempts on the same step.
  - Task explicitly involves canvas/chart/map/image-heavy content.
- On trigger: re-launch or reconfigure the MCP connection with vision enabled (`--caps=vision`), call `browser_take_screenshot`, let Gemini return coordinates, execute via `browser_mouse_click_xy` / `browser_mouse_move_xy`.
- After a successful vision-assisted action, take a fresh `browser_snapshot` and **drop back into DOM-first mode** — vision is invoked per-incident, not for the rest of the session.

### Phase 3 — Reliability layer
- Retry logic with backoff for flaky steps.
- Verification step after each action: re-snapshot and check expected state changed (e.g., new heading, form field populated) before proceeding — don't just trust the model's claim of success.
- Logging: record every tool call, snapshot diff, and fallback trigger for later debugging/audit.

### Phase 4 — Hardening for production
- Run headless by default; headed only for local debugging.
- Lock down MCP server exposure: no public port without a reverse proxy + auth, restrict `allowedOrigins` to the domains the agent is meant to touch.
- Version-pin the Playwright Docker base image; isolate `userDataDir` per session with correct permissions.
- Add human-in-the-loop confirmation for sensitive/destructive actions (payments, deletions, submissions) — pause and require explicit approval before executing.

### Phase 5 — Domain-specific site skills
- Build a small library of per-site "playbooks" — markdown files that capture the quirks of individual sites that trip up a generic agent (async modals, mislabeled buttons, canvas widgets, anti-bot behavior).
- Add a router in the agent loop: before acting on a new URL, match the hostname against the skills library and load the matching file(s) into context for that session.
- Full details, folder layout, and a starter template are in **Section 7** below.

### Phase 6 — Optional experiments
- Swap the vision-fallback engine from "general Gemini + raw screenshot" to Google's dedicated **Gemini Computer Use** model, which is purpose-built for screen-native action prediction — benchmark cost/accuracy against the generic approach.
- Evaluate WebMCP once more target sites adopt it, since it eliminates the DOM/vision guessing problem entirely for supporting pages.
- Explore exposing skills as a callable tool (`load_site_skill(domain)`) instead of static injection, so the agent can pull in a playbook mid-session if it navigates to a new domain unexpectedly.

### Phase 7 — Blocklist enforcement hook
- Add a hard-gate check that runs before **every** navigation, click-through, search result follow, or fetch — not just at task start — so the agent can't reach a disallowed domain mid-session either.
- Full design, matching rules, and enforcement points are in **Section 9** below.

### Phase 8 — Browser terminal integration
- Add a live, split-pane terminal alongside the chat UI so Gemini's actions (and the user's own commands) are visible in one shared session.
- Full backend options, feature requirements (history, spell-check/correct, copy-paste), and the recommended stack are in **Section 10** below.

---

## 5. Recommended Defaults (concrete config)

- **Primary MCP launch:** `npx @playwright/mcp@latest --headless --isolated`
- **Fallback MCP launch (triggered on demand):** `npx @playwright/mcp@latest --headless --isolated --caps=vision`
- **Viewport fixed** at a consistent size (e.g., `--viewport-size=1280x720`) so vision-mode coordinates stay stable across runs.
- **Step budget:** cap agent loops (e.g., 25–40 steps) to prevent runaway cost from an agent stuck in a retry loop.
- **Cost control:** track how often fallback triggers per session — a task that constantly needs vision may indicate a page that deserves a dedicated hand-written script instead of a general agent.

---

## 6. Risks & Open Questions

- **Nondeterminism:** even DOM-based agents can pick different (valid) paths across runs; add state-verification rather than relying purely on step-by-step trust.
- **Cost creep:** vision calls are meaningfully more expensive per step than snapshot calls — the fallback trigger logic in Phase 2 is the main lever for keeping this in check.
- **Security exposure:** an MCP server with browser control is a real attack surface if exposed without auth — treat the "no public port without auth" rule as non-negotiable.
- **Site coverage for WebMCP:** still early; don't architect around it as a dependency yet, just keep it on the roadmap.
- **Skill staleness:** site playbooks will drift out of date as target sites redesign their UI — treat each `SKILL.md` as something that needs periodic re-validation, not a write-once artifact.
- **Blocklist bypass via redirects:** an allowed URL that redirects to a blocked one defeats a naive pre-navigation check — always re-validate the final resolved URL, not just the requested one (see Section 9.2).
- **Terminal execution surface:** a live shell reachable by Gemini is a materially larger attack surface than browser control alone — command-level gating and sandboxing (Section 10.6) are not optional, even for a personal/internal tool.

---

## 7. Domain-Specific Site Skills — Detailed Design

Some sites need more than the generic DOM-first/vision-fallback loop can figure out on its own — persistent login quirks, anti-bot behavior, canvas-based widgets, elements with no accessible name, or timing issues that a fresh agent will rediscover (and fail on) every single run. The fix is a small, versioned library of per-site "playbooks" that get loaded into context only when relevant.

### 7.1 Folder layout

```
skills/
  amazon.com/SKILL.md
  linkedin.com/SKILL.md
  workday-generic/SKILL.md
  salesforce-lightning/SKILL.md
```

One folder or file per site (or per site-family, e.g. `workday-generic` for the many companies that run on the same Workday templates).

### 7.2 Routing logic

Add this as a step in the agent loop, before the first action on any new URL:

1. Parse the hostname of the target URL.
2. Look for an exact or pattern match in the `skills/` folder.
3. If found, read the file(s) and prepend their content to the prompt/context for that session.
4. If not found, proceed with the generic DOM-first/vision-fallback loop only.

This can be as simple as a dictionary lookup (`hostname → file path`) in your own orchestration code — no special Gemini feature is required for the static-injection approach.

### 7.3 SKILL.md template

```markdown
---
name: example-site-name
description: One-line summary of what this skill covers and when to use it
---
# <Site Name>

## When to use
- Any task URL matching <domain pattern>

## Known quirks
- <e.g. "Login form loads async — wait for [data-test-id=...] not just page load">
- <e.g. "Cookie consent modal blocks clicks until dismissed">
- <e.g. "Search results paginate via infinite scroll, not page numbers">

## Recommended approach
1. <Step-by-step guidance specific to this site>
2. <Note whether snapshot mode is sufficient or vision mode should be forced>
3. <Any login/session-state handling specifics>

## Known dead ends
- <Approaches that look reasonable but fail here, so the agent doesn't waste steps rediscovering them>

## Last verified
- <date> — <note on whether the site's UI has changed since>
```

### 7.4 What belongs in a skill file (and what doesn't)

**Include:**
- Selector/landmark quirks (elements with no accessible name, shadow DOM, custom widgets)
- Timing issues (async-loaded modals, delayed elements, infinite scroll)
- Anti-bot behavior specific to that site (rate limits, headless-mode breakage)
- An explicit call-out for when to force vision mode (e.g., "this dropdown is a canvas widget — always use `--caps=vision` here")
- Login/session handling notes (where persisted auth state lives, MFA quirks)
- Dead ends — approaches that look reasonable but fail, so the agent doesn't waste a run rediscovering them

**Don't include:**
- Generic Playwright/Gemini usage instructions — that belongs in the main agent prompt, not a site file
- Credentials or secrets — reference where they're stored (e.g., env var name), never the values themselves

### 7.5 Two ways to load skills

| Approach | Mechanics | Best for |
|---|---|---|
| **Static injection** | Orchestration code matches hostname → reads file → prepends to prompt | Simplest; fits directly into the Phase 1 per-step loop with one added lookup |
| **Skill-as-tool** | Expose `load_site_skill(domain)` as a Gemini-callable tool; model decides when to call it | Better if the agent may navigate across multiple domains mid-task and needs to pull in playbooks dynamically |

**Recommendation:** implement static injection first (Phase 5) since it requires no new tool-calling surface; revisit skill-as-tool in Phase 6 if the agent starts handling multi-domain tasks.

---

## 8. Gemini's Native Grounding Tools vs. the Playwright Agent

Not every "get Gemini info from the web" need requires the full browser agent. Two built-in Gemini tools cover lighter-weight cases:

| Need | Tool | Notes |
|---|---|---|
| "What's the latest on X" | `google_search` | Model runs the search itself, returns cited sources; no custom plumbing |
| "Summarize/extract data from this one page" | `url_context` | Fetches and reads full page content (HTML, JSON, plain text, XML, CSS, JS, CSV, RTF, images, PDFs) directly from a URL in the prompt |
| "Find pages about X, then read each one" | both together | Search discovers, URL context deep-reads each result |
| Multi-step crawling, forms, logins, JS-rendered content, clicking through a site | **Playwright MCP agent** (Phases 0–7) | Neither native tool renders JavaScript-heavy SPAs, handles logins, or performs multi-step interaction — that's what the rest of this plan is for |

**Practical rule:** reach for `google_search`/`url_context` first for read-only research tasks — they're cheaper and require no browser infrastructure at all. Escalate to the Playwright agent only when the task needs actual interaction (clicking, form-filling, authenticated sessions, multi-page navigation).

---

## 9. Blocklist Enforcement Hook

A single, centrally-enforced gate that blocks the agent from reaching disallowed domains — regardless of which tool or path it took to get there (direct navigation, a link it clicked, a search result it followed, or a `url_context` fetch).

### 9.1 Design principle

Don't rely on the model to "know" not to visit a blocked site. Enforce it in code, at every point where a URL is about to be acted on — navigation, click-through, fetch, and search-result following alike.

### 9.2 Enforcement points

| Path | Where the check must run |
|---|---|
| Direct navigation | Before `browser_navigate` executes |
| Clicking a link | Before the click resolves to a new URL, or immediately after navigation completes (verify-after-the-fact as a backstop) |
| Search-driven discovery | Filter `google_search` result URLs before they're offered to the model or fetched via `url_context` |
| Vision-fallback clicks | Same pre-navigation check applies even when the target was chosen via screenshot coordinates |
| Redirects | Re-check the final resolved URL after any redirect chain, not just the originally requested one — blocklists are trivially bypassed by an allowed URL that redirects to a blocked one |

### 9.3 Matching rules

- Support exact-domain, subdomain-wildcard (`*.example.com`), and path-prefix entries (`example.com/admin/*`) at minimum.
- Normalize URLs before matching: lowercase, strip trailing slashes, resolve `www.` variants, decode percent-encoding — otherwise trivial obfuscation slips through.
- Maintain the blocklist as external config (JSON/YAML), not hardcoded in agent logic, so it can be updated without a redeploy.

```yaml
# blocklist.yaml
domains:
  - "*.malicious-example.com"
  - "competitor-internal.example.com"
paths:
  - "example.com/admin/*"
```

### 9.4 Behavior on match

- Hard-block the action — do not execute the navigation/click/fetch.
- Return a clear tool-result error to Gemini (e.g., `{"error": "blocked_domain", "domain": "..."}`) so the model can adjust its plan rather than silently retrying.
- Log every block event (timestamp, requested URL, triggering action, task ID) for audit — repeated block attempts on the same task may indicate the task itself should be reviewed.

### 9.5 Implementation options

| Approach | Mechanics |
|---|---|
| **Wrapper around MCP tool calls** | Intercept `browser_navigate`, `browser_click`, etc. in your orchestration layer before forwarding to the MCP server; check target URL against blocklist first |
| **Proxy-level enforcement** | Route all browser traffic through a filtering proxy (e.g., domain-filtering reverse proxy) so the block happens at the network layer regardless of which tool triggered it — more robust, catches redirects and sub-resource loads too |
| **Both (recommended)** | Wrapper-level check for fast, clear model feedback; proxy-level check as a defense-in-depth backstop in case a path is missed at the application layer |

**Recommendation:** implement the wrapper-level check first (fast to build, integrates directly with the Phase 1 tool-calling loop), then add proxy-level filtering during Phase 4 hardening for defense in depth — this also naturally covers any blocked domains reached via `google_search`/`url_context` if those tools are used alongside the Playwright agent.

---

## 10. Browser Terminal Integration

A live terminal panel next to the chat UI, so Gemini's shell/agent actions and the user's own commands are both visible in one shared session.

### 10.1 Backend options (mix and match)

| # | Option | Mechanics | Best fit |
|---|---|---|---|
| 1 | **ttyd** | C, built on libwebsockets; wraps any CLI as a web app; built-in basic-auth, read-only/writable mode | Fastest to stand up; good default choice |
| 2 | **GoTTY** | Go equivalent of ttyd; same core idea, slightly fewer built-in options | Fits an already-Go-heavy stack |
| 3 | **Wetty** | Node.js-based, supports SSH/login | Fits naturally next to a Node/Express backend |
| 4 | **Custom xterm.js + node-pty + WebSocket** | You own every layer: frontend rendering, backend pty, transport | Full control over auth, command gating, and Gemini tool-call hooks |
| 5 | **Dinotty** | Server-side VTE (not just a passthrough pipe), session persistence across disconnects/refreshes, built-in file browser, plugin system | Best fit for long-running agent sessions that must survive a dropped connection |
| 6 | **WebContainer API** | Full Node.js runtime compiled to WebAssembly, runs entirely client-side, no backend server at all | Best when the goal is "Gemini generates and runs code live in-browser" rather than controlling a persistent host shell — same tech behind Claude Artifacts' code-running mode |
| 7 | **MCP-wrapped PTY** | A terminal/PTY session exposed as MCP tools (e.g. `node-terminal-mcp`) rather than raw shell writes | Cleanest integration point for command-gating logic; pairs naturally with Gemini CLI's native MCP support |

**Recommendation:** Option 4 or 1 for the visible terminal panel (fast, proven, standard xterm.js/node-pty pattern), combined with **Option 7** so Gemini's shell access goes through an MCP tool-calling layer rather than direct pty writes — this puts command approval/blocklist logic in one place instead of scattering it across the app. Reach for **Option 6 (WebContainer)** only if a fully sandboxed, serverless, in-browser code-execution model fits better than controlling a real host shell — note it has no real network access (proxied through the vendor's infrastructure) and doesn't support native Node bindings.

### 10.2 Frontend rendering

```javascript
import { Terminal } from '@xterm/xterm';
import { FitAddon } from '@xterm/addon-fit';
import { WebglAddon } from '@xterm/addon-webgl';
import { ClipboardAddon } from '@xterm/addon-clipboard';
import { SearchAddon } from '@xterm/addon-search';
import { WebLinksAddon } from '@xterm/addon-web-links';

const term = new Terminal({ scrollback: 10000 });
[new FitAddon(), new WebglAddon(), new ClipboardAddon(), new SearchAddon(), new WebLinksAddon()]
  .forEach(a => term.loadAddon(a));
```

### 10.3 Required features

| Feature | How it's satisfied |
|---|---|
| **Scrollback** | `scrollback: 10000` on the Terminal constructor (xterm.js core) |
| **Command history (↑/↓, Ctrl+R)** | Free from the underlying shell (bash/zsh) as long as `HISTFILE` persists per session; layer in **fzf** for fuzzy `Ctrl+R` search |
| **Copy/paste** | `@xterm/addon-clipboard` (official xterm.js addon); document the keybinding convention explicitly — xterm.js defaults to Shift+select-to-copy since Ctrl+C is reserved for SIGINT |
| **Spell-check/correct** | Not an xterm.js feature — a shell-level concern. Two options: (a) **fish shell** or **zsh + zsh-autosuggestions/zsh-syntax-highlighting** for live autosuggestion and inline correction as you type; (b) **`thefuck`** for post-hoc correction of a command that already failed |
| **Find-in-scrollback** | `@xterm/addon-search` |
| **Clickable links in output** | `@xterm/addon-web-links` |

Recommended default shell for the pty: **fish**, for autosuggestion + inline validity checking with the least setup; add `thefuck` as a fallback for typos already run.

### 10.4 Wiring Gemini to the terminal

One pty process, two writers — the user's own keystrokes and Gemini's tool-driven commands both land in the same visible session:

```javascript
// User input, from the terminal UI
socket.on('terminal.input', (data) => ptyProcess.write(data));

// Gemini's function call
function runShellCommand(command) {
  ptyProcess.write(command + '\n'); // same pty, same visible output
}
```

If using the MCP-wrapped approach (10.1, option 7), this becomes a tool call routed through the MCP server instead of a direct `ptyProcess.write()` — functionally the same outcome, cleaner interception point for gating (10.6).

### 10.5 Function-calling schema

```json
{
  "name": "run_shell_command",
  "description": "Run a shell command in the user's terminal session",
  "parameters": { "command": { "type": "string" } }
}
```

### 10.6 Security — non-negotiable before real use

A live pty reachable by Gemini is a materially larger attack surface than browser control alone:

- Run the pty inside a **Docker container** or restricted user account — never on the host directly.
- Apply the same allowlist/denylist pattern as Section 9's blocklist hook, but for commands: gate `ptyProcess.write()` (or the MCP tool call) against a deny-pattern list (`rm -rf`, `curl | sh`, credential-file access, etc.) before it executes.
- Add a **confirm-before-execute** step for destructive commands, mirroring the human-in-the-loop pattern from Phase 4.
- Log every command Gemini-originated command separately from user-typed ones, so an audit trail can distinguish who ran what.

---

## 11. Fact-Check Log

Verification pass across the plan's load-bearing technical claims, re-checked against current documentation as of August 22, 2026.

| Claim | Status | Note |
|---|---|---|
| Playwright MCP defaults to accessibility-tree snapshots, not screenshots | ✅ Confirmed | Consistent across README, official docs, and third-party guides |
| Vision mode enabled via `--caps=vision` (with `--vision` as legacy alias) | ✅ Confirmed | Verified directly against Playwright's own docs (`playwright.dev/mcp/vision-mode`) |
| Vision-mode tools: `browser_mouse_click_xy`, `browser_mouse_move_xy`, `browser_mouse_drag_xy` | ✅ Confirmed | Exact names verified |
| Capabilities are mixable (e.g. `--caps=vision,pdf,devtools`) | ✅ Confirmed | Also found `--caps=testing` for assertion primitives, not mentioned in the plan but not needed for this build |
| Snapshot mode is the token-cheaper, more reliable default; vision only for canvas/custom UI | ✅ Confirmed | Repeated consistently, including a concrete cost comparison (~114K tokens/task on snapshot-heavy flows vs. much higher with heavy vision use) |
| Google's dedicated Computer Use model exists as an alternative to general Gemini + screenshots | ✅ Confirmed, name corrected | Official name is **Gemini 2.5 Computer Use** (not "Gemini Computer Use" as shorthanded earlier) — a specialized model built on Gemini 2.5 Pro, released in public preview, available via Google AI Studio and Vertex AI. Reference implementation (`google/computer-use-preview`) explicitly recommends running in a sandboxed VM/container — reinforces this plan's Phase 4/10.6 sandboxing requirements independent of which model drives vision-fallback |
| `google_search` and `url_context` are real, current Gemini API tools | ✅ Confirmed | Both documented as of August 2026; combinable with each other and with custom function-calling tools |
| xterm.js addons (`clipboard`, `fit`, `webgl`, `search`, `web-links`) are real and current | ✅ Confirmed | All listed in xterm.js's official addon catalog |
| ttyd, GoTTY, Wetty are real, current browser-terminal tools | ✅ Confirmed | Still active/maintained as of 2026 sources |
| Dinotty as a session-persistent, server-side-VTE terminal server for AI agents | ✅ Confirmed | Verified against its own repo description and comparison table vs. ttyd/gotty/wetty |
| WebContainer API runs Node.js client-side via WebAssembly, powers StackBlitz/Claude Artifacts' code-running mode | ✅ Confirmed | Also confirmed real limitation: no native Node bindings, no real network (proxied) |
| WebMCP in Chrome Canary preview, early-stage | ✅ Still accurate | No material update found since original research |

**Corrections made to the plan as a result of this pass:** none required beyond the model-name precision noted above (already reflected in this log; the body text's shorthand "Gemini Computer Use" is accurate enough in context but the log preserves the precise name for anyone implementing Phase 6).

---

*Prepared as a working plan — adjust step budgets, fallback thresholds, and security settings to match your actual deployment environment before going to production.*
