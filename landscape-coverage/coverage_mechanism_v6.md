# Coverage Mechanism — v6 (Contrastive Constraint for ACTIVE Threads)
*Landscape coverage mechanism — v6*
*260314*

---

## Survey design (UNCHANGED from v5)

### Overview

A pre-run structural landscape survey fires once at run start (after Step V vocabulary derivation, before thread spawning). The survey generates 4 structurally extreme candidate claim types via an adversarial construction process, then generates one sketch per type, clusters them into 3 guaranteed non-overlapping regions, and seeds T1, T2, T3 from those regions.

**Core design principle (UNCHANGED from v5):** Structural diversity is constructed through conflict. The S0 adversarial process generates 4 initial claim types, then fires an adversary that identifies pairs which could produce structurally similar candidates. Flagged types are replaced with more extreme versions that make overlap impossible. Diversity is a product of adversarial pressure, not cooperative prompting.

**What changes from v5:** The ED step for ACTIVE threads is replaced — the soft nudge (advisory text) is replaced with an enforceable contrastive constraint that G1 can verify post-generation. S0a/S0b/S0c, S1–S5, G2, and G3 are all unchanged from v5. G1 gains one new sub-step (G1c) that fires only for ACTIVE-thread candidates. G3 gains one new handling rule for SAME_CLUSTER-flagged candidates.

All steps use haiku-bedrock. Survey total cost: ~$0.015, well within the $5 constraint.

---

### Step S0a — Seed taxonomy generation

**UNCHANGED from v5.**

---

### Step S0b — Adversarial overlap challenge

**UNCHANGED from v5.**

---

### Step S0c — Adversarially-informed revision

**UNCHANGED from v5.**

---

### Final taxonomy commit

**UNCHANGED from v5.**

---

### Step S1 — Sketch generation

**UNCHANGED from v5.**

---

### Step S2 — Structural verification (Layer 2 guarantee)

**UNCHANGED from v5.**

---

### Step S3 — 2-dimension scoring

**UNCHANGED from v5.**

---

### Step S4 — Structural clustering into 3 regions (Layer 3 guarantee)

**UNCHANGED from v5.**

---

### Step S5 — Thread seed selection

**UNCHANGED from v5.** `survey_result` stores STRUCTURAL_SIGNATURE, STRUCTURAL_SEED, and STRUCTURAL_DIMENSIONS (the three named axis values for each cluster) per cluster.

**New in v6:** S5 additionally stores `STRUCTURAL_DIMENSIONS` per cluster — a 3-tuple of named axis values drawn from the same vocabulary axes defined in Step V (central_frame, causal_mechanism, epistemic_register). These are the specific current values of the cluster on each structural axis; they are used in v6's contrastive constraint injection.

**Example (domain-agnostic):**
```
cluster[1].STRUCTURAL_DIMENSIONS = {
  central_frame: "comparative_counterfactual",
  causal_mechanism: "feedback_loop",
  epistemic_register: "empirical_predictive"
}
```

These values are extracted from STRUCTURAL_SIGNATURE (already produced by S4) via a single additional parse at S5 time — the STRUCTURAL_SIGNATURE already encodes these three axes; S5 now makes them explicit as a structured triple for downstream use in the contrastive constraint. No additional model call required for this extraction (deterministic parse of S4 output).

---

### Survey cost summary

**UPDATED from v5:** ~$0.009 per run (unchanged base) + negligible additional parse at S5 (no model call). Total survey cost unchanged.

---

## Gate mechanism (UPDATED — ACTIVE branch of ED replaced; G1c added; G3 SAME_CLUSTER rule added)

### Overview

The uniform 0.80 × current-best threshold is replaced with a region-aware diversity gate. The gate reads `survey_result` cluster signatures and assigns each incoming candidate to its nearest cluster via a single haiku-bedrock call per candidate (G1). The acceptance threshold adjusts based on that cluster's exploration status (G2, G3).

**New in v6:** The ACTIVE branch of Step ED (previously a soft nudge) is replaced with an enforceable contrastive constraint. G1 gains a new sub-step (G1c) that verifies contrastive compliance for ACTIVE-thread candidates. G3 gains a SAME_CLUSTER handling rule for candidates that fail contrastive compliance after one regeneration.

