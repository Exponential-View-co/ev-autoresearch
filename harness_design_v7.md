# Harness Design — v7 (Crossover Novelty Verification)
*Autoresearch escape loop — seventh iteration*
*260314*

---

## Escape trigger
Same as v2/v3/v4/v6: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4/v6.)*

**Basin clustering detection (unchanged from v6):** After every escape event in which the regression gate passes, the harness checks whether the loop is circling a basin. The check is: have the last 3+ consecutive escape events each produced a best seed that scores within 2% of the current best score?

> **Basin condition:** For each of the most recent N consecutive escape events (N ≥ 3), the highest-scoring seed from that event satisfies: `seed_score ≥ current_best_score × 0.98`. If this condition holds across 3+ consecutive events, basin clustering is detected and the crossover step fires **before** the next main-loop iteration.

The 2% threshold is applied to the **best seed from each escape event**, not the average score across seeds. A single strong outlier seed does not mask basin clustering — the check requires the best performer from each event to be near the top, meaning every escape event is converging on the same neighbourhood.

The basin clustering check fires independently of the adaptive axis discovery trigger (2+ consecutive gate failures). Both can be active simultaneously. Basin clustering detection can fire on events where the gate **passed** — the loop is still making progress by the regression gate's standard, but is circling the same peak.

---

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, Basin Clustering Crossover, and Crossover Novelty Verification.**

When the standard escape trigger fires, the harness runs the full Gated Antipode Escape (Steps 1–4 below). The adaptive axis discovery step (Step 5) fires when the regression gate has failed on 2+ consecutive escape events. The crossover step (Step 6) fires when basin clustering is detected — 3+ consecutive escape events whose best seed scores within 2% of the current best. Steps 1–5 are unchanged from v4/v6. Step 6 is changed in v7: a structural novelty verification step (6c-bis) is inserted between hybrid generation and mini-sprints.

---

### Step 1 — Structural dissection with live axis pool *(Unchanged from v3/v4/v6)*

A dedicated dissection agent reads the current-best and extracts structural elements from the **current axis pool**. The axis pool begins with the same three axes as v2/v3/v4/v6:

- **Frame** — the central metaphor or conceptual lens
- **Causal claim** — the primary mechanism asserted
- **Scope** — the level of analysis in play

The axis pool is a mutable ordered list, stored in the run's working state. It starts at length 3. Axes are added and retired dynamically (see Step 5 below). The dissection agent always reads from the current pool, not the fixed original set.

---

### Step 2 — Antipode generation *(Unchanged from v3/v4/v6)*

For each axis in the current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record (most gate-passing events) plus any newly added axes, capped at 5 seeds per escape event to contain cost.

---

### Step 3 — Extended mini-evaluation sprint *(Unchanged from v3/v4/v6)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Unchanged from v3/v4/v6)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** escape candidate replaces current-best. Discard counter resets to 0. The winning axis logs a "pass" event. Basin clustering counter is updated (see Step 6a).
- **Gate fails:** current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used in the failed escape logs a "fail" event. The escape event is excluded from the basin clustering check (only gate-passing events contribute to the basin window).

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Unchanged from v4/v6)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events — meaning the current axis pool is generating seeds that cluster rather than escaping to genuinely different regions.

When this condition is met, the dissection agent performs an additional analysis pass **before** resetting the discard counter and resuming the main loop:

#### 5a — Evaluator feedback analysis *(Unchanged from v3/v4/v6)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event. It identifies which dimensions scored lowest across all seeds in the failed escape (e.g., "D3 consistently 5/10 across all three seeds, with evaluator language referencing 'insufficient argumentative differentiation' and 'same epistemic stance'").

This is not structural analysis of the candidate itself — it is linguistic analysis of the evaluator's feedback, specifically the natural-language justifications the evaluator provided for its dimensional scores.

#### 5b — LLM-driven axis generation *(Unchanged from v3/v4/v6)*

The dissection agent generates **2 new structural axes** by prompting itself with the following:

