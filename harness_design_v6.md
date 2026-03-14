# Harness Design — v6 (Basin Clustering Detection)
*Autoresearch escape loop — sixth iteration*
*260314*

---

## Escape trigger
Same as v2/v3/v4: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4.)*

**Basin clustering detection (new in v6):** After every escape event in which the regression gate passes, the harness checks whether the loop is circling a basin. The check is: have the last 3+ consecutive escape events each produced a best seed that scores within 2% of the current best score?

> **Basin condition:** For each of the most recent N consecutive escape events (N ≥ 3), the highest-scoring seed from that event satisfies: `seed_score ≥ current_best_score × 0.98`. If this condition holds across 3+ consecutive events, basin clustering is detected and the crossover step fires **before** the next main-loop iteration.

The 2% threshold is applied to the **best seed from each escape event**, not the average score across seeds. A single strong outlier seed does not mask basin clustering — the check requires the best performer from each event to be near the top, meaning every escape event is converging on the same neighbourhood.

The basin clustering check fires independently of the adaptive axis discovery trigger (2+ consecutive gate failures). Both can be active simultaneously. Basin clustering detection can fire on events where the gate **passed** — the loop is still making progress by the regression gate's standard, but is circling the same peak.

---

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, and Basin Clustering Crossover.**

When the standard escape trigger fires, the harness runs the full Gated Antipode Escape (Steps 1–4 below). The adaptive axis discovery step (Step 5) fires when the regression gate has failed on 2+ consecutive escape events. The crossover step (Step 6) fires when basin clustering is detected — 3+ consecutive escape events whose best seed scores within 2% of the current best. Steps 1–5 are unchanged from v4. Step 6 is new in v6.

---

### Step 1 — Structural dissection with live axis pool *(Unchanged from v3/v4)*

A dedicated dissection agent reads the current-best and extracts structural elements from the **current axis pool**. The axis pool begins with the same three axes as v2/v3/v4:

- **Frame** — the central metaphor or conceptual lens
- **Causal claim** — the primary mechanism asserted
- **Scope** — the level of analysis in play

The axis pool is a mutable ordered list, stored in the run's working state. It starts at length 3. Axes are added and retired dynamically (see Step 5 below). The dissection agent always reads from the current pool, not the fixed original set.

---

### Step 2 — Antipode generation *(Unchanged from v3/v4)*

For each axis in the current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record (most gate-passing events) plus any newly added axes, capped at 5 seeds per escape event to contain cost.

---

### Step 3 — Extended mini-evaluation sprint *(Unchanged from v3/v4)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Unchanged from v3/v4)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** escape candidate replaces current-best. Discard counter resets to 0. The winning axis logs a "pass" event. Basin clustering counter is updated (see Step 6a).
- **Gate fails:** current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used in the failed escape logs a "fail" event. The escape event is excluded from the basin clustering check (only gate-passing events contribute to the basin window).

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Unchanged from v4)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events — meaning the current axis pool is generating seeds that cluster rather than escaping to genuinely different regions.

When this condition is met, the dissection agent performs an additional analysis pass **before** resetting the discard counter and resuming the main loop:

#### 5a — Evaluator feedback analysis *(Unchanged from v3/v4)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event. It identifies which dimensions scored lowest across all seeds in the failed escape (e.g., "D3 consistently 5/10 across all three seeds, with evaluator language referencing 'insufficient argumentative differentiation' and 'same epistemic stance'").

This is not structural analysis of the candidate itself — it is linguistic analysis of the evaluator's feedback, specifically the natural-language justifications the evaluator provided for its dimensional scores.

#### 5b — LLM-driven axis generation *(Unchanged from v3/v4)*

The dissection agent generates **2 new structural axes** by prompting itself with the following:

