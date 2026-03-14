# Harness Design — v9 (Tournament-Time Structural Divergence Check)
*Autoresearch escape loop — ninth iteration*
*260314*

---

## Escape trigger
Per-thread: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4/v6/v7/v8 — operating independently within each thread.)*

**Basin clustering detection (per-thread, unchanged from v6/v7/v8):** After every escape event in which the regression gate passes, each thread independently checks whether it is circling a basin. The check: have the last 3+ consecutive escape events within that thread each produced a best seed that scores within 2% of that thread's current best score?

> **Basin condition:** For each of the most recent N consecutive gate-passing escape events within a thread (N ≥ 3), the highest-scoring seed from that event satisfies: `seed_score ≥ thread_current_best_score × 0.98`. If this condition holds across 3+ consecutive events, basin clustering is detected and the crossover step fires **before** the next main-loop iteration within that thread.

The 2% threshold is applied to the **best seed from each escape event**, not the average. Basin clustering detection fires independently for each thread. Both the adaptive axis trigger (2+ consecutive gate failures) and the basin clustering trigger can be simultaneously active within a thread.

---

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, Basin Clustering Crossover, Crossover Novelty Verification, Lightweight Population Parallelism, and Tournament-Time Structural Divergence Check.**

All escape mechanisms from v8 (Steps 1–6d) are preserved and operate identically within each thread. The sole structural change is the addition of a **structural divergence check** that fires at the start of each tournament event, before the standard tournament procedure (T1–T5). This check detects cross-thread basin convergence and enforces re-diversification before tournament retirement and spawn can run.

---

### Step 1 — Structural dissection with live axis pool *(Per-thread, unchanged from v7/v8)*

A dedicated dissection agent reads the thread's current-best and extracts structural elements from that thread's **current axis pool**. Each thread's pool starts at length 3 and is mutable. Dissection always reads from the thread's own current pool.

---

### Step 2 — Antipode generation *(Per-thread, unchanged from v7/v8)*

For each axis in the thread's current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record plus any newly added axes, capped at 5 seeds per escape event.

---

### Step 3 — Extended mini-evaluation sprint *(Per-thread, unchanged from v7/v8)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Per-thread, unchanged from v7/v8)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ thread_current_best_score × 0.80

- **Gate passes:** escape candidate replaces thread's current-best. Discard counter resets to 0. The winning axis logs a "pass" event. Basin clustering counter is updated.
- **Gate fails:** thread's current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used logs a "fail" event. Failed events are excluded from the basin clustering window.

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Per-thread, unchanged from v7/v8)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events within the thread.

#### 5a — Evaluator feedback analysis *(Per-thread, unchanged from v7/v8)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event within the thread. It identifies which dimensions scored lowest across all seeds in the failed escape.

#### 5b — LLM-driven axis generation *(Per-thread, unchanged from v7/v8)*

The dissection agent generates **2 new structural axes** by prompting itself with the standard axis-generation prompt (pool list, evaluator language, orthogonality and generality requirements). No human review.

#### 5b-bis — Axis orthogonality verification *(Per-thread, unchanged from v7/v8)*

For each proposed axis, the harness:

**i.** Generates a **probe seed** (2–3 sentence sketch of what a candidate built on this axis would look like).

**ii.** Compares the probe seed against all currently active pool axes in a single opus call. Proposed axis passes if orthogonal to all existing pool axes.

**iii.** Up to 2 regeneration attempts on failure. Axis is dropped after 2 failed retries.

**The probe seed store** is per-thread and monotonically growing. Retired axes' probe seeds are retained. The store is initialised at thread start with probe seeds for the three starting axes assigned to that thread.

#### 5c — Axis pool management (retirement) *(Per-thread, unchanged from v7/v8)*

An axis is retired when it accumulates 3+ consecutive fail events with no intervening pass. Retirement is permanent within the thread. Retirement deferred if it would reduce the pool below 3 axes.

---

