# Harness Design — v5 (Structural Crossover Fallback)
*Autoresearch escape loop — fifth iteration*
*260314*

---

## Escape trigger
Same as v2/v3/v4: 10 consecutive discards on first trigger. Counter resets to 5 on first failed gate; resets to 3 on second consecutive failed gate. *(Unchanged from v2/v3/v4.)*

## Escape mechanism

**Gated Antipode Escape with Adaptive Axis Discovery, Orthogonality-Verified Pool, and Structural Crossover Stall Fallback.**

When the escape triggers, the harness runs v2/v3/v4's full Gated Antipode Escape. The adaptive axis discovery step fires when the regression gate has failed on 2+ consecutive escape events. Newly generated axes must pass orthogonality verification (v4) before pool admission. In v5, the single change is: when the adaptive axis step produces **zero new axes** (both proposed axes dropped after retries — stall state), instead of silently proceeding with an unchanged pool, the harness activates a **Structural Crossover Escape** — a fundamentally different search strategy that abandons the axis framework entirely and recombines structural elements from the run's historical best candidates.

---

### Step 1 — Structural dissection with live axis pool *(Unchanged from v3/v4)*

A dedicated dissection agent reads the current-best and extracts structural elements from the **current axis pool**. The axis pool begins with the same three axes as v2/v3/v4:

- **Frame** — the central metaphor or conceptual lens
- **Causal claim** — the primary mechanism asserted
- **Scope** — the level of analysis in play

The axis pool is a mutable ordered list, stored in the run's working state. It starts at length 3. Axes are added and retired dynamically (see Step 5 below). The dissection agent always reads from the current pool, not the fixed original set.

---

### Step 2 — Antipode generation *(Unchanged from v3/v4)*

For each axis in the current pool, the dissection agent derives its antipode (polarity inversion or 90° rotation), yielding N antipodal seed candidates (N = pool size, minimum 3). Seeds are evaluated in parallel during the mini-sprint. If the pool has grown beyond 3 axes, the harness evaluates seeds from the 3 axes with the strongest track record (most gate-passing events) plus any newly added axes, capped at 5 seeds per escape event to contain cost.

---

### Step 3 — Extended mini-evaluation sprint *(Unchanged from v3/v4)*

Each antipodal seed is run through a **5-iteration** greedy sub-run (same iteration agent + evaluator, same frozen evaluate.md). Unchanged.

---

### Step 4 — Regression-gated winner selection *(Unchanged from v3/v4)*

The highest-scoring antipodal seed after its 5-iteration sprint must clear the regression gate:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** escape candidate replaces current-best. Discard counter resets to 0. The winning axis logs a "pass" event.
- **Gate fails:** current-best retained. Discard counter resets to 5 (or 3 on second consecutive failure). Each axis used in the failed escape logs a "fail" event.

---

### Step 5 — Adaptive axis discovery with orthogonality verification *(Unchanged from v4 — except Step 5d is new)*

**Trigger condition:** The regression gate has failed on 2+ consecutive escape events — meaning the current axis pool is generating seeds that cluster rather than escaping to genuinely different regions.

When this condition is met, the dissection agent performs an additional analysis pass **before** resetting the discard counter and resuming the main loop:

#### 5a — Evaluator feedback analysis *(Unchanged from v3/v4)*

The dissection agent reads the evaluator's score breakdowns for the most recent failed escape event. It identifies which dimensions scored lowest across all seeds in the failed escape (e.g., "D3 consistently 5/10 across all three seeds, with evaluator language referencing 'insufficient argumentative differentiation' and 'same epistemic stance'").

This is not structural analysis of the candidate itself — it is linguistic analysis of the evaluator's feedback, specifically the natural-language justifications the evaluator provided for its dimensional scores.

#### 5b — LLM-driven axis generation *(Unchanged from v3/v4)*

The dissection agent generates **2 new structural axes** by prompting itself with the following:

> "You are a structural analysis agent. The current escape pool (axes: [list]) has failed to find a viable escape candidate on [N] consecutive attempts. The evaluator's feedback on the failed seeds consistently flagged: [quoted evaluator language from weakest dimensions]. Based on this feedback, propose 2 new perturbation axes that would generate meaningfully different candidates for this specific problem. Each axis must: (1) be orthogonal to all current pool axes, (2) be derivable from the current-best without human input, (3) be general enough to apply across essay/strategy/thesis structure. Name each axis, define its antipode operation, and explain why it addresses the evaluator's flagged weaknesses."

