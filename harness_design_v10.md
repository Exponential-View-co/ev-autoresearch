# Harness Design — v10 (Inter-Tournament Convergence Monitoring)
*Autoresearch escape loop — tenth iteration*
*260314*

---

## Escape trigger
Per-thread: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4/v6/v7/v8/v9 — operating independently within each thread.)*

**Basin clustering detection (per-thread, unchanged from v6/v7/v8/v9):** After every escape event in which the regression gate passes, each thread independently checks whether it is circling a basin. The check: have the last 3+ consecutive escape events within that thread each produced a best seed that scores within 2% of that thread's current best score?

> **Basin condition:** For each of the most recent N consecutive gate-passing escape events within a thread (N ≥ 3), the highest-scoring seed from that event satisfies: `seed_score ≥ thread_current_best_score × 0.98`. If this condition holds across 3+ consecutive events, basin clustering is detected and the crossover step fires **before** the next main-loop iteration within that thread.

The 2% threshold is applied to the **best seed from each escape event**, not the average. Basin clustering detection fires independently for each thread. Both the adaptive axis trigger (2+ consecutive gate failures) and the basin clustering trigger can be simultaneously active within a thread.

---

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, Basin Clustering Crossover, Crossover Novelty Verification, Lightweight Population Parallelism, Tournament-Time Structural Divergence Check, and Inter-Tournament Convergence Monitoring.**

All escape mechanisms from v9 (Steps 1–6d and Step SD) are preserved and operate identically within each thread and at each tournament boundary. The sole structural change is the addition of **inter-tournament convergence monitoring** (Step IC), a lightweight structural fingerprint check that runs every 5 main-loop iterations between tournament events. Step IC can trigger an early execution of Step SD when it detects potential convergence — it does not replace the scheduled 15-iteration tournament.

---

### Step 1 — Structural dissection with live axis pool *(Per-thread, unchanged from v7/v8/v9)*

A dedicated dissection agent reads the thread's current-best and extracts structural elements from that thread's **current axis pool**. Each thread's pool starts at length 3 and is mutable. Dissection always reads from the thread's own current pool.

---

### Step 2 — Antipode generation *(Per-thread, unchanged from v7/v8/v9)*

For each axis in the thread's current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record plus any newly added axes, capped at 5 seeds per escape event.

---

### Step 3 — Extended mini-evaluation sprint *(Per-thread, unchanged from v7/v8/v9)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Per-thread, unchanged from v7/v8/v9)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ thread_current_best_score × 0.80

- **Gate passes:** escape candidate replaces thread's current-best. Discard counter resets to 0. The winning axis logs a "pass" event. Basin clustering counter is updated.
- **Gate fails:** thread's current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used logs a "fail" event. Failed events are excluded from the basin clustering window.

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Per-thread, unchanged from v7/v8/v9)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events within the thread.

#### 5a — Evaluator feedback analysis *(Per-thread, unchanged from v7/v8/v9)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event within the thread. It identifies which dimensions scored lowest across all seeds in the failed escape.

#### 5b — LLM-driven axis generation *(Per-thread, unchanged from v7/v8/v9)*

The dissection agent generates **2 new structural axes** by prompting itself with the standard axis-generation prompt (pool list, evaluator language, orthogonality and generality requirements). No human review.

#### 5b-bis — Axis orthogonality verification *(Per-thread, unchanged from v7/v8/v9)*

For each proposed axis, the harness:

**i.** Generates a **probe seed** (2–3 sentence sketch of what a candidate built on this axis would look like).

**ii.** Compares the probe seed against all currently active pool axes in a single opus call. Proposed axis passes if orthogonal to all existing pool axes.

**iii.** Up to 2 regeneration attempts on failure. Axis is dropped after 2 failed retries.

**The probe seed store** is per-thread and monotonically growing. Retired axes' probe seeds are retained. The store is initialised at thread start with probe seeds for the three starting axes assigned to that thread.

#### 5c — Axis pool management (retirement) *(Per-thread, unchanged from v7/v8/v9)*