> "You are a structural analysis agent. The current escape pool (axes: [list]) has failed to find a viable escape candidate on [N] consecutive attempts. The evaluator's feedback on the failed seeds consistently flagged: [quoted evaluator language from weakest dimensions]. Based on this feedback, propose 2 new perturbation axes that would generate meaningfully different candidates for this specific problem. Each axis must: (1) be orthogonal to all current pool axes, (2) be derivable from the current-best without human input, (3) be general enough to apply across essay/strategy/thesis structure. Name each axis, define its antipode operation, and explain why it addresses the evaluator's flagged weaknesses."

The dissection agent's response is the canonical source of the proposed axes. No human review. The two proposed axes then enter the orthogonality verification step (5b-bis) before being admitted to the pool.

---

#### 5b-bis — Axis orthogonality verification *(Unchanged from v4)*

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

#### 5c — Axis pool management (retirement) *(Unchanged from v3/v4)*

An axis is **retired** from the pool when it has accumulated 3+ consecutive fail events with no intervening pass. Retirement is permanent for the current run. If retirement would reduce the pool below 3 axes, retirement is deferred until the adaptive step can replace it.

---

### Step 6 — Structural crossover on basin clustering detection *(New in v6 — the single change)*

**Trigger condition:** 3+ consecutive escape events in which the best seed scored within 2% of the current best (basin clustering detected — see Escape trigger section). The basin window is maintained in the run's working state and resets when a crossover fires or when a non-basin escape event occurs (i.e., a gate-passing event whose best seed scores more than 2% above the current best at the time of that event).

**Why this fires proactively:** The regression gate measures whether an escape candidate is good enough to replace the current best. It does not measure whether the escape is finding new territory. A loop can pass the regression gate on every escape event — making incremental progress — while still circling the same basin. Basin clustering detection catches this: if every escape event's best seed is landing within 2% of the top, the search is converging, not exploring. Crossover fires before the loop wastes further iterations confirming what the basin window already signals.

**Why the best-seed threshold (not average):** Using the average score would allow a single strong outlier seed to mask basin clustering — the other seeds could be close to the top while the average stays above 2%. The best-seed threshold is stricter: it requires that even the strongest performer from each escape event is still in the basin neighbourhood. If the best seed is within 2%, there is no standout escape direction in the current pool. Crossover is warranted.

---

#### 6a — Basin window maintenance

The harness maintains a running list of recent gate-passing escape events, recording the best seed score from each. Failed-gate escape events are excluded from the list. The list is rolling: only the most recent consecutive events that each meet the basin condition are counted. As soon as an event produces a best seed scoring more than 2% above the current best at that point, the window resets to zero.

After each gate-passing escape event, the harness evaluates:

> "Of the last N gate-passing escape events, how many had a best seed score within 2% of the current best at the time of that event?"

If N ≥ 3 and all N are in the basin window, crossover fires.

---

#### 6b — Structural decomposition of historical bests

When crossover fires, the harness selects the **top-3 historically best candidates** from the run's working state (measured by the best score achieved at any point during each candidate's lifespan, not just their final score before replacement). If fewer than 3 distinct candidates are on record, the crossover step uses however many are available (minimum 2; if only 1 is available, crossover is deferred until a second distinct candidate exists in history).

For each selected candidate, a dissection agent performs a **structural decomposition** — breaking the candidate into its atomic structural elements: the components that, if reassembled differently, would produce a meaningfully different candidate. The decomposition prompt is:

> "You are a structural decomposition agent. Read the candidate below and extract its atomic structural elements — the load-bearing components that define its character. These should be small enough to be combinable with elements from different candidates, but large enough to carry structural identity. Label each element with an emergent name (do not use pre-configured categories). Output a numbered list of 5–8 elements, each with: element name, 1-sentence definition, and a 1-sentence description of what this element contributes to the candidate's overall structure."

The decomposition agent produces a separate element inventory for each of the 3 historical bests. The inventories are then **aligned post-hoc** by the crossover agent: element labels are compared across inventories and mapped to functional equivalents where they exist (e.g., "central metaphor" in candidate A ≈ "governing analogy" in candidate B). This alignment is explicit — the crossover agent outputs a cross-candidate element map before generating hybrids.

