# The Math and Methods Behind goal-planner

This document covers the orchestration logic behind goal-planner — the algorithms for deciding what to do
next, breaking a goal into ordered steps, looping through execution, and sequencing under uncertainty —
with Python examples for each.

---

## 1. Means-Ends Analysis (the core routing logic)

### The formalism

A problem is a state space with a **current state** $s_0$, a **goal state** $s_g$, and a set of
**operators** (actions), each of which transforms one state into another and has **preconditions** for
when it can be applied. Means-ends analysis works by:

1. Compute the difference between current state and goal state.
2. Find an operator whose effects reduce that difference.
3. If the operator's preconditions aren't met, recursively set achieving them as a new subgoal.
4. Apply the operator; repeat until the difference is zero.

### Python: a minimal means-ends solver

```python
from dataclasses import dataclass

@dataclass
class Operator:
    name: str
    preconditions: set[str]
    add_effects: set[str]      # facts that become true
    remove_effects: set[str]   # facts that become false

def difference(state: set[str], goal: set[str]) -> set[str]:
    return goal - state  # what's missing from the goal that isn't true yet

def means_ends_analysis(state: set[str], goal: set[str], operators: list[Operator],
                          depth: int = 0, max_depth: int = 10) -> list[str]:
    if depth > max_depth:
        raise RecursionError("Max depth exceeded -- goal may be unreachable")

    diff = difference(state, goal)
    if not diff:
        return []  # goal already satisfied

    # Find an operator that reduces the difference
    for op in operators:
        if diff & op.add_effects:  # this operator produces at least one missing fact
            # Recursively achieve the operator's own preconditions first
            precondition_diff = op.preconditions - state
            plan = []
            working_state = set(state)
            if precondition_diff:
                sub_plan = means_ends_analysis(working_state, op.preconditions, operators,
                                                 depth + 1, max_depth)
                plan.extend(sub_plan)
                for sub_op_name in sub_plan:
                    sub_op = next(o for o in operators if o.name == sub_op_name)
                    working_state = (working_state - sub_op.remove_effects) | sub_op.add_effects

            plan.append(op.name)
            working_state = (working_state - op.remove_effects) | op.add_effects
            plan.extend(means_ends_analysis(working_state, goal, operators, depth + 1, max_depth))
            return plan

    raise ValueError(f"No operator found to reduce difference: {diff}")

operators = [
    Operator("research_market", preconditions=set(), add_effects={"market_understood"}, remove_effects=set()),
    Operator("write_business_plan", preconditions={"market_understood"},
             add_effects={"plan_written"}, remove_effects=set()),
    Operator("secure_funding", preconditions={"plan_written"},
             add_effects={"funded"}, remove_effects=set()),
    Operator("launch_product", preconditions={"funded"},
             add_effects={"product_launched"}, remove_effects=set()),
]

plan = means_ends_analysis(state=set(), goal={"product_launched"}, operators=operators)
print(" -> ".join(plan))
```

This is exactly what goal-planner's router does, but with each "operator" replaced by one of the five other
skills: the difference between "don't know what options exist" and "have a chosen approach" is closed by
invoking structured-brainstorm, and so on.

---

## 2. Hierarchical Task Network (HTN) Planning

### The formalism

HTN planning distinguishes **primitive tasks** (directly executable) from **compound tasks** (which must be
decomposed via a **method** into a sequence of simpler subtasks, which may themselves be compound). Planning
proceeds by recursively expanding compound tasks until only primitive tasks remain.

### Python: a minimal recursive HTN expander

