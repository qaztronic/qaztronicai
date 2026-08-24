# The Math and Methods Behind regression-test-strategy

This document covers the algorithms behind selecting, generating, and knowing when to stop adding
regression tests — with the graph theory, statistics, and small Python examples behind each.

---

## 1. Regression Test Selection via Dependency Graphs

### The graph model

Represent the codebase as a directed graph $G = (V, E)$ where $V$ is the set of functions/modules and an
edge $(u, v) \in E$ means "$u$ calls/depends on $v$." When code changes, you want every test whose execution
*reaches* a changed node.

This is a **reachability problem**: given a set of changed nodes $C \subseteq V$, find all tests $t$ such
that there's a path in the (reversed) graph from $t$'s entry point to some node in $C$. This is solved with
a simple breadth-first search (BFS) — no exotic algorithm needed, just applied carefully.

### Python: a minimal RTS implementation

```python
from collections import deque, defaultdict

def build_reverse_graph(call_graph: dict[str, list[str]]) -> dict[str, list[str]]:
    """call_graph: caller -> [callees]. Returns callee -> [callers]."""
    reverse = defaultdict(list)
    for caller, callees in call_graph.items():
        for callee in callees:
            reverse[callee].append(caller)
    return reverse

def affected_by_change(call_graph: dict[str, list[str]], changed_nodes: set[str]) -> set[str]:
    """BFS backward from changed nodes to find everything that could be impacted."""
    reverse_graph = build_reverse_graph(call_graph)
    affected = set(changed_nodes)
    queue = deque(changed_nodes)
    while queue:
        node = queue.popleft()
        for caller in reverse_graph.get(node, []):
            if caller not in affected:
                affected.add(caller)
                queue.append(caller)
    return affected

def select_tests(test_entry_points: dict[str, str], affected_nodes: set[str]) -> list[str]:
    return [test for test, entry in test_entry_points.items() if entry in affected_nodes]

# Toy call graph: A calls B and C; B calls D
call_graph = {"A": ["B", "C"], "B": ["D"], "C": [], "D": []}
test_entry_points = {"test_A": "A", "test_B": "B", "test_C": "C", "test_D": "D"}

# Suppose D changed
affected = affected_by_change(call_graph, changed_nodes={"D"})
print("Affected nodes:", affected)                       # {D, B, A}  -- C is untouched
print("Tests to run:", select_tests(test_entry_points, affected))
# test_D, test_B, test_A get selected; test_C is correctly skipped
```

This is the literal mechanism behind "run only tests reachable from the diff" — everything downstream in
the call graph from the changed code gets pulled in via BFS; everything else is safely skipped.

---

## 2. Mutation Testing

### The core metric

$$\text{Mutation score} = \frac{\text{killed mutants}}{\text{total mutants} - \text{equivalent mutants}}$$

A mutant is "killed" if at least one test fails against it; it "survives" if every test still passes despite
the injected bug — meaning no test actually verifies that piece of behavior.

### Python: a tiny mutation testing harness

```python
import copy

def original_function(x, y):
    return x if x > y else y   # returns the max of x and y

def mutant_flip_comparison(x, y):
    return x if x < y else y   # BUG: flipped > to <

def mutant_off_by_one(x, y):
    return x if x >= y + 1 else y   # BUG: off-by-one in the comparison

# A test suite -- deliberately weak, to show mutation testing catching the gap
def existing_tests(func):
    results = []
    results.append(func(5, 3) == 5)   # only tests the "x is bigger" case
    return all(results)

candidates = {
    "original": original_function,
    "mutant_flip_comparison": mutant_flip_comparison,
    "mutant_off_by_one": mutant_off_by_one,
}

for name, func in candidates.items():
    if name == "original":
        continue
    killed = not existing_tests(func)
    print(f"{name}: {'KILLED' if killed else 'SURVIVED -- test gap found'}")
```

Run this: `mutant_flip_comparison` survives the weak test suite (since it only ever tests `x > y`), which is
exactly the kind of gap mutation testing is designed to surface — a real bug the test suite would miss.
Adding a second test case like `func(3, 5) == 5` would kill it.

---

## 3. Delta Debugging

### The algorithm, ddmin

Given a failing test case (e.g., a long input that crashes a program), delta debugging finds a smaller
input that *still* triggers the failure, by repeatedly splitting the input into chunks and testing whether
removing each chunk still reproduces the failure.

### Python: a simplified ddmin

