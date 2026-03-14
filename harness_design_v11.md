# Harness Design — v11 (Domain-Configurable Fingerprint Vocabularies)
*Autoresearch escape loop — eleventh iteration*
*260314*

---

## Escape trigger
Per-thread: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4/v6/v7/v8/v9/v10 — operating independently within each thread.)*

**Basin clustering detection (per-thread, unchanged from v6/v7/v8/v9/v10):** After every escape event in which the regression gate passes, each thread independently checks whether it is circling a basin. The check: have the last 3+ consecutive escape events within that thread each produced a best seed that scores within 2% of that thread's current best score?

> **Basin condition:** For each of the most recent N consecutive gate-passing escape events within a thread (N ≥ 3), the highest-scoring seed from that event satisfies: `seed_score ≥ thread_current_best_score × 0.98`. If this condition holds across 3+ consecutive events, basin clustering is detected and the crossover step fires **before** the next main-loop iteration within that thread.

The 2% threshold is applied to the **best seed from each escape event**, not the average. Basin clustering detection fires independently for each thread. Both the adaptive axis trigger (2+ consecutive gate failures) and the basin clustering trigger can be simultaneously active within a thread.

---

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, Basin Clustering Crossover, Crossover Novelty Verification, Lightweight Population Parallelism, Tournament-Time Structural Divergence Check, Inter-Tournament Convergence Monitoring, and Domain-Configurable Fingerprint Vocabularies.**

All escape mechanisms from v10 (Steps 1–6d, Step SD, Step IC) are preserved and operate identically within each thread and at each tournament boundary. The sole structural change is the addition of **Step V — vocabulary derivation** at run start, which replaces the fixed three-element fingerprint vocabulary with a domain-derived vocabulary before any thread is spawned. Step IC1–IC5 are updated to use the derived vocabulary. Everything else is unchanged.

---

### Step V — Domain vocabulary derivation *(NEW in v11 — fires once at run start, before thread initialisation)*

**Purpose:** Derive a structural fingerprint vocabulary that is meaningful trip-wires for the specific problem domain — not fixed labels that were designed for writing/argumentation and are semantically inert for investment analysis, product strategy, or technical framing. A vocabulary derived from the problem statement has discriminative power where it matters: in the domain being worked.

**Trigger:** Fires once, immediately after receiving the problem statement and before spawning any threads or running the orthogonality check.

---

#### V1 — Problem statement classification

Before generating the vocabulary, the dissection agent classifies the problem statement into one of four archetype families. Classification is a heuristic guide — it does not constrain the vocabulary generation; it seeds the derivation with domain-relevant examples to steer the model toward specificity.

**Archetypes:**
- **Argumentation/thesis** — Writing, essays, academic arguments, opinion pieces, strategic narratives
- **Investment/strategy** — Market analysis, competitive positioning, deal thesis, portfolio decisions, strategic planning
- **Product/design** — Feature direction, user experience decisions, roadmap prioritisation, competitive product claims
- **Technical/analytical** — Architecture decisions, research methodology, experimental design, system behaviour modelling

Classification is a soft prior, not a hard category. Problems that span multiple archetypes (e.g. a strategic investment memo with analytical components) should produce a vocabulary that reflects the blend — dimension labels from both archetype families can appear.

---

#### V2 — Vocabulary derivation call

The harness makes a single short model call (Sonnet) with the following prompt:

---