---

#### 6c — Recombination and crossover sprint

The crossover agent generates **3 hybrid candidates** by sampling elements from different inventories according to the following constraint:

> Each hybrid must draw at least 2 elements from different source candidates. No hybrid may be dominated by elements from a single source candidate (no more than 60% of elements from one source). Elements must be compatible — the crossover agent discards combinations that would produce internal incoherence (e.g., a frame element that directly contradicts the causal claim element drawn from a different candidate) and regenerates until a coherent combination is found.

Each hybrid candidate is run through a **5-iteration greedy sub-run** (same iteration agent + evaluator, same frozen evaluate.md), identical in structure to the mini-sprint in Step 3. All three hybrids are run in parallel.

---

#### 6d — Regression-gated winner selection

The highest-scoring hybrid after its 5-iteration sprint must clear the same regression gate as standard escape candidates:

> **Gate condition:** crossover_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** crossover candidate replaces current-best. Discard counter resets to 0. Basin window resets to zero.
- **Gate fails:** current-best retained. Basin window resets to zero (crossover has fired; the basin has been addressed regardless of outcome). The discard counter is unchanged by the crossover step.

**No escalation path beyond crossover:** If the crossover candidates all fail the regression gate, the loop resumes with the current best and the existing axis pool. The basin window resets, so the next 3 escape events start a fresh clustering check. There is no further escalation — the harness accepts this as a genuine local optimum and continues exploring from it.

---

**Properties of the crossover mechanism:**
- **Draws on success signal** — recombines elements from historically strong candidates, not failure-mode patches
- **Fires in typical runs** — basin clustering is common in productive hill-climbing; crossover now activates as a standing search strategy, not a rare edge-case fallback
- **Sensitive trigger** — 2% threshold and best-seed measurement mean crossover fires early, before the loop wastes iterations confirming it's stuck
- **Bounded cost** — 3 hybrids × 5 iterations = 15 oracle evaluations (~$3–6 for the sprint); plus 3 decomposition calls + 1 alignment call (~$1.50–2.50 for decomposition); total per crossover event: ~$5–9. Well under $250.
- **Non-destructive** — regression gate protects current-best; crossover failure leaves the main thread intact
- **Basin window resets on crossover** — whether crossover passes or fails, the basin check starts fresh; the harness does not immediately re-trigger crossover on the next 3 events

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. *(Unchanged from v1/v2/v3/v4.)*

