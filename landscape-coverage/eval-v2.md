# Evaluation — coverage_mechanism_v2.md
*Evaluator v1.0 | 260314*

---

JUDGE: Karpathy
D1 (Coverage expansion): 6/10 — Three-layer structural verification (taxonomy → drift audit → clustering with non-overlap audit) is the best LLM-mediated guarantee possible, but it lacks a true embedding-space or contrastive property — diversity is asserted by haiku's judgement, not proven by construction.
D2 (Diversity maintenance): 7/10 — The 25-point spread (0.65 vs 0.90) is large enough to survive oracle noise and creates genuine continuous economic pressure that reshapes which candidates advance throughout the run — a real hill-climbing behavioural shift, not cosmetic. Docked because G1's "zero model call" claim for classifying free-form candidate text against a structural taxonomy is underspecified — either it requires a cheap LLM call (honest but unbudgeted) or it's doing shallow keyword matching (budgeted but unreliable).
D3 (Integration coherence): 8/10 — Drop-in consumer of v1's survey_result; slots into the existing gate position between oracle and tournament; graceful fallback to 0.80 uniform if survey absent. No existing mechanism needs modification.
D4 (Generality): 8/10 — Thresholds are domain-independent (they track harness state, not domain structure); cluster definitions are problem-derived via S0; works identically for thesis/strategy/investment/product/technical problems.
Geometric mean: 7.20

JUDGE: EV Platform Strategist
D1 (Coverage expansion): 6/10 — Survey is unchanged from v1; the structural taxonomy and three-layer audit are solid engineering but still rely on haiku's structural reasoning fidelity — no independent verification layer (e.g. embedding similarity check) to catch haiku's blind spots.
D2 (Diversity maintenance): 6/10 — The region-aware gate is architecturally clean and the continuous status reclassification (UNEXPLORED → ACTIVE → EXHAUSTED) is elegant. However, the G1 classification step is the load-bearing joint of the entire gate mechanism and its implementation is ambiguous — "reuses the S2 mechanism" but S2 was a batched haiku call, so either G1 silently adds a haiku call per candidate (changing the cost model) or does something shallower than claimed. This ambiguity is an integration risk.
D3 (Integration coherence): 8/10 — Cleanest possible integration: reads existing data structures, occupies existing gate slot, no mechanism changes required, fallback behaviour preserves v1 semantics. The only concern is the G1 ambiguity noted above — if it does require a model call, the "zero cost" claim is wrong and the per-run cost model changes.
D4 (Generality): 8/10 — Fully domain-independent by construction; derives everything from survey_result which is itself problem-derived.
Geometric mean: 6.93

JUDGE: VC Stress-Tester
D1 (Coverage expansion): 6/10 — Same survey as v1. The structural taxonomy is genuinely useful but the "guarantee" is only as good as haiku's structural reasoning — which is good enough for a 6 but not the 8+ that a formal structural property would earn.
D2 (Diversity maintenance): 5/10 — The mechanism design is sound in principle: continuous pressure, large spread, economically rational thresholds. But the chain from "gate adjusts thresholds" to "user gets noticeably better output" has an unproven link: does accepting a 0.65-quality candidate from an unexplored region actually lead to a better final output, or does it waste iterations on a weak starting point that the thread can't recover from? The 0.65 floor is low enough that this is a real risk. The mechanism bets on exploration value outweighing quality cost — a reasonable bet, but unvalidated.
D3 (Integration coherence): 7/10 — Clean integration, but the G1 ambiguity means the mechanism's actual behaviour at runtime could differ from the specification. A mechanism that integrates cleanly on paper but has an ambiguous core step gets 7, not 8.
D4 (Generality): 7/10 — Domain-independent in design. The threshold values (0.65/0.80/0.90) are fixed constants — they might be optimal for some problem types and suboptimal for others. No adaptive mechanism to tune the spread based on problem characteristics.
Geometric mean: 6.19

FINAL SCORE: 6.77
KEEP (vs previous best: 5.33)

WEAKEST ELEMENT: G1 cluster assignment mechanism — the load-bearing joint of the entire gate is underspecified. It claims zero model calls but classifying free-form candidate text against a structural taxonomy inherently requires either LLM judgement (unbudgeted) or a shallow heuristic (unreliable). This ambiguity undermines confidence in D2 across all three judges.

SUGGESTED DIRECTION: Make G1 concrete — either commit to a lightweight haiku call per candidate (budget it honestly; at ~$0.001/call it's still cheap) and add a structural-drift rejection rate, or define a non-LLM fingerprint-based classification (e.g. cosine similarity against cluster centroid embeddings from the survey sketches) that can be verified independently of LLM judgement.

---

*Evaluation complete — 260314*
