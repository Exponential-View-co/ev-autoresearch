# Coverage Mechanism — v7 (Structural Extremity Pass for Survey Diversity)
*Landscape coverage mechanism — v7*
*260314*

---

## Survey design (UPDATED — S0d extremity pass added after S0c)

### Overview

A pre-run structural landscape survey fires once at run start (after Step V vocabulary derivation, before thread spawning). The survey generates 4 structurally extreme candidate claim types via an adversarial construction process, then generates one sketch per type, clusters them into 3 guaranteed non-overlapping regions, and seeds T1, T2, T3 from those regions.

**Core design principle (UNCHANGED from v6):** Structural diversity is constructed through conflict. The S0 adversarial process generates 4 initial claim types, then fires an adversary that identifies pairs which could produce structurally similar candidates. Flagged types are replaced with more extreme versions that make overlap impossible. Diversity is a product of adversarial pressure, not cooperative prompting.

**What changes from v6:** A new step S0d is added after the two-round adversarial construction (S0a/S0b/S0c). After S0c completes its second round, S0d fires a sonnet-bedrock call that takes each of the 4 surviving claim types to its structural extreme — producing the boldest, most specific, and most unusual version of each type while preserving coherence. S0b then fires one final time (Round 3) on the extreme versions to check for any residual overlap. The result: claim types entering S1 are not merely adversarially separated — they are as structurally distant from one another as possible. All other survey steps (S1–S5), the gate mechanism (ED, G1–G3), and all v11 harness mechanisms are unchanged from v6.

---

### Step S0a — Seed taxonomy generation

**UNCHANGED from v6.**

---

### Step S0b — Adversarial overlap challenge

**UNCHANGED from v6** (haiku-bedrock call; fires on Round 1, Round 2, and — new in v7 — Round 3 after S0d extremity pass).

---

### Step S0c — Adversarially-informed revision

**UNCHANGED from v6** (deterministic application of S0b adversary feedback; fires after Round 1 and Round 2 only).

---

### Step S0d — Structural extremity pass (NEW in v7)

**Fires:** Once, after S0c Round 2 completes. The 4 claim types at this point have survived two rounds of adversarial overlap challenge. S0d takes them to their structural extremes before committing to the final taxonomy.

**Purpose:** The two-round adversarial process (S0b/S0c) ensures claim types are *separated* — it removes pairs that overlap. But separation is not the same as extremity. Two types can be non-overlapping yet both conservative, hedged, or structurally cautious. S0d pushes each surviving type to its logical extreme on the dimensions that make it structurally distinctive. The goal is maximum structural distance between types, achieved by maximisation rather than merely by separation. A claim type that makes an unusual structural bet — bold, specific, committing to a direction — is harder to confuse with any other type and generates candidates that genuinely differ in shape.

**Why sonnet-bedrock, not haiku-bedrock:** The adversarial rounds (S0b) use haiku-bedrock for cost efficiency in pairwise overlap judgement — a relatively coarse discrimination task. The extremity pass requires something more precise: for each claim type, identify *which structural properties* make it distinctive, then amplify those properties without breaking coherence. This requires distinguishing genuine structural novelty from surface variation — a task where haiku's structural reasoning is more prone to false equivalences. Sonnet-bedrock has reliably stronger structural reasoning for this step; it is better at identifying what makes a type genuinely unusual versus merely differently phrased. Cost: one sonnet call ≈ $0.004. Negligible.

**Inputs:**
- `final_taxonomy` after S0c Round 2 — the 4 surviving claim types (each described as a short structural description: what epistemic move the candidate makes, what structural position it occupies)

**Single sonnet-bedrock call. Prompt:**

> You are given 4 structural claim types. These types have already been adversarially tested to ensure they do not overlap structurally. Your task is to take each type to its structural extreme.
>
> For each of the 4 types:
> 1. Identify the core structural property that makes it distinctive — the specific epistemic move, causal logic, or framing that separates it from the other three types.
> 2. Push that property to its logical extreme: make the structural claim as **bold, specific, and unusual as possible** while remaining coherent.
> 3. The result should feel surprising and commit to an unusual structural bet — not a safe, hedged version of the original.
> 4. The goal is maximum structural distance from the other 3 types, achieved by sharpening what makes each type structurally unique — not by adding "novel" language to a familiar framing.
>
> Do NOT produce types that:
> - Sound different but make the same underlying structural move
> - Are extreme in rhetoric but conservative in structure ("the most extreme version" framed as a superlative claim, but with the same logic as the original)
> - Could be confused with any of the other 3 extreme types
>
> For each type, output:
> - `original`: the original claim type description
> - `distinctive_property`: the core structural property you identified (1 sentence)
> - `extreme_version`: the pushed, extreme version of the claim type (2–3 sentences describing the structural position)
> - `structural_distance_note`: why this extreme version is harder to confuse with the other 3 types (1 sentence)
>
> Input claim types:
> [{final_taxonomy after S0c Round 2 — 4 types, each as a short structural description}]