An axis is retired when it accumulates 3+ consecutive fail events with no intervening pass. Retirement is permanent within the thread. Retirement deferred if it would reduce the pool below 3 axes.

---

### Step 6 — Structural crossover on basin clustering detection *(Per-thread, unchanged from v7/v8/v9)*

**Trigger condition:** 3+ consecutive gate-passing escape events within the thread whose best seed scored within 2% of that thread's current best.

#### 6a — Basin window maintenance *(Per-thread, unchanged from v7/v8/v9)*

The thread maintains a rolling list of recent gate-passing escape events. Failed-gate events are excluded. The window resets when a crossover fires or when a non-basin escape event occurs (best seed more than 2% above thread's current best at time of event).

#### 6b — Structural decomposition of historical bests *(Per-thread, unchanged from v7/v8/v9)*

When crossover fires within a thread, the harness selects the **top-3 historically best candidates** from that thread's working state (by peak score). For each, a dissection agent produces a structural decomposition: 5–8 atomic structural elements with name, 1-sentence definition, and contribution description. The crossover agent aligns element inventories cross-candidate with an explicit cross-candidate element map.

#### 6c — Recombination and hybrid generation *(Per-thread, unchanged from v7/v8/v9)*

The crossover agent generates **3 hybrid candidates** satisfying:
- At least 2 elements from different source candidates per hybrid
- No more than 60% of elements from one source
- Internal coherence enforced (incoherent combinations discarded and regenerated)

Generated hybrids are held in working state pending novelty verification.

#### 6c-bis — Structural novelty verification for crossover hybrids *(Per-thread, unchanged from v7/v8/v9)*

After generating 3 hybrid candidates but before mini-sprints, each hybrid must pass structural novelty verification.

**i. Hybrid probe sketch generation:** A 2–3 sentence description of the hybrid's essential structural identity — governing logic, central tension, what makes it distinct.

**ii. Novelty judgement via opus:** Hybrid probe sketch compared against all 3 source candidate probe sketches. Verdict: NOVEL ("genuinely novel recombination — structural identity distinct from each source") or BLEND ("blended average — a weighted midpoint that explores no new territory").

Prompt: *"You are a structural novelty judge. You will receive a probe sketch of a crossover hybrid and probe sketches of its 3 source candidates. Determine whether the hybrid is a genuinely novel recombination or a blended average. A blended average takes safe, compatible elements from each source without producing structural identity none of the sources individually contains. A genuinely novel recombination produces emergent character — its defining logic, tension, or stance cannot be derived by averaging or interpolating the sources. Consider: does the hybrid have a structural identity that would surprise the authors of the source candidates? Or does it look like a committee-produced compromise? Return exactly one verdict: NOVEL or BLEND. Follow with a 1-sentence rationale."*

**iii. Resolution:**

- **NOVEL:** Hybrid cleared for mini-sprints.
- **BLEND → regenerate (up to 1 retry):** Augmented prompt foregrounds surprise: *"maximise structural distance from all three source candidates... choose surprising-and-coherent over coherent-and-average."* Regenerated hybrid re-judged by opus.
- **Still BLEND after retry → maximum-distance fallback:** Harness constructs replacement hybrid by taking the single most structurally distinctive element from each of the 3 source candidates (least semantic overlap with the other inventories) and combining exactly those 3 elements, discarding all bridging or compromise elements. Not re-judged — definitionally constructed for maximal structural distance.

Hybrid probe sketches (initial, regenerated, or fallback) stored per-thread alongside working state.

#### 6d — Crossover mini-sprint and regression-gated winner selection *(Per-thread, unchanged from v7/v8/v9)*

Each novelty-verified hybrid runs a **5-iteration greedy sub-run** against the same frozen evaluate.md. All verified hybrids run in parallel (within the thread).

Highest-scoring hybrid must clear the regression gate:
> `crossover_candidate_score ≥ thread_current_best_score × 0.80`

- **Gate passes:** Replaces thread's current-best. Discard counter resets to 0. Basin window resets to zero.
- **Gate fails:** Thread's current-best retained. Basin window resets to zero. Discard counter unchanged.

**No escalation beyond crossover:** If all crossover candidates fail the gate, the thread resumes with its current best. Basin window resets; next 3 escape events start a fresh clustering check.

---

## Population structure *(Unchanged from v8/v9 — three independent threads with tournament selection every 15 iterations, plus inter-tournament fingerprint check every 5 iterations)*

### Thread initialisation

The harness spawns **3 threads** at run start. Each thread is a complete, self-contained instance of the v9 harness — its own current-best candidate, its own axis pool, its own probe seed store, its own historical best inventory, its own escape trigger counter, its own basin window, its own discard counter, **and its own structural fingerprint log (new in v10).**

**Differentiated axis pool seeds:** The three threads are seeded with non-overlapping axis pools. At harness start, the harness generates probe seeds for candidate seed axes and verifies mutual orthogonality across all 9 starting axes (3 per thread × 3 threads) in a single opus call. The 9 axes are accepted only if all 9 are mutually orthogonal — otherwise the harness regenerates the set (up to 3 retries).

**Default starting axis pools (example assignment — actual pools verified by orthogonality check at run time):**

- **Thread 1 seed:** Frame (central metaphor / conceptual lens), Scope (level of analysis), Temporal frame (where in time the argument is situated)
- **Thread 2 seed:** Causal claim (primary mechanism asserted), Evidence base (what class of evidence anchors the argument), Audience stance (the implied reader's prior position the argument assumes)
- **Thread 3 seed:** Epistemic register (the type of claim being made — empirical/normative/predictive), Resolution mode (how the argument proposes to settle disagreement), Generative tension (the central contradiction the argument holds in productive suspension)

**Thread labelling:** Threads are labelled T1, T2, T3 and carry their label in working state. Labels are used for logging and tournament bookkeeping only.

---

### Independent operation (between tournaments)

Between tournament events, threads are fully independent. They do not share state, do not share candidates, do not share axis pools, do not share escape counters, and do not share basin windows. Each thread runs the full v9 mechanism independently. Threads run **concurrently** — the harness does not wait for one thread to complete an iteration before advancing another.

**NEW in v10:** After completing each main-loop iteration, each thread appends a structural fingerprint to its fingerprint log (see Step IC below). Every 5 iterations, the harness reads all three threads' most recent fingerprints and compares them. This inter-tournament monitoring runs between scheduled tournament events and can trigger an early Step SD check.

---

### Step IC — Inter-tournament convergence monitoring *(NEW in v10)*

**Purpose:** Shrink the maximum convergence detection window from 14 iterations (current gap between the previous tournament and the next scheduled one) to 4 iterations (maximum gap between fingerprint checks). Step IC is a trip-wire, not a verdict — it is intentionally cheap and slightly imprecise. False positives trigger an early Step SD check (cheap). False negatives are caught at the next fingerprint check or scheduled tournament. The key property: convergence waste is bounded at 4 iterations, not 14.

---

#### IC1 — Structural fingerprint generation (per-thread, per iteration)

After completing each main-loop iteration and updating its current-best, each thread's iteration agent generates a **structural signature** — a compact 3-element tuple describing the current-best's essential structural character:

```
[primary_frame, causal_direction, resolution_register]
```

**Element definitions:**

- **primary_frame** — The central metaphor, conceptual lens, or governing frame of the current-best. Examples: "scarcity logic", "network effects", "evolutionary pressure", "epistemic humility", "market failure", "systems intervention". One short phrase (2–5 words).

- **causal_direction** — The direction of the argument's primary causal claim. One of five fixed labels:
  - `bottom-up` — micro-level mechanisms produce macro outcomes
  - `top-down` — structural forces constrain individual behaviour
  - `bidirectional` — mutual reinforcement between levels
  - `lateral` — peer effects, diffusion, contagion
  - `emergent` — outcome arises from complexity, not reducible to any direction

- **resolution_register** — How the argument resolves or settles its central tension. One of five fixed labels:
  - `prescriptive` — identifies an action that should be taken
  - `diagnostic` — identifies a cause or mechanism without prescribing action
  - `predictive` — forecasts a trajectory or outcome
  - `reframing` — dissolves the tension by changing how the problem is understood
  - `irresolvable` — holds the tension as productive, refuses resolution

**Generation instruction to the iteration agent:**

> *"After updating your current-best this iteration, generate its structural signature as a 3-element tuple: [primary_frame, causal_direction, resolution_register]. primary_frame: 2–5 word phrase naming the governing metaphor or conceptual lens. causal_direction: exactly one of [bottom-up, top-down, bidirectional, lateral, emergent]. resolution_register: exactly one of [prescriptive, diagnostic, predictive, reframing, irresolvable]. Be approximate — this is a quick characterisation, not a precise structural analysis. Output format: SIGNATURE: [frame], [direction], [register]"*

**Cost:** ~50 tokens added to the standard iteration agent call — appended as a brief post-iteration step. No separate model call required; generated as a by-product of the iteration agent's normal output.

The signature is appended to the thread's **fingerprint log** — a simple per-thread list of (iteration_number, signature) entries. The log is maintained in working state alongside the thread's other state structures.

---

#### IC2 — Fingerprint comparison (every 5 iterations)

Every 5 main-loop iterations (at iteration counts 5, 10, 20, 25, etc. — excluding iteration counts that coincide with a scheduled tournament event at iteration 15, 30, etc.), the harness reads the most recent fingerprint from each of the three threads' logs and performs a pairwise comparison.

**Comparison rule:** For each pair (T1 vs T2, T1 vs T3, T2 vs T3), count how many of the 3 signature elements match:
- **primary_frame match:** exact string equality after lowercasing and stripping whitespace
- **causal_direction match:** exact label equality
- **resolution_register match:** exact label equality

If any pair shares **2 or more of 3 elements**, flag that pair as a **fingerprint convergence warning**.

**Matching is intentionally strict for causal_direction and resolution_register** (fixed vocabularies — exact matches only) and **intentionally loose for primary_frame** (exact string match — if two threads independently chose the same frame phrase, that is a strong signal). The result is a check that is cheap (arithmetic comparison, no LLM call), slightly imprecise (primary_frame vocabulary is not controlled, so two genuinely different frames that happen to share a phrase will produce a false positive), and asymmetrically biased toward false positives (triggering a cheap early SD check) rather than false negatives (missing genuine convergence).

**IC2 fires concurrently** — it reads from all three fingerprint logs at the check interval without interrupting any in-progress iteration, escape, or crossover event. The harness waits for any in-progress iteration to append its signature before running the comparison.

**Fingerprint comparison requires no LLM call.** It is a simple string-matching operation on the most recent log entries. Cost: negligible.

---

#### IC3 — Early tournament trigger on convergence warning

If any pairwise fingerprint comparison returns a convergence warning (2/3 elements match):

**i. Early Step SD fires immediately.** The harness does not wait for the next scheduled 15-iteration tournament boundary. It runs the full Step SD structural anatomy check (pairwise opus judgement of all three thread current-bests) immediately — using the same SD1–SD5 procedure defined in v9, unchanged.

**ii. If Step SD confirms convergence (CONVERGED verdict for any pair):** Re-diversification fires per the v9 SD3 mechanism. The SD ledger is updated with a note that this SD check was triggered by an inter-tournament fingerprint warning at iteration [N].

**iii. If Step SD returns DISTINCT for all pairs (false positive from fingerprint check):** No re-diversification. The SD ledger logs the false positive: iteration count, which pair triggered the fingerprint warning, the mismatching signature elements, and the DISTINCT verdict from the full check. The harness resumes normal operation. The fingerprint warning is cleared.

**Cost of an early SD check:** ~$0.08–0.12 (same single opus call as a tournament-triggered SD check). False positives are therefore cheap — they trigger a $0.10 confirmation step that returns DISTINCT and resumes. No wasted compute on unnecessary re-diversification.

**iv. The fingerprint check does NOT suspend thread operation.** Threads continue running their main-loop iterations concurrently with the Step SD execution. If Step SD confirms convergence and re-diversification is needed, the flagged thread pauses at the next safe checkpoint (end of current iteration, before the next escape or crossover event) to receive its new axis seed. This mirrors the same pause logic used at scheduled tournament time.

---

#### IC4 — Fingerprint check timing relative to scheduled tournaments

**The scheduled 15-iteration tournament is not replaced or modified.** Step IC is additive — it provides additional monitoring between tournaments. The two mechanisms interact as follows:

- At iteration 5: IC2 fingerprint check fires (if between tournament boundaries)
- At iteration 10: IC2 fingerprint check fires
- At iteration 15: Scheduled tournament fires (Step SD + T1–T5, per v9). No fingerprint check at this point — the full tournament subsumes it.
- At iteration 20: IC2 fingerprint check fires
- At iteration 25: IC2 fingerprint check fires
- At iteration 30: Scheduled tournament fires.

**If an early tournament was triggered by IC3** (e.g., at iteration 8 due to a fingerprint warning), the next scheduled tournament fires at iteration 8 + 15 = 23 (the tournament clock resets from the most recent tournament, whether scheduled or early). Fingerprint checks continue at 5-iteration intervals relative to the global iteration count — they are not reset by early tournaments.

**If a fingerprint check fires within 2 iterations of a scheduled tournament** (e.g., iteration 13 with a tournament at 15), the harness suppresses the IC2 check — the tournament SD check will run within 2 iterations regardless, making an early trigger redundant. Suppression threshold: ≤ 2 iterations before next scheduled tournament.

---

#### IC5 — Fingerprint log maintenance

The fingerprint log for each thread is a simple append-only list of (iteration_number, signature_tuple) pairs. It is part of the thread's working state and persists through tournament events. When a thread is retired (T3) or re-diversified (SD3), its fingerprint log is archived to the global state (tournament ledger appendix) for diagnostic purposes. New and re-diversified threads start with an empty fingerprint log.

The IC2 comparison always uses the **most recent signature** from each thread's log — not a historical average or rolling window. The comparison is a point-in-time snapshot of current structural character.

---

### Tournament selection *(Unchanged from v9 — Step SD + T1–T5)*

**Trigger:** Every 15 main-loop iterations completed across the total run (sum of all iterations across all active threads), OR when any thread reaches a convergence state (escape trigger has fired and the regression gate has failed on 3 consecutive escape events with no improvement), **OR when Step IC3 triggers an early tournament via fingerprint convergence warning.**

**Tournament procedure:** Step SD (Structural Divergence Check) fires first, then T1–T5. All procedures unchanged from v9.

#### Step SD — Structural divergence check *(Unchanged from v9 — fires at every tournament event, whether scheduled or early-triggered)*

SD1–SD5 unchanged. See v9 for full specification.

#### T1–T5 *(Unchanged from v9)*

T1 (current-best readout), T2 (global best promotion), T3 (lowest-scoring thread retirement), T4 (new thread spawn from global best), T5 (tournament ledger update) — all unchanged from v9.

T5 is extended to log fingerprint check history since the last tournament: number of checks run, any convergence warnings, whether each warning was confirmed by Step SD or was a false positive, and the false positive rate for the current run.

---

### Oracle consistency

All three threads evaluate candidates against the same fixed evaluate.md throughout the run. The oracle is frozen. Scores are directly comparable across threads. Tournament winner selection is valid because all scores are measured against the same evaluator. Unchanged.

---

### Global state structures

The run maintains two global state structures (unchanged from v9) plus one additional structure:

**Tournament ledger:** Unchanged from v9, extended to include: fingerprint check history since last tournament (iteration count of each check, warning flags, confirmation/false-positive verdicts), cumulative false positive rate.

**Global best history:** Unchanged from v9.

**Fingerprint check log (NEW in v10):** A global log of all IC2 checks run: iteration count, signatures from all three threads at check time, pairwise comparison results, any convergence warnings, and resolution (early SD triggered / false positive / suppressed). Maintained globally for diagnostic use. Does not affect thread operation.

---

### Termination condition *(Unchanged from v1–v9)*

The run terminates when the global iteration count reaches the configured limit (e.g., 30 iterations). Final output is the global best candidate from the last tournament ledger entry (or the highest-scoring current-best across all active threads if no tournament has fired since the last iteration).

---

## Perturbation direction logic

Unchanged from v9. Direction during standard escape is determined by antipode derivation from the thread's current axis pool. Direction during crossover is determined by structural element recombination across historical bests. Main-loop perturbation direction (evaluator's "SUGGESTED DIRECTION" guidance) unchanged. Cross-thread perturbation enrichment at tournament time unchanged.

---

## Re-integration

Unchanged from v9. Within each thread: escape candidate replaces thread's current-best only if it clears the regression gate (≥ 80%). Cross-thread re-integration: global best from any thread becomes the starting candidate for newly spawned threads (T4) and re-diversified threads (SD3).

---

## Estimated cost per run

~$91–145 for a standard 30-iteration run with 3 threads, 1–2 escape events per thread, 1 crossover event per thread, and 2 tournament events.

- Per-thread cost: ~$30–46 (unchanged from v7/v8/v9)
- 3 threads × $30–46 = $90–138 baseline
- Tournament orthogonality verification at run start: ~$0.10–0.15 (unchanged)
- Per-tournament SD check: ~$0.08–0.12 (unchanged)
- Per-tournament T4 re-seeding: ~$0.05–0.10 (unchanged)
- SD re-diversification (when triggered): ~$0.05–0.10 per re-diversified thread (unchanged)
- **IC2 fingerprint checks (NEW):** 4 checks per 30-iteration run (at iterations 5, 10, 20, 25) × ~0 LLM cost (string comparison) = negligible
- **Fingerprint generation (NEW):** ~50 tokens × 90 iterations (3 threads × 30 iterations) = ~4,500 additional tokens ≈ ~$0.04–0.07 at Bedrock Sonnet rates
- **Early SD checks triggered by fingerprint warnings (NEW):** Estimated 0–2 early checks per run (false positive rate ~30% of fingerprint warnings, ~1 warning per run) = 0–2 × $0.12 = $0.00–0.24
- Total IC overhead: ~$0.05–0.31 per run
- Total: ~$91–145 (v10 adds <$0.35 to v9 estimate)

Well under $250.

---

## What changed this iteration (v9 → v10)

**Element changed:** Population structure — added inter-tournament convergence monitoring (Step IC) operating every 5 iterations between tournament events. One change only. All escape mechanisms (Steps 1–6d), escape trigger, Step SD, thread initialisation, T1–T5, and all other elements preserved identically.

---

### The change — Inter-tournament convergence monitoring

**v9 state:** The structural divergence check (Step SD) fires at every tournament boundary — every 15 main-loop iterations. Between tournaments, convergence can happen silently. A thread pair can begin pulling toward the same attractor at iteration 1 of a tournament cycle and not be detected until the tournament fires at iteration 15 — up to 14 iterations of convergence waste before the correction mechanism sees it. VC Stress-Tester (v9 eval): "the mechanism only fires at tournament boundaries (every 15 iterations) — convergence between tournaments is undetected and can waste compute." EV Platform Strategist: "relies on a single opus call for structural judgement — a false negative lets convergence persist for another full tournament cycle (~15 iterations)." Consensus weakest element at 7.74.

**v10 change:** After each iteration, each thread generates a compact 3-element structural signature [primary_frame, causal_direction, resolution_register] appended to its fingerprint log (~50 tokens per iteration, no separate model call). Every 5 iterations, the harness compares the three most recent signatures pairwise. If any pair shares 2/3 elements, it fires an early Step SD check immediately.

**Why this design works as a trip-wire:**

The fingerprint is intentionally cheap and slightly imprecise. primary_frame is a free-text phrase (approximate, not controlled vocabulary), causal_direction and resolution_register use fixed 5-label vocabularies. Two threads sharing the same causal_direction AND resolution_register while also converging on the same frame phrase is a strong structural convergence signal — even if the frames don't perfectly match, two out of three matches across fixed vocabularies is meaningful. The trip-wire analogy is apt: stepping on it doesn't convict you of convergence, it escalates to the full inspection (Step SD). False positives cost $0.10 and return DISTINCT; false negatives are caught within 5 more iterations.

**The key property — convergence window shrinks from 14 to 4 iterations:**

In v9, convergence detected at tournament N was already present for up to 14 iterations before detection. In v10, the fingerprint check fires every 5 iterations. Worst case: convergence begins immediately after a fingerprint check, persists for 4 iterations, and is detected at the next check. Maximum detection window: 4 iterations. Maximum convergence waste: 4 iterations of redundant exploration before the trip-wire fires and escalates to Step SD.

**Why structural fingerprints instead of score proximity:**

Score proximity is already available — it's not used because it doesn't detect structural convergence (two threads can improve in parallel from the same basin). The fingerprint captures structural character at low cost, providing the same type of signal as Step SD's anatomy sketches but at 1/100th the cost (~50 tokens vs ~500 tokens + opus call). The fingerprint is a fast pre-filter; Step SD is the authoritative check.

**The false positive property is desirable:**

A false positive fingerprint warning triggers an early Step SD check ($0.10) and returns DISTINCT. The cost is minimal and the confirmation step runs the authoritative check. False positives from loose primary_frame matching are therefore not a problem — they are the designed behaviour of a trip-wire that errs toward early investigation.

**Relationship to Step SD:**

Step IC does not change Step SD. It adds an early trigger path to the existing mechanism. Every scheduled tournament still fires its Step SD check regardless of what IC has detected. Early IC-triggered SD checks are in addition to, not instead of, scheduled checks. The two mechanisms are complementary: IC provides high-frequency cheap monitoring; SD provides authoritative structural judgement at every tournament boundary and on early escalation.

**What this does NOT change:**
- Escape trigger mechanics (10 → 5 → 3 consecutive discards — unchanged from v2–v9)
- Antipode escape mechanism (Steps 1–4 — unchanged from v2–v9)
- Adaptive axis discovery (Step 5 — unchanged from v4–v9)
- Axis orthogonality verification (Step 5b-bis — unchanged from v4–v9)
- Probe seed store for axes (monotonically growing per-thread — unchanged from v4–v9)
- Basin clustering trigger (3+ consecutive events within 2% — unchanged from v6–v9)
- Historical best inventory and decomposition (Step 6b — unchanged from v6–v9)
- Hybrid generation constraints (60% cap, coherence filter — Step 6c unchanged)
- Structural novelty verification for crossover hybrids (Step 6c-bis — unchanged from v7–v9)
- Maximum-distance fallback for blended hybrids (Step 6c-bis — unchanged from v7–v9)
- Mini-sprint structure (5 iterations — unchanged from v2–v9)
- Regression gate threshold (0.80 — unchanged from v2–v9)
- Crossover winner selection and basin window reset (Step 6d — unchanged from v6–v9)
- No-escalation-beyond-crossover rule (unchanged from v6–v9)
- Progressive counter shortening on gate failure (unchanged from v2–v9)
- Axis retirement logic (3+ consecutive fails — unchanged from v3–v9)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- Population structure (3 threads — unchanged from v8/v9)
- Thread initialisation and axis seeding (unchanged from v8/v9)
- Independent operation between tournaments (unchanged from v8/v9)
- Tournament interval (every 15 iterations — unchanged from v8/v9, now also early-triggerable)
- Step SD procedure (SD1–SD5 — unchanged from v9)
- T1–T5 tournament procedure (unchanged from v8/v9)
- Global state structures (tournament ledger, global best history — unchanged from v9, extended)
- Termination condition (unchanged from v1–v9)
- evaluate.md (frozen throughout, shared oracle — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
