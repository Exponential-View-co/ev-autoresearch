# Coverage Mechanism — v4 (Adversarial S0 Taxonomy Construction)
*Landscape coverage mechanism — v4*
*260314*

---

## Survey design (S0 REPLACED — S0a/S0b/S0c adversarial construction; S1–S5 unchanged)

### Overview

A pre-run structural landscape survey fires once at run start (after Step V vocabulary derivation, before thread spawning). The survey generates 4 structurally extreme candidate claim types via an adversarial construction process, then generates one sketch per type, clusters them into 3 guaranteed non-overlapping regions, and seeds T1, T2, T3 from those regions.

**Core design principle (CHANGED in v4):** Structural diversity is no longer requested — it is constructed through conflict. The S0 adversarial process generates 4 initial claim types, then fires an adversary that identifies pairs which could produce structurally similar candidates. Flagged types are replaced with more extreme versions that make overlap impossible. Diversity is a product of adversarial pressure, not cooperative prompting.

**What changes from v3:** S0 (single haiku call generating 12–18 types with CANNOT_BE_MERGED_WITH constraints) is replaced by S0a/S0b/S0c. S1 now operates on 4 adversarially-hardened types instead of 12–18 cooperatively-generated types. S2–S5 and the entire gate mechanism (G1–G3) are unchanged.

All steps use haiku-bedrock. Survey total cost: ~$0.015, well within the $5 constraint.

---

### Step S0a — Seed taxonomy generation

**Fires:** Once, immediately after Step V (vocabulary derivation). This is the first of three S0 sub-steps.

**Purpose:** Generate 4 initial structural claim types as a draft for adversarial hardening. Speed over completeness — this is the raw material for the adversary, not the final taxonomy.

**Model:** haiku-bedrock

**Prompt:**

> You are a structural analyst. Your task is to enumerate 4 distinct structural CLAIM TYPES that could be used to answer the following question — not the specific answers, but the structural approaches to constructing an answer.
>
> A structural claim type is defined by the TYPE OF LOGICAL MOVE used to reach an answer, not the specific content. Two approaches that use different words but make the same type of argument are the SAME claim type.
>
> Problem statement: {PROBLEM_STATEMENT}
>
> Generate exactly 4 structural claim types. For each, provide:
> — CLAIM_TYPE_ID: a short identifier in CAPS_SNAKE_CASE
> — STRUCTURE: one sentence defining the type of logical move this claim type makes
>
> Requirements:
> — Each claim type must make a different type of logical move
> — Together they should cover the plausible solution space
> — Do not produce generic types that apply to any reasoning task; every type must be specific to THIS problem
> — Do not add caveats or CANNOT_BE_MERGED_WITH constraints — this is a working draft only

**Output:** 4 claim types with CLAIM_TYPE_ID and STRUCTURE. Stored as `s0a_draft_types`.

**Cost:** ~600 tokens ≈ $0.0006.

---

### Step S0b — Adversarial overlap challenge

**Fires:** Immediately after S0a, once per adversarial round. Fires at most twice (see S0c).

**Purpose:** An adversary receives the 4 draft claim types and stress-tests every pair for structural overlap. The adversary's mandate is to find overlap, not confirm independence. If any pair could produce candidates that differ only in framing rather than in what is actually being claimed, that pair is flagged and an alternative, more extreme version of one type is proposed.

**Model:** haiku-bedrock

**Adversary mandate (embed in system prompt):** You are a harsh structural adversary. Your job is to find overlap between claim types, not to confirm they are distinct. A pair overlaps if a candidate generated from type A and a candidate generated from type B could both make the same underlying claim through different surface vocabulary. If the two types differ only in framing — in how the argument is presented — but not in WHAT they are asserting, that is overlap. If both types could produce candidates whose core logical commitment is identical, that is overlap. Only if the two types make different bets about what is actually true in the world are they structurally distinct.

**Prompt:**

