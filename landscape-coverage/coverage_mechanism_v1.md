# Coverage Mechanism — v1 (Structural Survey Seeding)
*Landscape coverage mechanism — v1*
*260314*

---

## Survey design (CHANGED — this is the only change in v1)

### Overview

A pre-run structural landscape survey fires once at run start (after Step V vocabulary derivation, before thread spawning). The survey generates 15–20 brief candidate sketches across structurally diverse regions of the solution space, clusters them into 3 guaranteed non-overlapping regions, and seeds T1, T2, T3 from those regions — not from abstract axis labels.

**Core design principle:** Structural diversity is defined at the level of claim type, not surface phrasing. Two sketches that use different words but make the same type of logical move are in the same structural region. Clustering is at claim-structure level. The survey includes a three-layer structural verification mechanism to guarantee that the 3 seeded regions are non-overlapping by construction.

All steps use haiku-bedrock (fast, cheap). Survey total cost: ~$0.013, well within the $5 constraint.

---

### Step S0 — Structural approach taxonomy derivation

**Fires:** Once, immediately after Step V (vocabulary derivation), before thread spawning.

**Purpose:** Derive a structural taxonomy of 12–18 claim types for this specific problem. A claim type is defined by HOW it argues, not what it argues about. "Technology substitution claim" and "market-structure dislocation claim" are different claim types even if both argue that a company will win. The taxonomy is the foundation of all downstream steps: it defines the regions before any sketches are generated.

**Model:** haiku-bedrock

**Prompt:**

> You are a structural analyst. Your task is to enumerate the distinct structural CLAIM TYPES that could be used to answer the following question — not the specific answers, but the different structural approaches to constructing an answer.
>
> A structural claim type is defined by the TYPE OF LOGICAL MOVE used to reach an answer, not the specific content. Two approaches that use different words but make the same type of argument are the SAME claim type. Two approaches that use similar words but argue from structurally different bases are DIFFERENT claim types.
>
> Examples of structural distinction:
> - "X is better because its technology is superior" — claim type: TECHNOLOGY_SUPERIORITY (argues from technical comparison)
> - "X will win because users face high switching costs" — claim type: SWITCHING_COST_MOAT (argues from path-dependence and lock-in)
> - "X is undervalued because the market is mispricing the growth trajectory" — claim type: MARKET_MISPRICING (argues from expected value error)
> These are structurally distinct even if all three conclude "X wins."
>
> Problem statement: {PROBLEM_STATEMENT}
>
> Generate 12–18 structural claim types covering the realistic solution space. For each, provide:
> — CLAIM_TYPE_ID: a short identifier in CAPS_SNAKE_CASE (e.g. TECHNOLOGY_SUPERIORITY)
> — STRUCTURE: one sentence defining the type of logical move this claim type makes
> — CANNOT_BE_MERGED_WITH: name 1–2 other claim types that look similar but are structurally distinct, and state in one clause WHY they are distinct (what logical move differs)
>
> Requirements:
> — Each claim type must be structurally distinct (different logical move, not just different vocabulary)
> — Together they should cover the plausible solution space (no obvious structural approach missing)
> — Order them so that structurally similar types appear adjacent (makes later clustering easier)
> — Do not produce generic types that apply to any reasoning task; every type must be specific to THIS problem

**Cost:** ~1,000 tokens ≈ $0.001 at haiku-bedrock rates.

---

### Step S1 — Sketch generation

**Fires:** Immediately after S0 completes.

**Purpose:** Generate one 3-sentence directional sketch per claim type. Sketches are not full answers — they are structural bets: "if we approach this problem via claim type X, the answer would look like Y." Each sketch must commit to its assigned claim type; hedged or blended sketches are structurally useless.

**Model:** haiku-bedrock (batched — all 12–18 sketches in a single prompt)

**Prompt:**

> You will generate brief candidate sketches for the following problem: {PROBLEM_STATEMENT}
>
> Below is a list of structural claim types (each defines a different way of constructing an answer). For EACH claim type, write a 3-sentence sketch:
> — Sentence 1: The core claim (what the answer says — commit to a specific position)
> — Sentence 2: The key mechanism (why the claim is true — state the structural argument explicitly)
> — Sentence 3: The expected shape of a strong answer (what evidence or reasoning would make this compelling)
>
> CRITICAL: Each sketch must commit to the claim type it was generated for. Do not reframe, hedge, or blend claim types — write as if this approach is the correct one and you are pointing toward its best version.
>
> {CLAIM_TYPES_LIST}
>
> Output format: one sketch per claim type, clearly labelled with CLAIM_TYPE_ID.

