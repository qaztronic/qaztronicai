# The Math and Methods Behind research-strategy

This document covers the mechanics behind planning, corroborating, and scoring research findings, with
Python examples for each piece. Two steps (coverage planning and claim falsification) are delegated
entirely to other skills in the suite — see `structured-brainstorm-methods.md` and
`fact-falsifier-methods.md` for those. This document covers what's unique to research-strategy itself.

---

## 1. Source Triangulation

### The core idea, formalized

Triangulation's claim is simple: independent sources agreeing is stronger evidence than one source alone,
*if* the sources are actually independent (not all copying from the same original report). The naive way
to think about this is just counting corroborating sources, but that's misleading if the sources aren't
independent — three news outlets citing the same single press release isn't three pieces of evidence, it's
one.

A more careful model: if each source has an independent probability $p$ of correctly reporting a true
claim (and independently, some smaller probability of incorrectly reporting a false one as true), then the
probability that $k$ out of $n$ independent sources all agree on a true claim, by chance, drops fast as
$k$ grows — this is exactly why independent corroboration is strong evidence, and non-independent
corroboration is not.

### Python: why independence matters

```python
import numpy as np
rng = np.random.default_rng(4)

def simulate_corroboration(n_sources: int, p_independent_error: float, independent: bool, trials=50_000):
    """Estimate P(all sources agree on a FALSE claim by chance)."""
    false_positive_agreements = 0
    for _ in range(trials):
        if independent:
            # Each source independently has a small chance of being wrong
            errors = rng.random(n_sources) < p_independent_error
        else:
            # All sources share a single upstream error (e.g. same wire report) --
            # if the original is wrong, ALL downstream sources repeat the same error
            shared_error = rng.random() < p_independent_error
            errors = np.array([shared_error] * n_sources)

        if errors.all():
            false_positive_agreements += 1

    return false_positive_agreements / trials

p_err = 0.05
print("3 sources, truly independent:     ",
      simulate_corroboration(3, p_err, independent=True))
print("3 sources, all copying one origin:",
      simulate_corroboration(3, p_err, independent=False))
```

Run this: with genuinely independent sources, the chance that all three are simultaneously wrong is tiny
($0.05^3 \approx 0.0001$). With sources that share a single upstream origin, the chance is just $0.05$ —
five hundred times higher. This is the quantitative reason the skill's triangulation check specifically
asks for *unrelated* sources, not just *multiple* sources.

---

## 2. Recency Weighting

### The model: exponential decay of relevance

For time-sensitive facts, older sources should count for less. A standard way to formalize "less" is
exponential decay:

$$w(t) = e^{-\lambda (t_{\text{now}} - t_{\text{source}})}$$

where $\lambda$ controls how fast relevance decays (bigger $\lambda$ = faster-moving topic, shorter useful
shelf life for a source) and $t_{\text{now}} - t_{\text{source}}$ is the age of the source.

```python
import numpy as np
import datetime as dt

def recency_weight(source_date: dt.date, today: dt.date, half_life_days: float) -> float:
    """half_life_days: time after which a source's weight drops to 50%."""
    age_days = (today - source_date).days
    decay_rate = np.log(2) / half_life_days
    return np.exp(-decay_rate * age_days)

today = dt.date(2026, 8, 15)
sources = {
    "Company blog post":      dt.date(2026, 8, 1),
    "News article":           dt.date(2026, 6, 1),
    "Old industry report":    dt.date(2024, 1, 1),
}

# For a fast-moving topic (e.g. "current AI model rankings"), use a short half-life
for name, date in sources.items():
    w = recency_weight(date, today, half_life_days=90)
    print(f"{name}: weight = {w:.3f}")
```

A slower-moving topic (say, "how a well-established scientific process works") would use a much longer
half-life, since old sources there haven't actually lost much relevance — the skill's instruction to only
apply recency-weighting to genuinely time-sensitive claims is this choice of $\lambda$ (or half-life) being
topic-dependent, not a universal constant.

