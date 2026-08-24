# The Math and Methods Behind task-decomposer

This document covers the structural and mathematical foundations behind breaking work into small,
self-contained, validated tasks — with Python examples for each piece.

---

## 1. Work Breakdown Structure (recursive decomposition)

### The model: a rooted tree

A WBS is formally a rooted tree $T = (V, E)$ where the root is the whole project, each node's children are
its sub-deliverables, and leaf nodes are individual work packages. The **100% rule** is a completeness
constraint: at every level, the children of a node must sum to exactly the scope of that node — no more, no
less.

$$\text{scope}(v) = \sum_{c \, \in \, \text{children}(v)} \text{scope}(c)$$

### Python: a WBS tree with a completeness check

```python
from dataclasses import dataclass, field

@dataclass
class WBSNode:
    name: str
    scope_weight: float = 1.0          # this node's share of its parent's scope (0 to 1)
    children: list["WBSNode"] = field(default_factory=list)

    def is_leaf(self) -> bool:
        return len(self.children) == 0

    def check_100_percent_rule(self, tolerance: float = 1e-6) -> bool:
        """Recursively verify children's weights sum to 1.0 (100% of parent's scope)."""
        if self.is_leaf():
            return True
        total = sum(child.scope_weight for child in self.children)
        if abs(total - 1.0) > tolerance:
            print(f"VIOLATION at '{self.name}': children sum to {total:.2f}, not 1.0")
            return False
        return all(child.check_100_percent_rule(tolerance) for child in self.children)

root = WBSNode("Launch new product", children=[
    WBSNode("Build the product", scope_weight=0.5, children=[
        WBSNode("Backend", scope_weight=0.6),
        WBSNode("Frontend", scope_weight=0.4),
    ]),
    WBSNode("Go to market", scope_weight=0.5, children=[
        WBSNode("Marketing", scope_weight=0.5),
        WBSNode("Sales enablement", scope_weight=0.3),
        # deliberately missing 0.2 of scope, to demonstrate the check catching it
    ]),
])

root.check_100_percent_rule()
```

Running this prints a violation under "Go to market" — exactly the kind of gap the skill's MECE audit
(section 5 below) is designed to catch, but formalized here as a numeric constraint rather than a
qualitative review.

---

## 2. INVEST Criteria (the quality gate)

INVEST isn't mathematical, but it can be encoded as a predicate — a function that returns true only if
every criterion passes, which is exactly how the skill applies it as a gate before accepting a task.

```python
from dataclasses import dataclass

@dataclass
class Task:
    name: str
    depends_only_on_outputs: bool   # Independent
    has_room_to_adjust_approach: bool  # Negotiable
    moves_project_forward: bool     # Valuable
    can_be_sized_confidently: bool  # Estimable
    completable_in_short_unit: bool # Small
    has_concrete_check: bool        # Testable

def passes_invest(task: Task) -> tuple[bool, list[str]]:
    checks = {
        "Independent": task.depends_only_on_outputs,
        "Negotiable": task.has_room_to_adjust_approach,
        "Valuable": task.moves_project_forward,
        "Estimable": task.can_be_sized_confidently,
        "Small": task.completable_in_short_unit,
        "Testable": task.has_concrete_check,
    }
    failed = [name for name, passed in checks.items() if not passed]
    return len(failed) == 0, failed

t = Task("Implement login form", True, True, True, True, False, True)
ok, failed = passes_invest(t)
print(f"Passes INVEST: {ok}, failed criteria: {failed}")
# Fails "Small" -- signal to split this task further before accepting it
```

---

## 3. Design by Contract (the validation test itself)

### The formalism: Hoare triples

Design by Contract is built on Hoare logic, written as a **Hoare triple**:

$$\{P\} \; C \; \{Q\}$$

meaning: if precondition $P$ holds before running code $C$, then postcondition $Q$ is guaranteed to hold
after $C$ finishes. An **invariant** $I$ is a special case that must hold both before and after every public
operation on an object.

### Python: contracts as executable assertions

```python
def with_contract(precondition, postcondition):
    """A decorator that turns a plain function into one with an explicit contract."""
    def decorator(func):
        def wrapper(*args, **kwargs):
            assert precondition(*args, **kwargs), f"Precondition failed for {func.__name__}"
            result = func(*args, **kwargs)
            assert postcondition(result, *args, **kwargs), f"Postcondition failed for {func.__name__}"
            return result
        return wrapper
    return decorator

@with_contract(
    precondition=lambda balance, amount: amount > 0 and amount <= balance,
    postcondition=lambda result, balance, amount: result == balance - amount,
)
def withdraw(balance: float, amount: float) -> float:
    return balance - amount

print(withdraw(100, 30))   # OK: precondition and postcondition both hold -> 70

try:
    withdraw(100, 150)     # violates precondition: can't withdraw more than the balance
except AssertionError as e:
    print(f"Caught: {e}")
```

