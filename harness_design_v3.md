# Harness Design — v3 (Adaptive Axis Discovery)
*Autoresearch escape loop — third iteration*
*260314*

---

## Escape trigger
Same as v2: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2.)*

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery.**

When the escape triggers, the harness runs v2's full Gated Antipode Escape. A new upstream step fires when the axis pool has been exhausted or the gate has failed 2+ consecutive escape events: the dissection agent actively generates new structural axes — LLM-driven, derived from the evaluator's feedback language — expanding the perturbation vocabulary beyond the starting set of frame / causal claim / scope.

---

### Step 1 — Structural dissection with live axis pool *(modified from v2)*

A dedicated dissection agent reads the current-best and extracts structural elements from the **current axis pool**. The axis pool begins with the same three axes as v2:

- **Frame** — the central metaphor or conceptual lens
- **Causal claim** — the primary mechanism asserted
- **Scope** — the level of analysis in play

The axis pool is a mutable ordered list, stored in the run's working state. It starts at length 3. Axes are added and retired dynamically (see Step 5 below). The dissection agent always reads from the current pool, not the fixed original set.

---

### Step 2 — Antipode generation *(Unchanged from v2)*

For each axis in the current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record (most gate-passing events) plus any newly added axes, capped at 5 seeds per escape event to contain cost.

---

### Step 3 — Extended mini-evaluation sprint *(Unchanged from v2)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Unchanged from v2)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** escape candidate replaces current-best. Discard counter resets to 0. The winning axis logs a "pass" event.
- **Gate fails:** current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used in the failed escape logs a "fail" event.

---

### Step 5 — Adaptive axis discovery *(New mechanism — the single change in v3)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events — meaning the current axis pool is generating seeds that cluster rather than escaping to genuinely different regions.

When this condition is met, the dissection agent performs an additional analysis pass **before** resetting the discard counter and resuming the main loop:

#### 5a — Evaluator feedback analysis

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event. It identifies which dimensions scored lowest across all seeds in the failed escape (e.g., "D3 consistently 5/10 across all three seeds, with evaluator language referencing 'insufficient argumentative differentiation' and 'same epistemic stance'").

This is not structural analysis of the candidate itself — it is linguistic analysis of the evaluator's feedback, specifically the natural-language justifications the evaluator provided for its dimensional scores.

#### 5b — LLM-driven axis generation

The dissection agent generates **2 new structural axes** by prompting itself with the following:

> "You are a structural analysis agent. The current escape pool (axes: [list]) has failed to find a viable escape candidate on [N] consecutive attempts. The evaluator's feedback on the failed seeds consistently flagged: [quoted evaluator language from weakest dimensions]. Based on this feedback, propose 2 new perturbation axes that would generate meaningfully different candidates for this specific problem. Each axis must: (1) be orthogonal to all current pool axes, (2) be derivable from the current-best without human input, (3) be general enough to apply across essay/strategy/thesis structure. Name each axis, define its antipode operation, and explain why it addresses the evaluator's flagged weaknesses."

The dissection agent's response is the canonical source of the new axes. No human review. The two proposed axes are appended to the axis pool immediately.

**Examples of LLM-generated axes (illustrative, not pre-configured):**

*If evaluator flags D3 weakness as "same epistemic register across all seeds":*
→ The agent might generate: **Epistemic stance** (assertion → inquiry; or authoritative → speculative) and **Evidence type** (quantitative → qualitative; or historical → predictive).

*If evaluator flags D1 weakness as "seeds all operate at the same scale of analysis":*
→ The agent might generate: **Protagonist framing** (individual actor → systemic force → emergent property) and **Temporal horizon** (near-term effect → second-order consequence → regime change).

*If evaluator flags D2 weakness as "escapes feel like variations, not inversions":*
→ The agent might generate: **Narrative voice** (descriptive → prescriptive → diagnostic) and **Problem framing** (deficit to be solved → tension to be managed → feature to be exploited).

These are discovery examples only. In practice, the axes are generated entirely from the evaluator's feedback language. The harness has no pre-configured vocabulary of potential axes.

#### 5c — Axis pool management (retirement)

After each escape event, each axis logs either "pass" (gate cleared) or "fail" (gate failed). An axis is **retired** from the pool when it has accumulated 3+ consecutive fail events with no intervening pass.