---

## 3. Citation Snowball Sampling

### The model: graph traversal (again, but on citations instead of code)

Just like the call-graph reachability in regression-test-strategy, snowballing is BFS — but over a citation
graph instead of a code dependency graph. Backward snowballing follows a source's references; forward
snowballing follows what cites that source.

```python
from collections import deque

def snowball(seed_sources: set[str],
             references_of: dict[str, list[str]],   # backward: what a source cites
             cited_by: dict[str, list[str]],          # forward: what cites a source
             max_hops: int = 2) -> set[str]:
    found = set(seed_sources)
    frontier = deque((s, 0) for s in seed_sources)

    while frontier:
        source, hops = frontier.popleft()
        if hops >= max_hops:
            continue
        neighbors = references_of.get(source, []) + cited_by.get(source, [])
        for neighbor in neighbors:
            if neighbor not in found:
                found.add(neighbor)
                frontier.append((neighbor, hops + 1))

    return found

references_of = {
    "SeedPaper": ["FoundationalWork1", "FoundationalWork2"],
    "FoundationalWork1": ["EvenOlderWork"],
}
cited_by = {
    "SeedPaper": ["RecentFollowUp"],
    "RecentFollowUp": ["VeryRecentCritique"],
}

result = snowball({"SeedPaper"}, references_of, cited_by, max_hops=2)
print(sorted(result))
```

Note `max_hops` caps how far the search spreads — exactly the skill's instruction to only apply
snowballing to dimensions that turn up thin or contested results, rather than following every source's
entire citation web indefinitely (which would grow explosively).

---

## 4. Weight-of-Evidence Synthesis

### The math: log-odds weight of evidence

This is a direct extension of the Bayes factor idea from fact-falsifier, but expressed in a form
specifically designed for combining many pieces of evidence additively. The **weight of evidence** for a
single piece of evidence $E$ toward hypothesis $H$ is defined as:

$$W(E) = \log\left(\frac{P(E \mid H)}{P(E \mid \neg H)}\right)$$

The convenient property: weights of evidence from *independent* sources simply **add up**, which makes them
easy to accumulate into an overall confidence score across many sources of different strength.

```python
import numpy as np

def weight_of_evidence(p_given_true: float, p_given_false: float) -> float:
    return np.log(p_given_true / p_given_false)

# Each source has a different reliability profile, expressed as
# (P(source says X | X is true), P(source says X | X is false))
sources = {
    "Peer-reviewed study":     (0.95, 0.05),
    "Reputable news outlet":   (0.85, 0.20),
    "Anonymous forum post":    (0.60, 0.45),
    "Contradicting expert":    (0.10, 0.90),   # this one argues AGAINST the claim
}

total_weight = sum(weight_of_evidence(*profile) for profile in sources.values())
print(f"Total weight of evidence: {total_weight:.2f}")

# Convert total weight back to a probability, given a neutral 50/50 starting prior
prior_odds = 1.0
posterior_odds = prior_odds * np.exp(total_weight)
posterior_prob = posterior_odds / (1 + posterior_odds)
print(f"Posterior confidence: {posterior_prob:.1%}")
```

This is what the skill's "confidence read per finding" actually is under the hood: not a vague feeling, but
an accumulation of log-odds weights from each source, where strong independent corroboration adds a lot,
weak sources add a little, and a credible contradicting source actively subtracts.

---

## Putting it together

| Step | Tool |
|---|---|
| Coverage planning | delegated to structured-brainstorm (see that document) |
| Iterative query expansion | no fixed formula — reformulation heuristics |
| Source triangulation | independence-adjusted agreement probability |
| Recency weighting | exponential decay, half-life chosen by topic volatility |
| Citation snowballing | BFS over the citation graph, hop-limited |
| Claim falsification | delegated to fact-falsifier (see that document) |
| Evidence synthesis | weight-of-evidence (additive log-odds) |
