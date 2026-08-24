---
name: plan-accuracy
description: Re-read files fresh rather than trusting memory of an earlier/deleted version; verify technical claims before asserting them
metadata:
  type: feedback
---

Two related mistakes from the same session, worth not repeating:

1. **Don't reason from memory of a file's earlier state once it's been replaced or deleted.** When asked to
   plan "Phase 1" of `wip/browser-agent-implementation-plan.md`, assumed the phase numbering matched an
   *older* version of that same-named file read earlier in the conversation (which had since been deleted
   and replaced with a differently-structured 14-phase version). Led to asking the user about the wrong
   phase ("why the fuck are you asking me about phase 5?" — fair). Should have re-read the current file
   first, especially after the user is known to have edited/replaced/deleted files mid-session.

2. **Verify a technical "X requires Y" claim before asserting it, don't reason from general impression.**
   Claimed Node was needed for a later phase because it used `xterm.js`/`node-pty`, without distinguishing
   the browser-rendered piece (genuinely, unavoidably JS — true of any browser UI, not a backend-language
   argument) from the backend pty-spawning piece (has a direct Python equivalent, `pty`/`ptyprocess`). Only
   caught this because the user pushed back and asked "what is so special about xterm?" — the correction
   should have happened from re-reading the actual source material the first time, not after being
   challenged twice.

**Why:** both mistakes come from the same root cause — reasoning from a remembered impression instead of
re-checking the actual current source (a file's live content, or a library's actual architecture) before
stating something as fact in a plan.

**How to apply:** in planning work specifically ([[use-planning-skills]]), when citing a file the user
referenced, re-read it fresh rather than relying on an earlier read in the same conversation if there's any
chance it changed. When making an "X is required because of Y" claim that will shape a plan's recommended
approach, verify Y's actual mechanics (what genuinely can't be substituted vs. what's just "the existing
package happens to be in language Z") rather than stating it from a general/half-remembered understanding.