**The mechanism remains push + pull.** G3 creates pull — underexplored regions have lower acceptance thresholds. ED (UNEXPLORED and EXHAUSTED branches) creates push — it steers generation. The new ACTIVE contrastive constraint creates verifiable push: not "consider exploring elsewhere" but "your candidate must structurally differ from your current cluster on at least 2 of 3 named dimensions — and the harness will check."

---

### Step ED — Exploration directive injection (ACTIVE BRANCH CHANGED in v6)

**Fires:** At the start of each thread iteration, before the thread generates a candidate. Uses only `survey_result` global state. No model call (directive injection and dimension lookup are both template fills from stored state).

**Purpose:** Actively steer the thread toward unexplored structural territory. The gate adjusts what gets accepted; the directive adjusts what gets generated. In v5, the ACTIVE branch used advisory text that the iteration agent could absorb without acting on. In v6, the ACTIVE branch injects a contrastive constraint with named dimensional requirements that G1 can verify — making ACTIVE-thread exploration enforceable rather than advisory.

**Inputs (from `survey_result`):**
- `thread.assigned_cluster_id` — the cluster this thread was seeded from
- `cluster[id].status` — UNEXPLORED / ACTIVE / EXHAUSTED for each cluster
- `cluster[id].candidate_count` — number of accepted candidates so far
- `cluster[id].STRUCTURAL_SIGNATURE` — short descriptor (stored from S4; used in G1)
- `cluster[id].STRUCTURAL_SEED` — full 3-sentence sketch (stored from S5; used in EXHAUSTED redirect)
- **[NEW] `cluster[id].STRUCTURAL_DIMENSIONS`** — named 3-tuple: {central_frame, causal_mechanism, epistemic_register} (extracted at S5; used in ACTIVE contrastive constraint)

**Least-explored cluster resolution (deterministic, UNCHANGED from v5):**

```
least_explored = first cluster where status == UNEXPLORED (by cluster id order)
if no UNEXPLORED cluster:
  least_explored = ACTIVE cluster with lowest candidate_count
  (tie-break: lower cluster id wins)
if least_explored == thread.assigned_cluster_id:
  least_explored = next candidate in the same ordering
```

No model call. Pure state read.

**Directive logic by cluster status:**

| Thread's assigned cluster status | Directive type | What is injected |
|---|---|---|
| UNEXPLORED | None | Thread prompt is unmodified |
| ACTIVE | **Contrastive constraint** (CHANGED in v6) | Named dimensional requirements; G1 verifies compliance post-generation |
| EXHAUSTED | Hard redirect (UNCHANGED from v5) | Full structural seed of least-explored cluster replaces prompt context |

---

**UNEXPLORED — no directive:**

**UNCHANGED from v5.** The thread has just been seeded into a fresh region. No modification to its generation prompt. It is already pointing at good territory.

---

**ACTIVE — contrastive constraint (CHANGED in v6):**

**v5 behaviour (replaced):** Appended advisory text naming the least-explored cluster's STRUCTURAL_SIGNATURE. The thread could continue in its current direction; the nudge competed with generation inertia and provided no verifiable requirement.

**v6 behaviour:** Inject a contrastive constraint that names the current cluster's specific dimensional values and requires the generated candidate to differ on at least 2 of 3.

Append the following block to the thread's current generation prompt, after its existing context:

> **Structural divergence required.**
>
> Your current cluster occupies this structural position:
> - central_frame: [{cluster[thread.assigned_cluster_id].STRUCTURAL_DIMENSIONS.central_frame}]
> - causal_mechanism: [{cluster[thread.assigned_cluster_id].STRUCTURAL_DIMENSIONS.causal_mechanism}]
> - epistemic_register: [{cluster[thread.assigned_cluster_id].STRUCTURAL_DIMENSIONS.epistemic_register}]
>
> Generate a candidate that uses **different values on at least 2 of these 3 dimensions**. Your candidate must not simply restate or vary your current cluster's framing — it must make a genuine structural move away from the position above on at least 2 of the 3 named axes.
>
> Specifically:
> - Do not use central_frame = [{central_frame_current}] AND causal_mechanism = [{causal_mechanism_current}]
> - Do not use central_frame = [{central_frame_current}] AND epistemic_register = [{epistemic_register_current}]
> - Do not use causal_mechanism = [{causal_mechanism_current}] AND epistemic_register = [{epistemic_register_current}]
>
> Any one of the above three combinations would fail the structural divergence requirement. You must avoid all three.