> **VOCABULARY DERIVATION PROMPT:**
>
> You are the vocabulary architect for a structural fingerprint system. A structural fingerprint is a compact 3-element tuple used to detect when parallel search threads are converging on similar solutions — it acts as a cheap trip-wire.
>
> The fingerprint has exactly three dimensions. Each dimension is a structural property of a candidate — something that can vary independently of content and captures a meaningfully different aspect of how the candidate is structured. When two threads share the same label on two or more dimensions, that is a convergence signal.
>
> For trip-wires to be effective, the label sets must be:
> 1. **Specific to this domain** — labels must be meaningful distinctions that an expert in this domain would recognise as genuinely different structural choices
> 2. **Mutually exclusive within each dimension** — a candidate should be assignable to exactly one label per dimension without ambiguity
> 3. **Collectively exhaustive** — the label set should cover the realistic range of structural variation for this domain
> 4. **Orthogonal across dimensions** — the three dimensions should capture independent axes of structural variation (not correlated)
> 5. **Discriminative, not generic** — prefer labels that would surprise an expert in an adjacent domain; avoid labels that apply equally well to any reasoning task
>
> **Problem statement you are building the vocabulary for:**
> `{PROBLEM_STATEMENT}`
>
> **Domain archetype (soft prior):** `{ARCHETYPE}`
>
> **Examples of strong domain-specific vocabularies:**
>
> *Writing/thesis domain:*
> - Dim 1 — primary_frame: [scarcity logic, network effects, epistemic humility, market failure, evolutionary pressure, systems intervention]
> - Dim 2 — causal_direction: [bottom-up, top-down, bidirectional, lateral, emergent]
> - Dim 3 — resolution_register: [prescriptive, diagnostic, predictive, reframing, irresolvable]
>
> *Investment/strategy domain:*
> - Dim 1 — thesis_type: [market-structure dislocation, technology substitution, regulatory arbitrage, capital misallocation, platform accumulation, management quality play]
> - Dim 2 — risk_framing: [right-tail seeking, downside protection, scenario hedging, Kelly sizing, optionality preservation]
> - Dim 3 — time_horizon: [cycle trade, inflection point, compounding hold, terminal value bet]
>
> *Product/design domain:*
> - Dim 1 — user_problem_type: [friction elimination, capability unlock, social coordination failure, status signalling gap, workflow reinvention, trust deficit]
> - Dim 2 — solution_mechanism: [reduce steps, shift locus of control, change default, expose latent network, automate judgment, provide legibility]
> - Dim 3 — competitive_claim: [wedge expansion, category creation, incumbent displacement, ecosystem lock-in, experience moat, regulatory capture]
>
> *Technical/analytical domain:*
> - Dim 1 — problem_structure: [optimisation under constraint, search in combinatorial space, distribution shift, causal identification, systems stability, complexity emergence]
> - Dim 2 — solution_approach: [reduction to known problem, relaxation then tighten, empirical characterisation, formal proof, simulation, structural decomposition]
> - Dim 3 — validation_claim: [correctness guarantee, empirical validation, benchmark comparison, robustness bound, interpretability demonstration, deployment feasibility]
>
> **Your task:**
>
> Generate a vocabulary for the problem statement above. Do NOT copy an example vocabulary — generate one that is specific to THIS problem statement in THIS domain. The labels should be the kinds of structural distinctions an expert working on this problem would recognise as meaningfully different choices — not generic enough to apply to any reasoning task.
>
> Output format (strictly):
> ```
> DIM1_NAME: [label_a, label_b, label_c, label_d, label_e]
> DIM1_DEFINITION: <one sentence: what structural property does this dimension capture?>
> DIM2_NAME: [label_a, label_b, label_c, label_d]
> DIM2_DEFINITION: <one sentence>
> DIM3_NAME: [label_a, label_b, label_c, label_d, label_e]
> DIM3_DEFINITION: <one sentence>
> ```
> Each dimension must have 4–6 labels. Labels must be 1–4 words each. Do not use the dimension names or labels from the examples verbatim. Do not add explanation beyond the format above.

---

**Model:** Sonnet (not Opus — this is a light classification and label-generation task).
**Cost:** ~400–600 tokens (prompt + output) ≈ $0.02 at Sonnet Bedrock rates.
**Timeout:** If the derivation call fails or times out after 2 retries, fall back to the writing/thesis archetype vocabulary (the v10 defaults) — log the fallback to global state.

---

#### V3 — Vocabulary validation and freezing

After receiving the derivation output, the harness performs lightweight structural validation:

**i. Format check (no LLM call):**
- Exactly 3 dimensions output
- Each dimension has a name (non-empty), a definition (non-empty), and a label list
- Each label list has 4–6 entries
- No dimension has a label list where all labels are single common words (e.g., ["high", "low", "medium"] fails — too low discriminative power)

**ii. On format failure:** One retry with the same prompt. If retry also fails format check, fall back to the archetype-default vocabulary.

**iii. Freeze:** The validated vocabulary is stored as the **run-level fingerprint vocabulary** in global state. It is immutable from this point — no thread, escape event, crossover event, or tournament can modify it. All threads receive the frozen vocabulary as part of their initialisation state.

**iv. Frozen vocabulary stored in global state as:**
```
fingerprint_vocabulary: {
  dim1: { name: "<DIM1_NAME>", definition: "<DIM1_DEF>", labels: ["...", "...", ...] },
  dim2: { name: "<DIM2_NAME>", definition: "<DIM2_DEF>", labels: ["...", "...", ...] },
  dim3: { name: "<DIM3_NAME>", definition: "<DIM3_DEF>", labels: ["...", "...", ...] }
  derived_from_archetype: "<ARCHETYPE>",
  derivation_timestamp: "<ISO8601>",
  fallback_used: true/false
}
```

This structure replaces the implicit fixed vocabulary that was baked into the Step IC1 generation instruction in v10.

---

### Step 1 — Structural dissection with live axis pool *(Per-thread, unchanged from v7/v8/v9/v10)*

Unchanged. See v10.

---

### Step 2 — Antipode generation *(Per-thread, unchanged from v7/v8/v9/v10)*

Unchanged. See v10.

---

### Step 3 — Extended mini-evaluation sprint *(Per-thread, unchanged from v7/v8/v9/v10)*

Unchanged. See v10.

---

### Step 4 — Regression-gated winner selection *(Per-thread, unchanged from v7/v8/v9/v10)*

Unchanged. See v10.

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Per-thread, unchanged from v7/v8/v9/v10)*

Unchanged. See v10 for full specification of Steps 5a, 5b, 5b-bis, 5c.

---

### Step 6 — Structural crossover on basin clustering detection *(Per-thread, unchanged from v7/v8/v9/v10)*

Unchanged. See v10 for full specification of Steps 6a, 6b, 6c, 6c-bis, 6d.

---

## Population structure *(Unchanged from v8/v9/v10 — three independent threads with tournament selection every 15 iterations, plus inter-tournament fingerprint check every 5 iterations)*

### Thread initialisation

The harness spawns **3 threads** at run start. Each thread is a complete, self-contained instance of the v10 harness — its own current-best candidate, its own axis pool, its own probe seed store, its own historical best inventory, its own escape trigger counter, its own basin window, its own discard counter, and its own structural fingerprint log.

**NEW in v11:** Threads receive the frozen run-level fingerprint vocabulary at initialisation. Each thread's iteration agent is given the vocabulary as part of its operating context. The vocabulary is identical across all three threads — comparability preserved throughout the run.

**Differentiated axis pool seeds:** Unchanged from v8/v9/v10. The three threads are seeded with non-overlapping axis pools. At harness start (after Step V completes), the harness generates probe seeds for candidate seed axes and verifies mutual orthogonality across all 9 starting axes in a single Opus call. The 9 axes are accepted only if all 9 are mutually orthogonal — otherwise the harness regenerates the set (up to 3 retries).

**Default starting axis pools (example assignment — actual pools verified by orthogonality check at run time):** Unchanged from v9/v10.

**Thread labelling:** Threads are labelled T1, T2, T3. Unchanged.

**Initialisation order:**
1. Receive problem statement
2. Classify domain archetype (V1)
3. Derive and freeze fingerprint vocabulary (V2–V3)
4. Generate and verify orthogonal axis seeds (existing step, unchanged)
5. Spawn 3 threads with frozen vocabulary injected into each thread's context

---

### Independent operation (between tournaments) *(Unchanged from v8/v9/v10)*

