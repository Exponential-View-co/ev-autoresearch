# Harness Design — v4 (Axis Orthogonality Verification)
*Autoresearch escape loop — fourth iteration*
*260314*

---

## Escape trigger
Same as v2/v3: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3.)*

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery and Orthogonality-Verified Pool.**

When the escape triggers, the harness runs v2/v3's full Gated Antipode Escape. The adaptive axis discovery step (from v3) fires when the regression gate has failed on 2+ consecutive escape events. In v4, the single change is: before any newly generated axis is admitted to the pool, it must pass an **orthogonality verification check** against all currently active axes. Only axes that clear this check are added to the pool.

---

### Step 1 — Structural dissection with live axis pool *(Unchanged from v3)*

A dedicated dissection agent reads the current-best and extracts structural elements from the **current axis pool**. The axis pool begins with the same three axes as v2/v3:

- **Frame** — the central metaphor or conceptual lens
- **Causal claim** — the primary mechanism asserted
- **Scope** — the level of analysis in play

The axis pool is a mutable ordered list, stored in the run's working state. It starts at length 3. Axes are added and retired dynamically (see Step 5 below). The dissection agent always reads from the current pool, not the fixed original set.

---

### Step 2 — Antipode generation *(Unchanged from v3)*

For each axis in the current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record (most gate-passing events) plus any newly added axes, capped at 5 seeds per escape event to contain cost.

---

### Step 3 — Extended mini-evaluation sprint *(Unchanged from v3)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Unchanged from v3)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** escape candidate replaces current-best. Discard counter resets to 0. The winning axis logs a "pass" event.
- **Gate fails:** current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used in the failed escape logs a "fail" event.

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Step 5b modified — the single change in v4)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events — meaning the current axis pool is generating seeds that cluster rather than escaping to genuinely different regions.

When this condition is met, the dissection agent performs an additional analysis pass **before** resetting the discard counter and resuming the main loop:

#### 5a — Evaluator feedback analysis *(Unchanged from v3)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event. It identifies which dimensions scored lowest across all seeds in the failed escape (e.g., "D3 consistently 5/10 across all three seeds, with evaluator language referencing 'insufficient argumentative differentiation' and 'same epistemic stance'").

This is not structural analysis of the candidate itself — it is linguistic analysis of the evaluator's feedback, specifically the natural-language justifications the evaluator provided for its dimensional scores.

#### 5b — LLM-driven axis generation *(Unchanged from v3)*

The dissection agent generates **2 new structural axes** by prompting itself with the following:

> "You are a structural analysis agent. The current escape pool (axes: [list]) has failed to find a viable escape candidate on [N] consecutive attempts. The evaluator's feedback on the failed seeds consistently flagged: [quoted evaluator language from weakest dimensions]. Based on this feedback, propose 2 new perturbation axes that would generate meaningfully different candidates for this specific problem. Each axis must: (1) be orthogonal to all current pool axes, (2) be derivable from the current-best without human input, (3) be general enough to apply across essay/strategy/thesis structure. Name each axis, define its antipode operation, and explain why it addresses the evaluator's flagged weaknesses."

The dissection agent's response is the canonical source of the proposed axes. No human review. The two proposed axes then enter the orthogonality verification step (5b-bis) before being admitted to the pool.

---

#### 5b-bis — Axis orthogonality verification *(New in v4 — the single change)*

For each proposed axis coming out of step 5b, before appending it to the pool, the harness executes a three-part verification sequence:

**i. Probe seed generation**

The harness generates a **probe seed** for the proposed axis: a 2–3 sentence sketch of what a candidate built on this axis would look like — its essential character and structural identity, not a draft. The probe seed answers: "If I took the current-best and inverted/rotated it on this axis, what would the result feel like structurally?" It is deliberately brief — enough to convey the conceptual region the axis explores.

**Example probe seeds for the three starting axes, given a thesis-development problem:**

