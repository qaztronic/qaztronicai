---
name: fact-falsifier
description: Rigorously stress-tests a list of claimed "facts" by extracting their hidden assumptions and designing concrete experiments to prove or falsify each one. Use this skill whenever the user gives you a list of facts, claims, assertions, or "things we believe" and asks you to verify them, find the assumptions behind them, check if they're true, poke holes in them, be skeptical/doubtful about them, or design tests/experiments to confirm or disprove them. Also trigger on phrases like "fact check this list," "what are we assuming here," "stress test these claims," "red team this," or "how would we know if this were wrong." Make sure to use this skill even if the user only gives one or two facts rather than a long list — the same rigor applies at any scale.
---

# Fact Falsifier

A skill for taking a list of asserted facts and subjecting each one to structured doubt: decompose it into
its load-bearing assumptions, construct the strongest case against it, and design the cheapest experiment
that could prove it wrong. The default posture is **aggressively skeptical** — a fact is "unfalsified so far,"
never "confirmed," until real evidence has been checked against a real prediction.

## When to use this

Trigger any time the user hands you:
- A list of facts, claims, or assumptions to verify
- A report, pitch deck, or plan whose factual claims need to be checked
- A single strong claim they want pressure-tested
- Any request phrased as "is this actually true," "what would have to be true for this to hold," or
  "how would I know if I'm wrong"

Don't wait for a long list — even a single claim gets the full pipeline, just scaled down.

## The pipeline

Run these five stages in order for every fact. Stages 1–2 can be done for all facts in a batch; stages 3–5
are usually done fact-by-fact since the tests differ.

### Stage 1 — Fact extraction (make it falsifiable)

For each item on the user's list:
- Restate it as a single, specific, checkable claim. If it's vague ("many," "significant," "often"),
  either tighten it with the user or flag it explicitly as untestable-as-stated and propose a sharper version.
- Note the claim type, since this determines what stage 3 looks like:
  - **Empirical/measurable** (a number, a rate, an existence claim)
  - **Causal** ("X causes/drives/leads to Y")
  - **Definitional/conceptual** (depends on how a term is used)
  - **Forecast/judgment** (about the future, or a value call with no ground truth yet)

### Stage 2 — Assumption mining (Toulmin decomposition)

For each fact, break it into the classic argument components. Do this explicitly, in a table or list —
don't skip straight to a verdict:
- **Claim**: the fact as restated in stage 1
- **Grounds**: what evidence/data is actually being pointed to
- **Warrant**: the unstated logical bridge connecting the grounds to the claim (this is almost always
  where the real assumption lives)
- **Backing**: why the warrant itself should be trusted
- **Rebuttal conditions**: circumstances under which the claim would NOT hold even if the grounds are true

Also explicitly check each fact for these five assumption categories, since they're the most common places
claims quietly fail:
1. **Definitional** — are terms used the way the source/user means them?
2. **Causal** — could the relationship run backward, be bidirectional, or be confounded by a third factor?
3. **Generalization** — does this hold outside the sample, population, time period, or context it came from?
4. **Measurement** — is the method/instrument that produced this fact actually valid for what it's claiming?
5. **Temporal** — is this still true, or was it true once and treated as permanent?

List every assumption uncovered — a single fact often has 2-4 load-bearing ones.

### Stage 3 — Adversarial generation (red team)

For each assumption (not just each fact — each individual assumption), do a two-pass adversarial exercise:
1. **Steelman the opposition**: write the strongest real argument or alternative explanation that would
   make this assumption false. Use actual mechanisms — known biases (selection, survivorship, publication,
   confirmation), plausible confounders, base-rate comparisons, alternative causal directions — not a token
   objection. If you can't construct a real counter-argument, say so explicitly; that's useful information,
   not a failure.
2. **Causal check** (for causal claims specifically): sketch the causal structure informally — what else
   could produce the same observed pattern? Reverse causation? A common cause driving both? Selection into
   the sample? Name the specific alternative graph, not just "there could be confounders."
3. **Critique your own critique**: is the counter-argument actually strong, or is it a weak objection dressed
   up as skepticism? Rate it.

### Stage 4 — Falsification design

For each assumption, design the minimal test that could actually move the needle:
- State the **prediction**: what specific, observable result would this assumption predict?
- State the **falsifying result**: what specific, observable result would prove it wrong?
- State the **test**: the cheapest concrete way to get that observation — a specific search, a data pull,
  a natural experiment already in existing data, a controlled comparison, a check against a known base rate.
  Prefer tests that can be run now (web search, data analysis, document lookup) over ones that require new
  data collection, and say so when a cheap test exists.
- If the claim is a **forecast/judgment call with no ground truth available** (stage 1 type), swap the
  falsification test for a **Delphi-style check**: what would independent, knowledgeable people estimate,
  and how much do estimates disagree? Disagreement itself is informative.
- Actually run the test if you have the tools to do so (web_search, code execution, file reading) rather
  than only describing it. Don't just propose a test when you could execute it directly.

### Stage 5 — Confidence scoring and prioritization

After running or proposing tests:
- Give each fact a **Bayesian-style confidence read**: not a verdict, but a posterior sense of how
  confident to be given what stage 3–4 turned up — e.g. "still standing, no real counter-evidence found,"
  "weakened — plausible alternative explanation exists," "likely false — falsifying evidence found,"
  "untestable as stated." Never round this up to a clean "confirmed."
- Rank the facts (or their assumptions) by **expected value of further testing**: which ones would change
  a real decision if they flipped, versus which are decorative and not worth more effort? Spend remaining
  effort on the load-bearing ones first, and say explicitly which facts are low-stakes enough not to
  bother testing further.

## Output format

Default to a compact table per fact, then a short synthesis. For each fact:

| Fact | Type | Key assumption(s) | Strongest counter-argument | Test performed/proposed | Result | Confidence |
|---|---|---|---|---|---|---|

Then close with:
- **Facts that survived scrutiny** (still standing, no real counter-evidence)
- **Facts that are weakened or falsified**, with the specific evidence
- **Facts that are untestable as stated** and what would make them testable
- **Where to spend more effort next**, ranked by expected value of information

Keep the tone matter-of-fact and specific — cite mechanisms and evidence, not just "this could be wrong."
Avoid hedging every line into mush; the goal is sharp, falsifiable statements, not diplomatic vagueness.

## Notes on rigor vs. speed

- For a short list (1-5 facts) or high-stakes claims, run the full five-stage pipeline per fact.
- For a long list (10+), it's fine to batch stages 1-2 across all facts first, then triage: run stages 3-5
  in full only on the facts that are load-bearing for whatever decision the user is making, and give a
  lighter pass (assumptions + one obvious counter-argument, no full test design) to the rest — but say
  explicitly which facts got the light treatment so the user can ask for more.
- Always use available tools (web_search, code execution) to actually attempt falsification tests rather
  than only describing what a test would look like, whenever the fact is checkable that way.