Between tournament events, threads are fully independent. Unchanged. After completing each main-loop iteration, each thread appends a structural fingerprint to its fingerprint log using the domain-derived vocabulary (see Step IC1 below). Every 5 iterations, the harness reads all three threads' most recent fingerprints and compares them.

---

### Step IC — Inter-tournament convergence monitoring *(Updated in v11 — uses domain-derived vocabulary)*

**Purpose:** Unchanged from v10. Shrink the maximum convergence detection window from 14 iterations to 4 iterations. Step IC is a trip-wire. The core mechanism is identical to v10; the only change is that IC1's fingerprint generation instruction and IC2's comparison vocabulary are now drawn from the run-level fingerprint vocabulary derived in Step V, rather than the fixed [primary_frame/causal_direction/resolution_register] vocabulary hardcoded in v10.

---

#### IC1 — Structural fingerprint generation *(Per-thread, per iteration — updated in v11)*

After completing each main-loop iteration and updating its current-best, each thread's iteration agent generates a **structural signature** — a compact 3-element tuple using the **run-level vocabulary derived in Step V**.

The signature tuple format is:
```
[<DIM1_NAME_VALUE>, <DIM2_NAME_VALUE>, <DIM3_NAME_VALUE>]
```
where each value is exactly one label from the corresponding dimension's label list.

**Generation instruction to the iteration agent (parameterised by frozen vocabulary):**

> *"After updating your current-best this iteration, generate its structural signature as a 3-element tuple using the run vocabulary below. For each dimension, assign exactly one label from the provided list. Be approximate — this is a quick structural characterisation, not a precise analysis. Output format: SIGNATURE: [dim1_label], [dim2_label], [dim3_label]*
>
> *Run vocabulary:*
> *— {DIM1_NAME} ({DIM1_DEFINITION}): must be one of [{DIM1_LABELS}]*
> *— {DIM2_NAME} ({DIM2_DEFINITION}): must be one of [{DIM2_LABELS}]*
> *— {DIM3_NAME} ({DIM3_DEFINITION}): must be one of [{DIM3_LABELS}]*"*

The `{DIM_*}` placeholders are filled from the frozen vocabulary at run start and injected into each thread's iteration agent context. They are never changed during the run.

**Cost:** ~50–80 tokens added to the standard iteration agent call — appended as a brief post-iteration step. No separate model call required; generated as a by-product of the iteration agent's normal output. (Marginally higher than v10 due to vocabulary specification in the instruction — approximately 30 additional tokens per iteration call for the label-set enumeration.)

The signature is appended to the thread's **fingerprint log** — a simple per-thread list of (iteration_number, signature) entries. Structure unchanged from v10; vocabulary content domain-specific.

---

#### IC2 — Fingerprint comparison *(Every 5 iterations — updated in v11)*

Every 5 main-loop iterations (at iteration counts 5, 10, 20, 25, etc. — excluding iteration counts that coincide with a scheduled tournament event at iteration 15, 30, etc.), the harness reads the most recent fingerprint from each of the three threads' logs and performs a pairwise comparison.

**Comparison rule (unchanged from v10, now applied to derived vocabulary labels):**

For each pair (T1 vs T2, T1 vs T3, T2 vs T3), count how many of the 3 signature elements match:
- **dim1 match:** exact string equality after lowercasing and stripping whitespace
- **dim2 match:** exact label equality (from derived label set)
- **dim3 match:** exact label equality (from derived label set)

If any pair shares **2 or more of 3 elements**, flag that pair as a **fingerprint convergence warning**.

**Why derived vocabularies improve IC2 discriminative power:**

In v10, `causal_direction` had 5 fixed labels spanning the entire space of causal reasoning structures. For a problem in the investment domain, two threads could legitimately share the label `top-down` (structural market forces) while pursuing completely different theses — the label is too generic to distinguish structural character in that domain. With a derived vocabulary (e.g. `risk_framing: [right-tail seeking, downside protection, scenario hedging, Kelly sizing, optionality preservation]`), a match on `downside protection` is a genuine signal: two threads are structurally converging on the same risk posture, not just coincidentally applying the same causal abstraction.