The dissection agent's response is the canonical source of the proposed axes. No human review. The two proposed axes then enter the orthogonality verification step (5b-bis) before being admitted to the pool.

---

#### 5b-bis — Axis orthogonality verification *(Unchanged from v4)*

For each proposed axis coming out of step 5b, before appending it to the pool, the harness executes a three-part verification sequence:

**i. Probe seed generation**

The harness generates a **probe seed** for the proposed axis: a 2–3 sentence sketch of what a candidate built on this axis would look like — its essential character and structural identity, not a draft. The probe seed answers: "If I took the current-best and inverted/rotated it on this axis, what would the result feel like structurally?" It is deliberately brief — enough to convey the conceptual region the axis explores.

**ii. Orthogonality check via evaluator model**

The probe seed for the proposed axis is compared against the **probe seeds for all currently active pool axes** in a single opus call. The prompt is:

> "You are evaluating whether a proposed new perturbation axis explores a meaningfully different conceptual region from each existing axis. For each pair below (proposed axis vs. existing axis), answer: (a) Do their probe seeds describe structurally different types of candidates, or do they describe candidates that would feel like variants of each other? Answer 'orthogonal' or 'overlapping'. (b) One sentence of rationale.
>
> Proposed axis: [name + probe seed]
> Existing axes: [list of name + probe seed for each active pool axis]
>
> A proposed axis is ORTHOGONAL if candidates built on it would differ in kind from candidates built on existing axes — different argumentative gesture, different conceptual region, not just a different magnitude or emphasis of the same structural move. It is OVERLAPPING if candidates built on it would feel like variations within a region the existing pool already explores."

The call returns a verdict for each pair. The proposed axis passes if it is rated **orthogonal to all existing pool axes**. If it is rated overlapping with any existing axis, it fails.

**iii. Regeneration on failure (up to 2 retries)**

- **Pass:** The proposed axis, together with its probe seed, is appended to the pool. The probe seed is stored in the run's working state alongside the axis.
- **Fail (overlap detected):** The dissection agent regenerates the axis with an augmented prompt that includes: (a) the failing axis, (b) which existing axis it was found to overlap with, (c) the one-sentence rationale from the evaluator, and instructions to propose a replacement that avoids that conceptual region. Up to **2 regeneration attempts** per proposed axis.
- **Still failing after 2 retries:** The axis is dropped. It does not enter the pool. The adaptive step proceeds with however many axes cleared verification (potentially zero new axes if both proposed axes failed after retries).

**The probe seed store**

Each active axis in the pool carries a stored probe seed (generated or regenerated at the point of admission). The store is initialised at harness start with probe seeds for the three starting axes (frame / causal claim / scope), generated in a single call before the first iteration begins. This ensures the orthogonality check is functional from the very first adaptive step, not just once the first newly generated axis has been admitted.

The probe seed store is persisted in the run's working state alongside the axis pool. It grows as axes are added and shrinks as axes are retired. It is the harness's **map of explored conceptual regions** — a compact representation of where the pool has already been, used to refuse re-entry into those regions.

**Retired-axis probe seeds are retained in the store** even after the axis is retired. The map of explored regions is monotonically growing, even as the active pool shrinks and grows.

---

#### 5c — Axis pool management (retirement) *(Unchanged from v3/v4)*

After each escape event, each axis logs either "pass" (gate cleared) or "fail" (gate failed). An axis is **retired** from the pool when it has accumulated 3+ consecutive fail events with no intervening pass.