This *is* the validation test the skill attaches to a technical task: state the precondition, state the
postcondition, and the "test" is literally checking the postcondition holds whenever the precondition did.

---

## 4. Topological Sort (sequencing by dependency)

### The algorithm: Kahn's algorithm

Given a directed acyclic graph (DAG) of task dependencies, a topological sort produces an ordering where
every task appears after everything it depends on. Kahn's algorithm works by repeatedly removing nodes with
no remaining incoming edges (in-degree zero):

1. Compute in-degree (number of dependencies) for every node.
2. Put all zero-in-degree nodes in a queue.
3. Repeatedly pop a node, add it to the result, and decrement the in-degree of its neighbors — if any
   neighbor's in-degree hits zero, add it to the queue.
4. If you process fewer nodes than exist in the graph, there's a cycle (a genuine dependency problem).

```python
from collections import deque, defaultdict

def topological_sort(tasks: list[str], dependencies: list[tuple[str, str]]) -> list[str]:
    """dependencies: list of (task, depends_on) pairs."""
    in_degree = {t: 0 for t in tasks}
    graph = defaultdict(list)

    for task, depends_on in dependencies:
        graph[depends_on].append(task)
        in_degree[task] += 1

    queue = deque([t for t in tasks if in_degree[t] == 0])
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    if len(order) != len(tasks):
        remaining = set(tasks) - set(order)
        raise ValueError(f"Circular dependency detected among: {remaining}")

    return order

tasks = ["design_schema", "build_backend", "build_frontend", "integration_test"]
dependencies = [
    ("build_backend", "design_schema"),
    ("build_frontend", "design_schema"),
    ("integration_test", "build_backend"),
    ("integration_test", "build_frontend"),
]

print(topological_sort(tasks, dependencies))
```

If you add a dependency that loops back — say `("design_schema", "integration_test")` — the function raises
instead of silently returning a bad order, exactly matching the skill's instruction to flag circular
dependencies as a sign two "independent" tasks are secretly coupled.

---

## 5. MECE (the completeness and overlap audit)

### The set-theoretic definition

Given a universal set $U$ (the full scope of the effort) and a partition into subsets $S_1, S_2, \ldots,
S_n$ (the tasks), MECE requires two properties:

**Mutually exclusive** (no overlap): $S_i \cap S_j = \emptyset$ for all $i \neq j$

**Collectively exhaustive** (no gaps): $S_1 \cup S_2 \cup \cdots \cup S_n = U$

```python
def check_mece(universal_scope: set, task_scopes: dict[str, set]) -> dict:
    all_covered = set()
    overlaps = []

    task_names = list(task_scopes.keys())
    for i in range(len(task_names)):
        for j in range(i + 1, len(task_names)):
            overlap = task_scopes[task_names[i]] & task_scopes[task_names[j]]
            if overlap:
                overlaps.append((task_names[i], task_names[j], overlap))
        all_covered |= task_scopes[task_names[i]]

    gaps = universal_scope - all_covered

    return {
        "mutually_exclusive": len(overlaps) == 0,
        "overlaps": overlaps,
        "collectively_exhaustive": len(gaps) == 0,
        "gaps": gaps,
    }

universal_scope = {"login", "signup", "password_reset", "profile_edit", "billing"}
task_scopes = {
    "auth_task": {"login", "signup", "password_reset"},
    "account_task": {"profile_edit", "password_reset"},  # overlaps with auth_task on password_reset
    # "billing" is never covered -- a gap
}

result = check_mece(universal_scope, task_scopes)
print(result)
```

This prints both the overlap (`password_reset` claimed by two tasks — ambiguous ownership) and the gap
(`billing` covered by nothing) — the exact two failure modes the skill's final audit step is checking for,
made concrete and machine-checkable instead of a purely eyeball review.

---

## Putting it together

| Step | Tool |
|---|---|
| Recursive decomposition | WBS tree + the 100% rule |
| Quality gate per task | INVEST as a boolean predicate |
| Validation test | Design by Contract / Hoare triples |
| Sequencing | topological sort (Kahn's algorithm) over the dependency DAG |
| Completeness audit | MECE as a set-partition check |
