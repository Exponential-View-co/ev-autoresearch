# Harness Design — v8 (Lightweight Population Parallelism)
*Autoresearch escape loop — eighth iteration*
*260314*

---

## Escape trigger
Per-thread: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4/v6/v7 — now operating independently within each thread.)*

**Basin clustering detection (per-thread, unchanged from v6/v7):** After every escape event in which the regression gate passes, each thread independently checks whether it is circling a basin. The check: have the last 3+ consecutive escape events within that thread each produced a best seed that scores within 2% of that thread's current best score?

> **Basin condition:** For each of the most recent N consecutive gate-passing escape events within a thread (N ≥ 3), the highest-scoring seed from that event satisfies: `seed_score ≥ thread_current_best_score × 0.98`. If this condition holds across 3+ consecutive events, basin clustering is detected and the crossover step fires **before** the next main-loop iteration within that thread.

The 2% threshold is applied to the **best seed from each escape event**, not the average. Basin clustering detection fires independently for each thread. It can fire on events where the gate passed — the thread is making progress but circling the same peak. Both the adaptive axis trigger (2+ consecutive gate failures) and the basin clustering trigger can be simultaneously active within a thread.

---

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, Basin Clustering Crossover, Crossover Novelty Verification, and Lightweight Population Parallelism.**

All escape mechanisms from v7 (Steps 1–6d) are preserved and operate identically within each thread. The sole structural change is that three independent threads run this complete mechanism in parallel, each initialised with a distinct non-overlapping axis pool seed. Tournament selection governs knowledge-sharing between threads at fixed intervals. Everything below that is labelled *(Per-thread, unchanged from v7)* operates without modification inside each thread.

---

### Step 1 — Structural dissection with live axis pool *(Per-thread, unchanged from v7)*

A dedicated dissection agent reads the thread's current-best and extracts structural elements from that thread's **current axis pool**. Each thread's pool starts at length 3 and is mutable. Dissection always reads from the thread's own current pool.

---

### Step 2 — Antipode generation *(Per-thread, unchanged from v7)*

For each axis in the thread's current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record plus any newly added axes, capped at 5 seeds per escape event.

---

### Step 3 — Extended mini-evaluation sprint *(Per-thread, unchanged from v7)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Per-thread, unchanged from v7)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ thread_current_best_score × 0.80

- **Gate passes:** escape candidate replaces thread's current-best. Discard counter resets to 0. The winning axis logs a "pass" event. Basin clustering counter is updated.
- **Gate fails:** thread's current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used logs a "fail" event. Failed events are excluded from the basin clustering window.

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Per-thread, unchanged from v7)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events within the thread.

#### 5a — Evaluator feedback analysis *(Per-thread, unchanged from v7)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event within the thread. It identifies which dimensions scored lowest across all seeds in the failed escape.

#### 5b — LLM-driven axis generation *(Per-thread, unchanged from v7)*

The dissection agent generates **2 new structural axes** by prompting itself with the standard axis-generation prompt (pool list, evaluator language, orthogonality and generality requirements). No human review.

#### 5b-bis — Axis orthogonality verification *(Per-thread, unchanged from v7)*

For each proposed axis, the harness:

**i.** Generates a **probe seed** (2–3 sentence sketch of what a candidate built on this axis would look like).

**ii.** Compares the probe seed against all currently active pool axes in a single opus call. Proposed axis passes if orthogonal to all existing pool axes.

**iii.** Up to 2 regeneration attempts on failure. Axis is dropped after 2 failed retries.

**The probe seed store** is per-thread and monotonically growing. Retired axes' probe seeds are retained. The store is initialised at thread start with probe seeds for the three starting axes assigned to that thread.

#### 5c — Axis pool management (retirement) *(Per-thread, unchanged from v7)*

An axis is retired when it accumulates 3+ consecutive fail events with no intervening pass. Retirement is permanent within the thread. Retirement deferred if it would reduce the pool below 3 axes.

---

### Step 6 — Structural crossover on basin clustering detection *(Per-thread, unchanged from v7)*

**Trigger condition:** 3+ consecutive gate-passing escape events within the thread whose best seed scored within 2% of that thread's current best.

#### 6a — Basin window maintenance *(Per-thread, unchanged from v7)*

