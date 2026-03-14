# Coverage Mechanism — v5 (Exploration Directive)
*Landscape coverage mechanism — v5*
*260314*

---

## Survey design (UNCHANGED from v4)

### Overview

A pre-run structural landscape survey fires once at run start (after Step V vocabulary derivation, before thread spawning). The survey generates 4 structurally extreme candidate claim types via an adversarial construction process, then generates one sketch per type, clusters them into 3 guaranteed non-overlapping regions, and seeds T1, T2, T3 from those regions.

**Core design principle (UNCHANGED from v4):** Structural diversity is constructed through conflict. The S0 adversarial process generates 4 initial claim types, then fires an adversary that identifies pairs which could produce structurally similar candidates. Flagged types are replaced with more extreme versions that make overlap impossible. Diversity is a product of adversarial pressure, not cooperative prompting.

**What changes from v4:** The gate mechanism gains one new step — ED (Exploration Directive) — that fires before each candidate generation and injects a directional hint into the iteration agent's prompt. S0a/S0b/S0c, S1–S5, and G1–G3 are all unchanged.

All steps use haiku-bedrock. Survey total cost: ~$0.015, well within the $5 constraint.

---

### Step S0a — Seed taxonomy generation

**UNCHANGED from v4.**

---

### Step S0b — Adversarial overlap challenge

**UNCHANGED from v4.**

---

### Step S0c — Adversarially-informed revision

**UNCHANGED from v4.**

---

### Final taxonomy commit

**UNCHANGED from v4.**

---

### Step S1 — Sketch generation

**UNCHANGED from v4.**

---

### Step S2 — Structural verification (Layer 2 guarantee)

**UNCHANGED from v4.**

---

### Step S3 — 2-dimension scoring

**UNCHANGED from v4.**

---

### Step S4 — Structural clustering into 3 regions (Layer 3 guarantee)

**UNCHANGED from v4.**

---

### Step S5 — Thread seed selection

**UNCHANGED from v4.** `survey_result` stores both STRUCTURAL_SIGNATURE (short descriptor, used in G1 and ED soft nudge) and STRUCTURAL_SEED (full 3-sentence sketch from S1, used in ED hard redirect) for each cluster.

---

### Survey cost summary

**UNCHANGED from v4.** ~$0.009 per run.

---

## Gate mechanism (UPDATED — ED step added)

### Overview

The uniform 0.80 × current-best threshold is replaced with a region-aware diversity gate. The gate reads `survey_result` cluster signatures and assigns each incoming candidate to its nearest cluster via a single haiku-bedrock call per candidate (G1). The acceptance threshold adjusts based on that cluster's exploration status (G2, G3).

**New in v5:** Step ED (Exploration Directive) fires at the START of each thread iteration, BEFORE the thread generates a candidate. It reads the current cluster's exploration status from `survey_result` and injects a directional hint into the thread's generation prompt. No model call. Zero cost.

**The gate and the directive work together.** G3 creates pull — underexplored regions have lower acceptance thresholds. ED creates push — it tells the thread where to aim, not just what will be rewarded. Combined: pull + push for unexplored territory.

---

### Step ED — Exploration directive injection (NEW in v5)

**Fires:** At the start of each thread iteration, before the thread generates a candidate. Uses only `survey_result` global state. No model call.

**Purpose:** Actively steer the thread toward underexplored structural territory. The gate adjusts what gets accepted; the directive adjusts what gets generated. Without the directive, a thread operating in an EXHAUSTED cluster faces high rejection pressure but no instruction to look elsewhere — it keeps generating in its current region and keeps failing the gate. The directive closes this gap.

**Inputs (from `survey_result`):**
- `thread.assigned_cluster_id` — the cluster this thread was seeded from
- `cluster[id].status` — UNEXPLORED / ACTIVE / EXHAUSTED for each cluster
- `cluster[id].candidate_count` — number of accepted candidates so far
- `cluster[id].STRUCTURAL_SIGNATURE` — short descriptor (stored from S4; already used in G1)
- `cluster[id].STRUCTURAL_SEED` — full 3-sentence sketch (stored from S5 thread seeding)