**Cost:** ~4,000–5,000 tokens (prompt + 18 sketches × ~120 tokens each) ≈ $0.004–0.005.

---

### Step S2 — Structural verification (Layer 2 guarantee)

**Fires:** Immediately after S1.

**Purpose:** Verify that each sketch actually commits to its assigned claim type — not just lexically, but structurally. This catches the most common failure mode: haiku generating a sketch that uses the correct vocabulary but makes a structurally different argument. A sketch labelled MARKET_MISPRICING that argues "investors underestimate switching costs" has drifted — it is making a SWITCHING_COST_MOAT argument with mispricing language. The structural verification step catches and corrects this.

**Model:** haiku-bedrock (batched — all sketches in a single prompt)

**Prompt:**

> You are a structural auditor. Below are a set of brief sketches, each labelled with the structural claim type it was generated for. Your task: for each sketch, determine whether the sketch is ACTUALLY making the structural argument defined by its claim type, or whether it has drifted and is making a different type of argument.
>
> A sketch "drifts" when it uses the vocabulary of its assigned claim type but the underlying logical move belongs to a different claim type. The test: if you removed all claim-type-specific vocabulary from the sketch, which claim type would the underlying argument still resemble?
>
> Claim type definitions: {CLAIM_TYPES_LIST}
>
> Sketches to audit: {SKETCHES_LIST}
>
> For each sketch, output:
> — CLAIM_TYPE_ID: the original label
> — STRUCTURAL_MATCH: YES or NO
> — IF NO — ACTUAL_TYPE: which claim type does this sketch actually instantiate?
> — IF NO — REASON: one sentence describing the structural drift (what logical move does the sketch actually make?)

**Processing after audit:**
- Sketches with STRUCTURAL_MATCH = YES are accepted as verified.
- Sketches with STRUCTURAL_MATCH = NO are reassigned to their ACTUAL_TYPE.
- If reassignment causes two verified sketches to share the same claim type: score both in S3 and discard the lower-scoring one after scoring.
- If a claim type ends up with zero verified sketches: it is retained in the taxonomy (for clustering purposes) with a null sketch, but does not participate in S5 seed selection. The corresponding cluster's seed will be drawn from the next-best sketch in that cluster.

**Cost:** ~2,000–3,000 tokens ≈ $0.002–0.003.

---

### Step S3 — 2-dimension scoring

**Fires:** Immediately after S2, on all verified sketches.

**Purpose:** Score each verified sketch on two cheap proxy dimensions to identify the most promising direction within each cluster. These are NOT full oracle dimensions — they do not assess answer quality. They assess whether the approach is alive and direction-giving.

**Model:** haiku-bedrock (batched — all verified sketches in a single prompt)

**Dimensions:**
1. **Plausibility** (1–5): Would a domain expert judge this approach likely to produce a substantive, defensible answer? (1 = implausible or structurally incoherent; 5 = clearly promising direction)
2. **Resolution potential** (1–5): Does this sketch point toward a specific, concrete answer shape — or does it remain directionally vague? (1 = vague, could go anywhere; 5 = the shape of a strong answer is already visible from here)

**Prompt:**

> Score each of the following sketches on exactly two dimensions. Output only numbers — do not explain your reasoning.
>
> PLAUSIBILITY (1–5): Would a domain expert think this approach has genuine potential to produce a substantive, defensible answer to the problem? 1 = implausible or incoherent; 5 = clearly promising direction.
>
> RESOLUTION POTENTIAL (1–5): Does this sketch point toward a specific, concrete answer — or is it too vague to guide iteration? 1 = vague, could go anywhere; 5 = the shape of a strong answer is already visible.
>
> Do NOT score answer quality. Score only whether the approach is alive (plausibility) and direction-giving (resolution potential).
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

**Purpose:** Group the 12–18 verified claim types into exactly 3 structurally non-overlapping clusters. Each cluster becomes one thread's starting region. The clustering principle: two claim types belong in the same cluster if they make the SAME FUNDAMENTAL TYPE OF BET — they are structural variations of the same underlying logical move. Two claim types belong in different clusters if they make genuinely different bets that cannot be reduced to each other.

