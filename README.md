# ev-autoresearch

**Autonomous iterative improvement with landscape scanning.**

An escape harness for autoresearch loops — prevents local minima, scans the conceptual landscape, and produces better final outputs than pure hill-climbing.

Evolved via a meta-autoresearch loop: the harness design itself was iterated using the autoresearch pattern, starting from a null baseline (score 1.78) and converging to a multi-mechanism landscape scanner (score 7.82+) over 11 iterations.

---

## What this solves

Standard autoresearch (Karpathy 2026-03-07) is greedy hill-climbing: one change at a time, keep if better. It converges fast but gets trapped in local peaks — especially for writing, strategy, and thesis work where the solution landscape has many distinct conceptual regions.

This harness adds:
- **Antipode escape** — when stuck, derives the structural opposite of the current best on 3 axes and explores those regions
- **Adaptive axis discovery** — when antipodal seeds cluster, generates new axes from evaluator feedback
- **Axis orthogonality verification** — ensures new axes explore genuinely different regions (probe seed + LLM comparison)
- **Basin crossover** — detects when escape events are circling the same basin, fires recombination from top-3 historical bests
- **Hybrid novelty verification** — ensures crossover produces genuine recombinations, not blended averages
- **Population parallelism** — 3 independent threads with different axis seeds, tournament selection every 15 iterations
- **Inter-tournament monitoring** — lightweight structural fingerprint check every 5 iterations, triggers early re-diversification
- **Domain-configurable fingerprints** — fingerprint vocabularies derived from the problem statement at run start, not hardcoded to argumentation domains

---

## Design evolution

| Version | Mechanism | Score | Delta |
|---|---|---|---|
| v0 | Null baseline (pure hill-climbing) | 1.78 | — |
| v1 | Conceptual Antipode Escape | 6.89 | +5.11 |
| v2 | Regression gate + extended mini-sprints | 6.91 | +0.02 |
| v3 | Adaptive axis discovery | 7.32 | +0.41 |
| v4 | Axis orthogonality verification | 7.38 | +0.06 |
| v5 | ❌ Stall-only crossover fallback | 7.33 | −0.05 |
| v6 | Basin clustering crossover trigger | 7.49 | +0.11 |
| v7 | Crossover hybrid novelty verification | 7.65 | +0.16 |
| v8 | Population parallelism (3 threads) | 7.72 | +0.07 |
| v9 | Tournament-time divergence check | 7.74 | +0.02 |
| v10 | Inter-tournament convergence monitoring | 7.82 | +0.08 |
| v11 | Domain-configurable fingerprint vocabularies | 7.99 | +0.17 |

**Pattern:** Structural mechanism changes produce large jumps (v1: +5.11, v3: +0.41). Reliability and monitoring additions produce small but real gains. One discard (v5) — the mechanism was sound but the trigger was too narrow.

---

## Files

| File | Description |
|---|---|
| `program.md` | Iteration agent instructions for this design loop |
| `evaluate.md` | Fixed evaluator — 4 dimensions, 3 judges (Karpathy / EV Platform Strategist / VC Stress-Tester) |
| `harness_design_v0.md` → `harness_design_v11.md` | Design evolution — one change per version |
| `eval-v0-v1.md`, `eval-v2.md` ... `eval-v11.md` | Full evaluator output for each version |

---

## Usage

The current best design is `harness_design_v11.md`. To implement as an OpenClaw skill, see `auto-beta` skill (companion to `autoresearch` skill).

Cost estimate: ~$90–150/run at opus-bedrock evaluation rates (3× single-thread due to population parallelism). Well under $250.

---

## Origin

Evolved by R Mini Arnold (OpenClaw AI Chief of Staff) via meta-autoresearch on 2026-03-14. Based on Karpathy's autoresearch pattern (2026-03-07).

Part of the Exponential View AI toolchain.
