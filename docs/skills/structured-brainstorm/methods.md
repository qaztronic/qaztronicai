# The Math and Methods Behind structured-brainstorm

This document explains the mechanics behind each phase of the structured-brainstorm skill, with the
underlying math and small Python examples you can run yourself.

---

## 1. Divergent and Convergent Thinking (the two-phase structure)

Guilford's distinction isn't a formula, but it's worth being precise about what each mode is actually
optimizing for, because it explains why doing them in the wrong order (or skipping one) breaks brainstorms.

- **Divergent thinking** optimizes for *coverage of the possibility space* — you want to sample broadly
  before you know which region is best.
- **Convergent thinking** optimizes for *selection under a criterion* — you want to pick the best point
  from a space you've already sampled.

The failure mode of most brainstorming is running convergent thinking on a divergent-sized problem: judging
and picking too early, from too small a sample, so the "best" idea found is really just "the best of the
first five ideas that came to mind" rather than the best of the real space.

A simple way to see why sample size matters: if good ideas are rare (say, only 10% of possible ideas in a
category are actually strong), the probability you've seen at least one strong idea after generating $n$
ideas is:

$$P(\text{at least one strong idea}) = 1 - (1 - p)^n$$

```python
def prob_at_least_one_strong(p_strong: float, n_ideas: int) -> float:
    return 1 - (1 - p_strong) ** n_ideas

for n in [1, 3, 5, 10]:
    print(f"n={n:2d} ideas -> P(found a strong one) = {prob_at_least_one_strong(0.10, n):.1%}")
```

At $n=1$ you have a 10% chance of even having a strong idea in front of you to pick. At $n=10$, you're
above 65%. This is the quantitative case for the skill's "3-5 ideas per category" quota — stopping at one
or two per category is stopping before the sampling has done its job.

---

## 2. Thompson Sampling (why the skill keeps uncertain, high-upside ideas)

### The problem it solves

Imagine each idea is a "slot machine arm" with an unknown true quality. You want to identify the best one,
but you only have limited attention (analogous to "which ideas do I keep exploring"). A greedy strategy —
always picking whatever currently looks best — will systematically starve out ideas that are *actually*
better but happened to look mediocre on first impression.

### The math

Thompson sampling models your belief about each idea's quality as a probability distribution, not a point
estimate. For a simple version — each idea either "succeeds" (turns out great) or "fails" when evaluated —
the natural distribution is the **Beta distribution**, which is the conjugate prior for a Bernoulli/binomial
process:

$$\theta_i \sim \text{Beta}(\alpha_i, \beta_i)$$

where $\alpha_i$ is roughly "successes so far + 1" and $\beta_i$ is "failures so far + 1" for idea $i$. To
choose which idea to favor next, you *sample* a value from each idea's distribution and pick whichever
sample is highest — not whichever idea has the highest average so far.

```python
import numpy as np
rng = np.random.default_rng(2)

class Idea:
    def __init__(self, name, true_quality):
        self.name = name
        self.true_quality = true_quality   # unknown to the algorithm, used only to simulate feedback
        self.successes = 1                 # alpha (starts at 1: uninformative prior)
        self.failures = 1                  # beta

    def sample_belief(self):
        return rng.beta(self.successes, self.failures)

    def observe_feedback(self):
        # Simulate evaluating the idea once
        if rng.random() < self.true_quality:
            self.successes += 1
        else:
            self.failures += 1

ideas = [
    Idea("Idea A (looked great early)", true_quality=0.55),
    Idea("Idea B (looked mediocre early, actually strong)", true_quality=0.80),
    Idea("Idea C (consistently weak)", true_quality=0.20),
]

# Simulate 30 rounds of "evaluate whichever idea currently samples highest"
for round_num in range(30):
    sampled = {idea.name: idea.sample_belief() for idea in ideas}
    chosen = max(ideas, key=lambda i: i.sample_belief())
    chosen.observe_feedback()

for idea in ideas:
    mean_belief = idea.successes / (idea.successes + idea.failures)
    print(f"{idea.name}: estimated quality={mean_belief:.2f} (true={idea.true_quality})")
```

Run this a few times: Idea B, despite not looking best on the very first evaluation, keeps getting sampled
often enough (because its distribution has real probability mass on high values) that the algorithm
discovers its true strength — instead of a greedy top-1 approach permanently writing it off after one bad
early look.

---

