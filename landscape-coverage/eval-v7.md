# Evaluation — coverage_mechanism_v7.md
*Evaluator v1.0 | 260314*

---

## Summary

v7 adds a **Structural Extremity Pass (S0d)** after the two-round adversarial construction (S0a/S0b/S0c). A single sonnet-bedrock call pushes each of the 4 surviving claim types to its structural extreme — amplifying the distinctive property that makes each type unique. S0b then fires a third round (haiku) to verify no new overlaps were introduced by the extremity push. All gate mechanisms (ED, G1–G3) and remaining survey steps (S1–S5) are unchanged from v6.

**Core argument:** Separation (v6) ≠ extremity. Two types can be non-overlapping yet conservative. S0d adds maximisation on top of separation — pushing types as far apart as possible, not merely ensuring they don't overlap.

---

## Scores

```
JUDGE: Karpathy
D1 (Coverage expansion): 7/10 — Extremity pass amplifies divergence beyond mere separation, but the mechanism is still LLM-qualitative (sonnet assessing "which structural property is distinctive" and amplifying it); no formal structural guarantee that extreme types occupy genuinely different landscape regions rather than appearing extreme through rhetorical inflation.
D2 (Diversity maintenance): 7/10 — Gate logic is identical to v6; the indirect benefit of sharper seeds making G1c's contrastive constraint more discriminating is plausible but unverified — the gate's ceiling remains haiku's qualitative pairwise judgement.
D3 (Integration coherence): 9/10 — Cleanly inserted between S0c and S1; reuses S0b infrastructure for Round 3; graceful degradation path means the mechanism cannot produce a worse outcome than v6 — a well-designed upside-only addition.
D4 (Generality): 8/10 — Sonnet prompt operates on structural properties (epistemic moves, causal logic, framing) not domain content; no domain-specific configuration needed.
Geometric mean: 7.71

JUDGE: EV Platform Strategist
D1 (Coverage expansion): 7/10 — Sonnet upgrade for the extremity step is well-justified (structural reasoning > pairwise conflict detection), but the improvement is incremental — still asking an LLM to assess and amplify structural distinctiveness, which is a better heuristic, not a structural guarantee.
D2 (Diversity maintenance): 7/10 — Gate unchanged; the claim that sharper seeds make forbidden combinations harder to game is sound but the gate's runtime behaviour depends on haiku's qualitative judgement, which hasn't improved.
D3 (Integration coherence): 9/10 — Excellent integration discipline: drop-in addition, no existing mechanisms modified, graceful degradation to v6 state if Round 3 finds irreparable overlaps. Cost negligible (~$0.005).
D4 (Generality): 8/10 — Domain-agnostic by design; extremity prompt targets structural properties not content features; same generality as v6.
Geometric mean: 7.71

JUDGE: VC Stress-Tester
D1 (Coverage expansion): 7/10 — The extremity pass means bolder starting points, which should yield more structurally diverse candidates — but the D1 ceiling at 7 reflects that no mechanism in v7 structurally guarantees diversity; it's still "ask a better LLM to try harder," which is an improvement in degree, not in kind.
D2 (Diversity maintenance): 7/10 — Identical gate. The user's runtime experience of diversity maintenance is unchanged from v6 — threads still get the same gate logic with the same haiku judgements. Sharper seeds are a survey-side improvement, not a gate improvement.
D3 (Integration coherence): 9/10 — Cannot break what currently works; graceful degradation is a genuine safety property; the additional ~$0.005 is noise.
D4 (Generality): 8/10 — No new domain-specific configuration; same generality profile as v6.
Geometric mean: 7.71

FINAL SCORE: 7.71
DISCARD (vs previous best: 7.73)

WEAKEST ELEMENT: D1 — the LLM-qualitative ceiling. S0d adds maximisation (pushing types to extremes) on top of separation (removing overlaps), but both are LLM-qualitative operations. The extremity pass asks sonnet to amplify structural distinctiveness — a stronger heuristic than haiku's pairwise conflict detection, but still a heuristic. No mechanism in v7 structurally guarantees that the 4 claim types occupy genuinely non-overlapping regions of the conceptual landscape (as opposed to appearing non-overlapping to an LLM judge).

SUGGESTED DIRECTION: Break through the D1 ceiling by introducing embedding-space verification — after S0d produces extreme types, compute embeddings for each type's structural description and verify that pairwise cosine distances exceed a minimum threshold. This would be a structural (geometric) guarantee of diversity, not a qualitative (LLM-assessed) one, and is the kind of formally verifiable property that would move D1 above 7.
```

---

*Evaluated by fixed evaluator v1.0 — 260314*
