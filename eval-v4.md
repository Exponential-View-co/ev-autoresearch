# Evaluation — v4 (Axis Orthogonality Verification)
*Autoresearch escape loop — eval-v4*
*260314*

---

JUDGE: Karpathy
D1 (Landscape coverage): 8/10 — The monotonically growing probe seed store is a genuine structural property that prevents re-entry into exhausted regions; this is not variance increase, it is coverage enforcement with memory.
D2 (Escape reliability): 7/10 — Trigger and gate mechanics are solid (inherited), but the verification step can drop both proposed axes after retries, leaving the pool unchanged after consecutive failures — the mechanism can still stall if the verifier is too strict.
D3 (Perturbation quality): 8/10 — Probe seeds force instantiation before comparison, catching "sounds different, acts the same" axes that definitional checks would miss; the opus-level semantic judgement is the right tool for this.
D4 (Generality): 8/10 — Probe seeds are generated from whatever the current-best is, comparison is conceptual via opus — zero domain configuration required.
Geometric mean: 7.74

JUDGE: EV Platform Strategist
D1 (Landscape coverage): 8/10 — The probe seed store as a persistent map of explored conceptual regions is architecturally elegant; retained retired-axis seeds mean the map only grows, which is the right property for a long-running system.
D2 (Escape reliability): 6/10 — If both proposed axes fail verification after 2 retries each, the adaptive step adds nothing and the pool is unchanged — the system has detected it needs new territory but failed to find any, with no fallback mechanism.
D3 (Perturbation quality): 8/10 — Single opus call per axis covering probe generation and all pairwise comparisons is cost-efficient; the retry mechanism with failure rationale gives the dissection agent a targeted second chance rather than blind regeneration.
D4 (Generality): 8/10 — No domain-specific adaptation required; the entire verification pipeline operates on structural character sketches that work for essay, strategy, and thesis problems alike.
Geometric mean: 7.44

JUDGE: VC Stress-Tester
D1 (Landscape coverage): 7/10 — Preventing overlapping axes should translate to better final output, but the mechanism is still one step removed from proving that verified-orthogonal axes produce noticeably different *final candidates* rather than just different seeds.
D2 (Escape reliability): 6/10 — The double-drop scenario (both axes fail verification) is a real failure mode that could leave the harness stuck in exactly the situation the adaptive step was designed to resolve — consecutive gate failures with no new axes to try.
D3 (Perturbation quality): 7/10 — The probe seed comparison is a meaningful quality gate, but the 2–3 sentence sketch level of abstraction may not capture all the ways two axes can produce structurally similar full candidates once developed through a 5-iteration sprint.
D4 (Generality): 8/10 — Domain-general with no configuration; the probe seed + opus comparison pattern would work for any open-ended optimisation problem with semantic structure.
Geometric mean: 6.96

FINAL SCORE: 7.38
KEEP (vs previous best: 7.32)

WEAKEST ELEMENT: Escape reliability — the orthogonality verification can reject both proposed axes, leaving the pool unchanged after consecutive gate failures with no fallback, creating a potential stall state.
SUGGESTED DIRECTION: Add a fallback mechanism when both proposed axes fail verification — e.g., a "random conceptual restart" that generates a candidate from a maximally distant region without requiring axis-pool admission, ensuring the mechanism cannot stall indefinitely even when the verifier is strict.

---

*Evaluator v1.0 | Candidate: harness_design_v4.md | Previous best: v3 (7.32) | Decision: KEEP*