**Output:** 4 extreme versions, each with `extreme_version` as the new claim type description. The `original`, `distinctive_property`, and `structural_distance_note` fields are logged but not passed forward.

**S0b Round 3 — Extremity overlap check (haiku-bedrock):**

After S0d produces 4 extreme versions, S0b fires one final time — Round 3 — on the extreme versions. The adversary checks whether the extremity push has inadvertently created new overlaps (e.g., two types pushed in the same "bold" direction and converged structurally).

**Prompt modification for Round 3 (CHANGED from standard S0b prompt):**

> These 4 claim types have each been pushed to their structural extreme. Assess whether any pair now overlaps structurally — not due to hedging or conservatism, but because the extremity push moved two types in the same direction.
>
> For any overlapping pair:
> - Identify which of the two is the *less extreme* version (the one whose extremity push was less bold or less structurally distinctive)
> - Instruction: "Push [less extreme type] further — it must make a more unusual structural bet that moves away from [other type]. The goal is that the two types, at their respective extremes, feel like bets on completely different structural properties."
>
> If no pair overlaps after the extremity push: output "NO_OVERLAP — extremity pass complete." Commit `extreme_taxonomy` as the final taxonomy for S1.
>
> [{extreme_version} descriptions for all 4 types]

**Handling of Round 3 adversary output:**

| Round 3 outcome | Action |
|---|---|
| NO_OVERLAP | Commit `extreme_taxonomy` as `final_taxonomy`. Proceed to S1. |
| Overlap flagged for 1 pair | Apply adversary instruction: replace the less-extreme type with the further-pushed version. Commit updated `extreme_taxonomy` as `final_taxonomy`. Proceed to S1. (No fourth round — one correction pass only.) |
| Overlap flagged for 2+ pairs | Apply adversary instruction for each flagged pair. Commit updated `extreme_taxonomy` as `final_taxonomy`. Proceed to S1. (Same rule: one correction pass only.) |

**Why one correction pass only, not a third full adversarial cycle:** Two full adversarial rounds (S0b/S0c Round 1 and Round 2) have already guaranteed non-overlap before S0d fires. The extremity push is unlikely to introduce large overlaps — it amplifies divergence, not convergence. Round 3 is a safety check on the extremity pass's output, not a full re-do of adversarial separation. If one correction pass is insufficient, the final taxonomy still enters S1 with 4 adversarially-separated types (guaranteed by the prior two rounds); it simply may not be maximally extreme. This is a graceful degradation — the mechanism cannot produce a worse outcome than v6 in this path.

**State written:** `extreme_taxonomy` — 4 extreme claim type descriptions (replaces `final_taxonomy` from S0c Round 2 as the input to S1). Persisted in `survey_result` alongside the originals for logging.

**Cost:** 1 sonnet-bedrock call (S0d extremity generation) ≈ $0.004 + 1 haiku-bedrock call (S0b Round 3) ≈ $0.001 = **~$0.005 per run**. Negligible within the $5 constraint.

---

### Final taxonomy commit

After S0d and S0b Round 3: `extreme_taxonomy` (4 structurally extreme, adversarially-verified claim types) is committed as `final_taxonomy` for S1.

**CHANGED from v6:** `final_taxonomy` now contains extreme versions of the surviving types, not the S0c Round 2 versions. The claim types entering S1 are as structurally extreme and mutually distant as possible.

---

### Step S1 — Sketch generation

**UNCHANGED from v6** (operates on `final_taxonomy` — which now contains S0d extreme versions).

---

### Step S2 — Structural verification (Layer 2 guarantee)

**UNCHANGED from v6.**

---

### Step S3 — 2-dimension scoring

**UNCHANGED from v6.**

---

### Step S4 — Structural clustering into 3 regions (Layer 3 guarantee)

**UNCHANGED from v6.**

---

### Step S5 — Thread seed selection

**UNCHANGED from v6.** `survey_result` stores STRUCTURAL_SIGNATURE, STRUCTURAL_SEED, and STRUCTURAL_DIMENSIONS (the three named axis values for each cluster) per cluster. These now reflect the extreme taxonomy's structural properties — which are sharper and more structurally committed than the pre-extremity versions.