**Why named dimensions, not "be different":** A constraint that says "generate something different" gives the iteration agent no specific target and no verifiable failure criterion. Named axis values with explicit forbidden combinations are specific enough to be violated or satisfied — and specific enough for G1 to check via the same fingerprint mechanism used throughout. The iteration agent cannot satisfy the constraint by adding the word "novel" to a familiar framing; G1's structural fingerprint check will catch similarity on the named dimensions.

**Why 2 of 3, not all 3:** Requiring divergence on all three dimensions would overconstrain the generation space, potentially producing incoherent candidates or candidates that cannot be scored meaningfully. Divergence on 2 of 3 allows the iteration agent to anchor on one familiar dimension while genuinely exploring on the other two — preserving coherence while forcing structural movement.

**Why forbidden combinations (explicit negative), not positive targets:** The contrastive constraint specifies what the candidate must NOT be (via three forbidden 2-dimension combinations), not a specific target to aim at. This is intentional: positive targets would create a pull toward the least-explored cluster (as the EXHAUSTED hard redirect already does). The ACTIVE contrastive constraint's purpose is divergence from the current cluster, not convergence on a specific alternative. The iteration agent has freedom to choose where to go, as long as it moves away from two of the three current axes.

**State required:** `cluster[thread.assigned_cluster_id].STRUCTURAL_DIMENSIONS` — a structured triple already extracted at S5. No model call. Template fill from stored state. O(1) per iteration.

---

**EXHAUSTED — hard redirect (UNCHANGED from v5):**

Replace the thread's structural context entirely. Prepend the following block BEFORE any existing generation instructions in the thread's prompt:

> You are being redirected to a new structural territory.
>
> Your current region has been thoroughly explored. Do not generate variations of previous candidates — that territory is exhausted.
>
> Generate a candidate that explores THIS structural territory:
>
> [{STRUCTURAL_SEED of least-explored cluster — full 3-sentence sketch from S5}]
>
> Your candidate must make the structural move described above. Aim at this territory, not your previous one.

**UNCHANGED from v5.** The hard redirect remains appropriate for EXHAUSTED threads because the primary failure mode is generation inertia in an exhausted region, and a concrete seed provides a target. The ACTIVE contrastive constraint is a different intervention — it prevents premature narrowing without redirecting the thread entirely.

---

**Prompt injection mechanics (UNCHANGED from v5):**

The directive is injected at the harness level, not by the thread itself. The harness reads `survey_result` state, computes the directive, and assembles the final prompt before passing it to the iteration agent. The iteration agent receives one prompt per iteration — it does not observe that a directive has been prepended. From the iteration agent's perspective, it always receives a single generation prompt.

The contrastive constraint is a template fill from `STRUCTURAL_DIMENSIONS` stored in `survey_result`. No model call. O(1) per iteration.

---

**State tracking for ED (UPDATED):**

`survey_result` now additionally stores `STRUCTURAL_DIMENSIONS` per cluster (extracted at S5 — no new model call, deterministic parse of S4 STRUCTURAL_SIGNATURE output). ED reads this state — it does not write to it. One new field per cluster added to `survey_result`.

**New state written by G1c (below):** `thread.last_candidate_contrastive_status` — PASS / FAIL. Written by G1c after each ACTIVE-thread candidate verification. Read by the gate execution loop to decide whether to trigger one regeneration. Not persisted beyond the current iteration.

---

**Cost (ED):** Zero for UNEXPLORED and EXHAUSTED branches (unchanged from v5). Zero for ACTIVE contrastive constraint injection itself (template fill). The additional cost in v6 is G1c — one haiku-bedrock contrastive compliance check per ACTIVE candidate. See G1c below.

---

### Step G1 — Candidate cluster assignment (G1c SUB-STEP ADDED in v6)

**G1a (UNCHANGED from v5):** Single haiku-bedrock call per candidate. Returns CLUSTER_ID, RATIONALE, BOUNDARY_CASE.

**G1b (UNCHANGED from v5):** BOUNDARY_CASE tie-break — assign to less-explored cluster.

**G1c (NEW in v6) — Contrastive compliance verification:**

Fires only when the thread's assigned cluster status is ACTIVE (i.e., an ACTIVE-branch ED constraint was injected). Does not fire for UNEXPLORED or EXHAUSTED threads.

