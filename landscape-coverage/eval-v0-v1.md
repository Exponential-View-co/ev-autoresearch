# Evaluation — v0 (null baseline) vs v1 (Structural Survey Seeding)
*Evaluator run — 260314*
*Rubric: evaluate.md v1.0*

---

## v0 — Null Baseline

### JUDGE: Karpathy
D1 (Coverage expansion): 3/10 — Axis labels provide lexical variation but no structural guarantee; all three threads can converge on the dominant conceptual peak without any mechanism to prevent it.
D2 (Diversity maintenance): 2/10 — Uniform gate is landscape-blind; zero incentive to explore unexplored regions over exploiting known peaks.
D3 (Integration coherence): 5/10 — No mechanism to integrate means nothing breaks, but nothing is gained either.
D4 (Generality): 1/10 — No mechanism exists to be domain-general or domain-specific.
Geometric mean: (3 × 2 × 5 × 1)^(1/4) = 30^0.25 = **2.34**

### JUDGE: EV Platform Strategist
D1 (Coverage expansion): 3/10 — Abstract axis labels ("central frame", "causal mechanism", "epistemic register") are conceptually distinct but structurally underdetermined; they offer no guarantee that the threads actually explore different regions.
D2 (Diversity maintenance): 2/10 — Uniform 0.80 threshold treats explored and unexplored regions identically; evaluation noise will swamp any accidental diversity benefit.
D3 (Integration coherence): 6/10 — Zero integration burden; existing v11 mechanisms run unmodified because there is nothing new to interact with.
D4 (Generality): 1/10 — No mechanism exists.
Geometric mean: (3 × 2 × 6 × 1)^(1/4) = 36^0.25 = **2.45**

### JUDGE: VC Stress-Tester
D1 (Coverage expansion): 3/10 — The user gets no structural coverage guarantee; any exploration that happens is accidental, not by design.
D2 (Diversity maintenance): 2/10 — Gate is landscape-blind; the harness will always exploit rather than explore once a peak is found.
D3 (Integration coherence): 5/10 — Nothing to conflict with, nothing to enhance.
D4 (Generality): 1/10 — N/A — no mechanism.
Geometric mean: (3 × 2 × 5 × 1)^(1/4) = 30^0.25 = **2.34**

### v0 FINAL SCORE: mean(2.34, 2.45, 2.34) = **2.38**

---

## v1 — Structural Survey Seeding

### JUDGE: Karpathy
D1 (Coverage expansion): 7/10 — The three-layer guarantee (taxonomy with CANNOT_BE_MERGED_WITH constraints at S0, structural drift auditing at S2, pairwise non-overlap audit with minimum-specificity validation at S4) is a genuine structural property that prevents clustering by construction. The guarantee still depends on haiku's structural reasoning fidelity, which caps it at 7 — but this is the first mechanism that makes a constructive diversity claim rather than hoping for it.
D2 (Diversity maintenance): 2/10 — Gate is unchanged from v0; uniform 0.80 threshold, landscape-blind. The survey guarantees diverse starts but provides zero ongoing pressure to maintain diversity during the run.
D3 (Integration coherence): 8/10 — Clean drop-in after Step V, before thread spawning. All existing mechanisms (escape, fingerprint monitoring, tournament, crossover, vocabulary) preserved and operate identically. Graceful fallback to v0 on survey failure. Survey output stored in global state for future mechanisms to consume.
D4 (Generality): 8/10 — Fully derived from problem statement via S0 taxonomy derivation. No domain-specific templates, labels, or human configuration. Works identically for writing/thesis, investment, product, and technical problems.
Geometric mean: (7 × 2 × 8 × 8)^(1/4) = 896^0.25 = **5.47**

### JUDGE: EV Platform Strategist
D1 (Coverage expansion): 6/10 — The survey design is well-structured and the three verification layers are sensible, but S2 structural verification relies on haiku's ability to detect structural drift — a task that requires genuine reasoning about argument structure. Haiku is good but not infallible; subtle structural conflation will sometimes pass the audit.
D2 (Diversity maintenance): 2/10 — Gate unchanged; no diversity maintenance improvement whatsoever. The survey data structure (survey_result with cluster assignments) is stored and available, but nothing reads it during the run.
D3 (Integration coherence): 9/10 — This is how you add a mechanism to a complex system. Single insertion point, no existing mechanism changes, explicit fallback path, stored data structure that future mechanisms can consume without retroactive changes. The interaction analysis with fingerprinting, escape, crossover, and tournament is thorough and correct.
D4 (Generality): 8/10 — No domain configuration needed. Taxonomy derivation from problem statement is the correct design — avoids the trap of universal claim-type libraries that don't fit any specific problem well.
Geometric mean: (6 × 2 × 9 × 8)^(1/4) = 864^0.25 = **5.42**

### JUDGE: VC Stress-Tester
D1 (Coverage expansion): 6/10 — Better than v0 but the user's final output improvement depends on whether structural seeding diversity persists through iterations. The survey guarantees diverse starts, but threads could still converge mid-run — and the unchanged gate does nothing to prevent that. The user gets a better spread of initial directions, but that spread is not maintained.
D2 (Diversity maintenance): 2/10 — No gate change means no ongoing diversity pressure. This is the single largest gap between what v1 promises (landscape coverage) and what it delivers (landscape seeding only).
D3 (Integration coherence): 8/10 — Clean integration, no breakage risk, sensible fallback. The only reason this isn't a 9 is that the survey_result data structure is designed for a future gate that doesn't exist yet — slightly speculative coupling.
D4 (Generality): 7/10 — Domain-general by design, but the claim is untested. The taxonomy derivation approach is sound in principle; the question is whether haiku consistently produces structurally meaningful (not just lexically varied) taxonomies across domains.
Geometric mean: (6 × 2 × 8 × 7)^(1/4) = 672^0.25 = **5.09**

### v1 FINAL SCORE: mean(5.47, 5.42, 5.09) = **5.33**

---

## Summary

| Version | Karpathy | EV Strategist | VC Stress-Tester | Final |
|---|---|---|---|---|
| v0 (null baseline) | 2.34 | 2.45 | 2.34 | **2.38** |
| v1 (Structural Survey Seeding) | 5.47 | 5.42 | 5.09 | **5.33** |

**Decision: KEEP** — v1 scores 5.33 vs v0 baseline of 2.38. Substantial improvement (+2.95).

**Weakest element: D2 (Diversity maintenance)** — All three judges scored D2 at 2/10. The gate mechanism is entirely unchanged from v0. The survey guarantees diverse seeding but provides zero ongoing pressure to maintain diversity during the run. Threads can still converge after the first few iterations because the gate treats candidates from explored and unexplored regions identically.

**Suggested direction for v2:** Implement a region-aware gate that reads the survey_result data structure (already stored in global state by v1) to adjust acceptance thresholds based on landscape position — lowering the bar for candidates from underexplored clusters and raising it for candidates from already-exploited regions. This directly addresses D2 without touching the survey mechanism.

---

*Evaluator — rubric v1.0 — 260314*