**Example (domain-agnostic, illustrating how extremity amplifies distinctiveness):**
```
# Before S0d (post-S0c Round 2):
cluster[1].STRUCTURAL_DIMENSIONS = {
  central_frame: "comparative_counterfactual",
  causal_mechanism: "feedback_loop",
  epistemic_register: "empirical_predictive"
}

# After S0d (extreme version — same type, pushed to structural extreme):
cluster[1].STRUCTURAL_DIMENSIONS = {
  central_frame: "single_counterfactual_world_comparison",
  causal_mechanism: "nonlinear_self_reinforcing_cascade",
  epistemic_register: "empirical_predictive_with_confidence_bounds"
}
```

The axis values become more specific and structurally committed — this sharpens the contrastive constraint in the gate mechanism (ED/G1c), making the forbidden combinations more precise and harder to satisfy by surface variation.

---

### Survey cost summary

**UPDATED from v6:** ~$0.009 per run (base survey, unchanged) + **~$0.005 per run (S0d: 1 sonnet call + 1 haiku call)** = **~$0.014 per run**. Negligible.

---

## Gate mechanism (UNCHANGED from v6)

### Overview

**UNCHANGED from v6.** The contrastive constraint (G1c), SAME_CLUSTER handling (G3), ED ACTIVE branch, and all threshold tiers are identical to v6. The gate now operates on structurally more extreme cluster seeds (because `final_taxonomy` is more extreme post-S0d) — but no gate logic changes are required. More extreme seeds mean the contrastive constraint's forbidden dimension combinations are more specific, which makes G1c's compliance check more discriminating without any modification to G1c itself.

---

### Step ED — Exploration directive injection

**UNCHANGED from v6** (ACTIVE contrastive constraint, UNEXPLORED no-directive, EXHAUSTED hard redirect all identical).

---

### Step G1 — Candidate cluster assignment (including G1c)

**UNCHANGED from v6** (G1a cluster assignment, G1b boundary case tie-break, G1c contrastive compliance verification — all identical).

---

### Step G2 — Cluster status classification

**UNCHANGED from v6.**

---

### Step G3 — Threshold computation and gate decision

**UNCHANGED from v6** (UNEXPLORED 0.65, ACTIVE 0.80, EXHAUSTED 0.90, SAME_CLUSTER → ACTIVE threshold — all identical).

---

### Anti-gaming property (UNCHANGED from v6)

The contrastive constraint targets the three specific structural axes already tracked throughout IC1–IC5 monitoring. An iteration agent cannot satisfy the constraint by adding surface novelty — G1c operates on the same structural fingerprint used throughout. **v7 note:** The extreme taxonomy means those axis values are now more specific and structurally committed, making the forbidden combinations harder to satisfy by near-synonym substitution. The anti-gaming property is strengthened by S0d without any gate modification.

---

### Convergence safety (UNCHANGED from v6)

All v6 safety properties preserved. S0d adds a new survey-side safety property: S0d can only produce a better outcome than S0c Round 2 (more extreme types, better separated) or fall back to the S0c Round 2 state (Round 3 finds irreparable overlaps). The mechanism cannot produce a worse outcome than v6 for any individual run; S0d is an upside-only survey enhancement.

---

## Integration (UPDATED — S0d and S0b Round 3 added to survey execution order)

**Survey execution order (UPDATED from v6):**
1. Step V (vocabulary derivation) fires — unchanged from v11
2. **S0a** — Seed taxonomy: 4 initial claim types via haiku call
3. **S0b (Round 1)** — Adversarial challenge: 6-pair overlap assessment via haiku call
4. **S0c (Round 1)** — Revision: deterministic application of adversary feedback
5. **S0b (Round 2)** — Adversarial challenge on revised types via haiku call
6. **S0c (Round 2)** — Final revision → 4 adversarially-separated claim types
7. **[NEW] S0d** — Structural extremity pass: 4 surviving types passed to sonnet-bedrock call with instruction to push each to its structural extreme → 4 extreme versions produced
8. **[NEW] S0b (Round 3)** — Extremity overlap check: haiku adversary checks extreme versions for newly introduced overlap; applies one correction pass if flagged → `extreme_taxonomy` committed as `final_taxonomy`
9. **S1** — Sketch generation from `final_taxonomy` (4 adversarially-hardened, structurally extreme types)
10. **S2** — Structural verification (unchanged)
11. **S3** — 2-dimension scoring (unchanged)
12. **S4** — Structural clustering + signatures (unchanged)
13. **S5** — Thread seed selection (unchanged); `survey_result` stores STRUCTURAL_SIGNATURE, STRUCTURAL_SEED, and STRUCTURAL_DIMENSIONS per cluster — now reflecting extreme taxonomy values
14. Thread spawning from cluster seeds — unchanged from v11

