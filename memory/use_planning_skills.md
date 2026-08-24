---
name: use-planning-skills
description: Always invoke goal-planner (and its 5 sub-skills) when making any plan, not just when explicitly asked
metadata:
  type: feedback
---

For every plan produced in this environment — implementation plans, roadmaps, "how do I get from X to Y"
work — invoke the **goal-planner** skill rather than freehanding a plan. goal-planner orchestrates the other
five installed skills ([[repo-structure]] — all live in `.claude/skills/`):

- **structured-brainstorm** — generate/narrow options when the approach isn't already fixed
- **fact-falsifier** — stress-test assumptions the plan depends on
- **research-strategy** — fill missing information gaps
- **task-decomposer** — break the goal into small, validated tasks
- **regression-test-strategy** — verify nothing broke once work is implemented

**Why:** user explicitly asked that these 6 skills be used for every plan, not left as opt-in tools that
only fire on exact trigger phrasing.

**How to apply:** when a request calls for planning — even if phrased casually ("what should I do about
X," "help me get to Y") rather than with goal-planner's literal trigger phrases — invoke goal-planner
first rather than deciding ad hoc whether it "counts." Treat this as standing instruction, not a
per-request judgment call.
