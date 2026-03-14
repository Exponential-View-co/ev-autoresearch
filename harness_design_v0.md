# Harness Design — v0 (null baseline)
*Autoresearch escape loop — starting state*
*260314*

---

## Escape trigger
None. The loop runs until 10 consecutive discards, then stops and reports convergence.

## Escape mechanism
None. When 10 consecutive discards occur, the loop terminates. No escape attempted.

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential.

## Perturbation direction logic
None. The iteration agent is told to change ONE element per the priority order in program.md. Direction is driven entirely by the evaluator's "SUGGESTED DIRECTION" output from the previous iteration. If the loop converges, there is no mechanism to try a different direction.

## Re-integration
Not applicable — single thread, no branching.

## Estimated cost per run
~$18–20 for a standard 30-iteration run at opus-bedrock evaluation rates. Well under $250.

---

## Why this is the baseline problem

Pure greedy hill-climbing with this structure will reliably converge to a local optimum. The evaluator's "SUGGESTED DIRECTION" guidance steers the next iteration, but all suggestions are incremental improvements to the current peak — they cannot suggest jumping to a different conceptual region because they have no visibility of what other regions exist.

Example failure mode for a book thesis: the loop converges on "Scarcity OS → Learning Curve frame" as the central metaphor. Every subsequent iteration tries to sharpen this frame. But a completely different frame (e.g. "The Abundance Blindspot" or "The Cost Collapse") might score higher — and the loop will never find it because it can only make incremental changes to the current best.

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