**Least-explored cluster resolution (deterministic):**

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
| ACTIVE | Soft nudge | Structural signature of least-explored cluster appended to prompt |
| EXHAUSTED | Hard redirect | Full structural seed of least-explored cluster replaces prompt context |

---

**UNEXPLORED — no directive:**

The thread has just been seeded into a fresh region. No modification to its generation prompt. It is already pointing at good territory.

---

**ACTIVE — soft nudge:**

Append the following block to the thread's current generation prompt, after its existing context:

> Note: this structural territory is currently underexplored:
>
> [{STRUCTURAL_SIGNATURE of least-explored cluster}]
>
> You may continue generating in your current direction, but candidates that work within the territory above will be evaluated with a lower acceptance threshold.

**Why a nudge, not a redirect:** An ACTIVE thread still has productive territory remaining. Forcing a hard redirect would waste exploration potential in its current cluster. The soft nudge makes underexplored territory visible and signals that it is rewarded — the thread may naturally begin drifting that way, or it may stay in its current region. Both outcomes are acceptable at ACTIVE status.

---

**EXHAUSTED — hard redirect:**

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

**Why a hard redirect, not a nudge:** An EXHAUSTED thread has exhausted its local region. Continued generation there produces candidates with high structural similarity to existing ones — they will fail the gate at the EXHAUSTED threshold (0.90 × current-best) and will not meaningfully expand coverage. The directive must be strong enough that the thread understands it is being redirected, not merely advised. The full structural seed — a committed 3-sentence sketch — gives the thread a concrete target, not an abstract instruction.

**Hard redirect includes the full seed (not just the signature) because:**
- The signature (S4 output) is a short descriptor optimised for G1 classification — it identifies the cluster's structural territory but does not show the thread what a good candidate in that territory looks like
- The seed (S5 output) is a committed 3-sentence sketch — it models the logical move the thread should make, including core claim, key mechanism, and expected answer shape
- A thread being redirected cold needs a concrete example to aim at; a label alone is insufficient

---

**Prompt injection mechanics:**

The directive is injected at the harness level, not by the thread itself. The harness reads `survey_result` state, computes the directive, and assembles the final prompt before passing it to the iteration agent. The iteration agent receives one prompt per iteration — it does not observe that a directive has been prepended or that its context has changed. From the iteration agent's perspective, it always receives a single generation prompt.

**This is a prompt assembly operation, not a model call.** The directive text is template-filled from stored state. The harness holds both the template strings (defined once at harness initialisation from this spec) and the per-run `survey_result` data. Assembly is O(1) per iteration.

---

**State tracking for ED:**

`survey_result` already tracks `candidate_count` per cluster (updated by G3 on each accepted candidate). ED reads this state — it does not write to it. No new state fields required.

---

**Cost:** Zero. No model call. Template fill from existing `survey_result` state. Total ED overhead per run: negligible (O(iteration count) string operations, no API calls).

---

### Step G1 — Candidate cluster assignment

**UNCHANGED from v4.** Single haiku-bedrock call per candidate. Operates on the candidate the thread has generated (which may now have been steered by ED). Returns CLUSTER_ID, RATIONALE, BOUNDARY_CASE.

**Expected interaction with ED:** When ED fires a hard redirect, the thread generates a candidate aimed at the least-explored cluster's territory. G1 should assign this candidate to the least-explored cluster — the structural seed it was given came directly from that cluster's S5 seed. If G1 assigns it elsewhere (BOUNDARY_CASE = YES), the gate applies the least-explored cluster's threshold regardless (existing BOUNDARY_CASE tie-break rule: assign to less-explored cluster). No modification to G1 required.

---

### Step G2 — Cluster status classification

**UNCHANGED from v4.** Deterministic state lookup. Returns UNEXPLORED / ACTIVE / EXHAUSTED.

---

### Step G3 — Threshold computation and gate decision

**UNCHANGED from v4.**

| Cluster status | Acceptance threshold |
|---|---|
| UNEXPLORED | **0.65 × current-best** |
| ACTIVE | **0.80 × current-best** |
| EXHAUSTED | **0.90 × current-best** |

---

### Anti-gaming property