- *Frame (inverted):* "The candidate abandons its machine-learning metaphor entirely and frames the problem through evolutionary dynamics — entities with fitness functions, selection pressure, and niche competition replace gradient-descent language throughout."
- *Causal claim (inverted):* "The candidate's core mechanism reverses — where the original argues that adoption drives infrastructure investment, the inverted candidate argues infrastructure investment determines adoption ceiling and sequence."
- *Scope (rotated):* "The candidate lifts from firm-level analysis to system-level: individual firms disappear as protagonists, replaced by industry-wide dynamics, regulatory regimes, and cross-sector feedback loops."

These are illustrative. Probe seeds are always generated fresh from the current-best on each adaptive step call — they are not stored across runs.

**ii. Orthogonality check via evaluator model**

The probe seed for the proposed axis is compared against the **probe seeds for all currently active pool axes** in a single opus call. The prompt is:

> "You are evaluating whether a proposed new perturbation axis explores a meaningfully different conceptual region from each existing axis. For each pair below (proposed axis vs. existing axis), answer: (a) Do their probe seeds describe structurally different types of candidates, or do they describe candidates that would feel like variants of each other? Answer 'orthogonal' or 'overlapping'. (b) One sentence of rationale.
>
> Proposed axis: [name + probe seed]
> Existing axes: [list of name + probe seed for each active pool axis]
>
> A proposed axis is ORTHOGONAL if candidates built on it would differ in kind from candidates built on existing axes — different argumentative gesture, different conceptual region, not just a different magnitude or emphasis of the same structural move. It is OVERLAPPING if candidates built on it would feel like variations within a region the existing pool already explores."

The call returns a verdict for each pair. The proposed axis passes if it is rated **orthogonal to all existing pool axes**. If it is rated overlapping with any existing axis, it fails.

**iii. Regeneration on failure (up to 2 retries)**

- **Pass:** The proposed axis, together with its probe seed, is appended to the pool. The probe seed is stored in the run's working state alongside the axis.
- **Fail (overlap detected):** The dissection agent regenerates the axis with an augmented prompt that includes: (a) the failing axis, (b) which existing axis it was found to overlap with, (c) the one-sentence rationale from the evaluator, and instructions to propose a replacement that avoids that conceptual region. Up to **2 regeneration attempts** per proposed axis.
- **Still failing after 2 retries:** The axis is dropped. It does not enter the pool. The adaptive step proceeds with however many axes cleared verification (potentially zero new axes if both proposed axes failed after retries).

**The probe seed store**

Each active axis in the pool carries a stored probe seed (generated or regenerated at the point of admission). The store is initialised at harness start with probe seeds for the three starting axes (frame / causal claim / scope), generated in a single call before the first iteration begins. This ensures the orthogonality check is functional from the very first adaptive step, not just once the first newly generated axis has been admitted.

The probe seed store is persisted in the run's working state alongside the axis pool. It grows as axes are added and shrinks as axes are retired. It is the harness's **map of explored conceptual regions** — a compact representation of where the pool has already been, used to refuse re-entry into those regions.

**Why this is a semantic check, not a similarity metric**

The check uses the evaluator model (opus) for a specific reason: orthogonality here is a conceptual judgement, not a distance calculation. Two axes can have very different names and surface descriptions while generating structurally similar candidates. Cosine similarity on axis names or definitions would miss this. The probe seed approach forces each axis to instantiate into a candidate sketch, making the comparison concrete rather than definitional. The evaluator model has the reasoning capacity to judge whether two candidate sketches are exploring different structural territory — which is what "orthogonal" means in this context.