Conversely, in v10, two genuinely different investment threads that were structurally exploring different axes could both be assigned `emergent` (because both involved complex, non-linear dynamics) — the label is so broad it fails as a trip-wire. Domain-derived labels are narrow enough to be meaningfully exclusive.

**Matching remains intentionally strict** — exact string equality after normalisation. The label sets are controlled vocabularies (unlike v10's free-text primary_frame), so exact-match is appropriate for all three dimensions in v11. This improves precision: v10 used exact matching for causal_direction and resolution_register but relied on free-text coincidence for primary_frame (a known source of false positives). In v11, all three dimensions use controlled-vocabulary exact-matching — false positive rate from accidental lexical overlap is lower.

**IC2 fires concurrently** — unchanged from v10. Requires no LLM call. Cost: negligible.

---

#### IC3 — Early tournament trigger on convergence warning *(Unchanged from v10)*

Unchanged. If any pairwise fingerprint comparison returns a convergence warning (2/3 elements match): early Step SD fires immediately. SD1–SD5 procedure unchanged.

---

#### IC4 — Fingerprint check timing relative to scheduled tournaments *(Unchanged from v10)*

Unchanged. See v10 for full specification.

---

#### IC5 — Fingerprint log maintenance *(Unchanged from v10)*

Unchanged. Fingerprint log is append-only per thread. When a thread is retired or re-diversified, its log is archived to the global state tournament ledger appendix. New and re-diversified threads start with an empty fingerprint log and receive the (unchanged) frozen run-level vocabulary.

---

### Tournament selection *(Unchanged from v9/v10 — Step SD + T1–T5)*

**Trigger:** Every 15 main-loop iterations completed across the total run, OR when any thread reaches a convergence state, OR when Step IC3 triggers an early tournament via fingerprint convergence warning. Unchanged.

**Tournament procedure:** Step SD fires first, then T1–T5. All procedures unchanged from v9/v10.

#### Step SD — Structural divergence check *(Unchanged from v9/v10)*

SD1–SD5 unchanged. The SD structural anatomy check operates on the full text of thread current-bests — it is not affected by the fingerprint vocabulary change. The fingerprint trip-wire and SD authoritative check remain complementary and independent.

#### T1–T5 *(Unchanged from v9/v10)*

All unchanged. T5 logs fingerprint check history since the last tournament, including a reference to the frozen vocabulary used for the run.

---

### Oracle consistency *(Unchanged)*

All three threads evaluate candidates against the same fixed evaluate.md throughout the run. The oracle is frozen. Unchanged.

---

### Global state structures

The run maintains the same global state structures as v10 plus one addition:

**Tournament ledger:** Unchanged from v10, extended to include a reference to the frozen fingerprint vocabulary (vocabulary name, archetype classification, fallback used flag).

**Global best history:** Unchanged from v9/v10.

**Fingerprint check log:** Unchanged from v10. Extended to include the run-level vocabulary in the log header for diagnostic use.

**Run-level fingerprint vocabulary (NEW in v11):** Stored once in global state after Step V completes. Immutable. Contains DIM1, DIM2, DIM3 with name, definition, label list, archetype used, derivation timestamp, and fallback flag. Injected into each thread's context at initialisation.

---

## Perturbation direction logic *(Unchanged from v9/v10)*

Unchanged. Direction during standard escape is determined by antipode derivation from the thread's current axis pool. Direction during crossover is determined by structural element recombination across historical bests. Main-loop perturbation direction (evaluator's "SUGGESTED DIRECTION" guidance) unchanged. Cross-thread perturbation enrichment at tournament time unchanged.

---

## Re-integration *(Unchanged from v9/v10)*

Unchanged. Within each thread: escape candidate replaces thread's current-best only if it clears the regression gate (≥ 80%). Cross-thread re-integration: global best from any thread becomes the starting candidate for newly spawned threads (T4) and re-diversified threads (SD3).

---

## Estimated cost per run

