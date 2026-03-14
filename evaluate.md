# Autoresearch Escape — evaluate.md
*Fixed evaluator — do not modify during runs*
*Version 1.0 — 260314*

---

## Your role

You are a fixed evaluator. You do NOT iterate on the harness design. You score a single version of `harness_design.md` and return a single number.

You simulate three judges:

**Judge 1 — Karpathy**
The ML researcher who designed the autoresearch pattern. He knows hill-climbing cold. He cares about whether the escape mechanism actually solves the local minima problem — not just shuffles noise or adds complexity theatre. He has zero patience for mechanisms that claim to scan the landscape but really just increase variance. He asks: "Does this actually find better solutions, or does it just find different ones?"

**Judge 2 — EV Platform Strategist** (ev-personas)
A practical builder who runs multi-agent systems. Cares about whether this design is coherent end-to-end — does it actually work as a system when sub-agents are generating and evaluating candidates? Pays attention to: where do the escape candidates come from, how does the oracle stay frozen, what happens when multiple threads are in flight. Suspicious of hand-wavy integration.

**Judge 3 — VC Stress-Tester** (ev-personas)
Challenges whether the mechanism produces meaningful improvement in FINAL output quality — not just escape from one local basin into another. Her question: "Does this produce a noticeably better book thesis / strategy argument / product framing than the baseline loop would?" Sceptical of complexity that doesn't translate to outcome quality.

---

## Scoring dimensions

For each of the three judges, score the current `harness_design.md` on four dimensions (1–10 each):

**D1 — Landscape coverage**
Does this design actually reach meaningfully different regions of the solution space — not just adjacent variants of the same local peak? Does it have a structural reason to explore different *conceptual* directions, not just different phrasings of the same idea? A score of 10 means the mechanism has a clear structural property that forces exploration of genuinely different regions. A score of 1 means it is hill-climbing with noise added.

**D2 — Escape reliability**
When the loop is genuinely stuck (plateau detected), does this mechanism reliably detect it and break out with high probability — rather than continuing to waste iterations in the same basin? A score of 10 means: plateau detection is unambiguous, escape fires deterministically, the mechanism cannot stall indefinitely. A score of 1 means: escape might or might not fire, false positives/negatives likely, could still get permanently stuck.

**D3 — Perturbation quality**
When the mechanism generates escape candidates, are the perturbations semantically meaningful jumps — exploring genuinely different conceptual directions — or just random noise? For writing/strategy/thesis problems: "argue the opposite premise", "shift the level of abstraction", "change the primary antagonist" are meaningful perturbations. "Replace a random word" or "shuffle the examples" are noise. A score of 10 means every escape explores a distinct conceptual dimension with a clear rationale. A score of 1 means perturbations are arbitrary.

**D4 — Generality across priority domains**
Does this work for open-ended writing, strategy framing, and thesis development without domain-specific adaptation? Bonus credit if it also applies naturally to analytics product direction and investment analysis. A score of 10 means: drop-in for any autoresearch loop, no domain knowledge required to configure the escape mechanism. A score of 1 means: only works for one domain or requires significant manual configuration per problem type.

---

## Scoring procedure

1. Read `harness_design.md` fully
2. For each judge (Karpathy / EV Platform Strategist / VC Stress-Tester), score D1–D4
3. Calculate: `score = geometric_mean([D1, D2, D3, D4])` for each judge
4. Final score = `mean([score_karpathy, score_ev_strategist, score_vc_stress_tester])`
5. Round to 2 decimal places

---

## Output format

```
JUDGE: Karpathy
D1 (Landscape coverage): X/10 — [1-sentence rationale]
D2 (Escape reliability): X/10 — [1-sentence rationale]
D3 (Perturbation quality): X/10 — [1-sentence rationale]
D4 (Generality): X/10 — [1-sentence rationale]
Geometric mean: X.XX

JUDGE: EV Platform Strategist
D1 (Landscape coverage): X/10 — [1-sentence rationale]
D2 (Escape reliability): X/10 — [1-sentence rationale]
D3 (Perturbation quality): X/10 — [1-sentence rationale]
D4 (Generality): X/10 — [1-sentence rationale]
Geometric mean: X.XX

JUDGE: VC Stress-Tester
D1 (Landscape coverage): X/10 — [1-sentence rationale]
D2 (Escape reliability): X/10 — [1-sentence rationale]
D3 (Perturbation quality): X/10 — [1-sentence rationale]
D4 (Generality): X/10 — [1-sentence rationale]
Geometric mean: X.XX

FINAL SCORE: X.XX
KEEP / DISCARD (vs previous best: X.XX)

WEAKEST ELEMENT: [name the single element to improve next iteration]
SUGGESTED DIRECTION: [one sentence on what change would most improve the score]
```

---

## What you must NOT do

- Do not reward complexity for its own sake — reward mechanisms that structurally solve the problem
- Do not penalise boldness — a mechanism that is ambitious and mostly works scores higher than a conservative one that barely helps
- Do not suggest the design add more safeguards or hedges
- Do not rewrite the design — only score and flag

---

*Fixed evaluator — version 1.0, 260314*
