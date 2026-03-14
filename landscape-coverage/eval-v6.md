# Evaluation — coverage_mechanism_v6.md
*Evaluator v1.0 | 260314*

---

## Mechanism summary

v6 replaces the ACTIVE-branch soft nudge (advisory text) in the ED step with an enforceable contrastive constraint: named dimensional requirements on 3 structural axes (central_frame, causal_mechanism, epistemic_register), verified post-generation by a new G1c sub-step. Candidates that fail G1c twice are flagged SAME_CLUSTER and receive the ACTIVE threshold (0.80) — exactly the v5 outcome. Survey (S0–S5), ED UNEXPLORED/EXHAUSTED branches, G1a/G1b, G2, and base G3 thresholds are all unchanged from v5.

---

JUDGE: Karpathy
D1 (Coverage expansion): 7/10 — Survey is unchanged from v5; the adversarial S0 process is a genuine structural property (conflict-based diversity construction), but it remains LLM-judged rather than mathematically guaranteed — 7 is the ceiling for this approach.
D2 (Diversity maintenance): 8/10 — The contrastive constraint with G1c verification is a real enforcement mechanism: named forbidden combinations on specific axis values, checked against the same structural fingerprint used throughout IC1–IC5. This is qualitatively different from v5's advisory nudge — the iteration agent cannot satisfy it by absorbing and ignoring. One regeneration + graceful SAME_CLUSTER fallback is well-calibrated.
D3 (Integration coherence): 8/10 — Clean drop-in: G1c reads the existing G1a fingerprint (no additional extraction), SAME_CLUSTER routes to existing threshold tier, no existing mechanism modified. The only new state is an ephemeral per-iteration flag. No interference patterns with crossover, tournaments, or IC1–IC5.
D4 (Generality): 8/10 — STRUCTURAL_DIMENSIONS triple derives from Step V vocabulary axes, which are already domain-agnostic. The contrastive constraint template is a pure template fill from stored state. No domain-specific configuration required.
Geometric mean: 7.74

JUDGE: EV Platform Strategist
D1 (Coverage expansion): 7/10 — Survey unchanged; adversarial S0 provides structural diversity but the same approach as v5. No new coverage expansion mechanism.
D2 (Diversity maintenance): 8/10 — The verifiable contrastive constraint addresses the core weakness all three judges flagged in v5: ACTIVE is the majority thread status, and the soft nudge was unenforceable. Named forbidden combinations + G1c post-generation check is a practical enforcement pattern. The one-regeneration cap and SAME_CLUSTER fallback are production-sensible — no stall risk.
D3 (Integration coherence): 9/10 — Excellent integration hygiene. G1c piggybacks on G1a's existing fingerprint. SAME_CLUSTER maps to an existing threshold tier. No modification to oracle, fingerprint monitoring, escape mechanisms, crossover, tournament, or termination. The gate execution order update is additive — one new sub-step inserted between G1b and G2. Survey execution unchanged. Fallback behaviour preserved (reverts to 0.80 uniform if survey_result absent).
D4 (Generality): 8/10 — Domain-agnostic throughout. The contrastive constraint references whatever axis values Step V and S4/S5 produced — no hardcoded domain labels. G1c compliance check asks "do these axis values differ?" not "is this good for domain X?"
Geometric mean: 7.97

JUDGE: VC Stress-Tester
D1 (Coverage expansion): 7/10 — Survey unchanged from v5. The regions the search starts in are no more diverse than before.
D2 (Diversity maintenance): 7/10 — The contrastive constraint is a genuine improvement over advisory text, but the enforcement is still LLM-verified (haiku judging "structural difference" on named dimensions). The estimated 30% first-attempt failure rate and graceful SAME_CLUSTER fallback mean a meaningful fraction of ACTIVE iterations will still produce same-cluster candidates at the v5 threshold. The mechanism creates an upside path (candidates that pass G1c get exploration credit) but the floor is unchanged — and the floor is where the user experience is determined when the constraint doesn't bite. Incremental, not transformative.
D3 (Integration coherence): 8/10 — Clean integration. No existing mechanism modified. Graceful degradation. Cost well within budget (~$0.39 total overhead).
D4 (Generality): 8/10 — Domain-agnostic. Template fills from stored state. No user configuration required.
Geometric mean: 7.48

---

FINAL SCORE: 7.73
KEEP (vs previous best: 7.49)

WEAKEST ELEMENT: D1 (Coverage expansion) — unchanged survey means all three judges hit the 7/10 ceiling. The contrastive constraint improved D2 for the ACTIVE phase, but D1 is now the binding constraint on the overall score. No amount of gate refinement will push past this ceiling while the survey's structural diversity guarantee remains LLM-judged adversarial rather than formally verifiable.

SUGGESTED DIRECTION: Attack the D1 ceiling directly. Consider embedding-space verification of cluster separation at S4 — e.g., compute structural fingerprint embeddings for each cluster seed and enforce minimum cosine distance between all pairs, rather than relying on LLM-judged adversarial overlap assessment. This would give D1 a structural/mathematical guarantee that could push it above 7.

CONVERGENCE NOTE: v5 → v6 delta is +0.24. The mechanism is approaching diminishing returns on gate-side improvements — D2 gains are real but bounded by the SAME_CLUSTER fallback floor, and D1 is static. The loop should either attack D1 (survey-side structural guarantee) or consider whether 7.73 is an acceptable convergence point.

---

*Evaluated by: Evaluator Agent v1.0 | 260314*
