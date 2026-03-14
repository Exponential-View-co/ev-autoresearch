# Landscape Coverage Mechanism

Complementary addition to the ev-autoresearch escape harness. Addresses the D1 (landscape coverage) ceiling in `auto-beta` by adding a pre-run survey phase and region-aware gate.

Evolved via autoresearch loop, 2026-03-14. Started at 2.38 (null), converged at 7.73 (v6) over 7 iterations, 1 discard.

---

## What this adds to auto-beta

| Phase | Mechanism |
|---|---|
| Pre-run | Structural survey maps the landscape, seeds 3 threads from genuinely distinct regions |
| Per-iteration | Region-aware gate lowers threshold for unexplored regions, raises it for exhausted ones |
| Per-iteration | Exploration directive steers iteration agent toward underexplored clusters |
| Per-iteration | Contrastive constraint enforces structural divergence for ACTIVE threads |

---

## Design evolution

| Version | Mechanism | Score | Delta |
|---|---|---|---|
| v0 | Null — no survey, uniform gate | 2.38 | — |
| v1 | Structural survey seeding (3-layer guarantee) | 5.33 | +2.95 |
| v2 | Region-aware diversity gate (0.65/0.80/0.90 tiers) | 6.77 | +1.44 |
| v3 | Concrete G1 cluster assignment (haiku call) | 7.08 | +0.31 |
| v4 | Adversarial S0 construction (S0a/S0b/S0c) | 7.37 | +0.29 |
| v5 | Exploration directive (push + pull) | 7.49 | +0.12 |
| v6 | Contrastive constraint for ACTIVE threads | 7.73 | +0.24 |
| v7 | ❌ Structural extremity pass (discarded) | 7.71 | −0.02 |

**D1 ceiling note:** The oracle evaluator has a hard rule — D1 (coverage expansion) cannot exceed 7/10 without a formally verifiable diversity guarantee. LLM-qualitative adversarial operations reach 7 but cannot break through. The path above 7 on D1 requires embedding-space verification (cosine distance floor between cluster seeds). This is the next frontier if needed.

---

## Integration into auto-beta

See `auto-beta` skill (`~/.openclaw/skills/auto-beta/SKILL.md`) — the landscape coverage mechanism operates as Phase 0 (pre-run survey) + continuous gate adjustment during Phase 3.

---

## Cost

~$0.38/run total (survey + gate + G1 calls). Negligible relative to the ~$90–150 base run cost.