**Gate execution order per candidate (UNCHANGED from v6):**
1. ED: Harness reads thread's assigned cluster status from `survey_result`
2. ED: Computes least-explored cluster (pure state — no model call)
3. ED: Assembles generation prompt (ACTIVE → contrastive constraint; UNEXPLORED → no directive; EXHAUSTED → hard redirect — all unchanged from v6)
4. Iteration agent generates candidate
5. Oracle scores candidate
6. G1a: haiku cluster assignment → CLUSTER_ID, RATIONALE, BOUNDARY_CASE
7. G1b: BOUNDARY_CASE tie-break if applicable
8. G1c: If thread's assigned cluster is ACTIVE → haiku contrastive compliance check; FAIL → one regeneration; FAIL again → SAME_CLUSTER = true
9. G2: cluster status classification
10. G3: threshold computation + gate decision; SAME_CLUSTER → ACTIVE threshold
11. If gate passes: enter tournament pool; `candidate_count` incremented
12. If gate fails: discard; thread continues

**No change to oracle, fingerprint monitoring (IC1–IC5), escape mechanisms (Steps 1–6d), crossover (Step 6), tournament (Step SD), Step V, thread parallelism, termination condition, or fallback behaviour.**

---

## Cost estimate (UPDATED)

Base cost unchanged from v11: ~$91–146 per run.

**Survey overhead (UPDATED from v6):** ~$0.009 per run (base survey unchanged) + ~$0.005 (S0d: 1 sonnet call + 1 haiku Round 3 call) = **~$0.014 per run**.

**Gate overhead (UNCHANGED from v6):** ~$0.37–0.39 per run (G1a + G1c base + G1c regeneration).

**Updated total mechanism overhead per run:** ~$0.014 (survey) + ~$0.37–0.39 (gate) = **~$0.38–0.40 per run**.

**Budget check:** $5.00 constraint. ~$0.40 overhead. $4.60 headroom. Comfortably within constraint.

---

## Domain generality (UNCHANGED from v6)

- S0d prompt is domain-agnostic: the sonnet call is instructed to identify and amplify the *structural* properties that make each type distinctive — not domain-specific content features. The same extremity prompt applies whether the domain is investment thesis writing, product strategy, or technical analysis.
- S0b Round 3 is domain-agnostic: same adversarial overlap check, same format, applied to whatever extreme versions S0d produced.
- The resulting `extreme_taxonomy` feeds S1 through S5 — all of which are already domain-agnostic (unchanged from v6).
- G1–G3, ED, and all gate mechanisms derive their domain-specificity entirely from `survey_result` state produced by S4/S5. No new domain-specific configuration introduced by S0d.

---

## What changed this iteration (v6 → v7)

**Element changed:** Survey construction — S0d structural extremity pass added after S0c Round 2, plus S0b Round 3 extremity overlap check. All gate mechanisms (ED, G1–G3) and all other survey steps (S1–S5) are unchanged from v6.

---

### The change — Structural extremity pass: push D1 above the LLM-adversary ceiling

**v6 state:** The adversarial survey construction (S0a/S0b/S0c) guarantees *separation* — it removes overlapping claim types through two rounds of adversarial conflict. The adversary (haiku-bedrock) makes pairwise qualitative judgements: "do these two types overlap structurally?" If yes, the less distinctive type is replaced. After two rounds, 4 non-overlapping types remain.

**The ceiling at 7/10:** All three v6 judges capped D1 at 7/10. The reason is structural: the adversarial construction guarantees non-overlap, but not extremity. Two types can be non-overlapping yet both conservative, hedged, or structurally cautious — occupying nearby but non-identical regions of the structural landscape. The adversary's job is conflict detection, not extremity maximisation. Haiku is sufficient for conflict detection; it is less reliable for identifying *which structural property* makes a type genuinely distinctive and then amplifying it. The adversary asks "do these overlap?" — not "is this type as bold and unusual as it could be?"

