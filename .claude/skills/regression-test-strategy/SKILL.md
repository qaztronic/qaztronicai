---
name: regression-test-strategy
description: Use this skill whenever the user wants to design, select, prioritize, or generate regression tests for a codebase — including questions like "what tests should I run for this change," "how do I speed up my test suite without missing bugs," "which tests should I add," or "when should I stop adding tests." Also trigger when the user asks about test selection strategy, mutation testing, test coverage gaps, flaky/legacy test suites, or how to know if a regression suite is "good enough." This skill deliberately optimizes for efficient bug-catching over exhaustive re-running or 100% code coverage — it does not treat "run everything" or "cover everything" as the goal. Do NOT use for writing individual unit tests from scratch with no selection/strategy question involved, or for non-regression testing topics (e.g. load testing, UAT sign-off processes) unless they intersect with test selection or suite scoping.
---

# Regression Test Strategy

A skill for deciding **which regression tests to run, which new tests to add, and when to stop** — optimized
for catching real bugs efficiently, not for maximizing coverage percentage or suite size. Coverage numbers are
a weak proxy for bug-catching; this skill treats them as a diagnostic, never a target.

## Core stance

- **Selecting tests to run** and **generating tests to add** are different problems — don't conflate them.
- 100% code/suite coverage is not the goal and is usually not achievable or worth the cost. See "On coverage"
  below before recommending a change that pushes toward exhaustive running.
- A regression suite's job is to make regressions loud immediately, not to re-verify unrelated code on every change.

## Part 1 — Selecting which existing tests to run

Default recommendation, in order of application:

1. **Call-graph / dependency-based Regression Test Selection (RTS)** — build a static or dynamic call graph
   from the change, walk it to find all affected functions/modules, and select tests exercising any node in
   that affected subgraph. This is the default "what to run" filter — more reliable than raw coverage overlap
   because it catches indirect breakage, not just directly-touched lines.
2. **Risk-based prioritization** — within the selected set (or the full suite, when time allows), order tests
   by historical failure rate, recent code churn in the area, and business criticality of the feature. Run
   highest-risk first so a time-boxed run fails fast on what's likely broken.
3. **Test Impact Analysis (coverage-overlap selection)** — use as a cheaper fallback when a call graph isn't
   available (e.g. dynamically typed/scripted code where static graphs are unreliable). Weaker than #1 because
   it only catches directly-touched code, not indirect impact.
4. **Two-tier cadence** — run the RTS-selected subset on every commit for fast feedback; run the full suite on
   a slower cadence (nightly, pre-release, or pre-merge-to-main). This is how you get safety without paying
   full-suite cost on every change — recommend this explicitly when the user is worried RTS alone feels risky.

Skip pairwise/combinatorial generation and golden-master snapshotting from this default stack unless the
codebase specifically calls for them (see Part 2, gap categories 4 and 6).

## Part 2 — Generating new tests to fill gaps

Use the **structured-brainstorm** skill's two-phase method for this step whenever the user is asking "what
tests am I missing" or "what should I add" — this is exactly the kind of divergent-then-convergent search
that skill is built for, and gap-finding is where shallow, first-idea-only thinking is most costly.

**Phase 1 (divergence) — use these as the category dimensions**, generating a batch of candidate new tests
under each rather than stopping at the first obvious gap:
1. **Mutation-guided gaps** — inject small faults (mutants) into the changed code; any mutant that survives
   (no existing test kills it) is a candidate new test. This is the most targeted gap-finder — it finds
   places existing tests execute code but don't actually assert anything meaningful.
2. **Found-bug codification** — for every regression bug found in the wild, use delta debugging to minimize
   the failing input, then codify that minimal case as a new permanent test. Never let a found bug go
   un-codified — this is the highest-value category since it's a bug you know is real.
3. **Combinatorial/pairwise gaps** — for features with multiple interacting parameters/config flags, generate
   pairwise (or n-wise) combinations not currently covered.
4. **Legacy/golden-master gaps** — for code with no tests at all, capture current behavior as a baseline
   snapshot so future diffs get flagged for review, even before finer-grained tests exist.
5. **ODC category gaps** — classify existing found-bugs by type/trigger (Orthogonal Defect Classification);
   candidate new tests are ones that would target a defect *category* not yet represented, not just another
   instance of an already-well-tested category.

**Phase 2 (convergence)** — score each candidate on expected bug-catching value vs. cost to write/maintain,
then narrow to a shortlist. Per the brainstorm skill's convergence logic: don't just take the highest-scored
candidates — deliberately keep 1-2 lower-confidence-but-high-upside candidates (e.g. an odd mutant that's
cheap to kill, or a rare-but-catastrophic combinatorial case) that a naive top-N cut would drop.

If the structured-brainstorm skill isn't available, still run this as a divergence-then-convergence pass
manually: generate candidates across all 5 categories before scoring any of them, and don't collapse to a
shortlist in one cut.

## Part 3 — Knowing when to stop adding tests

Primary stopping signal:

- **Mutation score plateau** — track % of injected mutants killed across successive test additions to a
  module. Stop adding tests there once the score plateaus despite new tests — new tests are now only
  catching equivalent/duplicate mutants, not new bug classes.

Secondary/confirming signals — use alongside the primary signal, not instead of it:

- **ODC category coverage** — stop when new tests stop surfacing *new* defect categories, only repeats within
  categories already well-covered.
- **Reliability growth model** (e.g. Goel-Okumoto) — if the project tracks defect-discovery-over-time data,
  fit a curve and watch for the flattening that signals diminishing returns from more testing effort.
- **Capture-recapture defect estimation** — if multiple independent reviewers/testers are finding bugs, use
  overlap rate between their found-bug sets to estimate remaining undiscovered defects statistically.

**Explicitly do not use raw code/branch coverage percentage as a stopping signal.** It's fine as a secondary
diagnostic (e.g. flag a module at 20% coverage as suspicious), but high coverage famously does not equal low
bug rate, and treating "coverage went up" as "we can stop" will systematically under-test the highest-risk,
hardest-to-cover code.

## On coverage (read before recommending "run/cover everything")

If a user asks whether this approach reaches 100% coverage, or pushes toward exhaustive running/covering:
- Distinguish **code coverage**, **suite-run coverage**, and **behavioral coverage** — they're different
  numbers and none of them being 100% is a defect in this approach; it's the design.
- RTS and risk-based selection deliberately run *less than* the full suite per change; recommend the two-tier
  cadence (Part 1, #4) as the answer to "but what if RTS misses something," not "just run everything."
  100% behavioral coverage is generally infeasible for nontrivial software (path explosion, infinite input
  space) — don't imply it's an achievable target.
- If the user's priority is genuinely coverage-completeness over speed/cost (e.g. safety-critical code),
  say so explicitly and adjust: recommend running the full suite by default and reserve RTS for fast local
  iteration only, plus track a coverage floor that must never regress.

## Output format

When asked to design a regression strategy for a specific codebase/change, structure the answer as:
1. **Selection**: what runs on this change, and why (Part 1)
2. **Gaps**: brainstormed candidate new tests, using the two-phase method (Part 2), with the ranked shortlist
3. **Stop condition**: what signal will tell them they've added enough for now (Part 3)
4. **Coverage caveat**: one line noting what this approach does and doesn't guarantee re: coverage
