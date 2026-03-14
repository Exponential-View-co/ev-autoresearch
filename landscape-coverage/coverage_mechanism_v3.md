# Coverage Mechanism — v3 (Concrete G1 Cluster Assignment)
*Landscape coverage mechanism — v3*
*260314*

---

## Survey design (UNCHANGED from v2, minor S4 addition)

### Overview

A pre-run structural landscape survey fires once at run start (after Step V vocabulary derivation, before thread spawning). The survey generates 15–20 brief candidate sketches across structurally diverse regions of the solution space, clusters them into 3 guaranteed non-overlapping regions, and seeds T1, T2, T3 from those regions — not from abstract axis labels.

**Core design principle:** Structural diversity is defined at the level of claim type, not surface phrasing. Two sketches that use different words but make the same type of logical move are in the same structural region. Clustering is at claim-structure level. The survey includes a three-layer structural verification mechanism to guarantee that the 3 seeded regions are non-overlapping by construction.

All steps use haiku-bedrock (fast, cheap). Survey total cost: ~$0.013, well within the $5 constraint.

---

### Step S0 — Structural approach taxonomy derivation

**Fires:** Once, immediately after Step V (vocabulary derivation), before thread spawning.

**Purpose:** Derive a structural taxonomy of 12–18 claim types for this specific problem. A claim type is defined by HOW it argues, not what it argues about. The taxonomy is the foundation of all downstream steps.

**Model:** haiku-bedrock

**Prompt:**

> You are a structural analyst. Your task is to enumerate the distinct structural CLAIM TYPES that could be used to answer the following question — not the specific answers, but the different structural approaches to constructing an answer.
>
> A structural claim type is defined by the TYPE OF LOGICAL MOVE used to reach an answer, not the specific content. Two approaches that use different words but make the same type of argument are the SAME claim type. Two approaches that use similar words but argue from structurally different bases are DIFFERENT claim types.
>
> Problem statement: {PROBLEM_STATEMENT}
>
> Generate 12–18 structural claim types covering the realistic solution space. For each, provide:
> — CLAIM_TYPE_ID: a short identifier in CAPS_SNAKE_CASE
> — STRUCTURE: one sentence defining the type of logical move this claim type makes
> — CANNOT_BE_MERGED_WITH: name 1–2 other claim types that look similar but are structurally distinct, and state in one clause WHY they are distinct
>
> Requirements:
> — Each claim type must be structurally distinct (different logical move, not just different vocabulary)
> — Together they should cover the plausible solution space
> — Order them so that structurally similar types appear adjacent
> — Do not produce generic types that apply to any reasoning task; every type must be specific to THIS problem

**Cost:** ~1,000 tokens ≈ $0.001 at haiku-bedrock rates.

---

### Step S1 — Sketch generation

**Fires:** Immediately after S0 completes.

**Purpose:** Generate one 3-sentence directional sketch per claim type. Sketches are structural bets, not full answers.

**Model:** haiku-bedrock (batched — all 12–18 sketches in a single prompt)

**Prompt:**

> You will generate brief candidate sketches for the following problem: {PROBLEM_STATEMENT}
>
> Below is a list of structural claim types. For EACH claim type, write a 3-sentence sketch:
> — Sentence 1: The core claim (commit to a specific position)
> — Sentence 2: The key mechanism (state the structural argument explicitly)
> — Sentence 3: The expected shape of a strong answer
>
> CRITICAL: Each sketch must commit to the claim type it was generated for. Do not reframe, hedge, or blend claim types.
>
> {CLAIM_TYPES_LIST}
>
> Output format: one sketch per claim type, clearly labelled with CLAIM_TYPE_ID.

**Cost:** ~4,000–5,000 tokens ≈ $0.004–0.005.

---

### Step S2 — Structural verification (Layer 2 guarantee)

**Fires:** Immediately after S1.

**Purpose:** Verify that each sketch actually commits to its assigned claim type structurally, not just lexically. Catches the most common failure mode: haiku generating a sketch that uses the correct vocabulary but makes a structurally different argument.

**Model:** haiku-bedrock (batched — all sketches in a single prompt)

**Prompt:**