**Input:**
- The generated candidate's structural fingerprint (produced as part of G1a — the same fingerprint used throughout IC1–IC5 monitoring; no additional extraction required)
- `cluster[thread.assigned_cluster_id].STRUCTURAL_DIMENSIONS` — the named 3-tuple for the thread's current cluster

**Single haiku-bedrock call. Prompt:**

> Given the following structural fingerprint of a generated candidate:
> [{candidate structural fingerprint from G1a}]
>
> And the following structural dimensions of the candidate's current cluster:
> - central_frame: [{central_frame_current}]
> - causal_mechanism: [{causal_mechanism_current}]
> - epistemic_register: [{epistemic_register_current}]
>
> Does the candidate's structural fingerprint differ from the cluster's values on at least 2 of these 3 dimensions?
> A dimension "differs" if the candidate uses a structurally distinct value — not a synonym or minor restatement of the current cluster value.
>
> Return JSON: {"contrastive_pass": true/false, "dimensions_that_differ": ["central_frame"|"causal_mechanism"|"epistemic_register"], "rationale": "..."}

**Cost:** ~$0.001 per call (haiku-bedrock). Fires only for ACTIVE-thread candidates.

**Returns:**
- `contrastive_pass`: true / false
- `dimensions_that_differ`: list of which dimensions show genuine divergence
- `rationale`: brief justification (used in logging only; not fed back to iteration agent)

**Compliance handling:**

| G1c result | Action |
|---|---|
| `contrastive_pass: true` | Proceed to G2/G3 normally; candidate is NOT marked SAME_CLUSTER |
| `contrastive_pass: false` (first attempt) | Trigger one regeneration: same ACTIVE contrastive constraint prompt, same thread context. G1c fires again on the regenerated candidate. |
| `contrastive_pass: false` (second attempt) | Accept candidate as-is; mark `SAME_CLUSTER = true`. Do NOT discard or block — blocking risks stalling threads. |

**Why one regeneration, not unlimited:** Unlimited regeneration could stall a thread in an infinite loop if the iteration agent consistently fails the constraint (e.g., the problem domain has limited structural variation available on the named axes). One regeneration gives the iteration agent a second chance — the same constraint text, the same dimensional requirements — before falling back gracefully. This is a soft enforcement mechanism, not a hard gate.

**Why SAME_CLUSTER is a flag, not a discard:** Discarding SAME_CLUSTER candidates would create a dead zone where ACTIVE threads can neither satisfy the constraint nor produce accepted candidates — the iteration would stall. The SAME_CLUSTER flag routes the candidate to G3's ACTIVE threshold (0.80) rather than the UNEXPLORED threshold (0.65). This is the correct penalty: the candidate gets no exploration credit but is still eligible to be accepted if it is sufficiently strong.

**State written:** `thread.last_candidate_contrastive_status` ← PASS / FAIL. Ephemeral (current iteration only). Not persisted in `survey_result`.

---

### Step G2 — Cluster status classification

**UNCHANGED from v5.** Deterministic state lookup. Returns UNEXPLORED / ACTIVE / EXHAUSTED.

---

### Step G3 — Threshold computation and gate decision (SAME_CLUSTER RULE ADDED in v6)

**UNCHANGED from v5** for the three base threshold tiers:

| Cluster status | Acceptance threshold |
|---|---|
| UNEXPLORED | **0.65 × current-best** |
| ACTIVE | **0.80 × current-best** |
| EXHAUSTED | **0.90 × current-best** |

**New in v6 — SAME_CLUSTER handling:**

If a candidate is flagged `SAME_CLUSTER = true` (failed G1c twice), it receives the **ACTIVE threshold (0.80 × current-best)** regardless of G2 cluster status.

This rule exists because SAME_CLUSTER candidates are structurally indistinguishable from the current cluster — they failed a verifiable contrastive check twice. They should not receive the lower UNEXPLORED threshold (0.65), which is reserved for candidates that have genuinely moved into underexplored territory. Applying the ACTIVE threshold preserves the incentive structure: divergent candidates are rewarded; non-divergent candidates are not penalised beyond the base ACTIVE rate.

**SAME_CLUSTER does not apply the EXHAUSTED threshold (0.90).** A SAME_CLUSTER candidate is a candidate that tried (and failed) to diverge — it is not the same as a candidate generated in an already-exhausted region. The penalty is withholding the exploration discount, not imposing additional penalty.

---

### Anti-gaming property (UPDATED from v5)

**v5:** Adversarially-hardened cluster signatures resist surface-diverse but structurally-similar candidates.

