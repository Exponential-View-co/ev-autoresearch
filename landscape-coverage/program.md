# Landscape Coverage Mechanism — program.md
*Iteration agent instructions*
*Version 1.0 — 260314*

---

## What you are doing

You are iterating on the design of a **landscape coverage mechanism** — a two-part addition to the auto-beta harness that addresses the current D1 (landscape coverage) ceiling.

The mechanism has two components:
1. **Pre-run survey** — fast, cheap scan of the solution landscape before deep iteration begins, identifying 3 structurally diverse starting regions to seed the parallel threads
2. **Region-aware diversity gate** — replaces the current uniform 0.80 × current-best acceptance threshold with a gate that lowers the bar for candidates from under-explored regions, rewarding exploration throughout the run

These two parts are designed to work together: the survey maps the landscape at the start; the gate keeps the harness exploring rather than converging prematurely.

---

## Context

The current auto-beta harness (v11) scores 7.99 overall but the VC Stress-Tester gives D1 (landscape coverage) a 7/10. The root cause: the 3 parallel threads are seeded from abstract axis labels (e.g. "central frame", "causal mechanism", "epistemic register") which may cluster in the same conceptual region. The uniform gate is blind to whether a candidate is from an explored or unexplored region.

---

## The experiment file

You are iterating on `coverage_mechanism.md`. It contains:

1. **Survey design** — how the pre-run survey works: what it generates, how many sketches, what model, how it identifies diverse regions, how it seeds the 3 threads
2. **Gate mechanism** — how the region-aware gate works: what counts as "explored", how the threshold adjusts, what prevents gaming
3. **Integration** — how these two parts connect to each other and to the existing auto-beta mechanisms (crossover, adaptive axes, tournament, fingerprint monitoring)
4. **Cost estimate** — rough token/dollar overhead per run
5. **Domain generality** — how this works without domain-specific configuration

---

## Rules for iteration

- Change **ONE element** per iteration
- Priority order:
  1. Survey design (the core — how do you actually map the landscape cheaply?)
  2. Gate mechanism (how do you reward unexplored regions without breaking convergence?)
  3. Integration (how do the two parts interact with existing mechanisms?)
  4. Domain generality (does this work for writing, strategy, investment, product?)

**Direction of travel:** bolder. The survey should generate genuine diversity, not just variety. The gate should meaningfully change search behaviour, not just add a small discount.

**Hard constraints:**
- Survey must cost <$5 total
- Gate must not prevent convergence (the harness must still reach a final answer)
- Both parts must work without domain-specific tuning from the user
- Must integrate cleanly with the existing v11 harness

---

## What the evaluator measures

Four dimensions scored by 3 judges (Karpathy / EV Platform Strategist / VC Stress-Tester):

| Dimension | Core question |
|---|---|
| D1 Coverage expansion | Does the survey actually find diverse starting regions, not just different-sounding ones? |
| D2 Diversity maintenance | Does the gate reward exploration throughout — not just at init? |
| D3 Integration coherence | Do survey + gate work together without breaking existing mechanisms? |
| D4 Generality | Works across writing, strategy, investment, product without manual configuration? |

Single metric: geometric mean per judge → arithmetic mean across 3 judges.

---

*Version 1.0 — 260314*