> You are a structural auditor. Below are a set of brief sketches, each labelled with the structural claim type it was generated for. For each sketch, determine whether the sketch is ACTUALLY making the structural argument defined by its claim type, or whether it has drifted.
>
> A sketch "drifts" when it uses the vocabulary of its assigned claim type but the underlying logical move belongs to a different claim type.
>
> Claim type definitions: {CLAIM_TYPES_LIST}
>
> Sketches to audit: {SKETCHES_LIST}
>
> For each sketch, output:
> — CLAIM_TYPE_ID: the original label
> — STRUCTURAL_MATCH: YES or NO
> — IF NO — ACTUAL_TYPE: which claim type does this sketch actually instantiate?
> — IF NO — REASON: one sentence describing the structural drift

**Processing after audit:**
- Sketches with STRUCTURAL_MATCH = YES are accepted as verified.
- Sketches with STRUCTURAL_MATCH = NO are reassigned to their ACTUAL_TYPE.
- If reassignment causes two verified sketches to share the same claim type: score both in S3 and discard the lower-scoring one.
- If a claim type ends up with zero verified sketches: retained in taxonomy for clustering (null sketch); not used in S5 seed selection.

**Cost:** ~2,000–3,000 tokens ≈ $0.002–0.003.

---

### Step S3 — 2-dimension scoring

**Fires:** Immediately after S2, on all verified sketches.

**Purpose:** Score each verified sketch on two cheap proxy dimensions to identify the most promising direction within each cluster.

**Model:** haiku-bedrock (batched — all verified sketches in a single prompt)

**Dimensions:**
1. **Plausibility** (1–5): Would a domain expert judge this approach likely to produce a substantive, defensible answer?
2. **Resolution potential** (1–5): Does this sketch point toward a specific, concrete answer shape?

**Prompt:**

> Score each of the following sketches on exactly two dimensions. Output only numbers.
>
> PLAUSIBILITY (1–5): Would a domain expert think this approach has genuine potential? 1 = implausible; 5 = clearly promising.
> RESOLUTION POTENTIAL (1–5): Does this sketch point toward a specific, concrete answer? 1 = vague; 5 = the shape of a strong answer is already visible.
>
> Do NOT score answer quality. Score only whether the approach is alive and direction-giving.
>
> Problem: {PROBLEM_STATEMENT}
>
> {VERIFIED_SKETCHES}
>
> Output format: one row per sketch — CLAIM_TYPE_ID | PLAUSIBILITY | RESOLUTION_POTENTIAL

**Cost:** ~1,500–2,000 tokens ≈ $0.001–0.002.

---

### Step S4 — Structural clustering into 3 regions (Layer 3 guarantee)

**Fires:** Immediately after S3.

**Purpose:** Group the 12–18 verified claim types into exactly 3 structurally non-overlapping clusters. Each cluster becomes one thread's starting region.

**Model:** haiku-bedrock (1 call)

**Prompt:**

> You have a list of structural claim types for this problem: {PROBLEM_STATEMENT}
>
> Claim type definitions: {CLAIM_TYPES_LIST}
>
> Your task: group these claim types into exactly 3 clusters such that:
> 1. Within each cluster: all claim types make the same FUNDAMENTAL TYPE OF BET
> 2. Across clusters: each cluster makes a DIFFERENT TYPE OF BET that cannot be reduced to either other cluster
> 3. The 3 clusters together cover the full set of verified claim types (no orphans)
>
> After grouping, produce a STRUCTURAL NON-OVERLAP AUDIT:
> — For each pair of clusters (C1 vs C2, C1 vs C3, C2 vs C3), state in one sentence WHY those two clusters cannot be merged.
> — If you cannot state a clear, specific structural distinction for any pair, that pair MUST be merged and remaining claim types re-split into a new third cluster.
>
> Output format (strictly):
> CLUSTER_1: {comma-separated CLAIM_TYPE_IDs} | STRUCTURAL_LABEL: {one sentence: what type of bet does this cluster make?} | STRUCTURAL_SIGNATURE: [{dim1_label}, {dim2_label}, {dim3_label}]
> CLUSTER_2: {comma-separated CLAIM_TYPE_IDs} | STRUCTURAL_LABEL: {one sentence} | STRUCTURAL_SIGNATURE: [{dim1_label}, {dim2_label}, {dim3_label}]
> CLUSTER_3: {comma-separated CLAIM_TYPE_IDs} | STRUCTURAL_LABEL: {one sentence} | STRUCTURAL_SIGNATURE: [{dim1_label}, {dim2_label}, {dim3_label}]
> NON_OVERLAP_AUDIT:
> C1 vs C2: {why they cannot be merged — one sentence}
> C1 vs C3: {why they cannot be merged — one sentence}
> C2 vs C3: {why they cannot be merged — one sentence}