**Properties of the verification step:**
- **Cheap:** 1 opus call per proposed axis (covers probe seed generation AND all pairwise comparisons in a single prompt). Plus up to 2 retry calls per axis if the first fails.
- **Stateless regeneration:** Each retry is fully specified — the dissection agent receives the failure rationale and can target a genuinely different region.
- **Pool is self-curating over a long run:** The probe seed store grows to represent all regions the pool has ever explored. Any new axis must clear all of them. Over many adaptive steps, the pool becomes increasingly precise at identifying and refusing overlapping proposals — it converges on axes that genuinely expand the coverage map.
- **Zero manual configuration:** The probe seeds and their comparisons are generated entirely from the current-best candidate and the evaluator model. No designer input required at any step.

---

#### 5c — Axis pool management (retirement) *(Unchanged from v3)*

After each escape event, each axis logs either "pass" (gate cleared) or "fail" (gate failed). An axis is **retired** from the pool when it has accumulated 3+ consecutive fail events with no intervening pass.

Retirement is permanent for the current run. If retirement would reduce the pool below 3 axes, the retirement is deferred until the adaptive step can replace it (i.e., retirement and new axis generation are coupled — you don't fall below 3).

The axis pool state (current axes, their pass/fail event history, probe seeds, retired axes) is persisted in the run's working state so it survives across iterations.

Note: **retired axes' probe seeds are retained in the probe seed store** even after the axis is retired. This is the key property that makes the pool self-curating over a long run: the harness not only refuses to re-explore regions currently in the pool, it refuses to re-explore regions it has already exhausted and retired. The map of explored regions is monotonically growing, even as the active pool shrinks and grows.

---

**Properties of this mechanism (v4 additions in bold):**
- Directed — escape targets derived from current-best anatomy (inherited from v1/v2/v3)
- Semantically distant — antipodes structurally guaranteed to differ on at least one major axis (inherited from v1/v2/v3)
- Regression-protected — main thread cannot be replaced by inferior candidate (inherited from v2/v3)
- Deeper seed development — 5-iteration sprints (inherited from v2/v3)
- Adaptive — axis pool expands when fixed axes cluster; new axes LLM-derived from evaluator feedback language (inherited from v3)
- Self-correcting — failed axes are retired; new axes address specifically flagged weaknesses (inherited from v3)
- Domain-general — axis vocabulary derived from evaluator's language, not pre-configured (inherited from v3)
- **Structurally verified** — newly generated axes are checked for orthogonality against all active pool axes before admission; overlap triggers regeneration or drop
- **Self-curating** — the pool builds a persistent map of explored conceptual regions (probe seed store) and refuses to re-enter them, including regions from retired axes
- **Monotonic coverage expansion** — the probe seed store only grows; over a long run, the pool becomes increasingly precise at identifying genuinely new territory
- Autonomous — no human input at any step (all steps including axis generation and orthogonality verification fire automatically)
- Open-ended writing/strategy compatible — probe seed generation and comparison operate on conceptual structure, not domain-specific features
- Bounded cost — standard escape: 3 seeds × 5 iterations = 15 oracle evaluations (~$3–6); adaptive step: 1 dissection call + 1–3 opus verification calls per proposed axis (~$1–2 extra per adaptive step vs v3); harness initialisation: 1 call to generate 3 starting probe seeds (~$0.20 once per run; negligible); total per escape event with full adaptation: ~$6–11 at opus-bedrock rates. Well under $250.

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. *(Unchanged from v1/v2/v3.)*

## Perturbation direction logic
Direction during escape is determined by antipode derivation from the current axis pool (mutable — same as v3). Main-loop perturbation logic (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1/v2/v3.

The axis pool and probe seed store are the two state structures that perturbation direction logic reads. The probe seed store does not change how antipodes are derived — only which axes are admitted to the pool in the first place.

## Re-integration
The escape candidate replaces the current-best *only if it clears the regression gate* (≥ 80% of current-best score). *(Unchanged from v2/v3.)* The adaptive axis step (including orthogonality verification) fires *before* the loop resumes, so new verified axes are available on the next escape event, not the current one.

## Estimated cost per run
~$27–38 for a standard 30-iteration run with 1–2 escape events (vs ~$26–35 in v3). The orthogonality verification adds ~$1–3 per adaptive step: 1 opus call per proposed axis for probe seed generation + comparison (2 proposed axes = 2 calls minimum), plus up to 2 retry calls per failing axis. Harness initialisation adds a one-time cost of ~$0.20 for the 3 starting probe seeds. Verification only fires on adaptive steps, which only fire on consecutive gate failures — cost increase is bounded and triggered only when the mechanism is actively needed. Well under $250.

---

## What changed this iteration (v3 → v4)

**Element changed:** Axis orthogonality verification — a structural gate on pool admission. D1 (Landscape coverage) and D3 (Perturbation quality).

---

### The change — Axis orthogonality verification (Step 5b-bis)

**v3 state:** When the dissection agent generated new axes in Step 5b, those axes were appended to the pool immediately on the basis of the dissection agent's own assertion that they were orthogonal to existing axes. The LLM generated the axis AND adjudicated its own orthogonality claim. Nothing external validated this. Two generated axes could describe structurally similar candidate types while having different names and definitions — the pool would grow but coverage would not, and escape events would be wasted on overlapping conceptual territory.

**v4 change:** Before any proposed axis is admitted to the pool, it must clear an orthogonality verification step:

1. A **probe seed** is generated — a 2–3 sentence sketch of what a candidate built on this axis would look like structurally. This concretises the axis into an instantiated candidate character, making the comparison concrete rather than definitional.

2. The probe seed is compared against the stored probe seeds of all currently active pool axes, in a **single opus call** that judges: "Do these probe seeds describe structurally different types of candidates, or variants within the same region?" The evaluator model (opus) is used because this is a conceptual reasoning task, not a similarity metric.

3. If the proposed axis is rated **overlapping** with any existing axis: the dissection agent regenerates it (up to 2 retries), receiving the failure rationale to guide the replacement toward a genuinely different region.

4. If still overlapping after 2 retries: the axis is **dropped**.

5. Critically: **retired axes' probe seeds are retained in the probe seed store.** This means the probe seed store is a monotonically growing map of explored conceptual regions — the harness can refuse to re-enter regions it has already exhausted, not just regions currently active in the pool.

---

### Why this addresses the evaluator's weakest element (Axis orthogonality)

All three v3 judges flagged the same weakness: "axis orthogonality is asserted but never verified." The evaluator suggested a semantic distance check between new-axis seed outputs and existing-axis seed outputs. V4 implements this at the axis level (before seeds are generated and evaluated), using probe seeds rather than full candidate evaluations — which is cheaper and fires at exactly the right point in the pipeline.

The probe seed approach is a deliberate design choice: it forces the axis to instantiate into a candidate sketch before the comparison. This makes the check conceptually grounded. Two axes that sound different but would produce structurally similar candidates will have similar probe seeds — the overlap is detectable at this level, before the harness invests in a full 5-iteration mini-sprint.

The use of the evaluator model (opus) for the comparison is also deliberate: this is not a task for cosine similarity or BM25. Structural orthogonality in essay/strategy/thesis problems requires reasoning about what different argumentative gestures feel like, which is a high-level semantic judgement. Opus is the right tool.

---

### What this does NOT change:
- Escape trigger (still 10 consecutive discards on first trigger; counter resets to 5/3 on failed gate)
- Starting axis pool (frame / causal claim / scope — still the initial set)
- Axis generation mechanism (still LLM-driven from evaluator feedback language — unchanged from v3)
- Antipode derivation logic (inversion or 90° rotation — unchanged)
- Mini-sprint length (5 iterations — unchanged from v2/v3)
- Regression gate threshold (0.80 — unchanged from v2/v3)
- Progressive counter shortening on gate failure (unchanged from v2/v3)
- Axis retirement logic (3+ consecutive fails — unchanged from v3)
- Population structure (single thread — unchanged)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- evaluate.md (frozen throughout — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