**This is NOT similarity clustering** (which would group by surface similarity). This is structural reduction: "are these two claim types ultimately instances of the same class of argument?"

**Model:** haiku-bedrock (1 call)

**Prompt:**

> You have a list of structural claim types for this problem: {PROBLEM_STATEMENT}
>
> Claim type definitions: {CLAIM_TYPES_LIST}
>
> Your task: group these claim types into exactly 3 clusters such that:
> 1. Within each cluster: all claim types make the same FUNDAMENTAL TYPE OF BET — they are structural variations of each other (different instances of the same underlying logical move, not just similar-sounding arguments)
> 2. Across clusters: each cluster makes a DIFFERENT TYPE OF BET that cannot be reduced to either other cluster
> 3. The 3 clusters together cover the full set of verified claim types (no orphans)
>
> After grouping, produce a STRUCTURAL NON-OVERLAP AUDIT:
> — For each pair of clusters (C1 vs C2, C1 vs C3, C2 vs C3), state in one sentence WHY those two clusters cannot be merged — what is the irreducible structural distinction between them?
> — If you cannot state a clear, specific structural distinction for any pair, that pair MUST be merged and the remaining claim types re-split into a new third cluster.
>
> Output format (strictly):
> CLUSTER_1: {comma-separated CLAIM_TYPE_IDs} | STRUCTURAL_LABEL: {one sentence: what type of bet does this cluster make?}
> CLUSTER_2: {comma-separated CLAIM_TYPE_IDs} | STRUCTURAL_LABEL: {one sentence}
> CLUSTER_3: {comma-separated CLAIM_TYPE_IDs} | STRUCTURAL_LABEL: {one sentence}
> NON_OVERLAP_AUDIT:
> C1 vs C2: {why they cannot be merged — one sentence with specific structural distinction}
> C1 vs C3: {why they cannot be merged — one sentence with specific structural distinction}
> C2 vs C3: {why they cannot be merged — one sentence with specific structural distinction}

**Validation (no LLM call):** The harness checks:
- Every CLAIM_TYPE_ID from S0 appears in exactly one cluster
- Each cluster contains at least 1 verified sketch (with a non-null sketch from S2)
- The NON_OVERLAP_AUDIT contains a non-empty, non-generic reason for each pair (rejected if any audit reason is "they are different" or similar content-free phrase — minimum 8 words required)

**On validation failure:** Up to 2 retries with the same prompt. If retries fail, fall back to v0 axis-label seeding and log `survey_fallback: true` in global state.

**Cost:** ~1,500–2,000 tokens ≈ $0.001–0.002.

---

### Step S5 — Thread seed selection

**Fires:** Immediately after S4 validation passes.

**Purpose:** Select the best sketch from each cluster to seed the corresponding thread.

**Model:** No LLM call needed.

**Selection rule:** For each cluster, compute `seed_score = plausibility × resolution_potential` (from S3 scores) for all sketches assigned to that cluster. Select the sketch with the highest seed_score as the cluster's seed. On tie: prefer higher plausibility (directional credibility matters more than concreteness for seeding). On tie again: prefer the sketch from the claim type earliest in the S0 output ordering.

**Thread seeding:** The selected sketch replaces the abstract axis label used in v0. Each thread receives:

> "You are Thread {N}. Your starting region is the {CLUSTER_N_STRUCTURAL_LABEL} region of the solution space. Begin from this directional sketch:
>
> {SELECTED_SKETCH}
>
> Your task is to iterate and develop this direction — not to explore structurally different regions (those are assigned to other threads). Stay within your structural region; do not drift toward the other clusters' structural bets."

This seeding instruction is injected into each thread's initialisation context alongside the frozen fingerprint vocabulary from Step V.

**Landscape map stored in global state:**
```
survey_result: {
  taxonomy: [{claim_type_id, structure, cluster_assignment, sketch, plausibility, resolution_potential}, ...],
  clusters: [
    {cluster_id: 1, structural_label, claim_types: [...], seed_sketch, seed_score},
    {cluster_id: 2, structural_label, claim_types: [...], seed_sketch, seed_score},
    {cluster_id: 3, structural_label, claim_types: [...], seed_sketch, seed_score}
  ],
  non_overlap_audit: {c1_vs_c2: "...", c1_vs_c3: "...", c2_vs_c3: "..."},
  survey_total_cost_usd: <estimate>,
  fallback_used: true/false
}
```