**Note on STRUCTURAL_SIGNATURE (NEW in v3):** Each cluster now outputs a 3-element structural signature alongside the STRUCTURAL_LABEL. Each element is a domain vocabulary label drawn from Step V, together forming a machine-readable fingerprint of the structural bet the cluster makes. This is the only change to the S4 prompt in v3 — required to support G1's concrete haiku call. Generated in the same S4 call at no additional cost.

**Signature format example** (for an investment thesis problem):
```
STRUCTURAL_SIGNATURE: [market_mispricing, growth_trajectory, expected_value_error]
```

**Validation (no LLM call):** The harness checks:
- Every CLAIM_TYPE_ID from S0 appears in exactly one cluster
- Each cluster contains at least 1 verified sketch (non-null from S2)
- The NON_OVERLAP_AUDIT contains a non-empty, non-generic reason for each pair (minimum 8 words)
- Each cluster's STRUCTURAL_SIGNATURE contains exactly 3 non-empty labels

**On validation failure:** Up to 2 retries with the same prompt. If retries fail, fall back to v0 axis-label seeding and log `survey_fallback: true` in global state.

**Cost:** ~1,500–2,000 tokens ≈ $0.001–0.002 (marginal increase for STRUCTURAL_SIGNATURE output — negligible).

---

### Step S5 — Thread seed selection

**Fires:** Immediately after S4 validation passes.

**Purpose:** Select the best sketch from each cluster to seed the corresponding thread.

**Model:** No LLM call needed.

**Selection rule:** For each cluster, compute `seed_score = plausibility × resolution_potential` (from S3 scores). Select the sketch with the highest seed_score. On tie: prefer higher plausibility. On second tie: prefer the sketch from the claim type earliest in S0 output ordering.

**Thread seeding:** The selected sketch replaces the abstract axis label used in v0. Each thread receives:

> "You are Thread {N}. Your starting region is the {CLUSTER_N_STRUCTURAL_LABEL} region of the solution space. Begin from this directional sketch:
>
> {SELECTED_SKETCH}
>
> Your task is to iterate and develop this direction — not to explore structurally different regions (those are assigned to other threads). Stay within your structural region; do not drift toward the other clusters' structural bets."

**Landscape map stored in global state:**
```
survey_result: {
  taxonomy: [{claim_type_id, structure, cluster_assignment, sketch, plausibility, resolution_potential}, ...],
  clusters: [
    {cluster_id: 1, structural_label, structural_signature: [dim1, dim2, dim3], claim_types: [...], seed_sketch, seed_score},
    {cluster_id: 2, structural_label, structural_signature: [dim1, dim2, dim3], claim_types: [...], seed_sketch, seed_score},
    {cluster_id: 3, structural_label, structural_signature: [dim1, dim2, dim3], claim_types: [...], seed_sketch, seed_score}
  ],
  non_overlap_audit: {c1_vs_c2: "...", c1_vs_c3: "...", c2_vs_c3: "..."},
  survey_total_cost_usd: <estimate>,
  fallback_used: true/false
}
```

This map is immutable after S5 completes. Available to all mechanisms throughout the run.

---

### Survey cost summary

| Step | Model | Calls | Tokens (est.) | Cost (est.) |
|---|---|---|---|---|
| S0 — Taxonomy derivation | haiku-bedrock | 1 | ~1,000 | ~$0.001 |
| S1 — Sketch generation | haiku-bedrock | 1 (batched) | ~5,000 | ~$0.005 |
| S2 — Structural verification | haiku-bedrock | 1 (batched) | ~2,500 | ~$0.003 |
| S3 — 2-dimension scoring | haiku-bedrock | 1 (batched) | ~2,000 | ~$0.002 |
| S4 — Structural clustering + signatures | haiku-bedrock | 1 | ~2,000 | ~$0.002 |
| **Total** | haiku-bedrock | **5 calls** | **~12,500 tokens** | **~$0.013** |

Budget: $5.00 constraint. Actual usage: ~$0.013. Headroom: $4.987.

