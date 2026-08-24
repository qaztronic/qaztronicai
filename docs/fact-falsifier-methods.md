# The Math and Methods Behind fact-falsifier

This document walks through the actual mechanics — mathematical and algorithmic — behind each stage of the
fact-falsifier skill. Each section builds the idea from first principles, then shows a small, runnable
Python example so you can see the mechanism directly rather than just take it on faith.

---

## 1. Toulmin's Argument Model (assumption mining)

Toulmin's model isn't mathematical — it's a structural decomposition — but it's worth formalizing because
the skill uses it as a parsing step. Think of an argument as a small directed graph:

```
Grounds ──warrant──> Claim
              ^
              |
           Backing
              |
          Rebuttal (attacks the warrant or the claim)
```

The key move is that **grounds alone never support a claim** — you always need a warrant, an inference rule
that licenses the jump from "here's some evidence" to "therefore, this is true." Most bad arguments aren't
wrong about their grounds; they're silent about a shaky warrant.

**Worked example:**
- Claim: "This drug is safe for widespread use."
- Grounds: "In the trial, 95% of patients had no adverse reaction."
- Warrant (usually unstated): "A 95% no-reaction rate in a trial population generalizes to the broader
  population who will use the drug."
- This warrant is exactly where you'd probe: was the trial population representative? Was 95% actually
  a strong result for this drug class, or unusually weak? That's the rebuttal question.

A tiny Python representation, just to make the structure concrete and checkable:

```python
from dataclasses import dataclass, field

@dataclass
class ToulminArgument:
    claim: str
    grounds: list[str]
    warrant: str
    backing: str = ""
    rebuttals: list[str] = field(default_factory=list)

    def is_load_bearing(self) -> bool:
        """An argument is 'load-bearing' if it has a warrant that hasn't been backed."""
        return self.warrant != "" and self.backing == ""

arg = ToulminArgument(
    claim="This drug is safe for widespread use.",
    grounds=["95% of trial patients had no adverse reaction."],
    warrant="A 95% no-reaction trial rate generalizes to the general population.",
)
print(arg.is_load_bearing())  # True — this is exactly what fact-falsifier should flag
```

---

## 2. Bayesian Updating and Bayes Factors

This is the actual mathematical engine behind "confidence, not verdicts."

### Bayes' theorem, derived

Start from the definition of conditional probability:

$$P(H \mid E) = \frac{P(E \mid H) \, P(H)}{P(E)}$$

where $H$ is a hypothesis (e.g., "the claim is true") and $E$ is evidence. $P(H)$ is your **prior** belief,
$P(H \mid E)$ is your **posterior** belief after seeing the evidence, and $P(E \mid H)$ is the
**likelihood** — how probable the evidence is if the hypothesis is true.

### The odds form (this is what fact-falsifier actually tracks)

Compare two competing hypotheses $H_0$ (claim is false) and $H_1$ (claim is true). Divide their two
posteriors:

$$\underbrace{\frac{P(H_1 \mid E)}{P(H_0 \mid E)}}_{\text{posterior odds}} = \underbrace{\frac{P(H_1)}{P(H_0)}}_{\text{prior odds}} \times \underbrace{\frac{P(E \mid H_1)}{P(E \mid H_0)}}_{\text{Bayes factor}}$$

The **Bayes factor (BF)** is the piece that represents "how much did this specific evidence move me?" A BF
of 10 means the evidence is 10 times more likely under $H_1$ than under $H_0$. This is exactly why the
skill never says "confirmed" — it just multiplies the running odds by each new piece of evidence.