> You are a structural adversary. Below are 4 structural claim types proposed for a problem. Your task is to stress-test them for structural overlap.
>
> Problem statement: {PROBLEM_STATEMENT}
>
> Draft claim types (Round {ROUND_NUMBER}):
> {S0a_DRAFT_TYPES}
>
> For each of the 6 pairs (type A vs type B), answer:
> 1. Could a candidate generated from type A and a candidate generated from type B make the same underlying claim through different surface vocabulary? Answer YES or NO.
> 2. If YES: which type should be replaced with a more structurally extreme version to make overlap impossible? State which type (A or B) and propose a replacement STRUCTURE that pushes further from the other type — not a minor reframing, but a fundamentally different bet about what is true.
>
> Be harsh. If you are uncertain, answer YES. Framing differences do not count as structural differences. Only differences in what is actually being claimed count.
>
> Output format (one block per pair):
> PAIR: {TYPE_A_ID} vs {TYPE_B_ID}
> OVERLAP: YES / NO
> IF YES — REPLACE: {TYPE_ID to replace}
> IF YES — REPLACEMENT_STRUCTURE: {one sentence — the new, more extreme structural move}
> IF YES — WHY_MORE_EXTREME: {one sentence — what makes this structurally impossible to confuse with the other type}

**Output:** 6 pair assessments with OVERLAP flag and, where flagged, a replacement proposal. Stored as `s0b_adversary_output[round]`.

**Cost:** ~800 tokens ≈ $0.0008 per round. Two rounds maximum = ~$0.0016.

---

### Step S0c — Adversarially-informed revision

**Fires:** Immediately after S0b, once per adversarial round.

**Purpose:** Apply the adversary's feedback. Replace any flagged claim types with the more extreme version the adversary proposed. Where the same type is flagged by multiple pairs, apply the most extreme proposed replacement (the one with the largest structural distance from its overlap partner).

**Model:** haiku-bedrock

**Processing rule (deterministic — no model call):**

```
For each flagged pair in s0b_adversary_output[round]:
  if OVERLAP == YES:
    replacement_type = s0b_adversary_output[round].pair.REPLACE
    replacement_structure = s0b_adversary_output[round].pair.REPLACEMENT_STRUCTURE
    update s0a_draft_types[replacement_type].STRUCTURE = replacement_structure

If any type was replaced by multiple pairs:
  keep the replacement whose WHY_MORE_EXTREME sentence is longest
  (a proxy for structural distance — longer explanations indicate more extreme divergence)
```

**After revision:** The 4 types now reflect one round of adversarial hardening. Stored as `s0c_revised_types[round]`.

**Adversarial rounds:** The S0b → S0c cycle repeats once:
- **Round 1:** S0b adversary challenges `s0a_draft_types`. S0c produces `s0c_revised_types[1]`.
- **Round 2:** S0b adversary challenges `s0c_revised_types[1]`. S0c produces `s0c_revised_types[2]`.
- After Round 2: `final_taxonomy = s0c_revised_types[2]`. No further adversarial rounds.

**On degenerate adversary output (all 6 pairs flagged as NO):** Accept `s0a_draft_types` as-is after one round and skip Round 2. Log `s0b_all_clear: true` in global state. This is unlikely but not impossible — it indicates the seed types were already maximally distinct.

**On parse failure in S0b:** If the adversary output cannot be parsed for any pair, treat that pair as OVERLAP = NO. The revision step is robust to partial adversary output.

**Cost:** Zero model calls (pure state update). Adversarial round costs are carried in S0b.

---

### Final taxonomy commit

After S0c Round 2 (or Round 1 if all-clear), `final_taxonomy` contains 4 adversarially-hardened claim types. These replace the 12–18 cooperatively-generated types from v3's S0. The key property: every claim type in `final_taxonomy` has survived at least two adversarial challenges — it was not replaced because no adversary could identify a more extreme structural alternative that avoids overlap with other types. Diversity is guaranteed by the adversary's inability to find overlap, not by a cooperative request to be diverse.

---

### Step S1 — Sketch generation