```python
def ddmin(input_sequence: list, test_fails) -> list:
    """Simplified minimizing delta debugging.
    test_fails(seq) should return True if that sequence still reproduces the failure."""
    n = 2  # number of chunks to try splitting into
    current = input_sequence

    while len(current) >= 2:
        chunk_size = max(1, len(current) // n)
        chunks = [current[i:i + chunk_size] for i in range(0, len(current), chunk_size)]

        reduced_successfully = False
        for i in range(len(chunks)):
            candidate = [item for j, chunk in enumerate(chunks) if j != i for item in chunk]
            if candidate and test_fails(candidate):
                current = candidate
                n = max(n - 1, 2)
                reduced_successfully = True
                break

        if not reduced_successfully:
            if n >= len(current):
                break
            n = min(n * 2, len(current))

    return current

# Simulate: the failure is triggered ONLY if both 'X' and 'Y' are present, in any input
def test_fails(seq):
    return "X" in seq and "Y" in seq

failing_input = list("aaaXbbbYccc")
minimal = ddmin(failing_input, test_fails)
print(f"Original length: {len(failing_input)}, minimized: {''.join(minimal)}")
# Delta debugging narrows this down close to just ['X', 'Y'] -- the actual root cause
```

This minimized failing case is exactly what regression-test-strategy converts into a new permanent test —
small, precise, and directly tied to the real bug rather than the noisy original bug report.

---

## 4. Capture-Recapture (estimating remaining defects)

### The Lincoln-Petersen estimator

Borrowed directly from wildlife ecology: to estimate a population size $N$ without counting every member,
catch and mark a sample, release them, then catch a second sample and see how many are already marked
(recaptures). If marking and catching are independent and random:

$$\hat{N} = \frac{n_1 \times n_2}{m_2}$$

where $n_1$ is the first sample size, $n_2$ is the second sample size, and $m_2$ is how many in the second
sample were already marked (i.e. found in the first). Applied to defects: two independent reviewers each
find some bugs; $m_2$ is how many bugs *both* reviewers found.

```python
def lincoln_petersen_estimate(reviewer1_defects: set, reviewer2_defects: set) -> float:
    n1 = len(reviewer1_defects)
    n2 = len(reviewer2_defects)
    m2 = len(reviewer1_defects & reviewer2_defects)   # found by BOTH
    if m2 == 0:
        return float("inf")  # cannot estimate -- no overlap at all
    return (n1 * n2) / m2

reviewer1 = {"bug_A", "bug_B", "bug_C", "bug_D"}
reviewer2 = {"bug_B", "bug_C", "bug_E"}

estimate = lincoln_petersen_estimate(reviewer1, reviewer2)
total_found = len(reviewer1 | reviewer2)
print(f"Total unique defects found so far: {total_found}")
print(f"Estimated true total defects: {estimate:.1f}")
print(f"Estimated defects still undiscovered: {estimate - total_found:.1f}")
```

A **small overlap** between reviewers ($m_2$ small relative to $n_1, n_2$) implies a *large* estimated total
population — meaning the reviewers are each finding mostly different bugs, which suggests there are still
many more undiscovered. A **large overlap** implies you're close to having found everything.

---

## 5. The Goel-Okumoto Reliability Growth Model

### The math

This models cumulative defects found by time $t$ as a non-homogeneous Poisson process with mean value
function:

$$m(t) = a \left(1 - e^{-bt}\right)$$

where $a$ is the expected total number of defects that will ever be found (as $t \to \infty$), and $b$ is
the rate at which they're discovered. As $t$ grows, $m(t)$ approaches $a$ and flattens — that flattening is
the "diminishing returns" signal.

### Python: fitting the curve to observed data

```python
import numpy as np
from scipy.optimize import curve_fit

def goel_okumoto(t, a, b):
    return a * (1 - np.exp(-b * t))

# Simulated observed cumulative defects found over 10 weeks of testing
weeks = np.array([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
cumulative_defects = np.array([8, 14, 19, 22, 24, 25.5, 26.5, 27, 27.3, 27.5])

(a_fit, b_fit), _ = curve_fit(goel_okumoto, weeks, cumulative_defects, p0=[30, 0.3])
print(f"Estimated total defects (a): {a_fit:.1f}")
print(f"Discovery rate (b): {b_fit:.3f}")

# Marginal value of one more week of testing right now, vs. earlier
for t in [2, 8]:
    marginal_gain = goel_okumoto(t + 1, a_fit, b_fit) - goel_okumoto(t, a_fit, b_fit)
    print(f"Expected new defects found in week {t+1}: {marginal_gain:.2f}")
```

Early on, one more week of testing finds several new defects; later, the marginal gain shrinks toward zero
— that shrinking marginal gain is the quantitative version of "stop adding tests here."

---

## Putting it together

| Part | Tool |
|---|---|
| Selecting existing tests to run | BFS reachability over the dependency graph |
| Generating new tests (finding gaps) | mutation testing's kill/survive signal |
| Codifying found bugs | delta debugging's ddmin |
| Estimating remaining unknown defects | Lincoln-Petersen capture-recapture |
| Deciding when to stop testing | Goel-Okumoto curve fitting + mutation score plateau |
