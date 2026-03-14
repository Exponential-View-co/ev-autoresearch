# Harness Design — v1 (Conceptual Antipode Escape)
*Autoresearch escape loop — first iteration*
*260314*

---

## Escape trigger
Same as v0: 10 consecutive discards. The loop does not terminate on trigger — it now fires the escape mechanism instead of stopping.

## Escape mechanism

**Conceptual Antipode Generation.**

When the escape triggers, the harness does not inject noise or make a random jump. Instead, it performs a *structural dissection* of the current-best candidate and derives escape targets from the anatomy of that candidate.

**Step 1 — Structural dissection.**
A dedicated dissection agent reads the current-best and extracts three structural elements:
- **Frame** — the central metaphor or conceptual lens (e.g. "Scarcity OS", "abundance blindspot", "learning curve race")
- **Causal claim** — the primary mechanism asserted (e.g. "cost collapse drives adoption", "regulation lags capability")
- **Scope** — the level of analysis in play (e.g. firm-level, sector-level, civilisational)

**Step 2 — Antipode generation.**
For each structural element, the dissection agent generates its *antipode* — not a random alternative, but the structurally determined opposite or a 90° rotation:
- Frame: invert the polarity (scarcity → abundance, or "enables" → "prevents"); and/or rotate to a different epistemic mode (mechanism → consequence; prediction → diagnosis)
- Causal claim: reverse the causal arrow (effect becomes cause); or replace the mechanism with its structural complement
- Scope: shift by two levels (firm → civilisational; sector → individual)

This yields 3 *antipodal seed candidates* — fully formed alternative framings derived from the current-best's own structure, guaranteed to be conceptually distant from it.

**Step 3 — Mini-evaluation sprint.**
Each antipodal seed is run through a short 3-iteration greedy sub-run (standard iteration agent + evaluator, same evaluate.md). This is enough to lift each seed to its local peak without full-run cost.

**Step 4 — Winner selection.**
The highest-scoring antipodal seed after its 3-iteration sprint becomes the new current-best. The main loop resumes from this new seed. The discards counter resets to 0.

**Properties of this mechanism:**
- Fully directed — each escape target is derived from the current-best's anatomy, not sampled randomly
- Semantically distant — antipodes are structurally guaranteed to differ in at least one major axis
- Autonomous — no human input; the dissection agent operates on the existing candidate
- Open-ended writing/strategy compatible — operates on semantic structure (frames, claims, scope), not syntax or domain-specific features
- Bounded cost — 3 seeds × 3 iterations = 9 additional oracle evaluations per escape event

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. (Unchanged from v0.)

## Perturbation direction logic
None beyond the evaluator's "SUGGESTED DIRECTION" guidance during the main loop. Direction during escape is determined entirely by antipode derivation — structured, not random. (Main-loop perturbation logic unchanged from v0.)

## Re-integration
The winning antipodal seed directly replaces the current-best. No merge logic required — single thread, clean replacement. The loop continues from the new seed as if it were a normal accepted iteration.

## Estimated cost per run
~$22–26 for a standard 30-iteration run with 1–2 escape events. Each escape event adds ~9 oracle evaluations (~$2–4 at opus-bedrock rates). Well under $250.

---

## What changed this iteration (v0 → v1)

**Element changed:** Escape mechanism (Priority 1 per program.md).

**v0 state:** No escape mechanism. 10 consecutive discards → loop terminates. Permanently stuck at local optimum.

**v1 change:** Escape mechanism added. 10 consecutive discards → fires Conceptual Antipode Generation. The escape derives 3 jump targets from the structural anatomy of the current-best (frame, causal claim, scope), runs each through a 3-iteration mini-sprint, and promotes the winner as the new current-best. Loop continues.

**Why this specific mechanism:**
The failure mode in v0 is that the evaluator can only suggest incremental improvements to the current peak — it has no visibility of what other conceptual regions exist. A random jump would address this but poorly: it would land in an arbitrary region, likely scoring lower and being immediately discarded.

Antipode generation solves this by exploiting what the loop *already knows*: the anatomy of the best candidate found so far. If the current peak uses "Scarcity OS" as its frame, the antipode is structurally forced to use something like "Abundance OS" or a polarity-inverted variant — which is a genuinely different conceptual region, not noise. The escape is directed by the topology of the current solution, not by a random walk.

For thesis and strategy work specifically, the three axes (frame, causal claim, scope) capture the principal dimensions along which strong arguments differ. Rotating on even one axis produces a candidate that the main loop's evaluator would have never suggested — satisfying D1 (landscape coverage) and D3 (perturbation quality) directly.

**What this does NOT change:**
- Escape trigger (still 10 consecutive discards)
- Population structure (still single thread)
- Perturbation direction logic during main loop (still evaluator-guided)
- Re-integration (still direct replacement)
- evaluate.md (still frozen throughout; antipode mini-sprints use the same oracle)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