**Rule of thumb interpretation** (Jeffreys' scale, roughly): BF of 1–3 is barely worth mentioning, 3–10 is
moderate evidence, 10–30 is strong, 30+ is very strong.

### Python: tracking belief across multiple pieces of evidence

```python
def update_odds(prior_odds: float, bayes_factor: float) -> float:
    """One Bayesian update step."""
    return prior_odds * bayes_factor

# Start neutral: 1:1 odds means "no idea yet"
odds = 1.0

# Evidence 1: a single corroborating source. Weak evidence -> small Bayes factor.
odds = update_odds(odds, bayes_factor=2.0)

# Evidence 2: an independent second source agrees. Stronger evidence now.
odds = update_odds(odds, bayes_factor=4.0)

# Evidence 3: a contradicting source appears. This pulls the odds back down.
odds = update_odds(odds, bayes_factor=0.3)

# Convert odds back to a probability for a human-readable confidence level
posterior_probability = odds / (1 + odds)
print(f"Final odds: {odds:.2f}:1, confidence: {posterior_probability:.1%}")
```

This is the literal computation behind phrases like "still standing" (odds staying near or above 1) versus
"weakened" (odds dropping toward or below 1) in the skill's output.

---

## 3. Causal Graphs and the Backdoor Criterion

When a claim says "X causes Y," the question is whether the observed association between X and Y is real,
or driven by something else entirely.

### Confounding, formally

A confounder $Z$ is a variable that causes both $X$ and $Y$. If you don't account for it, you'll see a
correlation between $X$ and $Y$ even if $X$ has zero causal effect on $Y$.

```
      Z (confounder)
     ╱ ╲
    ╱   ╲
   X     Y     <- X and Y are correlated purely through Z
```

### The backdoor criterion (Pearl)

If you can find a set of variables $Z$ that "blocks" every non-causal (backdoor) path from $X$ to $Y$
without blocking the real causal path, then you can compute the true causal effect by **adjusting** for
$Z$:

$$P(Y \mid \text{do}(X = x)) = \sum_{z} P(Y \mid X = x, Z = z) \, P(Z = z)$$

This says: instead of just looking at $P(Y \mid X=x)$ (which mixes in the confounding path), average the
effect of $X$ within each level of $Z$, then weight by how common that level of $Z$ is.

### Python: simulating confounding, then correcting for it

```python
import numpy as np
rng = np.random.default_rng(0)

n = 20_000
Z = rng.binomial(1, 0.5, n)                       # confounder: e.g. "patient is high-risk"
X = rng.binomial(1, 0.3 + 0.4 * Z, n)              # X depends on Z (e.g. "given the drug")
# Y depends on Z strongly, and X has ZERO true causal effect on Y here
Y = rng.binomial(1, 0.2 + 0.5 * Z, n)

# NAIVE (biased) estimate: just compare Y between X=1 and X=0
naive_effect = Y[X == 1].mean() - Y[X == 0].mean()

# ADJUSTED (backdoor) estimate: average within each level of Z, weighted by P(Z)
adjusted_effect = 0.0
for z in (0, 1):
    mask_z = Z == z
    p_z = mask_z.mean()
    effect_given_z = Y[mask_z & (X == 1)].mean() - Y[mask_z & (X == 0)].mean()
    adjusted_effect += p_z * effect_given_z

print(f"Naive (confounded) effect estimate:   {naive_effect:.3f}")
print(f"Backdoor-adjusted effect estimate:    {adjusted_effect:.3f}")
# The naive estimate will show a spurious positive effect;
# the adjusted estimate correctly lands near zero.
```

Run this and you'll see the naive estimate is misleadingly positive, while the adjusted one correctly
reveals there's no real effect — exactly the trap fact-falsifier is checking for when it asks "could a
confounder explain this?"

---

## 4. Expected Value of Information (EVOI)

This is the math behind deciding *which* claims are actually worth the effort of testing.

### The formula

$$\text{EVPI} = \mathbb{E}_{\text{state}}\left[\max_{\text{action}} \; \text{payoff}(\text{action}, \text{state})\right] - \max_{\text{action}} \; \mathbb{E}_{\text{state}}\left[\text{payoff}(\text{action}, \text{state})\right]$$

In words: EVPI (expected value of perfect information) is the difference between (a) the payoff you'd get
if you always knew the true state before deciding, versus (b) the payoff of the single best decision you'd
make without that knowledge. If that gap is small, more information isn't worth much — the claim isn't
load-bearing enough to test further.

### Python: a small decision table

```python
import numpy as np

# Two possible states of the world, with our current belief about their probability
states = ["fact_true", "fact_false"]
state_probs = {"fact_true": 0.6, "fact_false": 0.4}

# Two possible actions we could take, and the payoff of each action under each state
payoff = {
    ("proceed_with_plan", "fact_true"):  100,
    ("proceed_with_plan", "fact_false"): -80,
    ("hedge_the_plan",    "fact_true"):   40,
    ("hedge_the_plan",    "fact_false"):  20,
}
actions = ["proceed_with_plan", "hedge_the_plan"]

# (a) Expected payoff of the best action WITHOUT extra information
expected_payoffs = {
    a: sum(state_probs[s] * payoff[(a, s)] for s in states)
    for a in actions
}
best_without_info = max(expected_payoffs.values())

# (b) Expected payoff IF we always knew the true state first
with_perfect_info = sum(
    state_probs[s] * max(payoff[(a, s)] for a in actions)
    for s in states
)

evpi = with_perfect_info - best_without_info
print(f"Best action without more info: {max(expected_payoffs, key=expected_payoffs.get)}")
print(f"EVPI (value of resolving this claim): {evpi:.1f}")
```

If EVPI comes out small, that tells fact-falsifier this particular claim isn't worth a deep falsification
pass — the decision wouldn't change much even if you were certain about it. A large EVPI is the signal to
spend real effort.

---

## 5. The Delphi Method (for unfalsifiable judgment calls)

For forecasts and value judgments with no ground truth to test against, the skill substitutes structured
consensus-building instead of falsification.

### The mechanism

Delphi doesn't have a single formula — it's an iterative protocol:

1. Each independent expert gives an estimate, without seeing anyone else's.
2. The estimates are aggregated (commonly the median) and shown back to everyone, anonymized.
3. Each expert revises their estimate in light of the group's spread.
4. Repeat until the spread of estimates stabilizes (commonly measured by the interquartile range, IQR).

The interesting mathematical property is what's tracked as the stopping condition: not agreement on a
single number, but the **shrinking of disagreement** round over round.

### Python: simulating convergence

```python
import numpy as np
rng = np.random.default_rng(1)

n_experts = 7
true_unknown_value = 100  # the experts don't know this; we use it only to simulate their behavior

# Round 1: independent, noisy estimates
estimates = true_unknown_value + rng.normal(0, 25, n_experts)

def iqr(values):
    q75, q25 = np.percentile(values, [75, 25])
    return q75 - q25

for round_num in range(1, 5):
    group_median = np.median(estimates)
    spread = iqr(estimates)
    print(f"Round {round_num}: median={group_median:.1f}, IQR={spread:.1f}")

    # Each expert nudges 40% of the way toward the group median, plus fresh small noise
    estimates = estimates + 0.4 * (group_median - estimates) + rng.normal(0, 5, n_experts)

    if spread < 5:
        print("Consensus reached — stopping.")
        break
```

Running this shows the IQR shrinking round over round — that shrinking spread, not a forced single answer,
is what the skill treats as "satisfactory consensus."

---

## Putting it together

fact-falsifier's five stages map onto these tools directly:

| Stage | Tool |
|---|---|
| Extraction | plain restatement — no math needed |
| Assumption mining | Toulmin decomposition |
| Adversarial generation | backdoor criterion / confounder search |
| Falsification design | whichever test is cheapest given the claim type |
| Confidence scoring & prioritization | Bayes factor updating (score) + EVOI (prioritize); Delphi for judgment calls |