**UNCHANGED from v4.** Adversarially-hardened cluster signatures resist surface-diverse but structurally-similar candidates. The ED hard redirect, when active, points the thread at a seed derived from the same adversarial hardening process — the target territory is structurally genuine, not a surface restatement of the exhausted region.

---

### Convergence safety

**UNCHANGED from v4.** Four safeguards (ACTIVE floor, exhausted ceiling, no infinite loop, unassigned fallback) apply identically.

**Additional ED safety property:** The hard redirect points the thread at the least-explored cluster's structural seed — a seed that was selected by S5 as the best sketch in that cluster (`seed_score = plausibility × resolution_potential`). The thread is being redirected toward genuinely productive territory, not a random structural direction. This prevents the redirect from destabilising convergence by sending threads into structurally incoherent regions.

**ED does not fire on a thread's first iteration.** Thread initialisation already uses the cluster seed as the generation prompt (from S5). ED only activates from iteration 2 onward, once cluster status has begun to differentiate.

---

## Integration (UPDATED — ED step added)

**Updated survey execution order (UNCHANGED from v4):**
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
11. **S5** — Thread seed selection (unchanged); `survey_result` stores both STRUCTURAL_SIGNATURE and STRUCTURAL_SEED per cluster
12. Thread spawning from cluster seeds — unchanged from v11

**Updated gate execution order per candidate (ED ADDED):**
1. **[NEW] ED: Harness reads thread's assigned cluster status from `survey_result`**
2. **[NEW] ED: Computes least-explored cluster (pure state — no model call)**
3. **[NEW] ED: Assembles generation prompt (no directive / soft nudge / hard redirect based on cluster status)**
4. Iteration agent generates candidate (receives assembled prompt)
5. Oracle scores candidate
6. G1: haiku cluster assignment
7. G2: cluster status classification
8. G3: threshold computation + gate decision
9. If gate passes: enter tournament pool; `candidate_count` for assigned cluster incremented
10. If gate fails: discard; thread continues (loop to step 1)

**No change to oracle, fingerprint monitoring (IC1–IC5), escape mechanisms (Steps 1–6d), crossover (Step 6), tournament (Step SD), Step V, thread parallelism, termination condition, or fallback behaviour.**

---

## Cost estimate (UPDATED)

Base cost unchanged from v11: ~$91–146 per run.

**Survey overhead (UNCHANGED from v4):** ~$0.009 per run.

**Gate overhead (UNCHANGED from v4):** ~$0.27 per run (270 candidates × ~$0.001 per G1 call).

**ED overhead (NEW):** Zero. No model call. O(iteration count) string operations.

**Updated total mechanism overhead per run:** ~$0.009 (survey) + ~$0.27 (gate) + ~$0.00 (ED) = **~$0.28 per run** (functionally identical to v4 — ED adds no cost).

**Budget check:** $5.00 constraint. $0.28 overhead. $4.72 headroom. Unchanged.

---

## Domain generality (UNCHANGED from v4)

- Exploration directive templates are domain-agnostic — they reference structural seeds and signatures, not domain content
- STRUCTURAL_SEED and STRUCTURAL_SIGNATURE derive from the problem-specific adversarial taxonomy (S0a → hardening), not from domain-specific configuration
- The three directive types (none / soft nudge / hard redirect) and their trigger conditions (UNEXPLORED / ACTIVE / EXHAUSTED) are domain-independent
- G1–G3, S0a–S0c, S1–S5 are domain-agnostic throughout (unchanged)

---

## What changed this iteration (v4 → v5)

**Element changed:** Gate mechanism only — one new step (ED, Exploration Directive). All other gate steps (G1, G2, G3) unchanged from v4. Survey (S0a/S0b/S0c, S1–S5) unchanged from v4. All v11 harness mechanisms preserved.

---

### The change — Exploration directive: push to complement the gate's pull

**v4 state:** The gate (G1–G3) adjusts acceptance thresholds based on cluster exploration status. An EXHAUSTED cluster faces a 0.90 × current-best threshold — high pressure, low acceptance rate. But the iteration agent generating candidates in that cluster receives no instruction to look elsewhere. It continues generating variations of its current region, accumulating rejections, and spending iteration budget on candidates that will almost certainly fail. The gate pulls toward unexplored territory (by rewarding it); nothing pushes the thread toward it. Pull without push is slow — the thread has to discover underexplored territory through rejection pressure alone.

