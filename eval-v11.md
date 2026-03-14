# Evaluation — v11 (Domain-Configurable Fingerprint Vocabularies)
*Autoresearch escape loop — evaluator output*
*260314*

---

JUDGE: Karpathy
D1 (Landscape coverage): 8/10 — Three orthogonal-seeded threads with antipode exploration and crossover recombination provide structural reasons to reach genuinely different solution regions, not just adjacent variants.
D2 (Escape reliability): 8/10 — Progressive counter shortening + basin clustering + domain-specific fingerprint convergence monitoring form a triple-layered detection system that fires deterministically; vocabulary improvement makes convergence detection more precise.
D3 (Perturbation quality): 8/10 — Antipodes produce meaningful conceptual jumps (opposite premises), crossover recombines successful structural elements, and novelty verification prevents degenerate hybrids — these are not random noise.
D4 (Generality): 8/10 — Domain-configurable vocabulary directly addresses the v10 bottleneck; investment and product domains now get meaningful trip-wires, though axis pool seeds and perturbation directions remain domain-general.
Geometric mean: 8.00

JUDGE: EV Platform Strategist
D1 (Landscape coverage): 8/10 — Same 3-thread population with differentiated axis seeds; better convergence monitoring means less iteration budget wasted in converged states, marginally improving effective coverage.
D2 (Escape reliability): 8/10 — Clean end-to-end system: oracle frozen, threads independent, tournaments well-structured; vocabulary derivation has a conservative fallback path that degrades gracefully to v10 behaviour.
D3 (Perturbation quality): 8/10 — Well-motivated perturbations with layered quality controls (regression gate at 0.80, crossover novelty verification, structural divergence check at tournament time).
D4 (Generality): 9/10 — The vocabulary derivation is a well-engineered system-level improvement: one Sonnet call ($0.02), clean fallback, frozen vocabulary injected into all threads — makes the system genuinely drop-in for non-writing domains without manual configuration.
Geometric mean: 8.24

JUDGE: VC Stress-Tester
D1 (Landscape coverage): 7/10 — Sceptical that 3 threads with the same core exploration mechanism reliably reach different enough conceptual regions to produce noticeably better final output; differentiated axis seeds help but may not be sufficient for high-dimensional problem spaces.
D2 (Escape reliability): 8/10 — The combined detection mechanisms (progressive counter + basin clustering + domain-specific fingerprint convergence) are robust; the vocabulary improvement reduces both false positives and false negatives in convergence monitoring.
D3 (Perturbation quality): 8/10 — Antipodes and crossover hybrids produce meaningfully different framings; the regression gate ensures escape candidates are actual improvements, not just detours.
D4 (Generality): 8/10 — Cross-domain applicability is genuinely improved; an investment thesis problem now gets meaningful convergence detection — but the translation from better monitoring to better final output quality is indirect.
Geometric mean: 7.74

FINAL SCORE: 7.99
KEEP (vs previous best: 7.82)

WEAKEST ELEMENT: Landscape coverage for the sceptical judge (D1 = 7/10) — 3 threads with orthogonal axis seeds may not be sufficient to reliably explore genuinely different conceptual regions in high-dimensional problem spaces.
SUGGESTED DIRECTION: Add a diversity pressure mechanism to the regression gate or tournament selection that rewards structural distance from other threads' current-bests — not just absolute quality — so the system actively resists convergence rather than only detecting it after the fact.