```python
from dataclasses import dataclass

@dataclass
class Method:
    task_name: str
    preconditions: set[str]
    subtasks: list[str]

primitive_tasks = {"design_schema", "write_backend_code", "write_frontend_code", "run_integration_test"}

methods = [
    Method("build_backend", preconditions=set(), subtasks=["design_schema", "write_backend_code"]),
    Method("build_frontend", preconditions={"design_schema"}, subtasks=["write_frontend_code"]),
    Method("build_full_app", preconditions=set(),
           subtasks=["build_backend", "build_frontend", "run_integration_test"]),
]

def htn_decompose(task: str, state: set[str], depth: int = 0) -> list[str]:
    if task in primitive_tasks:
        return [task]

    applicable_methods = [m for m in methods if m.task_name == task and m.preconditions <= state]
    if not applicable_methods:
        raise ValueError(f"No applicable method for compound task '{task}' given state {state}")

    method = applicable_methods[0]
    plan = []
    working_state = set(state)
    for subtask in method.subtasks:
        sub_plan = htn_decompose(subtask, working_state, depth + 1)
        plan.extend(sub_plan)
        # Mark BOTH the compound subtask and every primitive step inside it as done,
        # so later sibling subtasks can see preconditions satisfied deep inside a branch
        # (e.g. "build_frontend" needs "design_schema", which happened inside "build_backend").
        working_state.add(subtask)
        working_state.update(sub_plan)
    return plan

full_plan = htn_decompose("build_full_app", state=set())
print(" -> ".join(full_plan))
```

This mirrors exactly what task-decomposer does when goal-planner calls it: "build_full_app" is a compound
task with no obvious single action, so it recursively decomposes through methods until only directly
executable primitive tasks remain, in dependency order.

---

## 3. Plan–Do–Check–Act (the outer loop)

PDCA doesn't have a single equation, but its *iteration structure* is worth making explicit, because that
structure is what prevents goal-planner from being a one-shot plan generator.

```python
def pdca_loop(goal_state: set[str], initial_state: set[str], operators, max_cycles=5):
    state = set(initial_state)
    cycle = 0

    while state != goal_state and cycle < max_cycles:
        cycle += 1
        print(f"--- Cycle {cycle} ---")

        # PLAN: build a plan to close the gap from current state to goal
        plan = means_ends_analysis(state, goal_state, operators)
        print(f"Plan: {plan}")

        # DO: execute the plan (here, simulated by applying operator effects)
        for op_name in plan:
            op = next(o for o in operators if o.name == op_name)
            state = (state - op.remove_effects) | op.add_effects

        # CHECK: did executing the plan actually satisfy the goal?
        gap = goal_state - state
        print(f"State after execution: {state}")
        print(f"Remaining gap: {gap if gap else 'none -- goal reached'}")

        # ACT: if there's still a gap, loop back to PLAN with the updated state
        if not gap:
            break

    return state

final_state = pdca_loop(goal_state={"product_launched"}, initial_state=set(), operators=operators)
```

In practice, the "CHECK" step is where regression-test-strategy and fact-falsifier get invoked (verifying
the work actually satisfies the goal, not just that steps were executed) — and a failed check is exactly
what sends the loop back to "PLAN" for another cycle rather than stopping.

---

## 4. Critical Path Method (finding what actually gates the goal)

### The math: forward pass and backward pass

Given a task graph with durations, CPM computes, for every task, its **earliest start (ES)**, **earliest
finish (EF)**, **latest start (LS)**, and **latest finish (LF)**. The **float** (slack) of a task is:

$$\text{float}(t) = LS(t) - ES(t)$$

Tasks with zero float are on the **critical path** — any delay in them delays the whole project.

Forward pass (compute ES/EF, going through the graph in topological order):
$$EF(t) = ES(t) + \text{duration}(t), \qquad ES(t) = \max_{p \, \in \, \text{predecessors}(t)} EF(p)$$

Backward pass (compute LS/LF, going through the graph in *reverse* topological order):
$$LS(t) = LF(t) - \text{duration}(t), \qquad LF(t) = \min_{s \, \in \, \text{successors}(t)} LS(s)$$

### Python: computing the critical path