### Step 6 — Structural crossover on basin clustering detection *(Per-thread, unchanged from v7/v8)*

**Trigger condition:** 3+ consecutive gate-passing escape events within the thread whose best seed scored within 2% of that thread's current best.

#### 6a — Basin window maintenance *(Per-thread, unchanged from v7/v8)*

The thread maintains a rolling list of recent gate-passing escape events. Failed-gate events are excluded. The window resets when a crossover fires or when a non-basin escape event occurs (best seed more than 2% above thread's current best at time of event).

#### 6b — Structural decomposition of historical bests *(Per-thread, unchanged from v7/v8)*

When crossover fires within a thread, the harness selects the **top-3 historically best candidates** from that thread's working state (by peak score). For each, a dissection agent produces a structural decomposition: 5–8 atomic structural elements with name, 1-sentence definition, and contribution description. The crossover agent aligns element inventories cross-candidate with an explicit cross-candidate element map.

#### 6c — Recombination and hybrid generation *(Per-thread, unchanged from v7/v8)*

The crossover agent generates **3 hybrid candidates** satisfying:
- At least 2 elements from different source candidates per hybrid
- No more than 60% of elements from one source
- Internal coherence enforced (incoherent combinations discarded and regenerated)

Generated hybrids are held in working state pending novelty verification.

#### 6c-bis — Structural novelty verification for crossover hybrids *(Per-thread, unchanged from v7/v8)*

After generating 3 hybrid candidates but before mini-sprints, each hybrid must pass structural novelty verification.

**i. Hybrid probe sketch generation:** A 2–3 sentence description of the hybrid's essential structural identity — governing logic, central tension, what makes it distinct.

**ii. Novelty judgement via opus:** Hybrid probe sketch compared against all 3 source candidate probe sketches. Verdict: NOVEL ("genuinely novel recombination — structural identity distinct from each source") or BLEND ("blended average — a weighted midpoint that explores no new territory").

Prompt: *"You are a structural novelty judge. You will receive a probe sketch of a crossover hybrid and probe sketches of its 3 source candidates. Determine whether the hybrid is a genuinely novel recombination or a blended average. A blended average takes safe, compatible elements from each source without producing structural identity none of the sources individually contains. A genuinely novel recombination produces emergent character — its defining logic, tension, or stance cannot be derived by averaging or interpolating the sources. Consider: does the hybrid have a structural identity that would surprise the authors of the source candidates? Or does it look like a committee-produced compromise? Return exactly one verdict: NOVEL or BLEND. Follow with a 1-sentence rationale."*

**iii. Resolution:**

- **NOVEL:** Hybrid cleared for mini-sprints.
- **BLEND → regenerate (up to 1 retry):** Augmented prompt foregrounds surprise: *"maximise structural distance from all three source candidates... choose surprising-and-coherent over coherent-and-average."* Regenerated hybrid re-judged by opus.
- **Still BLEND after retry → maximum-distance fallback:** Harness constructs replacement hybrid by taking the single most structurally distinctive element from each of the 3 source candidates (least semantic overlap with the other inventories) and combining exactly those 3 elements, discarding all bridging or compromise elements. Not re-judged — definitionally constructed for maximal structural distance.

Hybrid probe sketches (initial, regenerated, or fallback) stored per-thread alongside working state.

#### 6d — Crossover mini-sprint and regression-gated winner selection *(Per-thread, unchanged from v7/v8)*

Each novelty-verified hybrid runs a **5-iteration greedy sub-run** against the same frozen evaluate.md. All verified hybrids run in parallel (within the thread).

Highest-scoring hybrid must clear the regression gate:
> `crossover_candidate_score ≥ thread_current_best_score × 0.80`

- **Gate passes:** Replaces thread's current-best. Discard counter resets to 0. Basin window resets to zero.
- **Gate fails:** Thread's current-best retained. Basin window resets to zero. Discard counter unchanged.

**No escalation beyond crossover:** If all crossover candidates fail the gate, the thread resumes with its current best. Basin window resets; next 3 escape events start a fresh clustering check.

---

## Population structure *(Unchanged from v8 — three independent threads with tournament selection every 15 iterations)*

### Thread initialisation

The harness spawns **3 threads** at run start. Each thread is a complete, self-contained instance of the v7/v8 harness — its own current-best candidate, its own axis pool, its own probe seed store, its own historical best inventory, its own escape trigger counter, its own basin window, its own discard counter.

**Differentiated axis pool seeds:** The three threads are seeded with non-overlapping axis pools. At harness start, the harness generates probe seeds for candidate seed axes and verifies mutual orthogonality across all 9 starting axes (3 per thread × 3 threads) in a single opus call. The 9 axes are accepted only if all 9 are mutually orthogonal — otherwise the harness regenerates the set (up to 3 retries).

**Default starting axis pools (example assignment — actual pools verified by orthogonality check at run time):**

- **Thread 1 seed:** Frame (central metaphor / conceptual lens), Scope (level of analysis), Temporal frame (where in time the argument is situated)
- **Thread 2 seed:** Causal claim (primary mechanism asserted), Evidence base (what class of evidence anchors the argument), Audience stance (the implied reader's prior position the argument assumes)
- **Thread 3 seed:** Epistemic register (the type of claim being made — empirical/normative/predictive), Resolution mode (how the argument proposes to settle disagreement), Generative tension (the central contradiction the argument holds in productive suspension)

**Thread labelling:** Threads are labelled T1, T2, T3 and carry their label in working state. Labels are used for logging and tournament bookkeeping only.

---

### Independent operation (between tournaments)

Between tournament events, threads are fully independent. They do not share state, do not share candidates, do not share axis pools, do not share escape counters, and do not share basin windows. Each thread runs the full v7/v8 mechanism independently. Threads run **concurrently** — the harness does not wait for one thread to complete an iteration before advancing another.

---

### Tournament selection *(NEW step SD added in v9 — all other steps unchanged from v8)*

**Trigger:** Every 15 main-loop iterations completed across the total run (sum of all iterations across all active threads), OR when any thread reaches a convergence state (escape trigger has fired and the regression gate has failed on 3 consecutive escape events with no improvement — the thread has exhausted its axis pool's current escape capacity).

**Tournament procedure — v9 adds Step SD (Structural Divergence Check) as a pre-step before T1–T5:**

---

#### Step SD — Structural divergence check *(NEW in v9)*

**This step fires at every tournament event, before T1–T5.** It is a correction step, not a selection step. It runs even when all threads are performing well.

**SD1 — Pairwise structural anatomy judgement.**

After waiting for any in-progress escape or crossover events to complete (same waiting logic as T1), the harness extracts a **structural anatomy sketch** from each of the three thread current-bests. A structural anatomy sketch is a 3–5 sentence description of a candidate's essential structural identity: its governing logic, central framing approach, epistemic register, and primary argument stance. The sketch is generated by the dissection agent from the candidate text itself — not from the axis pool labels.

The harness then submits all three pairwise comparisons (T1 vs T2, T1 vs T3, T2 vs T3) to opus in a single call. For each pair, opus returns exactly one verdict: **DISTINCT** or **CONVERGED**.

**Convergence threshold is intentionally sensitive.** The oracle prompt for each pairwise comparison:

> *"You are a structural convergence judge. You will receive the structural anatomy sketches of two research candidates produced by independent search threads. Determine whether these two candidates occupy structurally distinct regions of the solution space, or whether they have converged to the same attractor basin.*
>
> *Use a sensitive convergence threshold. CONVERGED means: the two candidates share the same fundamental framing approach, operate in the same epistemic register, or would be recognisable as variants of the same core idea to a domain expert — even if their surface content differs and even if their scores differ substantially. Similar framing, same argumentative stance, same type of claim, same conceptual lens — any of these count as convergence.*
>
> *DISTINCT means: a domain expert would immediately recognise these as representatives of genuinely different approaches — different governing logic, different epistemic stance, different way of framing what the core problem is.*
>
> *Do NOT use score similarity as evidence. Two candidates can have similar scores from different peaks (fine — DISTINCT) or different scores from the same basin (still CONVERGED). Structural identity is the only criterion.*
>
> *Return exactly one verdict: DISTINCT or CONVERGED. Follow with a 1-sentence rationale naming the specific structural feature that determined your verdict."*

**SD2 — Flag converged threads for re-diversification.**

If any pairwise verdict is CONVERGED:
- In the converged pair, the **lower-scoring thread** is flagged for re-diversification.
- If both threads in a pair have the same score, the thread with higher iteration count (less efficient explorer) is flagged.
- If multiple pairs are CONVERGED and the same thread appears in more than one converged pair, that thread is flagged once (not multiply).
- If multiple distinct threads are flagged (e.g., T1–T2 converged, T2–T3 also converged, flagging T1 and T3), all flagged threads are re-diversified.

**SD3 — Re-diversification (fires before T1–T5).**

For each flagged thread:

**i. History preservation.** The flagged thread's current-best is retired to that thread's historical best inventory (marked with label "SD-retired: tournament [N]"). It is NOT discarded. Its probe sketch, if generated, is retained in the probe seed store. The flagged thread's full historical best inventory continues to be available for crossover decomposition (Step 6b) — re-diversification does not erase the thread's exploratory history.

**ii. New non-overlapping axis seed.** The harness generates a fresh 3-axis seed for the flagged thread using the run-start orthogonality mechanism: probe seed generation + opus orthogonality verification against **all currently active thread pool axes across all three threads** (not just the two surviving threads' starting axes — against each thread's current evolved pool). The new seed must be orthogonal to all active axes in all active thread pools. Up to 3 retries to find a valid non-overlapping triple.

The orthogonality check prompt for the new seed explicitly foregrounds maximum distance: *"The goal is not merely orthogonal — it is maximally distant from all currently active search regions. Prefer axes that would lead to candidates a domain expert would describe as a fundamentally different type of answer, not just a different variation on the existing approaches."*

**iii. Thread reinitialisation.** The flagged thread is reinitialised with:
- **Current-best:** set to the global best candidate (highest-scoring current-best across all threads at this tournament — same as T2's global best promotion, computed jointly)
- **Axis pool:** replaced with the new non-overlapping seed (3 axes)
- **Probe seed store:** cleared of active axis probe seeds; probe seeds for SD-retired and previously retired axes are retained in a read-only history store
- **Discard counter:** reset to 0
- **Basin window:** reset (empty)
- **Escape trigger counter:** reset to standard 10-discard threshold
- **Historical best inventory:** unchanged — all prior candidates (including the just-retired SD-retired candidate) remain in history

The reinitialised thread begins from the global best candidate's position but immediately diverges via its new axis pool. Divergence is structurally guaranteed by the orthogonality of the new axes against all active pools — not by the starting point.

**SD4 — SD ledger update.**

The harness logs to the tournament ledger: whether the divergence check fired, which pairs were judged CONVERGED, which threads were flagged, the structural rationale from opus for each CONVERGED verdict, the new axis labels assigned, and the iteration count at which re-diversification fired.

**SD5 — Resume tournament.**

After all re-diversification is complete, the standard tournament procedure T1–T5 runs. Re-diversified threads participate in T1 with their new current-best (the global best candidate) and their updated state. Note: re-diversified threads start from the global best, so they will not be the lowest-scoring thread at T3 (they share the global best score). The standard tournament therefore retires and replaces whichever non-re-diversified thread holds the lowest score at T1 — the re-diversification step and the retirement step are independent operations.

---

**SD design properties (key):**

- **Structural detection, not score detection.** Two threads can have identical scores from different peaks — DISTINCT, no re-diversification. Two threads can have a 20-point score gap but the same epistemic register — CONVERGED, re-diversification fires. Score is not used.
- **Sensitive threshold by design.** "Similar framing" or "same epistemic register" counts as convergence. The oracle does not wait for full structural collapse. Early detection prevents gradual attractor pull before it fully manifests.
- **Re-diversification preserves history.** The SD-retired candidate and all prior history remain available for crossover decomposition. The flagged thread's exploratory lineage is not erased — it is a navigational correction, not a restart.
- **Fires at every tournament, not just when stuck.** The check runs regardless of whether threads appear to be progressing. Convergence detection is proactive, not reactive.
- **Guaranteed divergence via axis orthogonality, not starting-point distance.** The new thread starts from the global best (optimal starting position) but is pushed into a structurally distinct region from iteration 1 by its new axis pool. The divergence mechanism is the axis pool, not the seed candidate.
- **Single opus call for all three pairwise comparisons.** Three verdicts returned in one call — cost overhead is minimal (~$0.08–0.12 per tournament event).

---

#### T1 — Current-best readout *(Unchanged from v8)*

The harness reads each thread's current-best candidate and its score. If any thread is mid-escape or mid-crossover when the tournament trigger fires, the tournament waits for that thread to complete its current escape/crossover event before proceeding.

---

#### T2 — Global best promotion *(Unchanged from v8)*

The thread holding the highest-scoring current-best is identified. Its current-best is designated the **global best** for this tournament. Logged to the tournament ledger. Tie broken by iteration count (fewer iterations = more efficient explorer wins).

---

#### T3 — Lowest-scoring thread retirement *(Unchanged from v8)*

The thread with the lowest current-best score is retired. Its working state is discarded. Its probe seed store is not transferred. Exception: deferral if thread is mid-escape or mid-crossover.

---

#### T4 — New thread spawn from global best *(Unchanged from v8)*

A new thread is initialised from the global best candidate with a fresh non-overlapping axis seed (3 axes, orthogonality-verified against all 9 currently active thread pool axes, up to 3 retries). Initialises with fresh discard counter (0), fresh basin window (empty), fresh escape trigger (standard 10-discard threshold), probe seed store initialised with probe seeds for its 3 starting axes only.

---

#### T5 — Tournament ledger update *(Unchanged from v8, extended to include SD fields)*

The harness logs: iteration count at tournament, all three thread scores, SD check results (pairs judged, verdicts, re-diversification actions if any), global best score and source thread, retired thread label and score, new thread spawn confirmation, new thread starting axis labels.

---

### Oracle consistency

All three threads evaluate candidates against the same fixed evaluate.md throughout the run. The oracle is frozen. Scores are directly comparable across threads. Tournament winner selection is valid because all scores are measured against the same evaluator.

---

### Global state structures

The run maintains two global state structures in addition to each thread's private state:

**Tournament ledger:** A log of all tournament events — timestamp (iteration count), thread scores at entry, SD divergence check results (pairwise verdicts, flagged threads, re-diversification actions), global best promoted, thread retired, thread spawned, new thread axis labels.

**Global best history:** A monotonically growing list of all candidates ever promoted to global best at any tournament, with scores and source thread labels. Available for crossover events within any thread (Step 6b draws from globally promoted candidates, not just per-thread history).

---

### Termination condition *(Unchanged from v1–v8)*

The run terminates when the global iteration count reaches the configured limit (e.g., 30 iterations). Final output is the global best candidate from the last tournament ledger entry (or the highest-scoring current-best across all active threads if no tournament has fired since the last iteration).

---

## Perturbation direction logic

Direction during standard escape is determined by antipode derivation from the thread's current axis pool (mutable — per-thread). Direction during crossover is determined by structural element recombination across historical bests (includes globally promoted candidates), with structural novelty verification before sprint. Main-loop perturbation direction (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1–v8 — operates per-thread.

**Cross-thread perturbation enrichment at tournament time:** When a new thread is spawned from the global best (Step T4), or when a thread is re-diversified (Step SD3), the global best's structural decomposition (if it exists from a prior crossover event) is available in working state. The new/re-diversified thread's first crossover event can draw on this decomposition as one of the historical bests — giving it richer material for recombination from iteration 1.

---

## Re-integration

Within each thread: escape candidate replaces thread's current-best only if it clears the regression gate (≥ 80% of thread's current-best score). Unchanged from v2–v8. Adaptive axis step fires before loop resumes on gate failure. Crossover fires before next main-loop iteration when basin clustering is detected.

**Cross-thread re-integration:** Global best from any thread becomes the starting candidate for newly spawned threads (T4) and re-diversified threads (SD3). The global best does not replace the current-bests of surviving non-re-diversified threads — they continue from their own current-bests.

---

## Estimated cost per run

~$90–142 for a standard 30-iteration run with 3 threads, 1–2 escape events per thread, 1 crossover event per thread, and 2 tournament events.

- Per-thread cost: ~$30–46 (unchanged from v7/v8 estimate)
- 3 threads × $30–46 = $90–138 baseline
- Tournament orthogonality verification at run start: ~$0.10–0.15 (1 opus call for 9-axis mutual orthogonality check)
- Per-tournament SD check: ~$0.08–0.12 (1 opus call, 3 pairwise verdicts)
- Per-tournament T4 re-seeding: ~$0.05–0.10
- SD re-diversification axis search (when triggered): ~$0.05–0.10 per re-diversified thread
- Estimated 2 tournament events per 30-iteration run, SD re-diversification firing on ~50% of tournaments (1 event): ~$0.30–0.60 additional
- Total: ~$90–142 with SD overhead negligible

Well under $250.

---

## What changed this iteration (v8 → v9)

**Element changed:** Tournament procedure — added structural divergence check (Step SD) as a pre-step before the standard T1–T5 tournament cycle. One change only. All escape mechanisms (Steps 1–6d), population structure, thread initialisation, independent operation, and T1–T5 tournament steps preserved identically.

---

### The change — Tournament-time structural divergence check

**v8 state:** Three independent threads with mutually orthogonal initialisation and fully independent operation between tournaments. At tournament time, the harness reads scores, promotes the global best, retires the lowest-scoring thread, and spawns a new thread from the global best with a fresh axis seed. There is no mechanism to detect whether surviving threads have structurally converged. Global best seeding — the only information that crosses thread boundaries — can act as an attractor: each successive tournament spawns a new thread from the same global best candidate, and over multiple tournaments, this seeding pressure can gradually pull all three threads toward the same structural basin even though their axis pools remain labelled as different. The harness has no awareness of this happening. By the time all three threads score similarly from the same region, the diversity that made population parallelism valuable has been silently eroded.

EV Platform Strategist (v8 eval): "global best seeding could gradually pull new threads toward the same attractor, and the design has no counter for this failure mode." This was the consensus weakest element at 7.72.

**v9 change:** At every tournament event, before T1–T5 runs, the harness compares all three thread current-bests structurally via pairwise opus judgements. If any pair is CONVERGED, the lower-scoring thread of the pair is re-diversified before the standard tournament cycle proceeds.

**What "converged" means and why the threshold is sensitive:**

Convergence is structural identity, not score proximity. The oracle is instructed to use a sensitive threshold: "similar framing", "same epistemic register", or "same type of claim" all count as CONVERGED. The harness does not wait for full structural collapse. The rationale: the damage from premature attractor convergence is gradual and invisible — by the time it is obvious (three threads scoring similarly from the same basin), multiple tournament cycles of compute have been wasted on redundant exploration. Early detection at the first sign of "similar framing" is cheap (one opus call per tournament) and prevents the failure mode from propagating.

Two threads can score identically from genuinely different peaks — this is fine and the oracle is explicitly instructed not to use score as evidence. Two threads can have a large score gap but operate in the same epistemic register — this is convergence and re-diversification fires.

**Re-diversification mechanism:**

The flagged thread's current-best is retired to history (not discarded — its entire exploratory lineage remains available for future crossover decomposition). The thread receives a new 3-axis seed verified orthogonal against all active thread pool axes across all three threads. The orthogonality check prompt explicitly foregrounds maximal distance: the goal is not just "different from existing axes" but "would lead to a fundamentally different type of answer." The thread reinitialises from the global best candidate (optimal starting position) but is structurally forced into a different region from iteration 1 by its new axis pool.

Divergence is guaranteed by the axis pool mechanism, not by the starting point. Starting from the global best is the correct initialisation — it gives the re-diversified thread the best available foundation to build from, while the orthogonal axis pool ensures its exploration direction is genuinely distinct from the other threads.

**Re-diversification fires before the standard tournament cycle (T1–T5):**

This ordering is deliberate. Re-diversification is a correction step, not a selection step. The standard tournament (retire lowest, spawn new) is selection pressure: it redirects compute from poor regions to promising ones. The divergence check is diversity maintenance: it ensures that selection pressure operates on genuinely distinct candidates. Running SD before T1–T5 means that when T3 retires the lowest-scoring thread, the threads entering the retirement decision are already structurally distinct — the harness is not simultaneously trying to enforce diversity and apply selection pressure in the same step.

**Why every tournament, not just when stuck:**

A harness that only checks for convergence when the loop "appears stuck" (scores plateauing, escape rates rising) detects convergence too late. The attractor pull from global best seeding can operate while threads are still progressing — threads can converge to the same basin and keep improving their score within it. Score-based stuck detection would never fire in this scenario. Structural divergence checks at every tournament event catch this failure mode regardless of whether score-based indicators are present.

**Cost of the change:** ~$0.08–0.12 per tournament event (one opus call for three pairwise verdicts). Additional $0.05–0.15 if re-diversification fires (axis orthogonality search). Total overhead: ~$0.30–0.60 for a standard 2-tournament run. Negligible relative to the $90–138 base cost.

---

### What this does NOT change:
- Escape trigger mechanics (10 → 5 → 3 consecutive discards — unchanged from v2–v8, per-thread)
- Antipode escape mechanism (Steps 1–4 — unchanged from v2–v8)
- Adaptive axis discovery (Step 5 — unchanged from v4–v8)
- Axis orthogonality verification (Step 5b-bis — unchanged from v4–v8)
- Probe seed store for axes (monotonically growing per-thread — unchanged from v4–v8)
- Basin clustering trigger (3+ consecutive events within 2% — unchanged from v6–v8)
- Historical best inventory and decomposition (Step 6b — unchanged from v6–v8, enriched by global history)
- Hybrid generation constraints (60% cap, coherence filter — Step 6c unchanged)
- Structural novelty verification for crossover hybrids (Step 6c-bis — unchanged from v7/v8)
- Maximum-distance fallback for blended hybrids (Step 6c-bis — unchanged from v7/v8)
- Mini-sprint structure (5 iterations — unchanged from v2–v8)
- Regression gate threshold (0.80 — unchanged from v2–v8)
- Crossover winner selection and basin window reset (Step 6d — unchanged from v6–v8)
- No-escalation-beyond-crossover rule (unchanged from v6–v8)
- Progressive counter shortening on gate failure (unchanged from v2–v8)
- Axis retirement logic (3+ consecutive fails — unchanged from v3–v8)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- Population structure (3 threads — unchanged from v8)
- Thread initialisation and axis seeding (unchanged from v8)
- Independent operation between tournaments (unchanged from v8)
- Tournament interval (every 15 iterations — unchanged from v8)
- T1–T5 tournament procedure (unchanged from v8)
- Global state structures (tournament ledger, global best history — unchanged from v8)
- Termination condition (unchanged from v1–v8)
- evaluate.md (frozen throughout, shared oracle — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