**v5 change:** Step ED (Exploration Directive) fires at the start of each thread iteration, before generation. It is a prompt assembly operation — no model call, zero cost — that reads `survey_result` cluster state and injects a directional hint into the iteration agent's prompt.

**Three directive types, matched to cluster status:**

**UNEXPLORED — no directive.** The thread is already in good territory. No modification. Preserves the seeding logic unchanged.

**ACTIVE — soft nudge.** The least-explored cluster's STRUCTURAL_SIGNATURE is appended to the thread's prompt as additional context: "this other territory is underexplored — [signature]." The thread may drift toward it or continue in its current region. Both are acceptable. The thread is being made aware, not commanded.

**EXHAUSTED — hard redirect.** The least-explored cluster's full STRUCTURAL_SEED (3-sentence sketch from S5) replaces the thread's structural context. The language is direct: "Your current region has been thoroughly explored. Do not generate variations of previous candidates. Generate a candidate that explores THIS structural territory: [seed]. Aim at this territory, not your previous one."

**Why the EXHAUSTED directive uses the full seed, not the signature:**
The STRUCTURAL_SIGNATURE is a short descriptor optimised for G1 classification — it identifies the territory but does not model what a good candidate there looks like. A thread being redirected cold needs a concrete target. The STRUCTURAL_SEED is a committed 3-sentence sketch showing the logical move, core claim, mechanism, and expected answer shape. It gives the thread something to aim at. A label without an example is an insufficient redirect.

**Why the EXHAUSTED directive is strong, not hedged:**
An EXHAUSTED thread has reached the end of productive territory in its current region. Soft language ("consider exploring") would be absorbed as background context and have minimal effect — the thread's generation inertia (its existing context and prior candidates) would dominate. The directive must be strong enough to override that inertia. The language is explicit: you are being redirected, not advised.

**The gate and the directive are complementary, not redundant:**
- Gate (pull): rewards exploration by lowering the acceptance threshold for underexplored regions
- Directive (push): steers generation toward underexplored regions before the candidate is created

Pull alone requires the thread to stumble into unexplored territory and be rewarded. Push alone without reward could create noise — threads generating off-target candidates that then fail the gate. Together, they create a coherent exploration incentive: the thread is directed toward a region, and when it arrives there, the gate accepts its candidates more readily.

**Why this addresses the D2 failure mode directly:**

The VC Stress-Tester scored D2 at 6/10 for v4: "The gate still relies on per-candidate haiku classification and fixed threshold tiers; there is no structural property guaranteeing that the gate provably shifts thread trajectories away from explored regions rather than applying a threshold adjustment that evaluation noise can overwhelm."

The exploration directive addresses this by operating at the generation level, not the acceptance level. A thread that has received a hard redirect is not hoping to stumble onto rewarded territory — it is explicitly aimed there before generating. The trajectory shift is structural (what the thread generates) not statistical (what the gate accepts). Combined with the gate's acceptance reward for that territory, the trajectory shift is reinforced.

Karpathy's implicit concern — that a gate relying solely on threshold adjustment cannot provably shift trajectories — is addressed by adding a mechanism that operates upstream of generation. The gate becomes one half of a two-part system: acceptance reward + directional push.

**What this does NOT change:**
- Survey (S0a/S0b/S0c, S1–S5 — unchanged from v4)
- Gate logic (G1 cluster assignment, G2 status classification, G3 thresholds — unchanged from v4)
- Oracle (frozen, shared — unchanged)
- Fingerprint monitoring (IC1–IC5 — unchanged from v11)
- Escape mechanisms (Steps 1–6d — unchanged from v11)
- Tournament structure (Steps SD, T1–T5 — unchanged from v11)
- Step V vocabulary derivation (unchanged from v11)
- Crossover (Step 6 — unchanged from v11)
- Termination condition (unchanged)
- Fallback behaviour (reverts to 0.80 uniform if survey_result absent — ED does not fire if survey_result is absent)

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
*Loop dir: ~/clawd/landscape-coverage-autoresearch/*