Retirement is permanent for the current run. If retirement would reduce the pool below 3 axes, the retirement is deferred until the adaptive step can replace it (i.e., retirement and new axis generation are coupled — you don't fall below 3).

The axis pool state (current axes, their pass/fail event history, retired axes) is persisted in the run's working state so it survives across iterations.

---

**Why this mechanism is bolder than human-configured axis expansion:**

Human-configured fallback axes (e.g., a static list of "backup perturbation types") are frozen at design time. They cannot respond to the specific problem the loop is working on. A harness for thesis development and one for investment strategy would need different fallback vocabularies — undermining generality.

LLM-driven axis generation inverts this: the dissection agent reads the evaluator's actual language and proposes axes *derived from what the evaluator found weak in this specific run*. If the evaluator has been flagging "same epistemic stance" for three consecutive escape events, the adaptive step surfaces that pattern and constructs axes that address it directly. The harness reasons about its own failure modes and corrects them without requiring the designer to anticipate those modes.

This is what makes the mechanism domain-general. The evaluator's feedback language is always problem-specific (it describes weaknesses in the current candidate, in plain language). The dissection agent's axis generation reads that language. Therefore the generated axes are always specific to the current problem, regardless of domain.

---

**Properties of this mechanism (v3 additions in bold):**
- Directed — escape targets derived from current-best anatomy (inherited from v1/v2)
- Semantically distant — antipodes structurally guaranteed to differ on at least one major axis (inherited from v1/v2)
- Regression-protected — main thread cannot be replaced by inferior candidate (inherited from v2)
- Deeper seed development — 5-iteration sprints (inherited from v2)
- **Adaptive** — axis pool expands when fixed axes cluster; new axes are LLM-derived from evaluator feedback language
- **Self-correcting** — failed axes are retired; new axes are generated to address the specific weaknesses the evaluator flagged
- **Domain-general** — axis vocabulary is not pre-configured; it is always derived from the evaluator's language about the current problem
- Autonomous — no human input at any step (all steps including axis generation fire automatically)
- Open-ended writing/strategy compatible — axis generation operates on evaluator feedback language, not domain-specific features
- Bounded cost — standard escape: 3 seeds × 5 iterations = 15 oracle evaluations (~$3–6); adaptive step adds 1 dissection-agent call (~$0.50); new axes add at most 2 additional seed evaluations per escape event (~$1–2 extra); total per escape event with adaptation: ~$5–9 at opus-bedrock rates

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. *(Unchanged from v1/v2.)*

## Perturbation direction logic
Direction during escape is determined by antipode derivation from the current axis pool (now mutable — see Step 5). Main-loop perturbation logic (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1/v2.

The axis pool is the only new state that perturbation direction logic reads. It does not change how antipodes are derived — only which axes they are derived from.

## Re-integration
The escape candidate replaces the current-best *only if it clears the regression gate* (≥ 80% of current-best score). *(Unchanged from v2.)* The adaptive axis step fires *before* the loop resumes, so new axes are available on the next escape event, not the current one.

## Estimated cost per run
~$26–35 for a standard 30-iteration run with 1–2 escape events (vs ~$24–30 in v2). The adaptive step adds ~$1.50–3 per triggered event (1 dissection call + up to 2 additional seed evaluations). Adaptive step fires only when gates fail consecutively — it does not add cost on successful escapes. Well under $250.

---

## What changed this iteration (v2 → v3)

**Element changed:** Perturbation direction logic — D1 (Landscape coverage). A single structural addition to the escape mechanism: adaptive axis discovery.

---

### The change — Adaptive axis discovery

**v2 state:** The axis pool was fixed at 3 structural elements (frame / causal claim / scope), defined at design time and unchanged throughout the run. When all 3 antipodal seeds failed the regression gate, the harness had no mechanism for discovering new directions. It could only re-fire the same 3 axes under progressively shorter escape intervals.

**v3 change:** When the gate fails on 2+ consecutive escape events (indicating that the current axis pool is generating clustered seeds, not genuinely different regions), the dissection agent performs an adaptive step:

1. Reads the evaluator's feedback on the failed seeds' weakest dimensions
2. Generates 2 NEW structural axes, LLM-driven, derived from the evaluator's natural-language feedback
3. Appends those axes to the pool for use in the next escape event
4. Retires axes with 3+ consecutive fail events (preventing dead-weight axes from persisting)

The axis generation is fully LLM-driven — the dissection agent actively proposes the new axes by reasoning over the evaluator's language. There is no pre-configured vocabulary of fallback axes.

---

### Why this addresses D1 (Landscape coverage)

V2's evaluator feedback (all three judges, D1 scores 6–7/10) was consistent: the 3-axis pool bounds coverage. The harness can find different basins only if those basins are reachable by inverting frame, causal claim, or scope. For writing/strategy/thesis problems, the landscape is larger than 3 axes — epistemic stance, evidence type, protagonist framing, temporal horizon, narrative register, and others are all genuinely distinct structural axes that a fixed 3-axis pool will never explore.

V3 addresses this without pre-configuring a larger fixed axis list (which would be domain-specific and require designer knowledge of the target domain). Instead, the harness discovers new axes from the evaluator's own feedback — which is always problem-specific, always in plain language, and always available. The result is a landscape scanner that expands its own vocabulary based on what the oracle reports as missing, rather than relying on what the designer anticipated would be missing.

---

### What this does NOT change:
- Escape trigger (still 10 consecutive discards on first trigger; counter resets to 5/3 on failed gate)
- Starting axis pool (frame / causal claim / scope — still the initial set)
- Antipode derivation logic (inversion or 90° rotation — unchanged)
- Mini-sprint length (5 iterations — unchanged from v2)
- Regression gate threshold (0.80 — unchanged from v2)
- Progressive counter shortening on gate failure (unchanged from v2)
- Population structure (single thread — unchanged)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- evaluate.md (frozen throughout — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