~$91–145 for a standard 30-iteration run with 3 threads, 1–2 escape events per thread, 1 crossover event per thread, and 2 tournament events.

- Per-thread cost: ~$30–46 (unchanged from v7/v8/v9/v10)
- 3 threads × $30–46 = $90–138 baseline
- Tournament orthogonality verification at run start: ~$0.10–0.15 (unchanged)
- Per-tournament SD check: ~$0.08–0.12 (unchanged)
- Per-tournament T4 re-seeding: ~$0.05–0.10 (unchanged)
- SD re-diversification (when triggered): ~$0.05–0.10 per re-diversified thread (unchanged)
- IC2 fingerprint checks: 4 checks per 30-iteration run × ~0 LLM cost = negligible (unchanged)
- Fingerprint generation (updated): ~70 tokens × 90 iterations ≈ ~$0.05–0.08 at Bedrock Sonnet rates (marginal increase from ~$0.04–0.07 in v10 due to vocabulary enumeration in instruction — ~+$0.01)
- Early SD checks triggered by fingerprint warnings: 0–2 × $0.12 = $0.00–0.24 (unchanged)
- **Step V — vocabulary derivation (NEW):** 1 Sonnet call × ~500 tokens ≈ **$0.02**
- Total IC overhead: ~$0.07–0.34 per run (v11 adds ~$0.02 to v10)
- Total: ~$91–146 (v11 adds <$1.00 to v10 estimate)

Well under $250.

---

## What changed this iteration (v10 → v11)

**Element changed:** Population structure / Step IC — added domain-configurable fingerprint vocabulary derivation (Step V) at run start. One change only. All escape mechanisms (Steps 1–6d), escape trigger, Step SD, thread initialisation, T1–T5, tournament structure, and all other elements preserved identically.

---

### The change — Domain-configurable fingerprint vocabularies

**v10 state:** The IC1 fingerprint used a fixed three-dimension vocabulary hardcoded at design time: `[primary_frame (free text), causal_direction (5 fixed labels: bottom-up/top-down/bidirectional/lateral/emergent), resolution_register (5 fixed labels: prescriptive/diagnostic/predictive/reframing/irresolvable)]`. These labels are appropriate for argumentation, strategy, and writing domains. For investment analysis, a candidate labelled `top-down` could be structurally anything — a macro thesis, a regulatory argument, a competitive moat claim, or a capital allocation rationale. The label is too abstract to function as a meaningful trip-wire. For a product direction problem, `prescriptive` says nothing about whether two threads are converging on similar user problem framings or similar solution mechanisms. All three judges scored D4 at 7/10: "domain-specific fingerprint vocabularies are the bottleneck for generality."

**v11 change:** Step V fires once at run start, before thread spawning, making a single Sonnet call (~$0.02) that:
1. Classifies the problem statement into an archetype family (soft prior for seeding)
2. Derives 3 structural fingerprint dimensions with 4–6 domain-specific labels each — specific enough to function as meaningful trip-wires for the target domain
3. Validates the output (format + discriminative power check) with fallback to v10 defaults if derivation fails
4. Freezes the vocabulary in global state and injects it into all thread contexts at initialisation

**Why domain-specific labels are better trip-wires:**

The IC trip-wire works by detecting when two threads independently converge on the same structural label. This signal is only useful if the label carves the domain's solution space at the right joints. For investment analysis, the structural joints that matter are: what type of thesis (market structure vs technology substitution vs management quality), what risk posture, what time horizon — not what causal direction in the abstract sense. A convergence warning on `thesis_type: technology substitution` in two threads is a genuine signal: both threads are building the same class of investment argument. A convergence warning on `causal_direction: bottom-up` in the v10 system is nearly meaningless — half of all investment theses involve micro-level mechanisms producing macro outcomes.

**Specificity → discriminative power → lower false negative rate:**

Generic labels (e.g., `emergent`) apply to a wide range of structurally distinct candidates — two threads that are genuinely different but both involve complex dynamics will share the label accidentally. Domain-specific labels apply to a narrower range: if two threads share `risk_framing: Kelly sizing`, they are making the same structural claim about position sizing logic — genuine convergence, not an accidental lexical match. The false negative rate (missing genuine convergence) decreases as labels become more discriminative.