**The failure mode:** If claim types are structurally cautious — plausible, coherent, but hedged into safe structural territory — then the sketches they generate (S1), the clusters they define (S4), and the seeds they provide to threads (S5) all reflect this conservatism. The threads start in adequately separated but structurally mild regions. The contrastive constraint (v6's G1c) then verifies that candidates diverge from these mild seeds — but divergence from a mild starting point does not guarantee genuinely unusual structure. The coverage expansion is real but bounded by the structural ambition of the initial claim types.

**v7 change:** After two adversarial rounds have guaranteed non-overlap, S0d fires a single sonnet-bedrock call with one instruction: *take each type to its structural extreme.* The prompt asks sonnet to identify the core structural property that makes each type distinctive — the specific epistemic move, causal logic, or framing that separates it from the other three — and push that property to its logical extreme, producing the boldest, most specific, and most unusual version while remaining coherent.

The key design properties of S0d:

1. **Extremity by amplification, not replacement.** S0d does not generate new claim types from scratch; it takes the 4 adversarially-separated survivors and pushes each one further in the direction it was already heading. This preserves the adversarial separation guarantee from S0b/S0c — the extreme versions remain non-overlapping because they were already non-overlapping before being pushed further apart.

2. **Sonnet for structural reasoning, not haiku.** The adversarial rounds use haiku for cost efficiency in pairwise conflict detection. S0d needs something more precise: *identifying which structural property to amplify*, then amplifying it without breaking coherence or inadvertently converging with another type. Sonnet's structural reasoning is reliably stronger for this task — it distinguishes genuine structural novelty from surface variation better than haiku.

3. **S0b Round 3 as safety check on extremity.** After S0d, S0b fires one final time on the extreme versions. It checks whether the extremity push accidentally created new overlaps (e.g., two types pushed in the same "bold" direction and converged). If flagged, the less-extreme type gets one correction pass. This ensures the non-overlap guarantee from Rounds 1–2 is preserved through the extremity pass.

4. **Maximisation, not separation.** v6's adversarial rounds create diversity by separation (removing overlap). S0d creates diversity by maximisation (pushing each type to its structural extreme). These are complementary: separation ensures the types are distinct; maximisation ensures they are as far apart as possible. The combined effect is that claim types entering S1 are both non-overlapping (adversarial guarantee) and as structurally extreme as the sonnet call can produce (extremity guarantee).

5. **Direction: surprising, not merely different-sounding.** The S0d prompt explicitly instructs against two failure modes: types that *sound* different but make the same underlying structural move, and types that are extreme in rhetoric but conservative in structure ("the MOST extreme version of X" framed as a superlative but with the same logic as X). The goal is structural bets that feel unusual — a claim type that commits to an unexpected causal logic, epistemic register, or framing, not one that adds "radically" to a familiar description.

**Why this addresses the D1 ceiling:**

The 7/10 ceiling on D1 reflects the judges' assessment that LLM-adversarial separation is a real structural property but not a formally verifiable one — haiku making qualitative pairwise judgements is reliable enough to be worth doing, but not reliable enough to guarantee genuinely extreme diversity. S0d does not replace the adversarial guarantee with a formal one (that would require embedding-space verification, as the VC Stress-Tester suggested). Instead, it adds a maximisation step that *complements* the adversarial separation with an explicit push toward structural extremity — raising the floor on how bold and unusual the starting claim types are.

The key property: a claim type that makes an unusual structural bet is harder to confuse with any other type, harder to satisfy by surface variation in G1c's contrastive check, and more likely to seed a thread that genuinely explores unusual structural territory. The diversity guarantee is still as strong as the adversary's judgement (S0b/S0c), but what the adversary is now verifying *after* S0d is whether *extreme* types overlap — a harder constraint to satisfy, meaning non-overlap of extreme types implies greater structural distance than non-overlap of conservative types.

**Cost:** 1 sonnet call + 1 haiku call = ~$0.005. Negligible. Total survey overhead rises from ~$0.009 to ~$0.014 per run — still comfortably within the $5 constraint.

**What this does NOT change:**
- Survey steps S1–S5 (unchanged from v6; they operate on whatever `final_taxonomy` contains)
- Gate mechanism: ED, G1a, G1b, G1c, G2, G3 — all unchanged from v6
- v11 harness mechanisms: oracle, fingerprint monitoring (IC1–IC5), escape mechanisms, crossover, tournament, termination — all unchanged
- Step V vocabulary derivation (unchanged from v11)
- Fallback behaviour (reverts to 0.80 uniform if survey_result absent — unchanged)

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
*Loop dir: ~/clawd/landscape-coverage-autoresearch/*