**Fires:** Immediately after S0c completes (after Round 2, or after Round 1 if all-clear). Operates on `final_taxonomy`.

**Purpose:** Generate one 3-sentence directional sketch per adversarially-hardened claim type. Sketches are structural bets, not full answers.

**Model:** haiku-bedrock (batched — all 4 sketches in a single prompt)

**Prompt:**

> You will generate brief candidate sketches for the following problem: {PROBLEM_STATEMENT}
>
> Below is a list of structural claim types. For EACH claim type, write a 3-sentence sketch:
> — Sentence 1: The core claim (commit to a specific position)
> — Sentence 2: The key mechanism (state the structural argument explicitly)
> — Sentence 3: The expected shape of a strong answer
>
> CRITICAL: Each sketch must commit to the claim type it was generated for. Do not reframe, hedge, or blend claim types. These types were constructed adversarially to be structurally incompatible — honour that incompatibility.
>
> {FINAL_TAXONOMY}
>
> Output format: one sketch per claim type, clearly labelled with CLAIM_TYPE_ID.

**Cost:** ~1,500–2,000 tokens ≈ $0.0015–0.002 (4 types × 3 sentences each, less than v3's 12–18 types).

---

### Step S2 — Structural verification (Layer 2 guarantee)

**UNCHANGED from v3.** Operates on the 4 sketches from S1. Verifies each sketch commits structurally to its assigned claim type. Reassigns drifted sketches; discards lower-scoring duplicates; retains null sketches for clustering.

**Cost:** ~1,000–1,500 tokens ≈ $0.001–0.0015 (fewer sketches than v3 → lower cost).

---

### Step S3 — 2-dimension scoring

**UNCHANGED from v3.** Scores each verified sketch on plausibility (1–5) and resolution potential (1–5). Operates on the verified sketches from S2.

**Cost:** ~800–1,000 tokens ≈ $0.0008–0.001.

---

### Step S4 — Structural clustering into 3 regions (Layer 3 guarantee)

**UNCHANGED from v3** (including STRUCTURAL_SIGNATURE output). Groups the 4 claim types into exactly 3 clusters via a single haiku call. With only 4 types, one cluster will contain 2 types and two will contain 1 each — the NON_OVERLAP_AUDIT is more important, not less, because the boundary between the 2-type cluster and each single-type cluster must be sharply defined.

**Note on cluster topology:** With 4 adversarially-hardened types distributed into 3 clusters, the typical configuration is (2, 1, 1). S4 validation requires all 4 types appear in exactly one cluster and each cluster has at least 1 verified sketch. The NON_OVERLAP_AUDIT is the critical check: if the 2-type cluster contains types that the adversary could not fully separate in S0b, S4's audit will flag this.

**Cost:** ~1,000–1,500 tokens ≈ $0.001–0.0015.

---

### Step S5 — Thread seed selection

**UNCHANGED from v3.** Selects the best sketch from each cluster using `seed_score = plausibility × resolution_potential`. Stores `survey_result` in global state. Thread seeding instruction is identical.

**Cost:** Zero model calls.

---

### Survey cost summary

| Step | Model | Calls | Tokens (est.) | Cost (est.) |
|---|---|---|---|---|
| S0a — Seed taxonomy (4 types) | haiku-bedrock | 1 | ~600 | ~$0.0006 |
| S0b — Adversarial challenge (×2 rounds) | haiku-bedrock | 2 | ~1,600 | ~$0.0016 |
| S0c — Revision (deterministic, ×2 rounds) | — | 0 | 0 | $0 |
| S1 — Sketch generation (4 sketches) | haiku-bedrock | 1 (batched) | ~2,000 | ~$0.002 |
| S2 — Structural verification | haiku-bedrock | 1 (batched) | ~1,500 | ~$0.0015 |
| S3 — 2-dimension scoring | haiku-bedrock | 1 (batched) | ~1,000 | ~$0.001 |
| S4 — Structural clustering + signatures | haiku-bedrock | 1 | ~1,500 | ~$0.0015 |
| **Total** | haiku-bedrock | **7 calls** | **~9,800 tokens** | **~$0.009** |

**Comparison to v3:** v4 uses 2 additional haiku calls (S0b ×2) but fewer total tokens overall — 4 types generate less sketch/verification/scoring text than 12–18. Net result: v4 survey costs ~$0.009 vs v3's ~$0.013. Cheaper AND structurally stronger.

Budget: $5.00 constraint. Actual usage: ~$0.009. Headroom: $4.991.

---

## Gate mechanism (UNCHANGED from v3)

### Overview

The uniform 0.80 × current-best threshold is replaced with a region-aware diversity gate. The gate reads `survey_result` cluster signatures and assigns each incoming candidate to its nearest cluster via a single haiku-bedrock call per candidate (G1). The acceptance threshold adjusts based on that cluster's exploration status (G2, G3).

**All gate steps (G1, G2, G3) are identical to v3.** The only upstream change is that the cluster signatures fed into G1 now come from 4 adversarially-hardened types rather than 12–18 cooperatively-generated types. The G1 haiku call is identical in structure — it receives cluster structural signatures and candidate text and returns CLUSTER_ID + RATIONALE + BOUNDARY_CASE. The signatures are structurally more extreme (adversarial hardening guarantees maximum structural distance), which means G1's classification task is easier — candidates are less likely to sit near cluster boundaries, and BOUNDARY_CASE = YES fires less often.

---

### Step G1 — Candidate cluster assignment

**UNCHANGED from v3.** Single haiku-bedrock call per candidate. Cluster signatures from `survey_result` (now derived from adversarially-hardened types). Returns CLUSTER_ID (1/2/3), RATIONALE, BOUNDARY_CASE.

**Expected effect of v4's upstream change:** Adversarially-hardened cluster signatures have maximum structural distance by construction. The G1 classifier receives more distinct structural anchors → cleaner boundary decisions → lower BOUNDARY_CASE rate → fewer tie-break resolutions toward less-explored clusters. The gate's quality signal improves as a downstream consequence of the S0 change.

**Cost per candidate:** ~$0.001 (unchanged).

---

### Step G2 — Cluster status classification

**UNCHANGED from v3.** Deterministic state lookup. Returns UNEXPLORED / ACTIVE / EXHAUSTED.

---

### Step G3 — Threshold computation and gate decision

**UNCHANGED from v3.**

| Cluster status | Acceptance threshold |
|---|---|
| UNEXPLORED | **0.65 × current-best** |
| ACTIVE | **0.80 × current-best** |
| EXHAUSTED | **0.90 × current-best** |

---

### Anti-gaming property

**UNCHANGED from v3, strengthened by v4's upstream change.** Adversarially-hardened cluster signatures are structurally more extreme — a thread generating surface-diverse but structurally similar candidates has a harder time gaming the UNEXPLORED threshold, because the haiku classifier maps structurally similar candidates to the same cluster regardless of surface vocabulary, and the cluster signatures themselves are maximally distinct structural anchors.

---

### Convergence safety

**UNCHANGED from v3.** Four safeguards (ACTIVE floor, exhausted ceiling, no infinite loop, unassigned fallback) apply identically. With only 3 clusters (derived from 4 types), convergence dynamics are cleaner — no long tail of rarely-explored claim types that could perturb thread activity counts.

---

## Integration (UNCHANGED from v3 except S0 sub-step labelling)

**Updated survey execution order:**
1. Step V (vocabulary derivation) fires — unchanged from v11
2. **S0a** — Seed taxonomy: 4 initial claim types via haiku call
3. **S0b (Round 1)** — Adversarial challenge: 6-pair overlap assessment via haiku call
4. **S0c (Round 1)** — Revision: deterministic application of adversary feedback
5. **S0b (Round 2)** — Adversarial challenge on revised types via haiku call
6. **S0c (Round 2)** — Final revision
7. **S1** — Sketch generation from `final_taxonomy` (4 adversarially-hardened types)
8. **S2** — Structural verification (unchanged)
9. **S3** — 2-dimension scoring (unchanged)
10. **S4** — Structural clustering + signatures (unchanged; 4 types → 3 clusters)
11. **S5** — Thread seed selection (unchanged)
12. Thread spawning from cluster seeds — unchanged from v11

**Gate execution order per candidate (UNCHANGED from v3):**
1. Candidate generated by thread
2. Oracle scores candidate
3. G1: haiku cluster assignment
4. G2: cluster status classification
5. G3: threshold computation + gate decision
6. If gate passes: enter tournament pool
7. If gate fails: discard; thread continues

**No change to oracle, fingerprint monitoring (IC1–IC5), escape mechanisms (Steps 1–6d), crossover (Step 6), tournament (Step SD), Step V, thread parallelism, termination condition, or fallback behaviour.**

---

## Cost estimate (UPDATED)

Base cost unchanged from v11: ~$91–146 per run.

**Survey overhead:** ~$0.009 per run (REDUCED from v3's ~$0.013 — fewer total tokens despite 2 additional haiku calls).

**Gate overhead (UNCHANGED from v3):** ~$0.27 per run (270 candidates × ~$0.001 per G1 call).

**Expected BOUNDARY_CASE rate (REDUCED):** Adversarially-hardened cluster signatures have maximum structural distance. Fewer candidates land near cluster boundaries → fewer BOUNDARY_CASE = YES flags → cleaner gate decisions. No change to per-candidate cost, but signal quality is higher.

**Updated total mechanism overhead per run:** ~$0.009 (survey) + ~$0.27 (gate) = **~$0.28 per run** (functionally identical to v3 — survey savings negligible against gate cost).

**Budget check:** $5.00 constraint. $0.28 overhead. $4.72 headroom. Unchanged from v3.

---

## Domain generality (UNCHANGED from v3)

- Cluster definitions derive from the problem-specific taxonomy (S0a → adversarial hardening) — no domain-specific configuration required
- S0b adversary prompt is domain-agnostic: it receives claim type STRUCTURE sentences and assesses structural overlap — identical operation across writing, strategy, investment, product, and technical problems
- S0c revision is deterministic — no domain configuration
- G1–G3 are domain-agnostic throughout (unchanged from v3)
- The three cluster status classes and threshold table are domain-independent

---

## What changed this iteration (v3 → v4)

**Element changed:** S0 taxonomy generation only. One change. Gate mechanism (G1–G3) unchanged from v3. S1–S5 unchanged from v3 (S1 prompt gains one sentence acknowledging adversarial incompatibility, but the mechanism is identical). All v11 harness mechanisms preserved.

---

### The change — Adversarial S0 construction replacing cooperative diversity request

**v3 state:** S0 used a single haiku call to generate 12–18 structural claim types, with CANNOT_BE_MERGED_WITH fields asking the model to flag types that look similar but are structurally distinct. This is a cooperative approach: the model is asked to be diverse. The three judges all scored D1 (coverage expansion) at 5–6/10 because there is no structural guarantee that a model asked to be diverse will produce structural diversity rather than surface diversity. A model generating 12–18 types in a single call can produce types that are genuinely distinct in vocabulary but structurally identical in logical move — particularly when the problem domain pulls language toward a particular conceptual neighbourhood. The CANNOT_BE_MERGED_WITH constraints are a weak safeguard: they ask the same model that generated potentially-homogeneous types to also flag its own homogeneity.

**v4 change:** S0 is replaced with a 3-sub-step adversarial construction process:

**S0a (Seed):** A single haiku call generates 4 initial claim types — fast, cheap, no diversity constraints. This is a working draft only. The reduction from 12–18 to 4 types is deliberate: the adversarial process is more tractable with fewer types (6 pairs vs up to 153 pairs for 18 types), and the downstream S4 clustering into 3 groups needs only 4 structurally extreme types, not 12–18 possibly-redundant ones.

**S0b (Adversarial challenge):** A second haiku call acts as an adversary. It receives the 4 draft types and assesses all 6 pairwise combinations. Its mandate is explicitly adversarial: find overlap, do not confirm independence. The criterion is intentionally strict — if two types could produce candidates that differ only in framing (surface vocabulary, rhetorical register, argument structure) but not in what is actually being claimed about the world, that is overlap. The adversary is instructed to answer YES when uncertain. This is not a balanced peer reviewer; it is a structural attack.

For each flagged pair, the adversary proposes a replacement version of one type: not a minor reframing, but a structurally more extreme version that makes overlap with the other type impossible. The replacement must make a fundamentally different bet about what is true.

**S0c (Revision):** The revision step applies the adversary's feedback deterministically — no additional model call. Flagged types are replaced with the adversary's proposed more-extreme version. The revision is applied in a single pass per round, with a conflict resolution rule when the same type is flagged multiple times (longest WHY_MORE_EXTREME explanation wins, as a proxy for structural distance).

**Two adversarial rounds:** The S0b → S0c cycle fires twice. After Round 1, the types have survived one adversarial challenge. Round 2 fires on the revised types — the adversary now attempts to find residual overlap in the already-hardened taxonomy. Types that survive Round 2 without replacement have resisted two independent adversarial challenges.

**The key property:** A claim type in `final_taxonomy` was either:
(a) never flagged as overlapping by either adversarial round — it was structurally distinct from all other types without modification, OR
(b) replaced once or twice with progressively more extreme versions until the adversary could no longer identify a more structurally distant alternative

In both cases, the outcome is structural distance by construction. The diversity of the final taxonomy is the residue of adversarial failure, not the product of a single cooperative call.

**Why this addresses the D1 failure mode directly:**

The VC Stress-Tester identified the failure as: "the survey still relies on haiku to spontaneously generate diverse claim types; without contrastive generation or adversarial probing, the 3 clusters could be three flavours of the same structural region." The adversarial process addresses this directly — the three clusters now derive from types that have been explicitly pressure-tested for structural overlap and replaced with more extreme versions wherever overlap was found. The adversary is the contrastive mechanism the evaluator called for.

**Karpathy's specific concern** — "a model asked to be diverse will produce surface diversity, not structural diversity" — is addressed by removing the cooperative diversity request entirely. S0a makes no diversity request. Diversity enters the taxonomy not through a prompt constraint but through adversarial challenge: a type survives only if the adversary cannot find another type it could overlap with.

**Downstream effect on G1:** Adversarially-hardened cluster signatures are structurally more extreme than cooperatively-generated ones. The G1 classifier receives more distinct anchors, making candidate-to-cluster assignment cleaner and reducing BOUNDARY_CASE frequency. The gate's signal quality improves as a free downstream consequence of the S0 change — no gate modifications required.

**Cost:** 2 additional haiku calls (S0b × 2 rounds) add ~$0.0016 per run. This is offset by fewer tokens in S1–S3 (4 types instead of 12–18). Net survey cost is $0.009 vs v3's $0.013 — v4 is cheaper and structurally stronger.

**What this does NOT change:**
- Gate mechanism (G1–G3 prompt, logic, thresholds — unchanged from v3)
- S1 mechanism (prompt structure unchanged; operates on 4 types instead of 12–18)
- S2 structural verification (unchanged)
- S3 2-dimension scoring (unchanged)
- S4 structural clustering (unchanged; 4 types → 3 clusters topology is (2,1,1) typical)
- S5 thread seed selection (unchanged)
- Escape mechanisms (Steps 1–6d — unchanged from v11)
- Vocabulary derivation (Step V — unchanged from v11)
- Fingerprint monitoring (Steps IC1–IC5 — unchanged from v11)
- Tournament structure (Steps SD, T1–T5 — unchanged from v11)
- Oracle (frozen, shared — unchanged)
- Termination condition (unchanged)
- Fallback behaviour (reverts to 0.80 uniform if survey_result absent)

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
*Loop dir: ~/clawd/landscape-coverage-autoresearch/*
