# A Guide to the Skill Suite

This document explains the reasoning behind six Claude skills — what each one does, the specific algorithms
and methods each is built from, and why those methods were chosen. It's written for people who want to
understand *why* a skill behaves the way it does, not just what button to press.

The six skills are:

1. **fact-falsifier** — stress-tests claimed facts by mining their assumptions and testing them
2. **structured-brainstorm** — generates and narrows ideas with deliberate coverage and mix-and-match
3. **regression-test-strategy** — selects, generates, and stops adding regression tests
4. **task-decomposer** — breaks work into small, self-contained, individually validated tasks
5. **research-strategy** — runs research as a structured, evidence-scored process
6. **goal-planner** — orchestrates all five of the above to turn a stated goal into an implementation plan

Three of the skills (regression-test-strategy, task-decomposer, goal-planner) call on the others as
sub-steps rather than reinventing them — this is noted in each section.

---

## 1. fact-falsifier

**What it does:** Takes a list of claimed facts and, instead of accepting or rejecting them outright, breaks
each one into its supporting assumptions, builds the strongest case against each assumption, and designs a
concrete test to prove or disprove it — landing on a confidence level rather than a verdict.

### Toulmin's model of argument

The core decomposition step comes from philosopher Stephen Toulmin's structure for analyzing arguments,
first proposed in his 1958 book *The Uses of Argument*. Toulmin broke an argument into six parts: the
**claim** (what's being asserted), the **grounds** (the evidence offered), the **warrant** (the usually
unstated logical bridge connecting grounds to claim), the **backing** (why the warrant itself should be
trusted), the **qualifier** (how strongly the claim is being made), and the **rebuttal** (the conditions
under which the claim wouldn't hold) [1][2]. The skill uses this structure deliberately because the warrant
— the part people skip past — is usually where a claim's real, unexamined assumption is hiding.

### Bayesian updating / Bayes factors

Rather than a true/false verdict, the skill scores confidence using Bayesian logic: a Bayes factor is the
ratio of how well the evidence supports one hypothesis over a competing one, and it's used to update a prior
belief into a posterior one as new evidence comes in [3]. This is why the skill's output uses phrases like
"still standing" or "weakened" instead of "confirmed" — nothing is ever fully closed off, only updated.

### Causal graphs and do-calculus

For any claim that implies causation, the skill checks for confounders and reverse causation using the
logic behind Judea Pearl's do-calculus: a set of rules for determining whether a causal effect can be
identified from observational data, given a causal graph of how variables might relate [4]. The skill
doesn't run the full mathematical apparatus, but borrows its central question — could a hidden common cause
or a reversed arrow produce the same observed pattern?

### Expected Value of Information (EVOI)

For prioritizing which facts are worth the effort of testing further, the skill borrows from decision
theory's expected value of information: a measure of how much a decision would improve, on average, from
resolving a given uncertainty [5][6]. Facts that wouldn't change any real decision if they flipped get
deprioritized, even if they're technically uncertain.

### Delphi method

For claims that are forecasts or judgment calls with no available ground truth, the skill substitutes a
lighter version of the Delphi method: an iterative, anonymous technique developed at the RAND Corporation
in the 1950s for building consensus among independent experts through repeated rounds of estimation and
feedback [7]. Disagreement between independent estimates is itself useful signal.

---

## 2. structured-brainstorm

**What it does:** Runs brainstorming as two distinct phases — broad, gap-checked idea generation, then
uncertainty-aware narrowing — followed by an explicit recombination pass that looks for ideas worth merging.

### Divergent and convergent thinking (Guilford)

The two-phase structure follows a distinction introduced by psychologist J.P. Guilford in the 1950s:
**divergent thinking**, the ability to generate many different ideas or approaches to an open-ended problem,
and **convergent thinking**, the ability to narrow many options down to the best one [8][9]. Guilford's
research found that conventional intelligence tests almost entirely measured convergent thinking, which is
part of why brainstorming so easily collapses into "list the first few ideas and pick one" — it defaults to
convergent-only thinking without ever doing the divergent step properly.

### Thompson sampling

The instruction to deliberately pull forward high-uncertainty, high-upside ideas during narrowing — rather
than just taking the top-scored options — mirrors the logic behind Thompson sampling, a method from
probability theory for balancing exploration against exploitation. Instead of always picking the option with
the highest known score, Thompson sampling selects options in proportion to their probability of actually
being the best one, which means uncertain-but-promising options still get picked sometimes rather than being
starved out [10][11].

### Simulated annealing

The instruction to narrow gradually — permissive on the first pass, strict only on the final pass — mirrors
simulated annealing, an optimization technique inspired by the physical process of annealing metal: cooling
it slowly so atoms can settle into a low-energy, defect-free structure, versus cooling it fast and locking in
a worse one [12][13]. In both cases, committing to a strict cutoff too early locks in whatever looked good
on first impression rather than what's actually best.

### Mix-and-match recombination

The final phase, which scans surviving ideas for combinations that are stronger together than apart, isn't
drawn from a single named algorithm — it's a deliberate addition to keep the shortlist from being treated as
final. It reflects the same principle as divergent thinking's dimension-based generation: ideas from
different categories often solve different parts of a problem, and forcing a single winner throws away that
complementarity.

---

## 3. regression-test-strategy

**What it does:** Decides which existing tests to run on a change, generates new tests to close real gaps
(delegating that step to structured-brainstorm), and defines when to stop adding tests.

### Regression Test Selection (RTS) via dependency/call graphs

Rather than rerunning an entire test suite on every change, the skill defaults to call-graph-based
selection: building a graph of what code depends on what, then selecting only the tests that exercise
code reachable from the change. This is a well-studied technique in software engineering research, with
approaches ranging from static call-graph analysis to dynamic dependency tracking [14][15].

### Mutation testing

To find real gaps in a test suite rather than superficial ones, the skill uses mutation testing: injecting
small, deliberate faults ("mutants") into the code and checking whether the existing tests catch them. A
mutant that survives — meaning every test still passes despite the injected bug — reveals a real hole in
test coverage; a mutant that's "killed" (caught by a failing test) shows the tests are doing real work
there [16][17]. The **mutation score** (percentage of mutants killed) is the skill's primary stopping
signal, precisely because it measures whether tests catch bugs rather than just whether code got executed.

### Delta debugging

When a regression bug is found in the wild, the skill converts it into a permanent test using the logic of
delta debugging: an algorithm developed by Andreas Zeller and Ralf Hildebrandt that systematically shrinks
a failing test case down to the smallest input that still reproduces the failure [18][19]. This turns a
messy real-world bug report into a minimal, precise regression test.

### Orthogonal Defect Classification (ODC)

As a secondary stopping signal, the skill checks whether new tests are surfacing genuinely new categories
of defects, using the logic of Orthogonal Defect Classification — a method developed at IBM Research by Ram
Chillarege in the late 1980s and early 1990s for categorizing software defects by type and by the
circumstance ("trigger") that caused them to surface [20][21]. If new tests only find repeats within
already-well-covered categories, that's a sign the test suite has covered the relevant failure-mode space.

### Capture-recapture and reliability growth models

For teams with enough historical defect data, the skill points to two additional statistical signals: the
capture-recapture method, borrowed from ecology (where it's used to estimate a wildlife population from the
overlap between multiple independent samples) and adapted to estimate how many defects likely remain
undiscovered from the overlap between independent reviewers' findings [22][23]; and reliability growth
models such as Goel-Okumoto, which model defect discovery as a non-homogeneous Poisson process and use the
flattening of that curve over time as a signal of diminishing returns from further testing [24][25].

---

## 4. task-decomposer

**What it does:** Recursively splits an effort into small, self-contained tasks, checks each against a
quality bar, attaches a concrete validation test to each, sequences them by dependency, and audits the
whole set for gaps or overlaps.

### Work Breakdown Structure (WBS)

The top-down recursive splitting step follows the Work Breakdown Structure method formalized by the Project
Management Institute: a deliverable-oriented hierarchical decomposition of the total work of a project,
where each descending level represents an increasingly detailed definition of the work, continuing until
the "100% rule" is satisfied — the WBS captures the entirety of the project's scope, no more and no
less [26][27].

### INVEST criteria

Each candidate task is checked against INVEST, a set of quality criteria for well-formed units of work
coined by Bill Wake in 2003 for agile user stories: **I**ndependent, **N**egotiable, **V**aluable,
**E**stimable, **S**mall, and **T**estable [28][29]. The skill borrows this directly because it's precisely
a checklist for "is this task actually self-contained and verifiable," which is the property the user asked
for by name.

### Design by Contract

For technical, well-defined tasks, the validation test is built using Design by Contract, a method
introduced by Bertrand Meyer in the 1980s as part of the Eiffel programming language: every unit of work has
a **precondition** (what must be true before it runs), a **postcondition** (what becomes true after it
completes successfully), and optionally an **invariant** (what must remain true throughout) [30][31]. The
test for whether a task is genuinely done becomes a direct check of its postcondition given its precondition.

### Topological sort over a dependency graph

Once tasks exist, the skill models them as a directed graph and orders them via topological sort — a
standard graph algorithm that produces a valid linear ordering of nodes such that every dependency comes
before what depends on it, and which also reveals circular dependencies (a sign two "independent" tasks are
actually coupled) when no valid ordering exists.

### MECE (Mutually Exclusive, Collectively Exhaustive)

The final completeness audit uses the MECE principle, developed by Barbara Minto at McKinsey & Company in
the 1960s as part of her Pyramid Principle: a full set of categories should have no overlap between items
(mutually exclusive) and should leave nothing out (collectively exhaustive) [32][33]. Applied to a task
breakdown, this becomes: does every task own distinct work with no duplication, and does the full set of
tasks cover the entire original scope with no gaps?

---

## 5. research-strategy

**What it does:** Plans the dimensions of a research question before searching, runs iterative search
within each dimension, corroborates and recency-weights sources continuously, goes deeper only where
warranted, falsifies load-bearing claims, and closes with confidence-scored synthesis.

### Coverage planning via structured-brainstorm

The skill opens by delegating to **structured-brainstorm**'s divergence phase to define the research
question's dimensions before any searching happens — treating "what angles does this question need
covering from" exactly like generating brainstorm categories, for the same reason: shallow research
clusters around whatever the first few searches happen to surface.

### Source triangulation

The continuous corroboration check is based on data triangulation, a concept from qualitative research
methodology formalized by sociologist Norman Denzin: using multiple independent data sources to corroborate
a finding, which increases the credibility of a conclusion and helps surface where sources actually
disagree [34][35]. The skill's rule of thumb — don't treat a claim as established until it shows up in 2-3
independent sources — is a simplified, practical version of this principle.

### Citation snowball sampling

For dimensions that turn up thin or contested results, the skill follows citation snowballing: a literature-
search technique of tracing a strong source's references backward (what it relies on) and, where possible,
forward (what cites it) to build out a fuller picture of a topic than a single layer of search results would
give [36][37]. This is standard practice in systematic literature reviews, where a substantial fraction of
included sources are typically found this way rather than through direct database search.

### Claim falsification via fact-falsifier

For any finding that's actually load-bearing for the user's decision, the skill delegates to
**fact-falsifier** to mine assumptions and search for genuine counter-evidence before treating the claim as
settled — applied selectively, not to every fact gathered, since running a full falsification pass on
routine facts would be excessive.

### Weight-of-evidence synthesis

The closing step scores each major finding by the strength of its supporting evidence — number of
independent corroborating sources, source authority, recency, and presence of contradicting evidence —
rather than presenting every finding with equal confidence. This follows directly from the triangulation
and recency-weighting principles above, applied at the level of the whole report rather than a single claim.

---

## 6. goal-planner

**What it does:** Given a stated goal, decides which of the other five skills to invoke at each stage of
building a concrete implementation plan, wrapped in an iterative execute-and-validate loop.

### Means-ends analysis

The core routing logic — at each step, identify the gap between the current state and the goal state, then
pick the operation that reduces it — is means-ends analysis, a problem-solving strategy formalized by Allen
Newell and Herbert Simon in their 1950s–60s work on the General Problem Solver (GPS), one of the earliest
artificial intelligence programs [38][39][40]. GPS worked by recursively comparing a current state to a
goal state, finding the most significant difference, and selecting an operator (here, one of the five other
skills) to reduce that difference — exactly the pattern goal-planner's decision table encodes.

### Hierarchical Task Network (HTN) planning

The recursive decomposition of a goal into ordered, executable subtasks — delegated to **task-decomposer**
— follows the structure of Hierarchical Task Network planning: an AI planning paradigm where complex
("compound") tasks are decomposed via domain-specific methods into simpler subtasks, continuing until only
directly executable ("primitive") actions remain [41][42].

### Plan–Do–Check–Act (PDCA)

The outer loop wrapping the whole process is the Plan–Do–Check–Act cycle, which traces back to physicist
Walter Shewhart's work on statistical process control in the 1920s and was later popularized worldwide by
W. Edwards Deming as a tool for continuous improvement [43][44]. The skill uses PDCA specifically (rather
than the closely related OODA loop, discussed below) because PDCA's structure — commit to a plan, execute
it, check the results, then adjust — maps directly onto "build an implementation plan and then actually
implement it," which is what the user asked this skill to do.

### OODA loop (noted alternative)

A closely related framework, John Boyd's OODA loop (Observe, Orient, Decide, Act), developed in the 1970s
for military decision-making, was considered as the outer loop instead of PDCA [45][46]. The key structural
difference is that OODA is built for continuous, late-committing observation and rapid re-looping, whereas
PDCA is built around committing to a plan and then checking it against results. Since goal-planner's job is
specifically to produce and execute a concrete plan (not just react quickly to a changing situation), PDCA
was the better structural fit — but OODA's observe/orient framing is echoed in how the Plan stage's
information-gathering step (via research-strategy) is described.

### Model Predictive Control / rolling-horizon replanning

For goals that span a long or uncertain time horizon, the skill borrows the logic of Model Predictive
Control (also known as receding-horizon control): rather than committing to a fully detailed plan for the
entire horizon upfront, plan only the near-term window in detail, execute it, then re-plan the next window
using fresh information once the current one completes [47][48]. This avoids wasting planning effort on
distant, uncertain parts of a plan that will likely need to change anyway by the time they're reached.

### Critical Path Method (CPM)

Once task-decomposer has produced a dependency-ordered task graph, the skill uses the Critical Path Method
— developed in the late 1950s by Morgan Walker of DuPont and James Kelley of Remington Rand — to identify
the longest chain of dependent tasks, which determines the shortest possible time to reach the goal and
flags which tasks have zero slack (any delay in them delays the whole goal) versus which have room to
slip [49][50].

---

## Works Cited

1. Toulmin Argument Model — Speaking Intensive Program, University of Mary Washington. https://academics.umw.edu/speaking/resources/handouts/toulmin-argument-model/
2. "The Toulmin Model of Argumentation: Claims, Data, and Warrants, Oh My!" — Academy 4SC Learning Hub. https://learn.academy4sc.org/video/the-toulmin-model-of-argumentation-claims-data-and-warrants-oh-my/
3. "Bayesian Inference: An Introduction to Hypothesis Testing Using Bayes Factors" — *Nicotine & Tobacco Research*, Oxford Academic. https://academic.oup.com/ntr/article/22/7/1244/5613971
4. "Do-calculus" — Wikipedia. https://en.wikipedia.org/wiki/Do-calculus
5. "Expected value of perfect information" — Wikipedia. https://en.wikipedia.org/wiki/Expected_value_of_perfect_information
6. "Expected value of information — EVI, EVPI, and ESVI" — Analytica Docs. https://docs.analytica.com/index.php/Expected_value_of_information_--_EVI,_EVPI,_and_ESVI
7. "Delphi Method" — RAND Corporation. https://www.rand.org/topics/delphi-method.html
8. "Divergent thinking" — Wikipedia. https://en.wikipedia.org/wiki/Divergent_thinking
9. "Convergent thinking" — Wikipedia. https://en.wikipedia.org/wiki/Convergent_thinking
10. "Analysis of Thompson Sampling for the multi-armed bandit problem" — Agrawal & Goyal, arXiv. https://arxiv.org/abs/1111.1797
11. "A Survey on Contextual Multi-armed Bandits" — arXiv. https://arxiv.org/pdf/1508.03326
12. "Simulated annealing" — Cornell University Computational Optimization Open Textbook. https://optimization.cbe.cornell.edu/index.php?title=Simulated_annealing
13. "A Comparison of Cooling Schedules for Simulated Annealing" — IRMA International. https://www.irma-international.org/viewtitle/10270/?isxn=9781599048499
14. "Efficient Incremental Code Coverage Analysis for Regression Test Suites" — arXiv. https://arxiv.org/pdf/2410.21798
15. "Names Are All You Need: Effective and Safe Regression Test Selection for Python" — arXiv. https://arxiv.org/html/2605.25356
16. "What is mutation testing?" — CircleCI. https://circleci.com/blog/what-is-mutation-testing/
17. "Extreme mutation testing in practice: An industrial case study" — arXiv. https://arxiv.org/pdf/2103.08480
18. "Simplifying and Isolating Failure-Inducing Input" — Zeller & Hildebrandt. https://homes.cs.washington.edu/~mernst/teaching/6.893/readings/zeller-tse.pdf
19. "Simplifying and Isolating Failure-Inducing Input: A Retrospective on Delta Debugging" — IEEE Transactions on Software Engineering, 2025. https://www.computer.org/csdl/journal/ts/2025/03/10859156/23X97jMgYjm
20. "Orthogonal defect classification" — Wikipedia. https://en.wikipedia.org/wiki/Orthogonal_defect_classification
21. "Process Control" — Chillarege Inc. https://www.chillarege.com/docs/Papers/odc-process-control/
22. "A Comprehensive Evaluation of Capture-Recapture Models for Estimating Software Defect Content" — ResearchGate. https://www.researchgate.net/publication/3188084_A_Comprehensive_Evaluation_of_Capture-Recapture_Models_for_Estimating_Software_Defect_Content
23. "Capture-recapture in software inspections after 10 years research" — ResearchGate. https://www.researchgate.net/publication/222300754_Capture-recapture_in_software_inspections_after_10_years_research_-_Theory_evaluation_and_application
24. "A Survey of Software Reliability Models" — arXiv. https://arxiv.org/pdf/1304.4539
25. "Application of Goel-Okumoto Model in Software Reliability Measurement" — Academia.edu. https://www.academia.edu/67033744/Application_of_Goel_Okumoto_Model_in_Software_Reliability_Measurement
26. "Work Breakdown Structure (WBS) - Basic Principles" — Project Management Institute. https://www.pmi.org/learning/library/work-breakdown-structure-basic-principles-4883
27. "Work breakdown structure" — Wikipedia. https://en.wikipedia.org/wiki/Work_breakdown_structure
28. "Writing meaningful user stories with the INVEST principle" — LogRocket Blog. https://blog.logrocket.com/product-management/writing-meaningful-user-stories-invest-principle/
29. "INVEST in Small User Stories!" — Boris Karl Schlein, Agile Insider. https://medium.com/agileinsider/invest-in-small-user-stories-3c83ef967997
30. "Design by Contract" — Bertrand Meyer, ETH Zürich. https://se.inf.ethz.ch/~meyer/publications/old/dbc_chapter.pdf
31. "Design by Contract - an overview" — ScienceDirect Topics. https://www.sciencedirect.com/topics/computer-science/design-by-contract
32. "MECE principle" — Wikipedia. https://en.wikipedia.org/wiki/MECE_principle
33. "Barbara Minto: 'MECE: I invented it, so I get to say how to pronounce it'" — McKinsey & Company. https://www.mckinsey.com/alumni/news-and-events/global-news/alumni-news/barbara-minto-mece-i-invented-it-so-i-get-to-say-how-to-pronounce-it
34. "An Introduction to Triangulation" — UNAIDS Monitoring and Evaluation Fundamentals. https://www.unaids.org/sites/default/files/sub_landing/files/10_4-Intro-to-triangulation-MEF.pdf
35. "The use of triangulation in qualitative research" — PubMed. https://pubmed.ncbi.nlm.nih.gov/25158659/
36. "'Snowballing' in Systematic Literature Review" — LinkedIn / summary of Wohlin methodology. https://www.linkedin.com/pulse/snowballing-systematic-literature-review-amanpreet-kohli
37. "Citation search" — Literature search, LibGuides at Radboud University. https://libguides.ru.nl/literaturesearch/snowball
38. "General Problem Solver" — Wikipedia. https://en.wikipedia.org/wiki/General_Problem_Solver
39. "Means–ends analysis" — Wikipedia. https://en.wikipedia.org/wiki/Means%E2%80%93ends_analysis
40. "General Problem Solver (A. Newell & H. Simon)" — InstructionalDesign.org. https://www.instructionaldesign.org/theories/general-problem-solver/
41. "Hierarchical Task Network (HTN) Planning in AI" — GeeksforGeeks. https://www.geeksforgeeks.org/artificial-intelligence/hierarchical-task-network-htn-planning-in-ai/
42. "Hierarchical Task Network" — ScienceDirect Topics. https://www.sciencedirect.com/topics/computer-science/hierarchical-task-network
43. "PDCA" — Wikipedia. https://en.wikipedia.org/wiki/PDCA
44. "PDCA Cycle - What is the Plan-Do-Check-Act Cycle?" — American Society for Quality (ASQ). https://asq.org/quality-resources/pdca-cycle
45. "OODA loop" — Wikipedia. https://en.wikipedia.org/wiki/OODA_loop
46. "The OODA Loop" — The Decision Lab. https://thedecisionlab.com/reference-guide/computer-science/the-ooda-loop
47. "What Is Model Predictive Control?" — MathWorks. https://www.mathworks.com/help/mpc/gs/what-is-mpc.html
48. "Stochastic Model Predictive Control" — Automatic Control Laboratory, ETH Zürich. https://control.ee.ethz.ch/research/theory/stochastic-model-predictive-control.html
49. "Origins of CPM - a Personal History" — Kelley, Walker & Sayer, Project Management Institute. https://www.pmi.org/learning/library/origins-cpm-personal-history-3762
50. "Critical Path Method (CPM) in Project Management" — PM Study Circle. https://pmstudycircle.com/critical-path-method-cpm-in-project-management/
