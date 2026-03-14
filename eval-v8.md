# Evaluation — v8 (Lightweight Population Parallelism)
*Autoresearch escape loop — eval-v8.md*
*260314*

---

JUDGE: Karpathy
D1 (Landscape coverage): 9/10 — This is the exact intervention: 3 threads with mutually orthogonal axis seeds verified in a single opus call structurally forces exploration of genuinely different regions simultaneously, not sequentially — a textbook population-based approach that directly lifts the coverage ceiling.
D2 (Escape reliability): 8/10 — Tournament retirement adds a population-level escape that catches threads stuck in basins that per-thread mechanisms miss; the combination of local escape (adaptive axes) and global escape (tournament retirement + re-seeding) covers both failure modes well, though the fixed 15-iteration interval is not adaptive to run dynamics.
D3 (Perturbation quality): 7/10 — Per-thread perturbation quality is unchanged from v7; the new-thread seeding from global best with fresh orthogonal axes is a meaningful population-level perturbation, but cross-thread enrichment of crossover material via global best history is infrequent (tournament-only) and doesn't improve the quality of individual antipode or crossover seeds.
D4 (Generality): 8/10 — Population structure is fully domain-agnostic; 3-thread tournament with orthogonal axis seeding requires zero domain knowledge to configure.
Geometric mean: 7.97

JUDGE: EV Platform Strategist
D1 (Landscape coverage): 9/10 — Directly addresses the single-thread coverage ceiling that all three v7 judges identified as the consensus weakest element; 3 verified-orthogonal concurrent explorations with divergent axis pools from iteration 1 is the right structural fix and the design is coherent about what is shared (oracle, global best history) vs what is private (everything else).
D2 (Escape reliability): 7/10 — Tournament retirement adds reliability at the population level, but there is no mechanism to detect when all three threads converge to the same basin despite orthogonal initialisation; global best seeding could gradually pull new threads toward the same attractor, and the design has no counter for this failure mode.
D3 (Perturbation quality): 7/10 — Within-thread perturbation mechanisms are identical to v7; cross-pollination via global best history enriching crossover decomposition is real but sparse (only at tournament time); the primary diversity gain comes from different starting axes, not from richer perturbation logic per se.
D4 (Generality): 8/10 — Fully general population mechanism; thread count, tournament interval, and axis seed count are the only parameters, all domain-independent.
Geometric mean: 7.71

JUDGE: VC Stress-Tester
D1 (Landscape coverage): 8/10 — 3× concurrent exploration directly increases the probability of finding the best final output; the tournament mechanism redirects compute from poor regions to promising ones; however, 3 threads is still a modest population and the coverage gain is linear (3×), not combinatorial.
D2 (Escape reliability): 7/10 — Tournament retirement prevents indefinite waste on stuck threads; but the fixed 15-iteration interval could waste compute if threads are stuck early (waiting too long to retire) or fragment productive exploration if all threads are progressing well (unnecessary retirement of a thread that just needs more time).
D3 (Perturbation quality): 7/10 — The quality of the final output depends on perturbation quality; per-thread mechanisms are unchanged, and the cross-thread enrichment is modest — the real question is whether 3 mediocre explorations beat 1 deep exploration, and the design doesn't structurally guarantee that the answer is yes.
D4 (Generality): 8/10 — Domain-agnostic; the cost increase (3×) is the only trade-off, and it is transparently stated.
Geometric mean: 7.48

FINAL SCORE: 7.72
KEEP (vs previous best: 7.65)

WEAKEST ELEMENT: Cross-thread convergence risk — no mechanism detects or responds when all threads converge to the same basin despite orthogonal initialisation; global best seeding could act as an attractor that gradually pulls all threads toward the same region.

SUGGESTED DIRECTION: Add a tournament-time structural divergence check that compares thread current-bests structurally (not just by score) and triggers forced re-diversification — e.g., the new thread receives a maximally distant seed rather than the global best — if threads have converged to structurally similar candidates.

---

*Evaluator — v1.0 rubric, 260314*