---

## Gate mechanism (CHANGED — G1 now uses a concrete haiku-bedrock call)

### Overview

The uniform 0.80 × current-best threshold is replaced with a **region-aware diversity gate**. The gate reads the `survey_result` cluster map and assigns each incoming candidate to its nearest cluster via a **single haiku-bedrock call per candidate**. The acceptance threshold is then set dynamically based on that cluster's current exploration status.

**Core design principle:** The gate adjusts the bar for entry based on how much value the harness has already extracted from a region. Candidates from under-explored regions face a lower bar. Candidates from actively-worked regions face the standard bar. Candidates from exhausted regions face a higher bar. The cluster assignment that drives these decisions is now made by the same model class that generated the cluster signatures — structural consistency guaranteed throughout.

---

### Step G1 — Candidate cluster assignment (CHANGED in v3 — concrete haiku-bedrock call)

**Fires:** Once per candidate, before the score comparison.

**Purpose:** Assign the incoming candidate to its nearest cluster in the survey map.

**Why a model call, not a heuristic:** Free-form candidate text generated by iterating threads cannot be reliably assigned to structural clusters by keyword matching or fingerprint voting. The cases that matter most — candidates near cluster boundaries, candidates generated from adaptive axes that were not present in the original S0 survey — are exactly the cases where surface-level matching fails:

- A candidate near the boundary between two clusters may use vocabulary from both; majority-vote fingerprinting returns the wrong cluster approximately 50% of the time in boundary cases.
- A candidate from an adaptive axis not in the original survey uses vocabulary the Phase 0 taxonomy never encountered; string matching returns no confident signal.

A haiku call reading the actual structural argument is reliable across all cases, including novel vocabulary and boundary cases. It uses the same model class that generated the cluster signatures — maximally consistent.

**The call:**

Single haiku-bedrock call per candidate. Full prompt:

> **System:** You are a structural classifier. You will be given a candidate text and three structural cluster signatures. Assign the candidate to the cluster whose structural argument most closely resembles the candidate's own structural argument.
>
> **Cluster signatures (from survey):**
> - Cluster 1: {survey_result.clusters[0].structural_label} | Signature: {survey_result.clusters[0].structural_signature}
> - Cluster 2: {survey_result.clusters[1].structural_label} | Signature: {survey_result.clusters[1].structural_signature}
> - Cluster 3: {survey_result.clusters[2].structural_label} | Signature: {survey_result.clusters[2].structural_signature}
>
> **Candidate text:**
> {CANDIDATE_TEXT}
>
> **Instruction:** Identify which cluster this candidate most structurally resembles. Focus on the TYPE OF LOGICAL MOVE the candidate makes — not its vocabulary or surface topic. Return ONLY:
> CLUSTER_ID: <1, 2, or 3>
> RATIONALE: <one sentence — what structural move does this candidate make, and why does that place it in this cluster rather than the others?>
> BOUNDARY_CASE: <YES or NO — YES if this candidate's structural argument could plausibly belong to a second cluster>

**Output parsing:**
- `CLUSTER_ID` parsed as integer (1, 2, or 3). On parse failure: retry once. On second failure: assign CLUSTER_ID = 0 (UNASSIGNED); gate defaults to 0.80 standard threshold.
- `RATIONALE` stored in candidate record for auditability.
- `BOUNDARY_CASE` triggers tie-breaking rule (see below).

**Boundary case tie-breaking:**
When `BOUNDARY_CASE = YES`, the gate does not average or split. It assigns to the cluster with lower current exploration status — UNEXPLORED is preferred over ACTIVE, which is preferred over EXHAUSTED. This rule rewards exploration in the exact cases where the harness most needs to be pushed toward new territory. If both candidate clusters are equally explored (both ACTIVE or both EXHAUSTED), the CLUSTER_ID returned by the model is used as-is.

**Candidate record update:**
```
candidate.cluster_id: <1|2|3|0>          # 0 = unassigned, gate uses 0.80 fallback
candidate.cluster_rationale: "<text>"     # stored for auditability
candidate.boundary_case: <true|false>
```

**Cost per candidate:** ~$0.001 (input: ~500 tokens — cluster context + candidate text; output: ~50 tokens at haiku-bedrock rates).