**v6 addition:** The contrastive constraint targets the three specific structural axes (central_frame, causal_mechanism, epistemic_register) that the fingerprint mechanism already tracks. An iteration agent cannot satisfy the constraint by adding surface novelty — the G1c compliance check operates on the same structural fingerprint used throughout IC1–IC5 monitoring. A candidate must exhibit genuine dimensional divergence on at least 2 of 3 named axes to pass. Surface restating with new vocabulary will fail G1c.

The cost of gaming: the iteration agent would need to produce a structurally divergent candidate (which is exactly what the constraint is asking for). If it does so successfully, the mechanism has succeeded. If it fails twice, the SAME_CLUSTER flag applies. In neither case can the iteration agent pocket the exploration reward without structural movement.

---

### Convergence safety (UPDATED from v5)

**UNCHANGED from v5** base safeguards: ACTIVE floor, exhausted ceiling, no infinite loop, unassigned fallback.

**v6 addition:** One regeneration cap in G1c compliance handling prevents the constraint from stalling a thread. After two failures, the candidate is accepted (with SAME_CLUSTER penalty) and the thread proceeds. No thread can be blocked by the contrastive constraint.

**Additional v6 safety property:** SAME_CLUSTER candidates are not discarded — they receive the ACTIVE threshold (0.80), the same bar they would have faced before v6. The mechanism cannot produce a worse outcome than v5 for any individual candidate: the SAME_CLUSTER path is exactly the v5 outcome (ACTIVE threshold, no exploration credit). The contrastive constraint is an upside mechanism — it creates a path to better outcomes (exploration credit, lower threshold) by verifying genuine divergence — not a downside risk mechanism.

**ED does not fire on a thread's first iteration (UNCHANGED from v5).** Thread initialisation already uses the cluster seed as the generation prompt (from S5). ED only activates from iteration 2 onward, once cluster status has begun to differentiate.

---

## Integration (UPDATED — G1c and SAME_CLUSTER rule added; gate execution order updated)

**Survey execution order (UNCHANGED from v5):**
1. Step V (vocabulary derivation) fires — unchanged from v11
2. **S0a** — Seed taxonomy: 4 initial claim types via haiku call
3. **S0b (Round 1)** — Adversarial challenge: 6-pair overlap assessment via haiku call
4. **S0c (Round 1)** — Revision: deterministic application of adversary feedback
5. **S0b (Round 2)** — Adversarial challenge on revised types via haiku call
6. **S0c (Round 2)** — Final revision
7. **S1** — Sketch generation from `final_taxonomy` (4 adversarially-hardened types)
8. **S2** — Structural verification (unchanged)
9. **S3** — 2-dimension scoring (unchanged)
10. **S4** — Structural clustering + signatures (unchanged)
11. **S5** — Thread seed selection (unchanged); `survey_result` stores STRUCTURAL_SIGNATURE, STRUCTURAL_SEED, and **[NEW] STRUCTURAL_DIMENSIONS** (3-tuple: central_frame, causal_mechanism, epistemic_register) per cluster
12. Thread spawning from cluster seeds — unchanged from v11

**Updated gate execution order per candidate (G1c and SAME_CLUSTER handling ADDED):**
1. ED: Harness reads thread's assigned cluster status from `survey_result`
2. ED: Computes least-explored cluster (pure state — no model call)
3. ED: Assembles generation prompt:
   - UNEXPLORED → no directive (unchanged)
   - **ACTIVE → contrastive constraint injected (CHANGED from v5: named dimensional requirements replace soft nudge)**
   - EXHAUSTED → hard redirect (unchanged)
4. Iteration agent generates candidate (receives assembled prompt)
5. Oracle scores candidate
6. G1a: haiku cluster assignment → CLUSTER_ID, RATIONALE, BOUNDARY_CASE
7. G1b: BOUNDARY_CASE tie-break if applicable (unchanged)
8. **[NEW] G1c: If thread's assigned cluster is ACTIVE → haiku contrastive compliance check**
   - **Returns CONTRASTIVE_PASS / CONTRASTIVE_FAIL**
   - **If FAIL → one regeneration (steps 4–8 repeated once, same constraint)**
   - **If FAIL again → mark candidate SAME_CLUSTER = true; proceed**