> "You are a structural analysis agent. The current escape pool (axes: [list]) has failed to find a viable escape candidate on [N] consecutive attempts. The evaluator's feedback on the failed seeds consistently flagged: [quoted evaluator language from weakest dimensions]. Based on this feedback, propose 2 new perturbation axes that would generate meaningfully different candidates for this specific problem. Each axis must: (1) be orthogonal to all current pool axes, (2) be derivable from the current-best without human input, (3) be general enough to apply across essay/strategy/thesis structure. Name each axis, define its antipode operation, and explain why it addresses the evaluator's flagged weaknesses."

The dissection agent's response is the canonical source of the proposed axes. No human review. The two proposed axes then enter the orthogonality verification step (5b-bis) before being admitted to the pool.

---

#### 5b-bis — Axis orthogonality verification *(Unchanged from v4/v6)*

For each proposed axis coming out of step 5b, before appending it to the pool, the harness executes a three-part verification sequence:

**i. Probe seed generation**

The harness generates a **probe seed** for the proposed axis: a 2–3 sentence sketch of what a candidate built on this axis would look like — its essential character and structural identity, not a draft.

**ii. Orthogonality check via evaluator model**

The probe seed for the proposed axis is compared against the probe seeds for all currently active pool axes in a single opus call. The proposed axis passes if it is rated **orthogonal to all existing pool axes**. If rated overlapping with any existing axis, it fails.

**iii. Regeneration on failure (up to 2 retries)**

- **Pass:** The proposed axis and its probe seed are appended to the pool.
- **Fail (overlap detected):** The dissection agent regenerates the axis with an augmented prompt that includes the failure rationale and instructions to target a genuinely different region. Up to **2 regeneration attempts** per proposed axis.
- **Still failing after 2 retries:** The axis is dropped.

**The probe seed store**

Each active axis in the pool carries a stored probe seed. Retired axes' probe seeds are retained — the store is monotonically growing. The harness refuses to re-enter regions it has already exhausted, not just regions currently active in the pool. The store is initialised at harness start with probe seeds for the three starting axes.

---

#### 5c — Axis pool management (retirement) *(Unchanged from v3/v4/v6)*

An axis is **retired** from the pool when it has accumulated 3+ consecutive fail events with no intervening pass. Retirement is permanent for the current run. If retirement would reduce the pool below 3 axes, retirement is deferred until the adaptive step can replace it.

---

### Step 6 — Structural crossover on basin clustering detection *(Mechanism unchanged from v6; novelty verification added in 6c-bis)*

**Trigger condition:** 3+ consecutive escape events in which the best seed scored within 2% of the current best (basin clustering detected — see Escape trigger section). The basin window is maintained in the run's working state and resets when a crossover fires or when a non-basin escape event occurs (i.e., a gate-passing event whose best seed scores more than 2% above the current best at the time of that event).

**Why this fires proactively:** The regression gate measures whether an escape candidate is good enough to replace the current best. It does not measure whether the escape is finding new territory. A loop can pass the regression gate on every escape event — making incremental progress — while still circling the same basin. Basin clustering detection catches this: if every escape event's best seed is landing within 2% of the top, the search is converging, not exploring. Crossover fires before the loop wastes further iterations confirming what the basin window already signals.

**Why the best-seed threshold (not average):** Using the average score would allow a single strong outlier seed to mask basin clustering — the other seeds could be close to the top while the average stays above 2%. The best-seed threshold is stricter: it requires that even the strongest performer from each escape event is still in the basin neighbourhood. If the best seed is within 2%, there is no standout escape direction in the current pool. Crossover is warranted.

---

#### 6a — Basin window maintenance *(Unchanged from v6)*

The harness maintains a running list of recent gate-passing escape events, recording the best seed score from each. Failed-gate escape events are excluded from the list. The list is rolling: only the most recent consecutive events that each meet the basin condition are counted. As soon as an event produces a best seed scoring more than 2% above the current best at that point, the window resets to zero.

After each gate-passing escape event, the harness evaluates:

> "Of the last N gate-passing escape events, how many had a best seed score within 2% of the current best at the time of that event?"

If N ≥ 3 and all N are in the basin window, crossover fires.

---

#### 6b — Structural decomposition of historical bests *(Unchanged from v6)*