**Full-run cost:** For a 90-iteration 3-thread run generating ~270 candidates total:
- ~$0.001 × 270 = **~$0.27 per run**
- Total run overhead (survey + gate): **~$0.28**

---

### Step G2 — Cluster status classification

**Fires:** Immediately after G1, once per candidate.

**Purpose:** Determine the current exploration status of the candidate's assigned cluster. Status is re-evaluated at gate-check time — not precomputed — so it reflects the harness's live state.

**Three status classes:**

| Status | Definition |
|---|---|
| **UNEXPLORED** | No thread is currently operating in this cluster, OR the thread originally assigned to this cluster has been retired and no replacement spawned into this region |
| **ACTIVE** | Exactly one thread is currently operating in this cluster (spawned, running, has not hit convergence) |
| **EXHAUSTED** | The thread operating in this cluster has hit the convergence condition OR been retired after extended stay (≥ per-thread retirement threshold from v11 harness) |

**Classification rule (deterministic, no model call):**

```
cluster_status(cluster_id):
  if cluster_id == 0:
    return ACTIVE  # unassigned fallback — apply standard 0.80 threshold
  active_threads = [t for t in threads if t.cluster_id == cluster_id and t.status == RUNNING]
  exhausted_threads = [t for t in threads if t.cluster_id == cluster_id and t.status in {CONVERGED, RETIRED}]

  if len(active_threads) == 0 and len(exhausted_threads) == 0:
    return UNEXPLORED
  elif len(active_threads) >= 1 and len(exhausted_threads) == 0:
    return ACTIVE
  else:
    return EXHAUSTED
```

**Thread cluster assignment:** Each thread carries an immutable `thread.cluster_id` set at spawn time from its survey seed. When a thread is retired and a new thread spawned (e.g. by escape mechanism), the new thread has no cluster assignment — it is assigned via G1 on its first candidate.

**Cost:** Zero model calls. Pure state lookup.

---

### Step G3 — Threshold computation and gate decision

**Fires:** Immediately after G2, completing the gate check.

**Threshold table:**

| Cluster status | Acceptance threshold |
|---|---|
| UNEXPLORED | **0.65 × current-best** |
| ACTIVE | **0.80 × current-best** (unchanged from v1) |
| EXHAUSTED | **0.90 × current-best** |

**Gate decision:**

```
threshold = THRESHOLD_TABLE[cluster_status(candidate.cluster_id)]
gate_pass = (candidate.score >= threshold)
```

**Why these specific values:**

- **0.65 (unexplored):** A 23-point reduction from the standard bar. The harness pays a quality cost to enter new territory. The 0.65 floor is not a rubber stamp: candidates must still reach 65% of current best.
- **0.80 (active):** Unchanged from v1. Standard convergence pressure for regions being worked.
- **0.90 (exhausted):** A 10-point increase above the standard bar. Re-entry to a mined region requires genuinely superior quality — only step-changes pass, incremental variations are rejected.

**The spread:** 0.65 vs 0.90 is a 25-point gap. A candidate scoring 0.72 × current-best passes from an UNEXPLORED cluster, fails from an ACTIVE cluster, fails hard from an EXHAUSTED cluster. The gate reshapes search behaviour throughout the entire run.

---

### Anti-gaming property

With the G1 haiku call now live, the anti-gaming property is strengthened. A thread cannot game the UNEXPLORED threshold by generating candidates that superficially resemble a different cluster's vocabulary — the haiku call reads the actual structural argument, not surface vocabulary. The same model class that generated the cluster signatures judges conformance to those signatures. Structural drift that fooled keyword matching will not fool the haiku classifier.

The `RATIONALE` field stored per candidate creates an auditable record: if a thread systematically generates candidates being assigned to UNEXPLORED clusters when its own cluster is EXHAUSTED, rationales can be inspected for structural coherence — and the inter-tournament fingerprint monitoring (IC1–IC5) independently detects re-convergence.

---

### Convergence safety

The diversity gate does not prevent convergence. Four safeguards:

1. **The ACTIVE floor (0.80):** Standard convergence pressure applies while threads are running in all three clusters.
2. **Exhausted cluster ceiling (0.90):** Once exhausted, only superior candidates re-enter — reinforcing convergence in completed regions.
3. **No infinite loop risk:** A cluster cannot stay UNEXPLORED indefinitely. The harness's existing thread-spawning logic will eventually spawn a thread into any unseeded region; once spawned, 0.80 applies.
4. **Unassigned fallback (cluster_id = 0):** Candidates that fail G1 classification twice receive the standard 0.80 threshold. The harness does not stall on classification failures.