## Perturbation direction logic
Direction during standard escape is determined by antipode derivation from the current axis pool (mutable — same as v3/v4). Direction during crossover is determined by structural element recombination across historical bests (new in v6). Main-loop perturbation logic (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1/v2/v3/v4.

The axis pool, probe seed store, and historical best inventory are the three state structures that perturbation direction logic reads. The historical best inventory is a lightweight record of each distinct current-best candidate the run has held, with its peak score — it grows monotonically and is the source of candidates for crossover decomposition.

## Re-integration
The escape candidate (antipodal or crossover) replaces the current-best *only if it clears the regression gate* (≥ 80% of current-best score). *(Unchanged from v2/v3/v4.)* The adaptive axis step fires *before* the loop resumes on gate failure. The crossover step fires *before* the next main-loop iteration when basin clustering is detected, regardless of whether the current escape event is a gate-pass or following a gate-pass run.

## Estimated cost per run
~$30–45 for a standard 30-iteration run with 1–2 escape events and 1 crossover event (vs ~$27–38 in v4). The crossover step adds ~$5–9 per activation (3 decomposition calls + alignment + 3 × 5-iteration sprints). Basin clustering is expected to fire 1–2 times in a typical productive run (the loop converges naturally as it improves). Well under $250.

---

## What changed this iteration (v4 → v6)

*(v5 was discarded — this iteration is v4 as the base, with one change.)*

**Element changed:** Crossover trigger — basin clustering detection replaces stall-only activation. Targets D1 (Landscape coverage) and D2 (Escape reliability).

---

### The change — Crossover trigger: basin clustering detection (Step 6)

**v4 state:** v4 did not include a crossover mechanism. The only structural escalation available was adaptive axis discovery (Step 5), which fired on gate failure — a reactive signal that the current pool has exhausted useful escape directions.

**v5 state (discarded):** v5 introduced a crossover mechanism but triggered it only on stall — specifically, when both newly proposed axes failed orthogonality verification after retries (0 new axes after full retry exhaustion). The v5 evaluators correctly identified that stall is a rare edge case. In most productive runs, the loop passes the gate and the adaptive step generates valid new axes, so crossover never fires. The mechanism existed but provided no landscape coverage or reliability benefit in typical runs.

**v6 change:** The crossover mechanism from v5 is retained **unchanged in its operation** (decompose top-3 historical bests, recombine elements, 5-iteration sprint, regression gate). Only the trigger changes.

**New trigger:** Crossover fires when **basin clustering is detected** — 3+ consecutive gate-passing escape events where the best seed from each event scores within 2% of the current best score at that point.

This is a proactive trigger, not a failure signal. The loop does not need to stall to earn a crossover. If every escape event's best seed is landing within 2% of the top, the axis-pool search is systematically circling the same peak — the harness detects this and fires crossover **before** the next standard escape iteration.

---

### Why best-seed threshold, not average

The 2% threshold is applied to the **best seed from each escape event**, not the average. The design intent is to detect that there is no standout escape direction in the current pool — that even the strongest candidate the pool can generate is stuck in the basin neighbourhood. Using the average would allow a single strong outlier to mask clustering. Best-seed is a stricter and more meaningful signal: if even the best performer is within 2%, the pool has no viable escape vector in the current landscape region.

---

### Why this makes crossover a standing search strategy

The evaluator's v5 critique was precise: "the mechanism fires only on rare stall events, providing no landscape coverage or reliability benefit in typical runs." Basin clustering is not rare. In productive hill-climbing, the loop regularly converges toward a local peak across multiple escape events — each event passes the regression gate (incremental progress is real) but the search is narrowing. The basin clustering check catches this convergence pattern and fires crossover at exactly the right moment: after enough evidence to be confident the loop is circling (N=3 events), but before the loop wastes further iterations confirming it.

The 2% threshold is intentionally sensitive. A looser threshold (e.g., 5%) would delay crossover until the loop is deeply stuck. At 2%, crossover fires early — in the growth phase of convergence, when the current pool is still finding small improvements but the landscape is narrowing. Early crossover means the recombination mechanism explores genuinely different territory while the main thread is still strong enough to generate coherent hybrids.

---

### What this does NOT change:
- Standard escape trigger (10 consecutive discards → 5 → 3 on failed gates — unchanged from v2/v3/v4)
- Antipode escape mechanism (Steps 1–4 — unchanged from v2/v3/v4)
- Adaptive axis discovery (Step 5 — unchanged from v4, still triggers on 2+ consecutive gate failures)
- Axis orthogonality verification (Step 5b-bis — unchanged from v4)
- Probe seed store (monotonically growing coverage map — unchanged from v4)
- Crossover mechanism internal operation (decompose top-3, recombine, 5-iteration sprint, regression gate — identical to v5, only trigger changes)
- Starting axis pool (frame / causal claim / scope — unchanged)
- Mini-sprint length (5 iterations — unchanged from v2/v3/v4)
- Regression gate threshold (0.80 — unchanged from v2/v3/v4)
- Progressive counter shortening on gate failure (unchanged from v2/v3/v4)
- Axis retirement logic (3+ consecutive fails — unchanged from v3/v4)
- Population structure (single thread — unchanged)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- evaluate.md (frozen throughout — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