## 3. Simulated Annealing (why narrowing happens gradually, not in one cut)

### The physical analogy

When metal is cooled quickly, atoms freeze into whatever position they're in — often a defective structure.
Cooled slowly, atoms have time to settle into a lower-energy, more stable arrangement. Simulated annealing
borrows this: early in a search, allow "worse" moves sometimes (to escape a false-looking local optimum);
late in the search, only accept improvements.

### The math

At a given "temperature" $T$, a worse candidate is accepted with probability:

$$P(\text{accept}) = \exp\left(-\frac{\Delta E}{T}\right)$$

where $\Delta E$ is how much worse the new candidate is. At high $T$, this probability is close to 1 (accept
almost anything, i.e. stay permissive — analogous to the skill's first, permissive narrowing pass). As $T$
cools toward 0, this probability collapses to nearly 0 for any negative move (i.e. become strict — analogous
to the skill's final pass).

```python
import numpy as np
rng = np.random.default_rng(3)

def score(idea_quality):
    return idea_quality  # higher is better

def simulated_annealing_narrow(ideas_with_quality, initial_temp=1.0, cooling_rate=0.85, steps=20):
    # Choose by INDEX, not by value -- rng.choice on a list of (str, float) tuples
    # would coerce everything to a single numpy string dtype and break the math below.
    n = len(ideas_with_quality)
    current = ideas_with_quality[rng.integers(n)]
    best = current
    temp = initial_temp

    for step in range(steps):
        candidate = ideas_with_quality[rng.integers(n)]
        delta = score(candidate[1]) - score(current[1])  # positive = candidate is better

        if delta > 0 or rng.random() < np.exp(delta / temp):
            current = candidate
            if score(current[1]) > score(best[1]):
                best = current

        temp *= cooling_rate  # cool down: become stricter over time

    return best

# (name, quality) pairs -- quality is a stand-in for a rough qualitative score
ideas = [("A", 0.4), ("B", 0.9), ("C", 0.3), ("D", 0.6), ("E", 0.2)]
result = simulated_annealing_narrow(ideas)
print(f"Selected: {result}")
```

The key behavioral point: early in the loop (`temp` still high), the algorithm sometimes moves to a
*worse*-looking candidate — exactly mirroring the skill's instruction to keep boundary cases in on the
first narrowing pass rather than cutting immediately to only the top scorers.

---

## 4. Mix-and-Match Recombination (Phase 2.5)

There's no single named algorithm for this step, but the underlying operation is combinatorics: given $n$
surviving ideas, the number of possible pairs to check for a good combination is:

$$\binom{n}{2} = \frac{n!}{2!(n-2)!}$$

This grows quickly, which is exactly why the skill says "scan for complementary pairs" rather than
"check every possible pair" — for a shortlist of 8 ideas, that's already 28 pairs, and real combination
value is rare, so an exhaustive check isn't worth it. Instead, a cheap heuristic filter (do these two ideas
solve *different* parts of the problem?) narrows the search before generating hybrids.

```python
from itertools import combinations

def find_candidate_pairs(ideas: dict[str, str]) -> list[tuple[str, str]]:
    """ideas: name -> which 'problem dimension' it addresses.
    A candidate pair is two ideas that address DIFFERENT dimensions
    (same-dimension ideas are usually substitutes, not complements)."""
    candidates = []
    for (name_a, dim_a), (name_b, dim_b) in combinations(ideas.items(), 2):
        if dim_a != dim_b:
            candidates.append((name_a, name_b))
    return candidates

ideas = {
    "Catchy name": "branding",
    "Punny name": "branding",
    "Freemium pricing": "monetization",
    "Referral program": "growth",
}
print(find_candidate_pairs(ideas))
# Only cross-dimension pairs are surfaced as real hybrid candidates —
# branding+branding pairs are filtered out since they're substitutes, not combinations.
```

---

## Putting it together

| Phase | Tool |
|---|---|
| 1. Divergence | coverage-by-category sampling — the $1-(1-p)^n$ logic for why quotas matter |
| 2. Convergence (scoring) | qualitative High/Medium/Low — no heavy math, deliberately, to avoid false precision |
| 2. Convergence (keep upside) | Thompson sampling logic |
| 2. Convergence (gradual cuts) | simulated annealing's cooling schedule |
| 2.5 Recombination | pairwise combinatorics, filtered by cross-dimension heuristic |