---

## Integration (UPDATED — G1 now an explicit model call)

**Updated gate execution order (per candidate):**
1. Candidate generated by thread
2. Oracle scores candidate (unchanged from v11)
3. **G1: Single haiku-bedrock call — candidate text + 3 cluster structural signatures → CLUSTER_ID + RATIONALE + BOUNDARY_CASE (CHANGED in v3 — replaces underspecified "S2 reuse")**
4. **G2: Classify cluster status — UNEXPLORED / ACTIVE / EXHAUSTED (unchanged from v2)**
5. **G3: Compute threshold from status table; gate decision (unchanged from v2)**
6. If gate passes: candidate enters tournament pool (unchanged from v11)
7. If gate fails: candidate discarded; thread continues (unchanged from v11)

**No change to oracle:** Oracle call fires before gate check (step 2). Gate uses oracle score but does not change how the oracle is called.

**Interaction with existing mechanisms:**

- **Survey (S0–S5):** S4 now outputs `structural_signature` per cluster — minor prompt addition, no extra LLM call, no structural change to survey logic. G1 consumes these signatures. No other survey steps change.
- **Fingerprint monitoring (IC1–IC5):** Unchanged. G1's haiku call reduces re-convergence rate; the fingerprint monitor remains the backstop.
- **Escape mechanisms (Steps 1–6d):** Unchanged. Escape-generated candidates are subject to G1–G3 like all candidates — and may well be assigned to an UNEXPLORED cluster, giving them a lower threshold. Desirable: escaped candidates exploring new territory face a lower bar.
- **Tournament / Step SD:** Unchanged. Gate determines which candidates enter the pool; tournament operates identically on that pool.
- **Crossover (Step 6):** Unchanged. Crossover candidates are subject to G1–G3.
- **Step V (vocabulary derivation):** Unchanged. Step V vocabulary is used in the `structural_signature` labels at S4; no new Step V changes required.

**Fallback behaviour:** If `survey_result` is absent (survey failed, fallback_used = true), the gate reverts to uniform 0.80 threshold from v1. G1's haiku call is not fired if there are no cluster signatures to compare against.

---

## Cost estimate (UPDATED)

Base cost unchanged from v11: ~$91–146 per run.

**Survey overhead:** ~$0.013 per run (unchanged from v2; negligible S4 marginal increase for structural_signature output absorbed within existing estimate).

**Gate overhead (UPDATED in v3):** One haiku-bedrock call per candidate.

| Metric | Value |
|---|---|
| Candidates per run | ~270 (90 iterations × 3 threads) |
| Cost per call | ~$0.001 |
| Gate overhead per run | **~$0.27** |
| Daily (3 runs) | ~$0.81 |
| Monthly (90 runs) | ~$24.30 |

**Updated total mechanism overhead per run:** ~$0.013 (survey) + ~$0.27 (gate) = **~$0.28 per run**. This is 0.2–0.3% of total run cost.

**Budget check:** $5.00 constraint applies to coverage mechanism additions. $0.28 overhead satisfies the constraint with $4.72 headroom.

---

## Domain generality

The region-aware gate inherits the domain generality of the survey that feeds it:

- Cluster definitions are derived from the problem-specific taxonomy (S0) — no domain-specific configuration required for the gate
- The G1 haiku call is domain-agnostic: it receives cluster signatures (problem-derived) and candidate text, and performs structural comparison — identical operation for writing, strategy, investment, product, and technical problems
- The three cluster status classes (UNEXPLORED / ACTIVE / EXHAUSTED) are domain-independent — they describe thread activity, not domain structure
- The threshold table (0.65 / 0.80 / 0.90) is domain-independent

---

## What changed this iteration (v2 → v3)

**Element changed:** G1 cluster assignment only. One change. Survey design remains identical to v1/v2 except for a minor addition to the S4 output format (STRUCTURAL_SIGNATURE per cluster — see below). Gate thresholds (G2, G3) unchanged from v2. All v11 harness mechanisms preserved.

---

### The change — Concrete G1 cluster assignment via haiku-bedrock call