Retirement is permanent for the current run. If retirement would reduce the pool below 3 axes, the retirement is deferred until the adaptive step can replace it (i.e., retirement and new axis generation are coupled — you don't fall below 3).

The axis pool state (current axes, their pass/fail event history, probe seeds, retired axes) is persisted in the run's working state so it survives across iterations.

---

#### 5d — Structural Crossover Fallback *(New in v5 — the single change)*

**Trigger condition:** The adaptive axis step (5a → 5b → 5b-bis) has produced **zero new verified axes** — both proposed axes were dropped after exhausting their 2 retries each. This is the stall state: the verifier has determined that every axis the dissection agent can generate from evaluator feedback is overlapping with existing pool territory, but the existing pool is not producing viable escape candidates either.

At this point, continuing to attempt axis generation from evaluator feedback would loop indefinitely (the same information is being fed back into the same generator). The harness instead executes a **Structural Crossover Escape** — a search strategy that operates entirely outside the axis framework.

---

**What the Structural Crossover Escape does:**

The mechanism is inspired by crossover in genetic algorithms: rather than perturbing a single candidate (what the axis framework does), it *recombines structural elements* drawn from the run's strongest historical candidates to produce hybrid candidates that occupy regions no single perturbation could reach.

**Step 5d-i — Historical best retrieval**

The harness retrieves the **top-3 scoring candidates** from run history (by final score, not sprint score). These are the candidates that have cleared the regression gate on previous escape events, plus the current-best. If the run has fewer than 3 distinct gate-clearing candidates (common early in the run), the harness uses however many exist, supplemented by the current-best counted once.

**Step 5d-ii — Structural element decomposition**

A fresh decomposition call reads each of the top-3 candidates and extracts a structured inventory of **atomic structural elements** — the smallest independent units that can be recombined across candidates without breaking semantic coherence. The prompt:

> "You are a structural decomposition agent. Read the following candidate [X]. Extract its structural elements as a labelled inventory. For each element, provide: (a) the element label (e.g., 'framing device', 'causal mechanism', 'epistemic stance', 'scope of analysis', 'argumentative gesture', 'evidence grounding strategy', 'narrative arc'), (b) a 1–2 sentence description of what this candidate does specifically on that element. Be precise and concrete — not what the element category is in general, but what this specific candidate does. Output as a structured list. Do NOT evaluate quality — only describe structure."

This call is made for each of the top-3 candidates, yielding three structured inventories of elements. The element labels are not fixed — the decomposition agent may surface different labels for different candidates if the structure warrants it. Labels are aligned post-hoc across inventories by the crossover agent in step 5d-iii.

**Step 5d-iii — Crossover candidate generation**

A crossover agent reads all three inventories and generates **3 hybrid candidates** by recombining elements across inventories. The recombination is not random draw — it is **targeted recombination guided by maximising structural distance from the current-best and from each other**. The prompt:

> "You are a structural crossover agent. You have three inventories of structural elements from the run's top-3 historical candidates. Your task: generate 3 hybrid candidates. For each hybrid: (a) select one element from each major structural category, drawing from *different candidates* where possible — the goal is that each hybrid contains no two consecutive elements from the same source candidate; (b) the combination must be internally coherent — elements from different candidates must be compatible enough that the resulting hybrid is a viable, non-contradictory piece of writing/argument/strategy; (c) each hybrid must be structurally different from the current-best in at least 2 major structural dimensions; (d) the three hybrids must be structurally different from each other — not minor variations.
>
> For each hybrid, output: (1) the element sourcing table (element label → source candidate), (2) a 3–5 sentence structural sketch of the hybrid — what kind of candidate this would be, what it argues, how it frames its claim, at what scope, from what epistemic stance. Do NOT write the full candidate. The sketch is a structural blueprint for the iteration agent to develop."

The crossover agent outputs 3 structural blueprints. These blueprints are NOT generated on any named axis — they are not antipodes, not axis-framework outputs. They come from a wholly different mechanism.

**Step 5d-iv — Blueprint development and sprint**

The iteration agent develops each blueprint into a full candidate (same process as main-loop iteration), then runs each through the same **5-iteration mini-sprint** (same frozen evaluate.md). This is identical to the standard escape sprint — the only difference is that the seeds are crossover blueprints, not antipodal seeds.

**Step 5d-v — Regression-gated winner selection (same gate)**

The highest-scoring hybrid candidate after its 5-iteration sprint must clear the same regression gate:

> **Gate condition:** escape_candidate_score ≥ current_best_score × 0.80

- **Gate passes:** the hybrid candidate replaces current-best. Discard counter resets to 0. The stall event is logged. The axis pool is **unchanged** — the crossover escape is not an axis-framework operation and does not modify the pool.
- **Gate fails:** current-best retained. The stall event is logged with outcome "crossover failed". Discard counter resets to 3. The main loop resumes with the unchanged axis pool.

**What the Crossover Escape does NOT do:**
- It does **not** modify the axis pool — the pool state is completely unaffected by crossover. The crossover mechanism is parallel to the axis framework, not inside it.
- It does **not** add probe seeds to the probe seed store — crossover hybrids are not built on named axes and so have no axis identity to store.
- It does **not** fire on every escape event — it fires **only** when the stall condition is detected (zero new axes from 5d-i/ii). All normal escape events use the axis-based antipode mechanism.
- It does **not** replace the axis framework — after the crossover escape resolves (pass or fail), the main loop continues with the existing axis pool. The crossover was a one-time bypass, not a regime change.

**Why this breaks the stall loop:**

The stall state arises because the dissection agent and the orthogonality verifier are locked in a feedback loop: the dissection agent generates axes from evaluator language → verifier rejects them as overlapping → the dissection agent tries again from the same signal source. More retries cannot escape this loop — the information source is unchanged.

The crossover mechanism breaks the loop because it **does not use the axis framework or evaluator feedback as its generative signal**. Instead it uses the historical candidates themselves — specifically, the structural elements that made them score well. This is a different information source. The crossover agent is not trying to find a new axis; it is trying to find a new *region of candidate space* by recombining elements from known-good regions. This is structurally guaranteed to produce different candidates from what the axis pool can generate, because the axis pool operates on the current-best only, whereas the crossover mechanism reads across the top-3 and constructs hybrids that are simultaneously related to multiple historical bests.

**Stall fallback design properties:**
- **Does not require human intervention** — all steps are autonomous: decomposition, crossover, development, sprint, gate
- **Does not repeat the failed axis generation** — operates from a completely different signal source (historical candidate structure, not evaluator feedback language)
- **Produces meaningfully different candidates** — hybrids draw from multiple historically strong candidates across multiple structural dimensions; no axis in the pool generates this kind of candidate
- **Conservative activation** — fires only on confirmed stall (zero new axes after full retry exhaustion); normal escape events are unaffected
- **No pool contamination** — does not modify the axis pool or probe seed store; the axis-based mechanism resumes unchanged after crossover resolves
- **Bounded cost** — 3 decomposition calls + 1 crossover call + 3 × 5-iteration sprints ≈ $6–9 per stall event (comparable to a standard escape event). Stall events are rare (require 2+ consecutive gate failures AND both proposed axes dropping after retries)

---

**Properties of this mechanism (v5 additions in bold):**
- Directed — escape targets derived from current-best anatomy (inherited from v1/v2/v3/v4)
- Semantically distant — antipodes structurally guaranteed to differ on at least one major axis (inherited from v1/v2/v3/v4)
- Regression-protected — main thread cannot be replaced by inferior candidate (inherited from v2/v3/v4)
- Deeper seed development — 5-iteration sprints (inherited from v2/v3/v4)
- Adaptive — axis pool expands when fixed axes cluster; new axes LLM-derived from evaluator feedback language (inherited from v3/v4)
- Self-correcting — failed axes are retired; new axes address specifically flagged weaknesses (inherited from v3/v4)
- Domain-general — axis vocabulary derived from evaluator's language, not pre-configured (inherited from v3/v4)
- Structurally verified — newly generated axes gated on orthogonality check before pool admission (inherited from v4)
- Self-curating — persistent probe seed store refuses re-entry into exhausted regions including retired axes (inherited from v4)
- Monotonic coverage expansion — probe seed store only grows over a run (inherited from v4)
- **Stall-proof** — when the axis framework and verifier reach a feedback-loop dead end (zero new axes), the harness activates a wholly different search strategy (structural crossover from top-3 historical bests) that operates outside the axis pool and is guaranteed to produce different candidates
- **No stall-induced infinite loops** — the crossover fallback terminates in a bounded number of calls regardless of gate outcome; even if the crossover candidates fail the gate, the harness does not re-enter the stall loop (it logs the outcome and resumes the main loop)
- Autonomous — no human input at any step
- Open-ended writing/strategy compatible — crossover operates on structural element inventories derivable from any essay/strategy/thesis candidate
- Bounded cost — standard escape: 3 seeds × 5 iterations = 15 oracle evaluations (~$3–6); adaptive step with verification: ~$6–11 per event; crossover fallback (stall only): ~$6–9 per stall event; stall events are rare and bounded. Well under $250.

---

## Population structure
Single thread. One current-best candidate at all times. One iteration agent, one evaluator, sequential. *(Unchanged from v1/v2/v3/v4.)*

## Perturbation direction logic
Direction during normal escape is determined by antipode derivation from the current axis pool (unchanged from v3/v4). Direction during crossover fallback is determined by structural element recombination from the top-3 historical bests — a separate signal source that does not interact with the axis pool. Main-loop perturbation direction (evaluator's "SUGGESTED DIRECTION" guidance) unchanged from v1/v2/v3/v4.

## Re-integration
The escape candidate replaces the current-best *only if it clears the regression gate* (≥ 80% of current-best score). *(Unchanged from v2/v3/v4.)* The adaptive axis step (including orthogonality verification) fires *before* the loop resumes. The crossover fallback fires inside the adaptive step, only when the stall condition is detected, and resolves before the loop resumes.

## Estimated cost per run
~$28–42 for a standard 30-iteration run with 1–2 escape events. Stall events are rare (require 2+ consecutive gate failures AND both proposed axes dropping after full retries). Each stall event adds ~$6–9 (3 decomposition calls + 1 crossover call + 3 mini-sprints). In runs without a stall event, cost is identical to v4. Well under $250.

---

## What changed this iteration (v4 → v5)

**Element changed:** Stall fallback mechanism — Structural Crossover Escape (Step 5d). Addresses D2 (Escape reliability).

---

### The change — Structural Crossover Stall Fallback (Step 5d)

**v4 state:** When the orthogonality verifier dropped both proposed axes after retries (stall state), the adaptive step produced zero new axes and the harness continued with the unchanged pool. This was a dead-end: the same axis pool that had already failed to produce viable escape candidates would be used for the next escape event, with no mechanism to break out of the failure mode. The dissection agent would attempt to generate new axes from evaluator feedback again — but it was receiving the same evaluator feedback from the same kinds of failed seeds, guaranteeing the same generation failure. True infinite loop.

**v5 change:** When the stall condition is confirmed (zero new axes from the full axis discovery + verification pipeline), the harness activates the **Structural Crossover Escape** before resuming the main loop.

The crossover mechanism is architecturally distinct from the axis framework in three ways:

1. **Different signal source** — the axis framework generates new directions from evaluator feedback language (what the evaluator said was weak). The crossover mechanism draws from the structural elements of the top-3 historical best candidates (what has historically scored well). These are opposite information sources: failure signal vs. success signal.

2. **Different operation** — the axis framework works by perturbation: take the current-best, identify an axis, invert or rotate on that axis. The crossover mechanism works by recombination: decompose multiple strong candidates into elements, recombine elements across candidates to form hybrids. Recombination can reach regions that no single perturbation, on any axis, would reach — because the hybrids are simultaneously related to multiple historical bests in multiple dimensions.

3. **No axis required** — crossover-generated candidates are not built on any named axis. They do not affect the axis pool or probe seed store. The axis framework resumes unchanged after the crossover resolves.

The design is deliberately analogous to crossover in genetic algorithms: when gradient-based local search stalls, switching to population-based recombination accesses a different region of the search landscape. The crossover fallback brings this property to the single-thread autoresearch loop without requiring persistent parallel populations — it is a temporary, triggered regime switch, not a permanent architecture change.

---

### Why this addresses the evaluator's weakest element (D2 — Escape reliability)

The v4 evaluators (all three judges) flagged the same failure mode: "both proposed axes fail verification → pool unchanged → no fallback → potential infinite stall." The v4 mechanism was self-described as having "no path forward" in this state.

V5 adds a guaranteed path forward. The crossover fallback:
- **Always terminates** — bounded calls regardless of outcome (no loop)
- **Never repeats the failed generator** — uses a completely different information source
- **Produces genuinely different candidates** — structural recombination from historical bests is not a variant of antipode perturbation
- **Does not depend on the stalled mechanism** — the crossover operates whether or not the axis pool grows, whether or not the verifier is strict

The one residual escape failure mode after v5: a crossover escape that fails the regression gate. In this case, the harness resumes with the unchanged pool and existing discard counter reset — it is not stuck (the main loop continues), but it has not found a better candidate. This is an honest failure (no viable candidate found in this escape event) rather than a loop failure (mechanism broken). The distinction matters: honest failure is bounded and recoverable; loop failure is not.

---

### What this does NOT change:
- Escape trigger (still 10 consecutive discards on first trigger; counter resets to 5/3 on failed gate)
- Starting axis pool (frame / causal claim / scope — still the initial set)
- Axis generation mechanism (still LLM-driven from evaluator feedback language — unchanged from v3/v4)
- Orthogonality verification (unchanged from v4)
- Probe seed store (unchanged from v4; crossover does not write to it)
- Antipode derivation logic (inversion or 90° rotation — unchanged)
- Mini-sprint length (5 iterations — unchanged from v2/v3/v4)
- Regression gate threshold (0.80 — unchanged from v2/v3/v4)
- Progressive counter shortening on gate failure (unchanged from v2/v3/v4)
- Axis retirement logic (3+ consecutive fails — unchanged from v3/v4)
- Population structure (single thread — unchanged)
- Main-loop perturbation direction (evaluator-guided — unchanged)
- evaluate.md (frozen throughout — unchanged)

---

*Autoresearch loop: autoresearch-escape | Started: 260314 | Loop dir: ~/clawd/autoresearch-escape/*
