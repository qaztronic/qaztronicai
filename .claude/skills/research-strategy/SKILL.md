---
name: research-strategy
description: Use this skill whenever the user asks for research, a research report, an investigation into a topic, or wants to know something that requires gathering and synthesizing information from multiple sources — e.g. "research X," "look into Y," "find out about Z," "what does the evidence say about...," "compile a report on...". Also trigger for competitive analysis, market research, literature reviews, due diligence, or any "find out what's true/current about this" request. This skill structures the research process itself (coverage planning, source triangulation, evidence scoring) rather than just firing off searches and summarizing the first results. Do NOT use for a single quick factual lookup with an obvious answer (e.g. "what's the capital of France") — go straight to a direct answer for those.
---

# Research Strategy

A skill for running research as a structured process — deliberate coverage of the question's dimensions,
continuous corroboration as evidence comes in, and confidence-scored synthesis at the end — instead of
firing off searches and summarizing whatever comes back first.

## Core stance

Shallow research clusters around whatever the first few searches surface and treats every source as equally
trustworthy. This skill fixes both: force coverage of the question's real dimensions before searching, and
never present a claim without an implicit or explicit read on how well-corroborated it is.

## Step 1 — Define coverage before searching (delegate to structured-brainstorm)

Before running any searches, use the **structured-brainstorm** skill's divergence phase to define the
dimensions of the research question — treat "what are the distinct angles this question needs covering
from" exactly like brainstorming idea categories. For example:
- "Should we enter market X" → competitive landscape, regulatory environment, customer demand, unit
  economics, timing risk
- "Is technology Y mature enough to adopt" → current state of the art, known failure modes, adoption by
  comparable orgs, cost/switching burden, trajectory of active development

Generate this dimension list explicitly and show it before searching — this is the research plan, not an
afterthought. If structured-brainstorm isn't available, still do this step manually: name 4-6 genuinely
distinct angles before writing a single query.

## Step 2 — Iterative query expansion within each dimension

For each dimension, don't fire a single fixed query. Start broad, read what comes back, then reformulate:
narrower terms if results are too broad, adjacent/synonym terms if results are thin, and — importantly —
terms that would surface *disagreement* or *contradicting* evidence, not just confirming evidence. A
dimension isn't done after one search; it's done when further reformulation stops changing the picture.

## Step 3 — Continuous corroboration and recency-weighting (run inline, not as a separate phase)

As sources come in, apply these two checks continuously rather than saving them for the end:
- **Source triangulation**: treat any single-source claim as provisional. Before treating a claim as
  established, look for it to be independently corroborated across 2-3 unrelated sources. If it only ever
  appears in one place, flag it as such rather than presenting it flatly.
- **Recency-weighting**: for anything time-sensitive (current events, prices, org leadership, active
  research/product areas), explicitly favor and surface the most recent sources, and flag when a source
  might be stale or superseded rather than treating all retrieved results as equally current.

## Step 4 — Go deeper only where it's warranted (citation-graph following)

For dimensions where Step 2's first pass turns up thin, contested, or unusually consequential results,
follow the citation graph: from a strong seed source, trace what it relies on (backward) and what engages
with or cites it (forward, if searchable) to reach primary sources and the actual current state of debate,
rather than stopping at the first layer of search results. Don't apply this to every dimension by default —
it's for the ones that need it, not a blanket step.

## Step 5 — Falsify load-bearing claims (delegate to fact-falsifier)

Before finalizing the report, identify which claims are actually load-bearing for whatever decision the
user is making — not every fact found, just the ones the conclusion actually depends on. Route those
specific claims through the **fact-falsifier** skill: mine their assumptions and actively search for the
strongest counter-evidence or alternative explanation before including them as settled. If fact-falsifier
isn't available, still do a lightweight version manually — ask "what would have to be true for this claim
to be wrong, and did I check for that" for each load-bearing claim.

Do not run this on routine, low-stakes facts gathered along the way — reserve it for claims that would
change the user's conclusion or decision if they turned out to be false.

## Step 6 — Evidence synthesis with confidence scoring

Don't present findings as a flat list of equally-certain facts. For the final synthesis, score each major
finding by weight of evidence:
- Number of independent corroborating sources (from Step 3)
- Source quality/authority
- Recency (from Step 3)
- Presence or absence of contradicting evidence (from Steps 4-5)

Report a confidence level alongside each major finding — e.g. "well-corroborated across N independent
sources," "single source, treat with caution," "actively contested — sources disagree" — rather than
letting everything read as equally solid.

## Output format

1. **Research plan**: the dimensions from Step 1, shown before/alongside the findings
2. **Findings per dimension**: synthesized in the user's own words (respect standard copyright/citation
   practices — paraphrase, cite sparingly, never reproduce large verbatim passages)
3. **Confidence read per major finding**: from Step 6
4. **Flags**: any claims still contested, single-sourced, or where Step 5's falsification pass weakened the
   original claim

## When to scale down

For a quick factual question with an obvious, uncontested answer, skip this whole pipeline and just answer
directly — this skill is for genuine research tasks (multi-source, decision-relevant, or open-ended), not
single-fact lookups.