```python
def critical_path_method(tasks: dict[str, float], dependencies: list[tuple[str, str]]):
    """tasks: name -> duration. dependencies: list of (task, depends_on)."""
    predecessors = {t: [] for t in tasks}
    successors = {t: [] for t in tasks}
    for task, dep in dependencies:
        predecessors[task].append(dep)
        successors[dep].append(task)

    order = topological_sort(list(tasks.keys()), dependencies)

    # Forward pass
    ES, EF = {}, {}
    for t in order:
        ES[t] = max((EF[p] for p in predecessors[t]), default=0)
        EF[t] = ES[t] + tasks[t]

    project_duration = max(EF.values())

    # Backward pass
    LF, LS = {}, {}
    for t in reversed(order):
        LF[t] = min((LS[s] for s in successors[t]), default=project_duration)
        LS[t] = LF[t] - tasks[t]

    float_time = {t: LS[t] - ES[t] for t in tasks}
    critical_path = [t for t in tasks if float_time[t] == 0]

    return {"duration": project_duration, "float": float_time, "critical_path": critical_path}

def topological_sort(tasks, dependencies):
    from collections import deque, defaultdict
    in_degree = {t: 0 for t in tasks}
    graph = defaultdict(list)
    for task, dep in dependencies:
        graph[dep].append(task)
        in_degree[task] += 1
    queue = deque(t for t in tasks if in_degree[t] == 0)
    order = []
    while queue:
        n = queue.popleft()
        order.append(n)
        for nb in graph[n]:
            in_degree[nb] -= 1
            if in_degree[nb] == 0:
                queue.append(nb)
    return order

tasks = {"design": 3, "backend": 5, "frontend": 4, "integration": 2}
deps = [("backend", "design"), ("frontend", "design"), ("integration", "backend"), ("integration", "frontend")]

result = critical_path_method(tasks, deps)
print(f"Project duration: {result['duration']}")
print(f"Float per task: {result['float']}")
print(f"Critical path: {result['critical_path']}")
```

`backend` takes longer than `frontend`, so `frontend` has slack (positive float) while `design`, `backend`,
and `integration` sit on the critical path with zero float — telling goal-planner exactly where to focus
Check-stage effort, since delays anywhere else don't threaten the goal's timeline.

---

## 5. Model Predictive Control (rolling-horizon replanning)

### The core idea

For long or uncertain goals, MPC doesn't try to optimize the entire future at once. Instead, at each step
it solves a much smaller optimization over a short horizon, applies only the *first* action from that
solution, then re-solves from scratch once new information arrives.

$$\text{at each step: solve} \min_{u_0, \ldots, u_{H-1}} \sum_{k=0}^{H-1} \text{cost}(x_k, u_k) \;\; \text{apply only } u_0, \;\; \text{repeat}$$

### Python: a toy rolling-horizon planner

```python
def rolling_horizon_plan(state, goal, operators, horizon=2, total_steps=6):
    """Plans only `horizon` steps ahead at a time, executes one step, then replans."""
    history = []
    for step in range(total_steps):
        if state == goal:
            print(f"Goal reached after {step} steps.")
            break

        # Plan only a short horizon ahead (here, reuse means-ends but cap depth)
        try:
            short_plan = means_ends_analysis(state, goal, operators, max_depth=horizon)
        except (ValueError, RecursionError):
            short_plan = means_ends_analysis(state, goal, operators)  # fall back to full plan if needed

        if not short_plan:
            break

        # Apply only the FIRST action, then stop and replan next iteration
        next_op = next(o for o in operators if o.name == short_plan[0])
        state = (state - next_op.remove_effects) | next_op.add_effects
        history.append(next_op.name)
        print(f"Step {step}: executed '{next_op.name}', state now includes {state}")

    return history

history = rolling_horizon_plan(state=set(), goal={"product_launched"}, operators=operators, horizon=2)
print("Execution history:", history)
```

The practical benefit: if new information changes the goal or the state partway through (a common real-world
situation goal-planner is built for), only the next short horizon needs replanning — not a full plan built
on assumptions that are now stale.

---

## Putting it together

| Part | Tool |
|---|---|
| Core routing (which skill to call) | means-ends analysis: reduce the gap between current and goal state |
| Task decomposition | HTN planning (delegated to task-decomposer) |
| Outer iteration | PDCA loop |
| Prioritizing Check-stage effort | Critical Path Method (float = 0 tasks) |
| Long/uncertain goals | Model Predictive Control / rolling-horizon replanning |