The thread maintains a rolling list of recent gate-passing escape events. Failed-gate events are excluded. The window resets when a crossover fires or when a non-basin escape event occurs (best seed more than 2% above thread's current best at time of event).

#### 6b — Structural decomposition of historical bests *(Per-thread, unchanged from v7)*

When crossover fires within a thread, the harness selects the **top-3 historically best candidates** from that thread's working state (by peak score). For each, a dissection agent produces a structural decomposition: 5–8 atomic structural elements with name, 1-sentence definition, and contribution description. The crossover agent aligns element inventories cross-candidate with an explicit cross-candidate element map.

#### 6c — Recombination and hybrid generation *(Per-thread, unchanged from v7)*

The crossover agent generates **3 hybrid candidates** satisfying:
- At least 2 elements from different source candidates per hybrid
- No more than 60% of elements from one source
- Internal coherence enforced (incoherent combinations discarded and regenerated)

Generated hybrids are held in working state pending novelty verification.

#### 6c-bis — Structural novelty verification for crossover hybrids *(Per-thread, unchanged from v7)*

After generating 3 hybrid candidates but before mini-sprints, each hybrid must pass structural novelty verification.

**i. Hybrid probe sketch generation:** A 2–3 sentence description of the hybrid's essential structural identity — governing logic, central tension, what makes it distinct.

**ii. Novelty judgement via opus:** Hybrid probe sketch compared against all 3 source candidate probe sketches. Verdict: NOVEL ("genuinely novel recombination — structural identity distinct from each source") or BLEND ("blended average — a weighted midpoint that explores no new territory").

Prompt: *"You are a structural novelty judge. You will receive a probe sketch of a crossover hybrid and probe sketches of its 3 source candidates. Determine whether the hybrid is a genuinely novel recombination or a blended average. A blended average takes safe, compatible elements from each source without producing structural identity none of the sources individually contains. A genuinely novel recombination produces emergent character — its defining logic, tension, or stance cannot be derived by averaging or interpolating the sources. Consider: does the hybrid have a structural identity that would surprise the authors of the source candidates? Or does it look like a committee-produced compromise? Return exactly one verdict: NOVEL or BLEND. Follow with a 1-sentence rationale."*

**iii. Resolution:**

- **NOVEL:** Hybrid cleared for mini-sprints.
- **BLEND → regenerate (up to 1 retry):** Augmented prompt foregrounds surprise: *"maximise structural distance from all three source candidates... choose surprising-and-coherent over coherent-and-average."* Regenerated hybrid re-judged by opus.
- **Still BLEND after retry → maximum-distance fallback:** Harness constructs replacement hybrid by taking the single most structurally distinctive element from each of the 3 source candidates (least semantic overlap with the other inventories) and combining exactly those 3 elements, discarding all bridging or compromise elements. Not re-judged — definitionally constructed for maximal structural distance.

Hybrid probe sketches (initial, regenerated, or fallback) stored per-thread alongside working state.

#### 6d — Crossover mini-sprint and regression-gated winner selection *(Per-thread, unchanged from v7)*

Each novelty-verified hybrid runs a **5-iteration greedy sub-run** against the same frozen evaluate.md. All verified hybrids run in parallel (within the thread).

Highest-scoring hybrid must clear the regression gate:
> `crossover_candidate_score ≥ thread_current_best_score × 0.80`

- **Gate passes:** Replaces thread's current-best. Discard counter resets to 0. Basin window resets to zero.
- **Gate fails:** Thread's current-best retained. Basin window resets to zero. Discard counter unchanged.

**No escalation beyond crossover:** If all crossover candidates fail the gate, the thread resumes with its current best. Basin window resets; next 3 escape events start a fresh clustering check.

---

## Population structure *(NEW in v8 — the single change)*

**Three independent threads with tournament selection every 15 iterations.**

### Thread initialisation

The harness spawns **3 threads** at run start. Each thread is a complete, self-contained instance of the v7 harness — its own current-best candidate, its own axis pool, its own probe seed store, its own historical best inventory, its own escape trigger counter, its own basin window, its own discard counter.

**Differentiated axis pool seeds:** The three threads are seeded with non-overlapping axis pools drawn from the orthogonal verification mechanism (Step 5b-bis). At harness start, before any thread executes an iteration, the harness generates probe seeds for candidate seed axes and verifies mutual orthogonality across all 9 starting axes (3 per thread × 3 threads) in a single opus call. The 9 axes are accepted only if all 9 are mutually orthogonal — otherwise the harness regenerates the set (up to 3 retries).

**Default starting axis pools (example assignment — actual pools verified by orthogonality check at run time):**

- **Thread 1 seed:** Frame (central metaphor / conceptual lens), Scope (level of analysis), Temporal frame (where in time the argument is situated)
- **Thread 2 seed:** Causal claim (primary mechanism asserted), Evidence base (what class of evidence anchors the argument), Audience stance (the implied reader's prior position the argument assumes)
- **Thread 3 seed:** Epistemic register (the type of claim being made — empirical/normative/predictive), Resolution mode (how the argument proposes to settle disagreement), Generative tension (the central contradiction the argument holds in productive suspension)

These are starting points: each thread's pool evolves independently via adaptive axis discovery (Step 5) from the first iteration. The three pools diverge continuously and are never synchronised.

**Thread labelling:** Threads are labelled T1, T2, T3 and carry their label in working state. Labels are used for logging and tournament bookkeeping only.

---

### Independent operation (between tournaments)

Between tournament events, threads are fully independent. They do not share state, do not share candidates, do not share axis pools, do not share escape counters, and do not share basin windows. There is no inter-thread communication of any kind between tournaments.

Each thread:
- Runs its own main hill-climbing loop (greedy: accept improvement, discard non-improvement)
- Runs its own escape trigger (10 consecutive discards → escape → adaptive axis step on 2+ consecutive gate failures)
- Runs its own basin clustering check (3+ consecutive gate-passing events within 2% → crossover)
- Evaluates all candidates against the same frozen evaluate.md (oracle is shared and frozen — comparisons are valid)
- Maintains its own working state (current-best, axis pool, probe seed store, historical best inventory, basin window, discard counter)

Threads run **concurrently** — the harness does not wait for one thread to complete an iteration before advancing another. Each thread advances at its own pace. Threads that trigger escapes or crossover events do so independently of other threads' states.

---

### Tournament selection

**Trigger:** Every 15 main-loop iterations completed across the total run (sum of all iterations across all active threads), OR when any thread reaches a convergence state (defined as: escape trigger has fired and the regression gate has failed on 3 consecutive escape events with no improvement — the thread has exhausted its axis pool's current escape capacity).

**Why 15 iterations:** The tournament interval is set to enable genuine cross-pollination. At 15 iterations, threads have diverged far enough from their starting positions to carry meaningful signal about different regions, but not so far that a low-performing thread has wasted capacity that could be redirected. The interval is short enough that a thread discovering a promising region promotes the global best before other threads have committed deeply to inferior regions.

**Tournament procedure (5 steps):**

**T1 — Current-best readout.** The harness reads each thread's current-best candidate and its score. If any thread is mid-escape or mid-crossover when the tournament trigger fires, the tournament waits for that thread to complete its current escape/crossover event (including all mini-sprints and novelty verification steps) before proceeding. This ensures all threads enter the tournament with a stable current-best, not a transient in-progress candidate.

**T2 — Global best promotion.** The thread holding the highest-scoring current-best is identified. Its current-best is designated the **global best** for this tournament. The global best is logged to the run's tournament ledger (score, source thread label, iteration count). If two threads are tied for highest score, the tie is broken by iteration count (the thread that reached this score in fewer iterations wins — it is the more efficient explorer). The global best is not immediately forced onto other threads — it becomes available as the source for thread re-seeding in T4.

**T3 — Lowest-scoring thread retirement.** The thread with the lowest current-best score is retired. Its working state (axis pool, probe seed store, historical best inventory, basin window, discard counter) is discarded. The retired thread's probe seed store is **not** transferred to other threads — each thread maintains its own coverage map. The only information that transfers at tournament time is the global best candidate (Step T2) and the re-seeding axis (Step T4).

Exception: If the lowest-scoring thread is currently executing an escape or crossover event when the tournament fires, retirement is deferred until that event completes (same waiting logic as T1). A thread mid-escape may produce a strong candidate that reverses its ranking — early retirement before completion would be wasteful.

**T4 — New thread spawn from global best.** A new thread (labelled with the retired thread's label, e.g., T3 is retired → new T3 is spawned) is initialised from the global best candidate. The new thread:

- Sets its current-best to the global best candidate
- Receives a **fresh non-overlapping axis seed** — 3 axes verified orthogonal to the current active threads' starting axes (using the run-start orthogonality check mechanism: probe seed generation + opus orthogonality verification against all 9 currently active thread pool axes). Up to 3 retries to find a non-overlapping triple.
- Initialises its probe seed store with probe seeds for its 3 starting axes only (no inherited probe seeds from the retired thread or global best thread)
- Initialises with fresh discard counter (reset to 0), fresh basin window (empty), fresh escape trigger (standard 10-discard threshold)
- Begins searching from the global best candidate's position but immediately diverges via its distinct axis pool

The rationale: the new thread gets the best known starting position (global best) and a fresh directional lens (new non-overlapping axes). It exploits what the best thread has found and explores in a genuinely different direction from iteration 1.

**T5 — Tournament ledger update.** The harness logs: iteration count at tournament, all three thread scores, global best score and source thread, retired thread label and score, new thread spawn confirmation, new thread starting axis labels.

---

### Oracle consistency

All three threads evaluate candidates against the same fixed evaluate.md throughout the run. The oracle is frozen — it does not change between threads or between tournament events. Scores are directly comparable across threads. Tournament winner selection is valid because all scores are measured against the same evaluator.

---

### Global state structures

The run maintains two global state structures in addition to each thread's private state:

**Tournament ledger:** A log of all tournament events — timestamp (iteration count), thread scores at entry, global best promoted, thread retired, thread spawned, new thread axis labels. Read-only by threads (threads do not use the ledger for search decisions). Written only by the tournament procedure.

**Global best history:** A monotonically growing list of all candidates ever promoted to global best at any tournament, with scores and source thread labels. Available for future crossover events within any thread — the crossover Step 6b draws from "historically best candidates in the run," which now means historically best candidates globally, not just within the thread's own lineage. This cross-thread historical inventory is the second mechanism by which threads share information — the first is the global best promotion at tournament time.

Note: Threads contribute to the global best history only at tournament time, not continuously. Between tournaments, each thread's historical best inventory is private.

---

### Termination condition (unchanged from v1/v2/v3/v4/v6/v7)

The run terminates when the global iteration count reaches the configured limit (e.g., 30 iterations). Partial iterations in progress are completed before termination. The final output is the global best candidate from the last tournament ledger entry (or the highest-scoring current-best across all active threads if no tournament has fired since the last iteration).

---

## Perturbation direction logic

Direction during standard escape is determined by antipode derivation from the thread's current axis pool (mutable — per-thread). Direction during crossover is determined by structural element recombination across historical bests (now includes globally promoted candidates from the tournament ledger), with structural novelty verification before sprint. Main-loop perturbation logic (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1/v2/v3/v4/v6/v7 — operates per-thread.

**Cross-thread perturbation enrichment at tournament time:** When a new thread is spawned from the global best (Step T4), the global best's structural decomposition (if it exists from a prior crossover event) is available in working state. The new thread's first escape event, if it triggers crossover, can draw on this decomposition as one of the historical bests — giving it richer material for recombination from iteration 1. This is the only way decomposition information crosses thread boundaries.

The axis pool (per-thread), probe seed store (per-thread), historical best inventory (per-thread, enriched by global history at tournament time), and hybrid probe sketch store (per-thread) are the four per-thread state structures. The tournament ledger and global best history are the two global state structures.

## Re-integration

Within each thread: escape candidate replaces thread's current-best only if it clears the regression gate (≥ 80% of thread's current-best score). Unchanged from v2/v3/v4/v6/v7. Adaptive axis step fires before loop resumes on gate failure. Crossover fires before next main-loop iteration when basin clustering is detected. Crossover hybrids must pass structural novelty verification (Step 6c-bis) before mini-sprints.

**Cross-thread re-integration:** Global best from any thread becomes the starting candidate for a newly spawned thread at tournament time. The global best does not replace the current-bests of surviving threads — it is available as a seed for the new thread only. Surviving threads continue from their own current-bests.

## Estimated cost per run

~$90–138 for a standard 30-iteration run with 3 threads, 1–2 escape events per thread, and 1 crossover event per thread. Breakdown:

- Per-thread cost: ~$30–46 (unchanged from v7 estimate per single thread)
- 3 threads × $30–46 = $90–138 baseline
- Tournament orthogonality verification at run start: ~$0.10–0.15 (1 opus call for 9-axis mutual orthogonality check)
- Per-tournament event: ~$0.05–0.10 (tournament ledger update, re-seeding axis verification: 1 opus call for new thread's 3-axis orthogonality check against 6 active axes)
- Estimated 2 tournament events per 30-iteration run: ~$0.10–0.20 additional
- Total: ~$90–138 with tournament overhead negligible

Well under $250. The 3× cost increase vs v7 is the direct price of 3× concurrent exploration capacity.

---

## What changed this iteration (v7 → v8)

**Element changed:** Population structure — replaced single-thread sequential exploration with a 3-thread parallel population. One change only. All escape mechanisms (Steps 1–6d) preserved identically within each thread.

---

### The change — Lightweight population parallelism

**v7 state:** Single thread. One current-best candidate at all times. All escape and crossover mechanisms operate sequentially within this single thread. To explore a different region of the solution space, the thread must first exhaust its current region (triggering escape), navigate to a new seed (via antipode generation or crossover), and then explore that region. The landscape is explored one region at a time. Every mechanism added since v1 improves the *quality* of what the single thread does when it escapes, but the architecture still imposes a hard coverage ceiling: you can only be in one region at once.

All three v7 evaluators named this explicitly. EV Platform Strategist: "the single-thread architecture still limits how many genuinely different regions are explored simultaneously; coverage improvement is defensive (blocking bad hybrids) not offensive (generating more diverse candidates upfront)." VC Stress-Tester: "the fundamental coverage strategy is unchanged from v6; the improvement blocks a failure mode rather than opening new search territory." Karpathy: "Introduce lightweight population parallelism."

**v8 change:** Three independent threads with tournament selection every 15 iterations.

**Thread initialisation:** Each thread starts with a different 3-axis seed, verified mutually orthogonal across all 9 starting axes before any iteration begins. The three threads diverge from iteration 1 because their axis pools are different — they explore genuinely different structural neighbourhoods from the start, not just after their first escape event.

**Independent operation:** Between tournaments, threads share nothing. Each thread runs the full v7 mechanism independently: its own hill-climbing loop, its own adaptive axis pool, its own escape trigger, its own crossover mechanism. They cannot converge prematurely because there is no shared state to converge on.

**Tournament selection at 15-iteration intervals:** Every 15 iterations (or on any thread's convergence), the harness:
1. Reads all thread scores
2. Promotes the global best to the tournament ledger
3. Retires the lowest-scoring thread
4. Spawns a new thread from the global best with a fresh non-overlapping axis seed

The tournament interval is short by design. At 15 iterations, threads have diverged enough to carry real signal but not so far that a low-performing thread has wasted capacity that could be redirected. The 15-iteration interval enables cross-pollination without premature convergence — the surviving high-scoring threads are not touched, they continue from their own current-bests.

**Why three threads (not two or four):** Three is the minimum odd number for clean tournament selection without tie-breaking complexity on the retirement decision. With three threads there is always a clear lowest-scoring candidate for retirement (ties broken by iteration count — the less efficient explorer is retired). Four threads adds cost without structural benefit at this stage.

**Cross-thread information sharing:** The only information that crosses thread boundaries is (a) the global best candidate at tournament time, used to seed the new thread, and (b) the global best history, available as a source of historical candidates for crossover decomposition. All other state (axis pools, probe seed stores, basin windows, escape counters) remains private to each thread.

**Why this is the right next change:** All previous iterations improved the quality of the single thread's escape events. The coverage ceiling was structural, not mechanical — no amount of improvement to escape quality can overcome the fact that sequential exploration visits regions one at a time. Population parallelism is the only intervention that directly addresses the coverage ceiling. The cost is a 3× increase in token usage ($90–138 vs $30–46) — this is the price of 3× concurrent exploration capacity and is well under the $250 threshold.

---

### What this does NOT change:
- Escape trigger mechanics (10 → 5 → 3 consecutive discards — unchanged from v2/v3/v4/v6/v7, now per-thread)
- Antipode escape mechanism (Steps 1–4 — unchanged from v2/v3/v4/v6/v7)
- Adaptive axis discovery (Step 5 — unchanged from v4/v6/v7)
- Axis orthogonality verification (Step 5b-bis — unchanged from v4/v6/v7)
- Probe seed store for axes (monotonically growing per-thread — unchanged from v4/v6/v7)
- Basin clustering trigger (3+ consecutive events within 2% — unchanged from v6/v7)
- Historical best inventory and decomposition (Step 6b — unchanged from v6/v7, now enriched by global history)
- Hybrid generation constraints (60% cap, coherence filter — Step 6c unchanged)
- Structural novelty verification for crossover hybrids (Step 6c-bis — unchanged from v7)
- Maximum-distance fallback for blended hybrids (Step 6c-bis — unchanged from v7)
- Mini-sprint structure (5 iterations — unchanged from v2/v3/v4/v6/v7)
- Regression gate threshold (0.80 — unchanged from v2/v3/v4/v6/v7)
- Crossover winner selection and basin window reset (Step 6d — unchanged from v6/v7)
- No-escalation-beyond-crossover rule (unchanged from v6/v7)
- Progressive counter shortening on gate failure (unchanged from v2/v3/v4/v6/v7)
- Axis retirement logic (3+ consecutive fails — unchanged from v3/v4/v6/v7)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- evaluate.md (frozen throughout, shared oracle — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
