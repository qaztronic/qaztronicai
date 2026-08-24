---
name: task-decomposer
description: Use this skill whenever the user wants to break down an effort, project, feature, or plan into small, self-contained, individually-validated tasks — e.g. "break this down into tasks," "decompose this project," "how should I split this work up," "turn this into a task list with tests." Also trigger when the user asks for a work breakdown structure, task dependencies/ordering, or a Definition of Done / validation criteria per task. Do NOT use for simple to-do lists with no decomposition or validation need, or for scheduling/timeline questions with no task-breakdown component.
---

# Task Decomposer

A skill for splitting an effort into tasks that are genuinely self-contained and individually verifiable —
not just a to-do list, but a structured breakdown where every task has an explicit validation test.

## Core stance

A task isn't "done" decomposing until each leaf task passes the INVEST bar (see below) and has a concrete,
checkable validation test attached. A breakdown without validation tests is just a to-do list wearing a
work-breakdown-structure costume.

## Step 1 — Recursive decomposition (Work Breakdown Structure)

Split the effort top-down: effort → deliverables → work packages → tasks. Keep splitting each branch until
the resulting task is small enough to estimate, assign, and verify independently. Don't stop at the first
level that "feels like enough" — push each branch until it hits atomicity or the INVEST bar in Step 2.

## Step 2 — Quality gate per task (INVEST)

For every candidate leaf task, check it against all six criteria before accepting it as final. If a task
fails any of these, split it further or rewrite it:

- **Independent** — can be done without waiting on a sibling task's internal details (dependencies on
  *outputs* are fine and expected; dependencies on another task's *implementation* are not)
- **Negotiable** — not so over-specified that there's no room to adjust approach
- **Valuable** — completing it visibly moves the effort forward, not just busywork
- **Estimable** — someone can size it with reasonable confidence
- **Small** — completable in a short, bounded unit of work (not a multi-week blob)
- **Testable** — there's a concrete way to check it's actually done (this feeds Step 3)

## Step 3 — Validation test per task

For each task that passed Step 2, attach a validation test using whichever fits the task type:

- **Design by Contract** (technical/well-defined tasks): state the precondition (what must be true going
  in), postcondition (what must be true coming out), and any invariant that must hold throughout. The test
  is literally "check the postcondition given the precondition."
- **Definition of Done checklist** (fuzzy, creative, or judgment-based tasks where pre/postconditions don't
  cleanly apply): a short list of observable completion criteria — concrete enough that someone unfamiliar
  with the task could check it off without guessing.

Every task gets one or the other. Don't leave a task without an attached test just because it seemed obvious.

## Step 4 — Sequencing (dependency graph)

Once tasks and their tests exist, model dependencies as a directed graph and topologically sort to get a
valid execution order. This step exists to catch two problems, not just to produce a schedule:

- **Circular dependencies** — a sign two "independent" tasks are secretly coupled and need to be re-split
- **Hidden coupling** — a task quietly depending on another's internals rather than its declared output,
  which is a Step 2 (Independent) violation that slipped through

## Step 5 — Completeness and overlap audit (MECE)

Run one pass over the *entire* task set, not per-task:

- **Collectively exhaustive** — does every part of the original effort map to at least one task? Anything
  in the original scope with no owning task is a gap.
- **Mutually exclusive** — do any two tasks overlap in ownership or duplicate work? Ambiguous ownership
  causes rework and untested seams later.

Flag both gaps and overlaps explicitly to the user rather than silently fixing them — the user may have
context on which is intentional.

## Output format

Present the breakdown as a task list with, for each task:
- **Task name** and one-line description
- **Depends on**: which other tasks must complete first (from Step 4's graph)
- **Validation test**: the contract (pre/post/invariant) or DoD checklist from Step 3
- **INVEST check**: brief confirmation it passed, or note if it's a judgment call

Close with:
- **Execution order** from the topological sort
- **MECE audit results**: any gaps or overlaps found, explicitly flagged for the user to resolve