This map is immutable after S5 completes. It is available to all mechanisms throughout the run and is the data structure a future region-aware gate (v2) would read to assess landscape position of new candidates.

---

### Survey cost summary

| Step | Model | Calls | Tokens (est.) | Cost (est.) |
|---|---|---|---|---|
| S0 — Taxonomy derivation | haiku-bedrock | 1 | ~1,000 | ~$0.001 |
| S1 — Sketch generation | haiku-bedrock | 1 (batched) | ~5,000 | ~$0.005 |
| S2 — Structural verification | haiku-bedrock | 1 (batched) | ~2,500 | ~$0.003 |
| S3 — 2-dimension scoring | haiku-bedrock | 1 (batched) | ~2,000 | ~$0.002 |
| S4 — Structural clustering | haiku-bedrock | 1 | ~2,000 | ~$0.002 |
| **Total** | haiku-bedrock | **5 calls** | **~12,500 tokens** | **~$0.013** |

Budget: $5.00 constraint. Actual usage: ~$0.013. Headroom: $4.987.

Haiku-bedrock pricing basis: ~$0.00025/1K input tokens, ~$0.00125/1K output tokens. All estimates use output-heavy pricing (conservative).

---

## Gate mechanism (UNCHANGED from v0)

Uniform threshold: all candidates must score ≥ current-best × 0.80 to be accepted. No adjustment for whether the candidate comes from an explored or unexplored region. The gate is blind to landscape position.

*(The gate mechanism will be addressed in v2 as Priority 2 in the iteration programme.)*

---

## Integration (UPDATED — survey fires before thread spawning)

**Updated initialisation order:**
1. Receive problem statement
2. Classify domain archetype (Step V1 — unchanged from v11)
3. Derive and freeze fingerprint vocabulary (Steps V2–V3 — unchanged from v11)
4. **Run landscape survey (Steps S0–S5 — NEW in v1)**
5. Generate and verify orthogonal axis seeds (unchanged from v11, but now seeded from survey cluster assignments rather than abstract axis labels — the 3 clusters inform the initial axis differentiation)
6. Spawn 3 threads with frozen vocabulary + survey seed sketches injected into each thread's context

**Interaction with existing mechanisms:**
- **Fingerprint monitoring (IC1–IC5):** Unchanged. The survey reduces initial convergence risk; fingerprint monitoring continues to detect re-convergence during the run. The two mechanisms are complementary: survey prevents convergent seeding; fingerprinting detects convergent drift.
- **Tournament / Step SD:** Unchanged. The structural divergence check at tournament time fires as before. Survey reduces the prior probability that SD fires early, but does not replace it.
- **Escape mechanisms (Steps 1–6d):** Unchanged. Within each thread, the escape mechanism operates identically to v11. The survey seed replaces the axis label as the starting point; it does not change escape trigger logic, antipode generation, or regression gate behaviour.
- **Crossover (Step 6):** Unchanged. Crossover draws from historical bests within the thread. The survey seed is just the initial current-best; once the thread runs iterations the seed is superseded.
- **Step V (vocabulary derivation):** Unchanged. Survey fires after Step V completes; the frozen vocabulary is available to the survey if needed (e.g. for claim-type labels to be aligned with vocabulary dimensions) but this alignment is not required for v1.

**Fallback behaviour:** If the survey fails (all retries exhausted at any step), the harness logs `survey_fallback: true` in global state and falls back to v0 axis-label seeding (central frame / causal mechanism / epistemic register). The run proceeds. v1 degrades gracefully to v0 behaviour on survey failure.

---

## Cost estimate (UPDATED)

Base cost unchanged from v11: ~$91–146 per run.

**Survey overhead (NEW in v1):** ~$0.013 per run (5 haiku-bedrock calls, ~12,500 tokens total).

**Updated total:** ~$91–146 (survey adds ~$0.01 — effectively negligible at run scale).

---

## Domain generality

The survey is fully domain-general:
- S0 derives claim types from the problem statement — no domain-specific configuration needed
- S1–S3 operate on those derived claim types — no hardcoded domain assumptions
- S4 clusters by structural bet type — the clustering principle (what fundamental class of argument) applies to writing, strategy, investment, product, and technical problems identically
- S5 selects seeds from survey output — domain-independent

