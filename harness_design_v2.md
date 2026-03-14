# Harness Design — v2 (Gated Antipode Escape)
*Autoresearch escape loop — second iteration*
*260314*

---

## Escape trigger
Same as v1: 10 consecutive discards. The loop does not terminate on trigger — it fires the escape mechanism instead of stopping.

## Escape mechanism

**Gated Antipode Escape.**

When the escape triggers, the harness performs v1's Conceptual Antipode Generation with two structural upgrades: extended mini-sprints (3 → 5 iterations) and a hard regression gate that the winning seed must clear before replacing the current-best.

**Step 1 — Structural dissection.** *(Unchanged from v1)*
A dedicated dissection agent reads the current-best and extracts three structural elements:
- **Frame** — the central metaphor or conceptual lens
- **Causal claim** — the primary mechanism asserted
- **Scope** — the level of analysis in play

**Step 2 — Antipode generation.** *(Unchanged from v1)*
For each structural element, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding 3 antipodal seed candidates guaranteed to be conceptually distant from the current-best.

**Step 3 — Extended mini-evaluation sprint.**
*(Changed: 3 → 5 iterations)*
Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). The additional 2 iterations give each seed meaningful room to develop in its local basin before scoring is compared. Seeds in richer basins — which may initially look weak — now have enough runway to separate from noise and register genuine signal.

**Step 4 — Regression-gated winner selection.**
*(New mechanism)*
The highest-scoring antipodal seed after its 5-iteration sprint is the escape candidate. Before promotion, it must clear the **regression gate**:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** the escape candidate replaces current-best. The discard counter resets to 0. The main loop resumes from the new seed.
- **Gate fails:** current-best is retained. The discard counter is reset to **5** (not 0) — this punishes the failed escape by shortening the runway before the next trigger fires, preventing the loop from stagnating at a local peak that no antipodal escape can improve. The loop continues from current-best, under pressure to escape sooner.

**Why 0.80:**
The gate is deliberately permissive. A 20% regression tolerance means the harness will accept an escape that is clearly directionally distinct but not yet fully polished — exactly what a seed at iteration 5 of its mini-sprint should be. Scores below 80% of the incumbent represent genuinely inferior territory: not just rough seeds, but structurally weaker framings that would pull the run backward. The gate rejects those while remaining open to high-potential seeds that need the main loop to realise their ceiling.

**What happens if no escape ever clears the gate (repeated failures):**
Each failed escape resets the counter to 5. After a second failed escape, the counter resets to 3. This progressive tightening means the harness fires escapes increasingly frequently until it either finds an acceptable seed or exhausts all perturbation directions at the current best. It does not get frozen.

**Properties of this mechanism:**
- Directed — escape targets derived from current-best anatomy, not sampled randomly (inherited from v1)
- Semantically distant — antipodes are structurally guaranteed to differ on at least one major axis (inherited from v1)
- Regression-protected — the main thread cannot be replaced by a clearly inferior candidate
- Deeper seed development — 5-iteration sprints give seeds enough runway to separate signal from noise
- Autonomous — no human input at any step
- Open-ended writing/strategy compatible — operates on frame/causal-claim/scope, not syntax or domain features
- Bounded cost — 3 seeds × 5 iterations = 15 oracle evaluations per escape event (vs 9 in v1); ~$3–6 per escape event at opus-bedrock rates

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. *(Unchanged from v1.)*

## Perturbation direction logic
Direction during escape is determined entirely by antipode derivation — structured, not random. Main-loop perturbation logic (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1.

## Re-integration
The escape candidate replaces the current-best *only if it clears the regression gate* (≥ 80% of current-best score). No merge logic required — single thread, clean conditional replacement. If the gate fails, current-best is retained and the discard counter resets to 5 (or 3 on second consecutive failure).

## Estimated cost per run
~$24–30 for a standard 30-iteration run with 1–2 escape events. Each escape event adds ~15 oracle evaluations (~$3–6 at opus-bedrock rates, vs ~$2–4 in v1). Well under $250.

---

## What changed this iteration (v1 → v2)

**Element changed:** Escape reliability — D2. A single structural upgrade to the escape mechanism's reliability component, comprising two tightly coupled parts.

---

### Part A — Extended mini-sprints (3 → 5 iterations)

**v1 state:** Each antipodal seed was evaluated over 3 greedy sub-iterations before the winner was selected.

**v2 change:** 5 iterations per seed.

**Rationale:**
A seed that lands in a rich conceptual basin may score modestly at iteration 1 and 2 — the evaluator's feedback needs 2–3 iterations to shape it toward its local peak. At only 3 iterations, a genuinely high-ceiling seed in a complex basin can be outscored by a mediocre seed in a simpler one that peaks faster. The result: the harness systematically under-selects for depth. Extending to 5 iterations gives each basin a fairer comparison — signal separates from noise, and the winner selection is more likely to reflect genuine landscape topology rather than which seed was easiest to polish in 3 steps.

For thesis and strategy work specifically (the primary domain), the most interesting framings are often the most complex to develop. A 3-iteration sprint consistently penalises them. 5 iterations corrects this bias without becoming a full run.

---

### Part B — Regression gate (new)

**v1 state:** The highest-scoring antipodal seed after its mini-sprint unconditionally replaced the current-best. No floor relative to current-best score existed.

**v2 change:** The escape candidate must score ≥ current-best × 0.80 to promote. Below that threshold, current-best is retained and the discard counter resets to 5 (shortening the next escape runway).

**Rationale:**
V1's unconditional replacement is structurally unsound for a search loop. Suppose the current-best scores 8.5. If the best antipodal seed, after its mini-sprint, scores 5.2 — the loop replaces 8.5 with 5.2 and restarts from a significantly weaker position. This isn't landscape exploration; it's self-sabotage. The evaluator correctly flagged this as the weakest element in v1.

The 0.80 threshold is deliberately bold rather than conservative. At 80%, the gate passes a seed scoring 6.8 against a current-best of 8.5 — that's a seed with room to grow, not a polished peer. It blocks only seeds below 6.8, which would represent a genuinely inferior framing. The gate's purpose is not to prevent exploration — it's to prevent *regression disguised as exploration*.

The shortened-runway penalty on gate failure (counter → 5, not 0) is the bold element. Rather than letting the loop coast on current-best for another full 10-discard cycle, a failed escape fires the next escape sooner. The harness leans into searching harder when an escape attempt comes back empty, rather than retreating to a long greedy stretch.

---

### What this does NOT change:
- Escape trigger (still 10 consecutive discards on first trigger; counter resets to 5 on failed gate)
- Structural dissection logic (frame / causal claim / scope)
- Antipode generation (inversion or 90° rotation per axis)
- Population structure (single thread)
- Main-loop perturbation direction (evaluator-guided)
- evaluate.md (frozen throughout; mini-sprints use the same oracle)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