**v2 state:** G1 claimed "zero model calls" — assigning free-form candidate text to structural clusters by "reusing the S2 mechanism" and performing a "table lookup." This was underspecified. The S2 mechanism is a batched haiku call over known sketches with pre-labelled claim types — not a mechanism applicable to unlabelled free-form candidate text arriving during iteration. The table lookup (CLAIM_TYPE_ID → CLUSTER_ID) presupposes the candidate has already been assigned a CLAIM_TYPE_ID, which requires either a model call or shallow heuristics. Three judges identified this as the load-bearing joint of the entire gate mechanism. A gate adjusting thresholds based on unreliable cluster assignments is worse than a uniform gate.

**v3 change:** G1 now uses a **single haiku-bedrock call per candidate**. The call receives:
1. The candidate text
2. The 3 cluster structural signatures from `survey_result` — each a 3-element domain vocabulary fingerprint produced by the S4 call
3. An instruction: assign this candidate to the cluster whose structural argument most closely resembles its own, return CLUSTER_ID (1/2/3) + one-sentence structural rationale + BOUNDARY_CASE flag

**Why Option A (haiku call) over Option B (fingerprint majority-vote):**

Option B (compute [dim1, dim2, dim3] fingerprint for candidate, majority-vote against cluster fingerprints) fails on the cases that matter most:

- **Boundary candidates:** A candidate equidistant between two clusters scores 1-1-1 across dimensions — a three-way tie producing arbitrary assignment. This is precisely the case where the gate's threshold selection matters most, and where Option B delivers noise.
- **Adaptive-axis candidates:** Candidates from adaptive axes not present in the original S0 survey use vocabulary the Phase 0 taxonomy never encountered. String matching returns no confident signal, defaulting to arbitrary assignment.
- **Majority vote failure rate:** In boundary cases, majority voting on 3 dimensions has approximately 50% error rate — a gate that is wrong half the time on the hardest cases provides less signal than a uniform gate.

Option A is reliable by construction across all cases:
- **Boundary cases:** The haiku model reads the actual structural argument and reasons about cluster fit, including articulating a rationale. The BOUNDARY_CASE flag makes ambiguity explicit rather than hiding it in a tie-breaking coin flip. Boundary cases are resolved toward exploration — the less-explored cluster wins — which is the correct direction for the gate's purpose.
- **Novel vocabulary:** The haiku model can interpret structural arguments in new vocabulary because it is reasoning about logical move type, not matching strings. Adaptive-axis outputs are handled correctly.
- **Consistency:** The same model class that generated the cluster signatures classifies candidates against those signatures. Structural reasoning is applied end-to-end.

The $0.27 additional cost per 90-iteration run is noise against the base cost of ~$91–146.

**Additional change to S4 output format:** The S4 clustering prompt now requests `STRUCTURAL_SIGNATURE: [dim1_label, dim2_label, dim3_label]` for each cluster alongside the existing STRUCTURAL_LABEL. Generated in the same S4 haiku call at no additional cost. Stored in `survey_result.clusters[n].structural_signature` and consumed exclusively by G1.

**What the v3 gate now guarantees:**
- Every candidate receives a cluster assignment from the same model class that generated the cluster taxonomy — structural consistency throughout the run
- Boundary cases are flagged explicitly and resolved toward exploration rather than handled arbitrarily
- The `RATIONALE` field creates an auditable per-candidate structural assignment trail — the first explainability record in the harness
- Threshold adjustments (0.65 / 0.80 / 0.90) are grounded in reliable cluster assignments — the gate's economic signal is trustworthy

**What this does NOT change:**
- Survey design (S0–S5 logic unchanged; S4 output gains `structural_signature` field only)
- Gate thresholds (G2, G3 unchanged from v2)
- Escape mechanisms (Steps 1–6d — unchanged from v11)
- Vocabulary derivation (Step V — unchanged from v11)
- Fingerprint monitoring (Steps IC1–IC5 — unchanged from v11)
- Tournament structure (Steps SD, T1–T5 — unchanged from v11)
- Thread parallelism and tournament interval (unchanged from v11)
- Oracle (frozen, shared — unchanged)
- Termination condition (unchanged)
- Fallback behaviour (reverts to 0.80 uniform if survey_result absent)

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
*Loop dir: ~/clawd/landscape-coverage-autoresearch/*