When crossover fires, the harness selects the **top-3 historically best candidates** from the run's working state (measured by the best score achieved at any point during each candidate's lifespan, not just their final score before replacement). If fewer than 3 distinct candidates are on record, the crossover step uses however many are available (minimum 2; if only 1 is available, crossover is deferred until a second distinct candidate exists in history).

For each selected candidate, a dissection agent performs a **structural decomposition** — breaking the candidate into its atomic structural elements: the components that, if reassembled differently, would produce a meaningfully different candidate. The decomposition prompt is:

> "You are a structural decomposition agent. Read the candidate below and extract its atomic structural elements — the load-bearing components that define its character. These should be small enough to be combinable with elements from different candidates, but large enough to carry structural identity. Label each element with an emergent name (do not use pre-configured categories). Output a numbered list of 5–8 elements, each with: element name, 1-sentence definition, and a 1-sentence description of what this element contributes to the candidate's overall structure."

The decomposition agent produces a separate element inventory for each of the 3 historical bests. The inventories are then **aligned post-hoc** by the crossover agent: element labels are compared across inventories and mapped to functional equivalents where they exist (e.g., "central metaphor" in candidate A ≈ "governing analogy" in candidate B). This alignment is explicit — the crossover agent outputs a cross-candidate element map before generating hybrids.

---

#### 6c — Recombination and hybrid generation *(Unchanged from v6)*

The crossover agent generates **3 hybrid candidates** by sampling elements from different inventories according to the following constraint:

> Each hybrid must draw at least 2 elements from different source candidates. No hybrid may be dominated by elements from a single source candidate (no more than 60% of elements from one source). Elements must be compatible — the crossover agent discards combinations that would produce internal incoherence (e.g., a frame element that directly contradicts the causal claim element drawn from a different candidate) and regenerates until a coherent combination is found.

The 3 hybrid candidates are held in working state. **They do not proceed to mini-sprints until they have passed the structural novelty verification in Step 6c-bis.**

---

#### 6c-bis — Structural novelty verification for crossover hybrids *(New in v7 — the single change)*

After generating the 3 hybrid candidates but before running any mini-sprints, each hybrid must pass a structural novelty check. The purpose is to enforce that crossover actually does what crossover is supposed to do — produce offspring that explore between peaks, not the average of peaks.

**Why this check is necessary:** The crossover agent's coherence filter (Step 6c) selects against *internally incoherent* combinations. This is a different objective from structural novelty. An internally coherent hybrid can still be a blended average — it takes the safe, compatible elements from each source and combines them without venturing into genuinely new structural territory. The analogy to axis orthogonality verification (Step 5b-bis) is precise: just as a proposed axis must be orthogonal to existing axes before it enters the pool, a crossover hybrid must be structurally distinct from all its source candidates before it is allowed to consume 5 sprint iterations.

**The verification sequence (applied to each hybrid independently):**

**i. Hybrid probe sketch generation**

For each hybrid, the harness generates a **probe sketch**: a 2–3 sentence description of the hybrid's essential structural identity — its governing logic, central tension, and what makes it distinct as a composition. The probe sketch captures character, not content — it is analogous to the axis probe seeds in Step 5b-bis.

The probe sketches for the 3 source candidates are retrieved from the structural decompositions already produced in Step 6b (or generated fresh if not available). Each source candidate's probe sketch is: a 2–3 sentence summary of that candidate's essential structural character, derived from its element inventory.

**ii. Novelty judgement via opus**

The hybrid's probe sketch is compared against the probe sketches of all 3 source candidates in a single opus call. The opus judge returns one of two verdicts:

- **"Genuinely novel recombination"** — the hybrid's structural identity is meaningfully distinct from each source candidate; it explores territory that none of the sources individually occupies.
- **"Blended average"** — the hybrid's structural identity is a weighted midpoint of its sources; it lacks distinctive character that could not be attributed to any source candidate by interpolation.

The verdict prompt is:

> "You are a structural novelty judge. You will receive a probe sketch of a crossover hybrid and probe sketches of its 3 source candidates. Your task: determine whether the hybrid is a genuinely novel recombination or a blended average. A blended average takes safe, compatible elements from each source and combines them without producing structural identity that none of the sources individually contains. A genuinely novel recombination produces a structural character that is emergent — its defining logic, tension, or stance cannot be derived by averaging or interpolating the sources. Consider: does the hybrid have a structural identity that would surprise the authors of the source candidates? Or does it look like a committee-produced compromise? Return exactly one verdict: NOVEL or BLEND. Follow with a 1-sentence rationale."

**iii. Resolution: NOVEL → proceed; BLEND → regenerate or replace**

**NOVEL verdict:** The hybrid is cleared for mini-sprints. It proceeds to Step 6d.

**BLEND verdict — regeneration path (up to 1 retry):**

The crossover agent regenerates the hybrid with an augmented prompt that makes structural distance the primary objective:

> "The hybrid you generated was judged a blended average of its sources. Regenerate this hybrid with one instruction: maximise structural distance from all three source candidates. Do not look for compatible middle ground. Find the combination of elements that produces the most surprising, unexpected structural identity — the one that looks least like any of the three sources individually, even if the result is jarring or unconventional. Coherence constraints remain, but if a choice is between coherent-and-average vs. surprising-and-coherent, choose surprising. Output a new hybrid element combination and a revised probe sketch."

The regenerated hybrid is run through the opus novelty check again (Step 6c-bis, ii). If it passes: proceed to mini-sprints. If it still returns BLEND: enter the maximum-distance fallback.

**BLEND verdict after retry — maximum-distance fallback:**

The harness replaces the failing hybrid with a **maximum-distance hybrid** constructed by the following explicit rule:

> "Identify the single most structurally distinctive element from each of the 3 source candidates — the element that is most unlike anything in the other two inventories. Combine exactly these 3 elements (one from each source), discarding all compromise or bridging elements. Do not attempt to smooth the combination. The result will likely be surprising or even uncomfortable — that is correct. The goal is to reach structural territory that none of the source candidates could reach incrementally. Generate a probe sketch of this maximum-distance hybrid."

The maximum-distance hybrid is not run through the novelty check again — it is definitionally constructed for maximal structural distance and proceeds directly to mini-sprints.

**The "most structurally distinctive element" selection rule:** The crossover agent selects, from each source candidate's element inventory, the element whose 1-sentence definition has the least semantic overlap with any element in the other two inventories. Overlap is judged by the crossover agent from the element definitions produced in Step 6b — no additional opus call is needed. If two elements tie for distinctiveness, the agent selects the one that contributes the most to the source candidate's overall structural identity (as described in its element contribution sentence from the decomposition).

**State tracking for the probe sketch store:** Each hybrid probe sketch — whether from initial generation, regeneration, or the maximum-distance fallback — is stored alongside the hybrid's working state. If the hybrid proceeds to mini-sprints and is later selected for decomposition in a future crossover event, its probe sketch is available for novelty comparison against future hybrids.

---

**Cost impact of 6c-bis:** Each novelty check is a single opus call comparing 4 probe sketches (1 hybrid + 3 source). At ~500 tokens per call: ~$0.05–0.10 per hybrid, $0.15–0.30 for all 3. Regeneration adds 1 additional generation call + 1 opus check per failing hybrid: ~$0.10–0.20 per regeneration. Maximum-distance fallback adds 1 generation call (~$0.05–0.10). Worst case (all 3 hybrids fail and require fallback): +$0.60–0.90 to the crossover event cost. Total per crossover event: ~$5.60–9.90 (vs ~$5–9 in v6). Immaterial.

---

#### 6d — Crossover mini-sprint and regression-gated winner selection *(Unchanged from v6)*

Each hybrid that has cleared the novelty verification in Step 6c-bis is run through a **5-iteration greedy sub-run** (same iteration agent + evaluator, same frozen evaluate.md), identical in structure to the mini-sprint in Step 3. All verified hybrids are run in parallel.

The highest-scoring hybrid after its 5-iteration sprint must clear the same regression gate as standard escape candidates:

> **Gate condition:** crossover_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** crossover candidate replaces current-best. Discard counter resets to 0. Basin window resets to zero.
- **Gate fails:** current-best retained. Basin window resets to zero (crossover has fired; the basin has been addressed regardless of outcome). The discard counter is unchanged by the crossover step.

