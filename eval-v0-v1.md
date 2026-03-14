# Autoresearch Escape — Evaluation: v0 vs v1
*VerExec Evaluator | 260314*

---

## v0 — Null Baseline

```
JUDGE: Karpathy
D1 (Landscape coverage): 1/10 — No mechanism exists to explore beyond the initial basin; the loop is pure greedy ascent.
D2 (Escape reliability): 1/10 — No escape mechanism; the loop terminates on plateau, guaranteeing permanent convergence to the first local optimum.
D3 (Perturbation quality): 2/10 — The evaluator's "SUGGESTED DIRECTION" provides semantically coherent nudges, but these are strictly incremental and cannot propose jumps to different conceptual regions.
D4 (Generality): 5/10 — The null harness is trivially domain-agnostic — nothing to configure means nothing to break, but also nothing to help.
Geometric mean: 1.78

JUDGE: EV Platform Strategist
D1 (Landscape coverage): 1/10 — Single thread, no branching, no population — explores exactly one trajectory from the initial seed.
D2 (Escape reliability): 1/10 — No escape exists; 10 discards = termination, full stop.
D3 (Perturbation quality): 2/10 — Evaluator guidance is coherent but structurally incapable of suggesting cross-basin jumps.
D4 (Generality): 5/10 — Domain-agnostic by absence; works anywhere but helps nowhere.
Geometric mean: 1.78

JUDGE: VC Stress-Tester
D1 (Landscape coverage): 1/10 — Converges to whatever local peak the initial seed is nearest; no visibility of alternative regions.
D2 (Escape reliability): 1/10 — Loop dies on plateau; the "escape" is giving up.
D3 (Perturbation quality): 2/10 — Incremental only; will never produce a qualitatively different thesis or framing.
D4 (Generality): 5/10 — No domain coupling, but no domain value either.
Geometric mean: 1.78

FINAL SCORE: 1.78
```

---

## v1 — Conceptual Antipode Escape

```
JUDGE: Karpathy
D1 (Landscape coverage): 7/10 — Antipode derivation on three structural axes (frame, causal claim, scope) forces jumps to genuinely different conceptual regions; however, three axes may not span the full landscape — dimensions like audience, evidence type, or narrative arc remain unexplored per escape event.
D2 (Escape reliability): 7/10 — Trigger is deterministic (10 discards), escape fires reliably, and mini-sprints give each seed a fair shake; however, no regression gate exists — if all 3 antipodes are weaker than current-best, the loop still replaces, risking downward jumps.
D3 (Perturbation quality): 8/10 — Antipode derivation from the candidate's own anatomy is structurally elegant: each perturbation has a clear semantic rationale and is guaranteed to differ meaningfully on at least one axis. Well-chosen for writing and strategy domains.
D4 (Generality): 7/10 — Frame/causal-claim/scope map naturally to thesis development, strategy framing, and open-ended writing; less obviously applicable to pure analytics or code optimisation, but covers the stated priority domains well.
Geometric mean: 7.24

JUDGE: EV Platform Strategist
D1 (Landscape coverage): 7/10 — Three structurally derived seeds per escape event provide reasonable basin coverage; multiple escape events across a run compound this.
D2 (Escape reliability): 6/10 — Deterministic trigger and execution are solid, but the dissection agent could produce weak antipodes with no quality gate — and the winner-takes-all selection has no floor relative to current-best, risking regression.
D3 (Perturbation quality): 7/10 — The dissection→antipode pipeline is well-specified; however, it relies on the dissection agent correctly identifying frame, causal claim, and scope from arbitrary candidates — non-trivial extraction that could fail silently.
D4 (Generality): 7/10 — The three axes are semantic primitives general enough for the stated priority domains without per-domain configuration.
Geometric mean: 6.74

JUDGE: VC Stress-Tester
D1 (Landscape coverage): 6/10 — Reaches different regions, but only 3 candidates per escape event; whether this is sufficient depends on the topology of the problem space. Some important basins may require more than one escape to discover.
D2 (Escape reliability): 6/10 — Fires reliably, but 3-iteration mini-sprints are very short — a promising seed in a complex basin might not develop enough signal to beat a polished incumbent, leading to false negatives on high-potential directions.
D3 (Perturbation quality): 8/10 — The perturbations are genuinely meaningful for the target domains: frame inversion, causal reversal, and scope shift each produce qualitatively different arguments. This is exactly what's needed for thesis exploration.
D4 (Generality): 7/10 — Good coverage of writing/strategy/thesis; investment analysis and product framing also fit naturally within the frame/claim/scope taxonomy.
Geometric mean: 6.70

FINAL SCORE: 6.89
KEEP (vs previous best: 1.78)

WEAKEST ELEMENT: Escape reliability — no regression gate prevents the loop from replacing a strong current-best with a weaker antipodal seed, and 3-iteration mini-sprints may undervalue promising seeds that need more iterations to develop.
SUGGESTED DIRECTION: Add a regression gate (only replace current-best if the winning antipodal seed exceeds a minimum score threshold relative to current-best, e.g. ≥80% of its score) and consider extending mini-sprint length to 5 iterations for fairer seed evaluation.
```

---

*VerExec Evaluator | autoresearch-escape | 260314*