The survey design avoids the domain-specificity trap by grounding "structural region" in the taxonomy derived from the specific problem, not in a fixed universal classification.

---

## What changed this iteration (v0 → v1)

**Element changed:** Survey design only. One change. Gate mechanism remains uniform (unchanged from v0). All v11 harness mechanisms (Steps 1–6d, Step V, Step IC, Step SD, T1–T5, tournament structure) preserved and operate identically.

---

### The change — Structural survey seeding

**v0 state:** No survey. Threads seeded from abstract axis labels chosen by the agent without landscape information: Thread 1 = "central frame / primary claim", Thread 2 = "causal mechanism / evidence type", Thread 3 = "epistemic register / resolution mode". These labels may cluster in the same conceptual region — if the problem has a dominant solution structure, all three labels may point to it. The VC Stress-Tester gave D1 a 7/10 specifically because of this: no structural guarantee of non-overlapping seeds.

**v1 change:** A 5-step pre-run structural landscape survey (Steps S0–S5, haiku-bedrock, ~$0.013) fires once at run start and:
1. **Derives a structural claim-type taxonomy (S0)** — 12–18 claim types defined at claim-structure level, each with an explicit `CANNOT_BE_MERGED_WITH` field that forces the model to articulate distinctions before generating the taxonomy
2. **Generates one sketch per claim type (S1)** — 3-sentence directional bets, one per taxonomy node, committed to the structural approach of that node
3. **Structurally verifies each sketch (S2)** — audits whether sketches actually commit to their assigned claim type (not just lexically) and reassigns drifted sketches to their actual structural type
4. **Scores each sketch on 2 cheap dimensions (S3)** — plausibility and resolution potential (proxies only, not full oracle)
5. **Clusters claim types into 3 structurally non-overlapping regions (S4)** — with a mandatory non-overlap audit requiring a stated structural distinction between each cluster pair
6. **Seeds T1, T2, T3 from those 3 regions (S5)** — concrete directional sketches from the highest-scoring sketch per cluster

### Why this guarantees structural non-overlap (not just encourages it)

v0 offered no guarantee: abstract axis labels are structurally underdetermined. v1 has three layers of structural guarantee:

**Layer 1 — Taxonomy design (S0).** The taxonomy is generated with a `CANNOT_BE_MERGED_WITH` constraint baked into every node. The model must articulate, before generating claim types, what makes each type structurally distinct from similar-looking types. This front-loads the structural differentiation into the taxonomy itself — before any sketch is generated, the regions are defined at the structural level.

**Layer 2 — Structural verification (S2).** Every sketch is audited against its assigned claim type. A sketch that uses the right vocabulary but makes a different structural argument is detected and reassigned. This catches the most common failure mode: lexical surface compliance with structural drift underneath. A sketch reassigned to a different claim type during S2 is structurally honest — it belongs where the audit says it belongs, not where it was generated.

**Layer 3 — Non-overlap audit (S4).** The clustering step requires the model to provide an explicit, non-empty, specific structural reason why each pair of clusters cannot be merged. This is not a soft encouragement — if the model cannot articulate a clear distinction, it must merge those clusters and split the remainder. The harness validates that all three audit reasons are non-empty and non-trivial (≥8 words, rejecting content-free phrases). A clustering that passes validation is one where the model has committed, in writing, to the structural reason each pair is distinct.

Together, these three layers provide a constructive guarantee: the seeded regions are non-overlapping by structural definition (Layer 1), by verified sketch commitment (Layer 2), and by explicit pairwise audit (Layer 3). The guarantee is not probabilistic encouragement — it is a structural definition enforced at generation time, at sketch level, and at clustering time.

### What this does NOT change:
- Gate mechanism (uniform 0.80 threshold — unchanged from v0/v11)
- All escape mechanisms (Steps 1–6d — unchanged from v11)
- Vocabulary derivation (Step V — unchanged from v11)
- Fingerprint monitoring (Steps IC1–IC5 — unchanged from v11)
- Tournament structure (Steps SD, T1–T5 — unchanged from v11)
- Thread parallelism and tournament interval (unchanged from v11)
- Oracle (frozen, shared — unchanged)
- Termination condition (unchanged)

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
*Loop dir: ~/clawd/landscape-coverage-autoresearch/*
