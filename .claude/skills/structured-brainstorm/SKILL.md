---
name: structured-brainstorm
description: Use this skill whenever the user asks to brainstorm, ideate, generate options/ideas, come up with possibilities, or explore alternatives for something — e.g. "brainstorm names for...", "give me ideas for...", "what are some ways to...", "help me think of options for...". Make sure to trigger this even if the user doesn't say the word "brainstorm" explicitly, as long as they're asking for a spread of possible ideas/options/approaches rather than a single answer or a factual lookup. Runs a two-phase process — broad, gap-free idea generation followed by uncertainty-aware narrowing — instead of just listing the first ideas that come to mind. Do NOT use this for requests that already specify a single clear direction (e.g. "write me a tagline in this style") or for narrow factual questions.
---

# Structured Brainstorm

A two-phase brainstorming method: force complete coverage of the idea space first, then narrow down without prematurely collapsing to "the obvious answer."

Most brainstorming — human or AI — fails in one of two ways: it either stays shallow and clusters around the first few obvious ideas (no coverage), or it generates a big pile of ideas and then just picks favorites (no principled narrowing). This skill fixes both failure modes.

## When to use this

Trigger on any request for a spread of ideas, options, names, approaches, features, angles, or solutions. This includes requests phrased as questions ("what are some ways to..."), not just explicit "brainstorm" requests.

Skip this skill (just answer directly) when:
- The user wants a single, specific piece of output (one email, one design, one answer)
- The user has already narrowed to 2-3 specific options and wants a comparison, not more ideas
- The request is a factual lookup, not an ideation task

## Phase 1: Divergence — Coverage, not just volume

The goal here is completeness: make sure no obvious region of the idea space got skipped because it wasn't the first thing that came to mind.

1. **Define the dimensions.** Before generating any ideas, identify 4-6 genuinely distinct categories/angles the topic could be approached from. These should be substantively different lenses, not superficial rewordings. For example:
   - Product features → UX, monetization, technical/infra, growth, risk/trust
   - Marketing campaign ideas → channel type, emotional angle, format, audience segment, budget tier
   - Names for a thing → literal/descriptive, metaphorical, invented/coined, evocative-of-feeling, competitor-contrast
   - If the topic doesn't obviously decompose, ask yourself "what are the different lenses someone could look at this problem through?" and use those as categories.

2. **Generate a quota per category.** Produce roughly 3-5 ideas in *each* category, not just wherever inspiration strikes. Do not skip a category because it feels less promising — thin or "boring" categories often surface the most differentiated ideas precisely because nobody else bothered to look there.

3. **Keep threads parallel, don't collapse early.** While generating, don't let one early idea dominate and pull all subsequent ideas toward it (e.g. all your "names" starting to rhyme with your first pick). If you notice this happening, deliberately branch back out.

4. **Output of Phase 1:** a coverage matrix — categories as rows, generated ideas as columns/bullets underneath each. This is shown to the user in full; nothing gets silently dropped at this stage.

## Phase 2: Convergence — Narrow without just picking favorites

The goal here is to select a shortlist that reflects genuine uncertainty-aware judgment, not just "which one did I like first."

1. **Score loosely, not precisely.** For each idea from Phase 1, assign a rough qualitative read on 2-3 criteria relevant to the task (e.g. feasibility, novelty, impact, fit). Use High/Medium/Low, not false-precision numeric scores — the goal is signal, not spreadsheet math.

2. **Don't just take the top scores (Thompson-sampling logic).** Resist collapsing straight to "the 5 highest-scored ideas." Deliberately pull forward at least 1-2 ideas that scored well on novelty/upside even if their overall confidence is lower/uncertain — high-uncertainty, high-upside ideas are exactly the ones a naive "take the top N" approach discards, and they're often the most valuable output of a brainstorm.

3. **Cool gradually (simulated-annealing logic).** If doing multiple narrowing passes (e.g. 20 ideas → 10 → 5), let the first pass be permissive — cut only the clearly weak or redundant ideas, keep boundary cases in. Only on the final pass apply a stricter bar. Don't jump straight from 20 to 5 in one cut; the intermediate pass is what protects unusual ideas from being cut too early by first-impression bias.

4. **Output of Phase 2:** a ranked shortlist (typically 5-8 ideas) with a one-line rationale for each — including *why* any lower-confidence-but-high-upside ideas made the cut despite not scoring highest.

## Phase 2.5: Recombination — Mix and match

Ideas from different categories are allowed, and encouraged, to combine. The shortlist from Phase 2 is not
required to be the final output as-is — treat it as raw material that can be recombined into something
stronger than any single idea alone.

1. **Scan for complementary pairs/sets.** Look across the shortlist (and it's fine to reach back into the
   full Phase 1 matrix, not just the shortlist) for ideas that solve different parts of the problem or that
   combine cleanly — e.g. one idea nails the mechanism, another nails the framing/name, a third nails the
   distribution angle. These are prime candidates for merging rather than choosing between.

2. **Generate hybrids explicitly.** For each promising combination, state the hybrid as its own new option:
   what it takes from each source idea, and why the combination is stronger than either parent alone (not
   just "these are both fine so here they are together" — there should be a real reason the combination adds
   value, like one idea's weakness being covered by the other's strength).

3. **Don't force it.** Not every shortlist has good combinations, and a forced hybrid is usually worse than
   a clean single idea. If nothing combines well, say so and move on with the Phase 2 shortlist as-is —
   this step should add options, not dilute good ones into a mediocre compromise.

4. **Output of Phase 2.5:** zero or more hybrid ideas, each labeled with its source ideas and rationale,
   added alongside (not replacing) the Phase 2 shortlist.

## Output format

Present all phases to the user, not just the final shortlist:

- **Phase 1 — Coverage matrix**: categories with their generated ideas, shown in full
- **Phase 2 — Ranked shortlist**: 5-8 ideas with one-line rationale each, explicitly noting which ones survived on upside rather than raw score
- **Phase 2.5 — Hybrids (if any)**: recombined ideas, each labeled with which source ideas it draws from and why the combination is stronger than its parts