**No escalation path beyond crossover:** If the crossover candidates all fail the regression gate, the loop resumes with the current best and the existing axis pool. The basin window resets, so the next 3 escape events start a fresh clustering check. There is no further escalation — the harness accepts this as a genuine local optimum and continues exploring from it.

---

**Properties of the crossover mechanism (updated for v7):**
- **Draws on success signal** — recombines elements from historically strong candidates, not failure-mode patches
- **Fires in typical runs** — basin clustering is common in productive hill-climbing; crossover activates as a standing search strategy, not a rare edge-case fallback
- **Sensitive trigger** — 2% threshold and best-seed measurement mean crossover fires early, before the loop wastes iterations confirming it's stuck
- **Hybrids are structurally verified** — novelty check prevents blended averages from consuming sprint capacity; the maximum-distance fallback guarantees a genuinely novel candidate even when the crossover agent defaults to safe recombinations
- **Maximum-distance fallback is bold** — it does not attempt to smooth or bridge; it deliberately combines the most foreign elements from each source, producing something that can look nothing like any individual parent
- **Bounded cost** — novelty verification adds ~$0.60–0.90 worst case per crossover event; total per event: ~$5.60–9.90; well under $250
- **Non-destructive** — regression gate protects current-best; crossover failure leaves the main thread intact
- **Basin window resets on crossover** — whether crossover passes or fails, the basin check starts fresh

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. *(Unchanged from v1/v2/v3/v4/v6.)*