9. G2: cluster status classification (unchanged)
10. G3: threshold computation + gate decision
    - **[NEW] If SAME_CLUSTER = true → apply ACTIVE threshold (0.80) regardless of G2 status**
    - Otherwise: apply G2 status threshold (unchanged)
11. If gate passes: enter tournament pool; `candidate_count` for assigned cluster incremented
12. If gate fails: discard; thread continues (loop to step 1)

**No change to oracle, fingerprint monitoring (IC1–IC5), escape mechanisms (Steps 1–6d), crossover (Step 6), tournament (Step SD), Step V, thread parallelism, termination condition, or fallback behaviour.**

**Interaction with crossover (Step 6):** SAME_CLUSTER-flagged candidates that pass G3 at the ACTIVE threshold can enter the tournament pool and be eligible for crossover. Crossover operates on candidate quality (oracle score) not structural origin — no modification required.

**Interaction with IC1–IC5 fingerprint monitoring:** G1c uses the same structural fingerprint computed in G1a. The IC1–IC5 monitoring system already produces this fingerprint per candidate; G1c reads it. No additional fingerprint extraction required.

---

## Cost estimate (UPDATED)

Base cost unchanged from v11: ~$91–146 per run.

**Survey overhead (UNCHANGED from v5):** ~$0.009 per run.

**Gate overhead (base, UNCHANGED from v5):** ~$0.27 per run (270 candidates × ~$0.001 per G1a call).

**ED overhead (UNCHANGED from v5):** Zero. No model call. Template fill.

**G1c overhead (NEW in v6):** ~$0.001 per ACTIVE-thread candidate. In a typical 90-iteration run, most candidates originate from ACTIVE-status threads (UNEXPLORED threads quickly become ACTIVE; EXHAUSTED threads are a minority). Estimated ~80–90% of candidates are ACTIVE at time of generation: ~72–81 G1c calls per run × ~$0.001 = **~$0.07–0.09 per run**.

For a regeneration call (G1c fires twice): only triggered when first G1c returns CONTRASTIVE_FAIL. If ~30% of ACTIVE candidates fail on first attempt: ~22–27 additional G1c calls × ~$0.001 = ~$0.02 additional.

**Updated total mechanism overhead per run:** ~$0.009 (survey) + ~$0.27 (G1a gate) + ~$0.00 (ED) + **~$0.09–0.11 (G1c contrastive checks)** = **~$0.37–0.39 per run**.

**Budget check:** $5.00 constraint. ~$0.39 overhead. $4.61 headroom. Comfortably within constraint.

---

## Domain generality (UPDATED from v5)

- Exploration directive templates are domain-agnostic — UNCHANGED from v5
- STRUCTURAL_DIMENSIONS triple (central_frame, causal_mechanism, epistemic_register) derives from Step V vocabulary axes — which are already domain-agnostic and derived from the problem statement without manual configuration. No domain-specific axis names required.
- The contrastive constraint template is domain-agnostic: it references whatever values the S4/S5 pipeline produced for the current problem domain. The same template applies whether the domain is investment thesis writing, product strategy, or technical analysis.
- G1c compliance check is domain-agnostic: it asks "do these named axis values differ?" not "is this a good candidate for domain X?"
- G1–G3, S0a–S0c, S1–S5, and ED (UNEXPLORED and EXHAUSTED branches) are domain-agnostic throughout (unchanged)

---

## What changed this iteration (v5 → v6)

**Element changed:** ED step — ACTIVE branch only. The soft nudge (advisory text appended to prompt) is replaced with an enforceable contrastive constraint (named dimensional requirements + G1c post-generation verification). G1 gains sub-step G1c. G3 gains SAME_CLUSTER handling rule. All other elements unchanged: survey (S0a/S0b/S0c, S1–S5), ED UNEXPLORED and EXHAUSTED branches, G1a/G1b, G2, base G3 thresholds, and all v11 harness mechanisms.

---

### The change — Contrastive constraint: make ACTIVE-thread divergence verifiable

**v5 state:** The ED step's ACTIVE branch appended advisory text to the thread's generation prompt: "this other territory is underexplored — [STRUCTURAL_SIGNATURE of least-explored cluster]. You may continue generating in your current direction, but candidates that work within the territory above will be evaluated with a lower acceptance threshold." This was a nudge — it informed the iteration agent of the exploration incentive, but provided no verifiable requirement. The iteration agent could absorb the text as background context, continue generating in its current structural region, and satisfy the prompt without structural movement. G1 had no mechanism to detect that the iteration agent had ignored the nudge.

