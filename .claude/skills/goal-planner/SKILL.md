---
name: goal-planner
description: Use this skill whenever the user states a goal and wants a concrete plan to implement/reach it — e.g. "make a plan to accomplish X," "how do I get from where I am to Y," "help me implement this goal," "what's the path to Z." This is an orchestration skill that decides, at each step of building the plan, which of five other skills to invoke (structured-brainstorm, fact-falsifier, research-strategy, task-decomposer, regression-test-strategy) rather than doing everything itself. Also trigger for "build a roadmap," "plan how to reach...," or "turn this goal into an actionable plan." Do NOT use for simple single-step requests with no real planning need, or when the user has already fully specified a step-by-step plan and just wants it executed.
---

# Goal Planner

An orchestration skill: given a stated goal, builds a concrete, executable plan to reach it by routing each
planning need to the right specialist skill rather than trying to do everything itself. This skill's own job
is deciding *which* skill to call *when* — not replacing any of them.

## Skills this orchestrates

- **structured-brainstorm** — generating and narrowing options
- **fact-falsifier** — stress-testing claims/assumptions the plan depends on
- **research-strategy** — gathering information the plan is currently missing
- **task-decomposer** — breaking the goal into small, self-contained, validated tasks
- **regression-test-strategy** — validating that implementation work hasn't broken what already worked

If any of these skills isn't available in the environment, do the equivalent step manually using its core
method (each is summarized inline below) rather than skipping the step outright.

## Core mechanism: the router

At every stage of planning, ask "what kind of gap am I trying to close right now?" and route accordingly
(means-ends analysis: identify the gap between current state and goal state, pick the operation that
reduces it). Use this decision table as the default routing logic:

| Planning need / gap | Route to |
|---|---|
| Don't know what approaches/options exist | **structured-brainstorm** |
| Not sure a belief the plan depends on is actually true | **fact-falsifier** |
| Missing information needed to plan (market, technical, factual) | **research-strategy** |
| Have a goal/approach but no concrete task breakdown | **task-decomposer** |
| Have implemented work and need to verify nothing broke / know when to stop testing | **regression-test-strategy** |

Multiple needs often apply at once — route to more than one skill in a stage when that's the case, don't
force a single call per stage.

## The outer loop: Plan → Do → Check → Act (PDCA)

Wrap the router in this iterative cycle. This skill is meant to reach implementation, not just produce a
one-shot document — expect to loop.

### 1. Plan

- **Frame the gap.** State the goal explicitly and the current state, in concrete terms. This is the
  means-ends starting point everything else routes off of.
- **Fill information gaps** (→ research-strategy) if the goal can't yet be planned confidently — e.g.
  unknowns about constraints, market, technical feasibility.
- **Generate and narrow approaches** (→ structured-brainstorm) if there's more than one plausible way to
  reach the goal. Skip this if the approach is already obviously determined by the goal itself.
- **Stress-test load-bearing assumptions** (→ fact-falsifier) for any belief the chosen approach depends on
  that hasn't actually been verified — don't let an unexamined assumption become the foundation of the plan.
- **Decompose into tasks** (→ task-decomposer): once an approach is chosen, break it into small,
  self-contained tasks, each with a validation test, sequenced by dependency. This produces the actual
  execution plan, including the critical path — which tasks gate the goal's completion vs. have slack.

### 2. Do

Execute the tasks in the sequence task-decomposer produced. This step is outside this skill's scope to
perform directly (it's real-world/implementation work), but the plan handed to the user or executing agent
should be concrete enough to act on without further clarification.

### 3. Check

- **Validate against the task-level tests** produced in task-decomposer's Step 3 (contract/DoD checks) as
  each task completes.
- **Run regression validation** (→ regression-test-strategy) once enough implementation exists that new
  work could plausibly break prior work — selecting what to (re)check, generating tests for gaps, and using
  its stopping criteria to know when enough validation has been done for this cycle.
- **Falsify claims of "done"** (→ fact-falsifier) for any claim that the goal has been reached, rather than
  accepting completion at face value — especially for judgment-based or high-stakes goals.

### 4. Act

- If Check reveals gaps (new information needed, an assumption that broke, a task that didn't actually
  satisfy the goal), loop back to **Plan** and re-route through whichever skill addresses that specific
  gap — don't restart the whole pipeline from scratch each cycle.
- If the goal spans a long or uncertain horizon, don't try to fully detail every future task upfront.
  Instead, plan the near-term horizon in full detail via the loop above, execute it, then replan the next
  horizon with fresh information (rolling-horizon / Model Predictive Control logic) — commit less detail to
  parts of the plan far from being executed.

## Sequencing and critical path

Once task-decomposer has produced the task graph, use its dependency ordering (already computed via
topological sort in that skill) to identify the critical path — the chain of tasks that actually gates
reaching the goal. Flag critical-path tasks to the user distinctly from slack tasks, since Check-stage
effort (regression validation, fact-falsification of "done" claims) is highest-value there.

## Output format

1. **Goal and current-state framing** — the gap this plan closes
2. **Routing summary** — which skills were invoked for this planning pass and why (brief, not a full replay
   of each skill's internal output)
3. **The plan itself** — task list from task-decomposer, with dependencies, validation tests, and the
   critical path flagged
4. **Open risks** — any assumptions flagged by fact-falsifier as weak/unverified, and any information gaps
   research-strategy couldn't fully close
5. **Check/Act cadence** — how and when this plan should be re-validated (regression-test-strategy) and
   replanned as execution proceeds

## When to scale down

For a goal that's simple enough to have one obvious approach and no real unknowns or assumptions, skip
straight to task-decomposer alone rather than running the full router/loop — this skill's overhead is for
goals with real ambiguity, unverified assumptions, or multi-stage execution, not trivial ones.