**Specificity → controlled vocabulary for all three dimensions → lower false positive rate:**

In v10, `primary_frame` was free text. Two threads that happened to independently write "network effects" as their frame would match — but so might "network effects argument" vs "network effects logic". In v11, all three dimensions use controlled vocabulary lists, so exact-match comparison is appropriate and precise. There is no free-text ambiguity.

**The vocabulary is frozen at run start — comparability preserved:**

All threads derive fingerprints from the same vocabulary throughout the run. Scores are comparable across threads; convergence warnings are interpretable. No per-thread vocabulary drift. No mid-run vocabulary changes. The freeze is absolute — even if the problem statement is partially reinterpreted during the run (which can happen in open-ended writing tasks), the vocabulary is not updated.

**Cost is negligible — $0.02 for a run-long improvement:**

One Sonnet call at run start. The vocabulary derivation adds $0.02 to the total run cost (which is $91–146 already). The benefit: every IC2 fingerprint comparison throughout the entire run uses labels that are meaningful for the domain. The cost-benefit ratio is approximately 1:4600 (one $0.02 derivation call improves the quality of 4 fingerprint checks × 3 thread-pairs × 30 iterations of fingerprint generation = 360 data points of structural monitoring).

**The fallback is conservative:**

If Step V fails after retries, the harness falls back to the v10 defaults (writing/thesis vocabulary). This means v11 degrades gracefully to v10 behaviour — it never fails harder than v10. The fallback is logged in global state.

**What this does NOT change:**
- Escape trigger mechanics (10 → 5 → 3 consecutive discards — unchanged from v2–v10)
- Antipode escape mechanism (Steps 1–4 — unchanged from v2–v10)
- Adaptive axis discovery (Step 5 — unchanged from v4–v10)
- Axis orthogonality verification (Step 5b-bis — unchanged from v4–v10)
- Probe seed store for axes (monotonically growing per-thread — unchanged from v4–v10)
- Basin clustering trigger (3+ consecutive events within 2% — unchanged from v6–v10)
- Historical best inventory and decomposition (Step 6b — unchanged from v6–v10)
- Hybrid generation constraints (60% cap, coherence filter — Step 6c unchanged)
- Structural novelty verification for crossover hybrids (Step 6c-bis — unchanged from v7–v10)
- Maximum-distance fallback for blended hybrids (Step 6c-bis — unchanged from v7–v10)
- Mini-sprint structure (5 iterations — unchanged from v2–v10)
- Regression gate threshold (0.80 — unchanged from v2–v10)
- Crossover winner selection and basin window reset (Step 6d — unchanged from v6–v10)
- No-escalation-beyond-crossover rule (unchanged from v6–v10)
- Progressive counter shortening on gate failure (unchanged from v2–v10)
- Axis retirement logic (3+ consecutive fails — unchanged from v3–v10)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- Population structure (3 threads — unchanged from v8/v9/v10)
- Thread initialisation and axis seeding (unchanged from v8/v9/v10)
- Independent operation between tournaments (unchanged from v8/v9/v10)
- Tournament interval (every 15 iterations — unchanged from v8/v9/v10)
- Inter-tournament fingerprint check interval (every 5 iterations — unchanged from v10)
- IC2 comparison rule (2/3 match = convergence warning — unchanged from v10)
- IC3 early tournament trigger (unchanged from v10)
- IC4 timing and suppression rules (unchanged from v10)
- IC5 fingerprint log maintenance (unchanged from v10)
- Step SD procedure (SD1–SD5 — unchanged from v9/v10)
- T1–T5 tournament procedure (unchanged from v8/v9/v10)
- Global state structures (tournament ledger, global best history, fingerprint check log — unchanged from v10, extended)
- Termination condition (unchanged from v1–v10)
- evaluate.md (frozen throughout, shared oracle — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