## Perturbation direction logic
Direction during standard escape is determined by antipode derivation from the current axis pool (mutable — same as v3/v4/v6). Direction during crossover is determined by structural element recombination across historical bests, now verified for structural novelty before sprint (new in v7). Main-loop perturbation logic (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1/v2/v3/v4/v6.

The axis pool, probe seed store, and historical best inventory are the three state structures that perturbation direction logic reads. The historical best inventory is a lightweight record of each distinct current-best candidate the run has held, with its peak score — it grows monotonically and is the source of candidates for crossover decomposition. The hybrid probe sketch store (new in v7) is a fourth state structure, appended to during crossover events and consulted by future novelty checks.

## Re-integration
The escape candidate (antipodal or crossover) replaces the current-best *only if it clears the regression gate* (≥ 80% of current-best score). *(Unchanged from v2/v3/v4/v6.)* The adaptive axis step fires *before* the loop resumes on gate failure. The crossover step fires *before* the next main-loop iteration when basin clustering is detected. Crossover hybrids must now pass structural novelty verification (Step 6c-bis) before they are eligible to run mini-sprints.

## Estimated cost per run
~$30–46 for a standard 30-iteration run with 1–2 escape events and 1 crossover event (vs ~$30–45 in v6). The novelty verification step adds ~$0.60–0.90 worst case per crossover event (3 opus calls + potential regeneration/fallback overhead). Well under $250.

---

## What changed this iteration (v6 → v7)

**Element changed:** Crossover hybrid quality control — structural novelty verification inserted between hybrid generation (Step 6c) and mini-sprints (Step 6d). New step: 6c-bis. Targets D3 (Perturbation quality) directly; expected to improve D1 (Landscape coverage) as a consequence.

---

### The change — Structural novelty verification for crossover hybrids (Step 6c-bis)

**v6 state:** After generating 3 hybrid candidates in Step 6c, the harness proceeded directly to mini-sprints. The crossover agent's only quality control was a coherence filter — discarding element combinations that would produce internal incoherence. Coherence and novelty are independent properties. A coherent hybrid can be a blended average: it takes the safe, compatible midpoint elements from each source and assembles them without producing structural identity that none of the sources individually contains. The v6 evaluators named this explicitly: "the crossover agent's discretion in judging element 'compatibility'... risks blended averages rather than genuinely novel structural combinations."

Compounding this: the orthogonality verification applied to proposed axes in Step 5b-bis had no equivalent for crossover hybrids. An axis probe seed was verified to be structurally distinct from all other pool axes before the axis was admitted. A crossover hybrid had no analogous verification before it consumed 5 sprint iterations.

**v7 change:** A structural novelty verification step (6c-bis) is inserted after hybrid generation and before mini-sprints. The mechanism is directly analogous to Step 5b-bis.

**The verification sequence:**

1. For each of the 3 hybrids, the harness generates a probe sketch — a 2–3 sentence description of the hybrid's essential structural identity (character, governing logic, central tension). The probe sketches for the 3 source candidates are derived from their existing decomposition inventories.

2. Each hybrid's probe sketch is compared against all 3 source candidate probe sketches in a single opus call. The opus judge returns NOVEL ("genuinely novel recombination — structural identity distinct from each source") or BLEND ("blended average — a weighted midpoint that explores no new territory").

3. **NOVEL → proceed to mini-sprints.** No change to the hybrid.

4. **BLEND → regenerate with an explicit maximise-structural-distance instruction (up to 1 retry).** The regeneration prompt foregrounds surprise and unconventionality over safety: "find the combination that looks least like any of the three sources individually, even if the result is jarring." The regenerated hybrid is re-judged by opus.

5. **Still BLEND after retry → maximum-distance fallback.** The harness constructs a replacement hybrid by an explicit rule: take the *single most structurally distinctive element* from each of the 3 source candidates (the element with least semantic overlap with either of the other inventories) and combine exactly those 3 elements, discarding all bridging or compromise elements. This fallback is not re-judged — it is definitionally constructed for maximal structural distance. It is expected to look like nothing any individual source candidate could have produced incrementally.

---

### Why the maximum-distance fallback must be bold

The maximum-distance fallback is not a graceful degradation. It is a deliberate choice to produce a hybrid that may be surprising or uncomfortable — one that combines the most alien elements from each source rather than looking for coherent middle ground. The rationale: if an agent, given explicit instructions to maximise structural distance, still produces a blended average, the only way to guarantee non-average output is to specify the construction mechanically. Taking one maximally distinctive element from each source and discarding everything else is that mechanical specification. The result explores territory that no individual source could reach incrementally, and that a human crossover agent optimising for safety would never select.

The design principle behind this: crossover's entire purpose is to reach places between peaks that neither parent could reach by small steps. A blended average is the *opposite* of this — it finds the centroid of the peaks. The maximum-distance fallback enforces the original purpose of the operator.

---

### Why the analogy to Step 5b-bis is exact

Step 5b-bis verifies that a proposed axis is orthogonal to existing axes before it enters the pool and generates seeds. The check: probe seed comparison via opus, with regeneration on failure and drop on repeated failure. The structural novelty check in Step 6c-bis is the same pattern applied to hybrids before they enter mini-sprints. The check: probe sketch comparison via opus, with regeneration on failure and maximum-distance fallback on repeated failure. Both steps ensure that the mechanism they gate — axis-based escape and crossover respectively — actually does what it claims to do rather than producing variants of what the system has already explored.

---

### What this does NOT change:
- Standard escape trigger (10 consecutive discards → 5 → 3 on failed gates — unchanged from v2/v3/v4/v6)
- Antipode escape mechanism (Steps 1–4 — unchanged from v2/v3/v4/v6)
- Adaptive axis discovery (Step 5 — unchanged from v4/v6)
- Axis orthogonality verification (Step 5b-bis — unchanged from v4/v6)
- Probe seed store for axes (monotonically growing coverage map — unchanged from v4/v6)
- Basin clustering trigger (3+ consecutive events within 2% — unchanged from v6)
- Historical best inventory and decomposition (Step 6b — unchanged from v6)
- Hybrid generation constraints (60% cap, coherence filter — Step 6c unchanged)
- Mini-sprint structure (5 iterations — unchanged from v2/v3/v4/v6)
- Regression gate threshold (0.80 — unchanged from v2/v3/v4/v6)
- Crossover winner selection and basin window reset (Step 6d — unchanged from v6)
- No-escalation-beyond-crossover rule (unchanged from v6)
- Progressive counter shortening on gate failure (unchanged from v2/v3/v4/v6)
- Axis retirement logic (3+ consecutive fails — unchanged from v3/v4/v6)
- Population structure (single thread — unchanged)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- evaluate.md (frozen throughout — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