**The failure mode:** ACTIVE is the majority status during a run. Most iterations fire the ACTIVE branch of ED. If the ACTIVE branch provides no enforcement mechanism, then most iterations provide no structural push — only the acceptance pull from G3 (0.80 threshold for ACTIVE vs 0.65 for UNEXPLORED). The pull is real but weak: an ACTIVE thread that generates a strong candidate in its current cluster will pass at 0.80; there is no penalty for staying in place. The majority of the run operates in this low-pressure regime. D2 suffers as a result.

**v6 change:** The ACTIVE branch now injects a contrastive constraint instead of advisory text. The constraint:

1. **Names the specific current axis values** of the thread's assigned cluster on all three structural dimensions (central_frame, causal_mechanism, epistemic_register) — values already stored in `survey_result` as STRUCTURAL_DIMENSIONS.

2. **States explicit forbidden combinations** — three 2-dimension combinations that would fail the constraint. The iteration agent cannot satisfy the constraint by adding surface novelty; it must produce a candidate with genuinely different values on at least 2 of the 3 named axes.

3. **Is verified post-generation by G1c** — a haiku-bedrock call that checks the generated candidate's structural fingerprint against the cluster's STRUCTURAL_DIMENSIONS triple. Returns CONTRASTIVE_PASS / CONTRASTIVE_FAIL. No fingerprint extraction overhead (G1a already produces the fingerprint; G1c reads it).

4. **Triggers one regeneration on failure** — same constraint text, same thread context, one more attempt. This gives the iteration agent a second chance without risking infinite loops.

5. **Falls back gracefully** — if both attempts fail, the candidate is marked SAME_CLUSTER and receives the ACTIVE threshold (0.80) — exactly the v5 outcome. The constraint cannot produce a worse result than v5; it can only produce better results when the iteration agent successfully diverges.

6. **SAME_CLUSTER candidates are not awarded exploration credit** — they receive the ACTIVE threshold, not the UNEXPLORED threshold (0.65). This preserves the incentive: genuine divergence is rewarded; superficial divergence is caught by G1c.

**Why this addresses the D2 failure mode identified by all three judges:**

All three judges flagged the ACTIVE soft nudge as weak. Karpathy: "soft nudge for ACTIVE threads remains advisory and could be ignored by the iteration agent." EV Platform Strategist: "the soft nudge for ACTIVE threads is appended text that competes with existing thread context and generation inertia — it may be absorbed without effect." VC Stress-Tester: "evaluation noise in the gate still dominates for ACTIVE-status iterations, which are the bulk of any run."

The contrastive constraint addresses all three concerns simultaneously:
- It cannot be ignored (G1c verifies compliance post-generation)
- It does not compete with generation inertia (it specifies what the candidate must NOT be, not what it should be — it constrains without prescribing a specific target)
- It operates at the generation level for the majority of iterations (ACTIVE is the most common status), not just for EXHAUSTED threads

**The key property:** contrastive constraint + G1 verification creates a verifiable divergence requirement. The iteration agent can't just add the word "different" to a familiar candidate — the fingerprint check catches structural similarity on the named dimensions. The constraint is specific enough to be violated or satisfied: not "be different" but "differ on at least 2 of these 3 named dimensions with these specific current values."

**What this does NOT change:**
- Survey (S0a/S0b/S0c, S1–S5 — unchanged from v5)
- ED UNEXPLORED branch (none — unchanged from v5)
- ED EXHAUSTED branch (hard redirect — unchanged from v5)
- Gate logic: G1a cluster assignment, G1b BOUNDARY_CASE tie-break, G2 status classification, G3 base thresholds (UNEXPLORED 0.65, ACTIVE 0.80, EXHAUSTED 0.90 — unchanged from v5)
- Oracle (frozen, shared — unchanged)
- Fingerprint monitoring (IC1–IC5 — unchanged from v11)
- Escape mechanisms (Steps 1–6d — unchanged from v11)
- Tournament structure (Steps SD, T1–T5 — unchanged from v11)
- Step V vocabulary derivation (unchanged from v11)
- Crossover (Step 6 — unchanged from v11)
- Termination condition (unchanged)
- Fallback behaviour (reverts to 0.80 uniform if survey_result absent — contrastive constraint does not fire if survey_result is absent)

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
*Loop dir: ~/clawd/landscape-coverage-autoresearch/*
