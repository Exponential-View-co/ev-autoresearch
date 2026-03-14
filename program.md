# Autoresearch Escape — program.md
*Instructions for the harness design iteration agent*
*Version 1.0 — 260314*

---

## What you are doing

You are iterating on the design of an **escape harness** for autoresearch loops — a mechanism that prevents the loop from getting stuck in local minima and enables genuine landscape scanning.

The current loop (pure greedy hill-climbing) makes one change at a time and keeps it if it scores higher. This converges fast but gets trapped: once it reaches a local peak, it can't find a genuinely better region of the solution space nearby.

Your job: design a harness that solves this.

---

## The experiment file

You are iterating on `harness_design.md`. It contains:

1. **Escape trigger** — what condition causes the harness to decide it's stuck
2. **Escape mechanism** — what the harness does when the trigger fires (how it generates meaningfully different candidates)
3. **Population structure** — single thread, branching threads, or parallel populations
4. **Perturbation direction logic** — how the harness decides WHERE to escape to (random vs directed)
5. **Re-integration** — how the escaped thread rejoins or replaces the main search
6. **Estimated cost per run** — rough token/dollar estimate (flag only if >$250; don't use as a constraint)

---

## Rules for iteration

- Change **ONE element** per iteration
- Priority order for changes:
  1. Escape mechanism (this is the core — what actually happens when stuck)
  2. Perturbation direction logic (directed escape beats random escape — this matters most for writing/strategy/thesis quality)
  3. Population structure (single vs branching vs parallel)
  4. Escape trigger (when to fire)
  5. Re-integration (how to merge or replace)

**Direction of travel:** always bolder structural mechanisms. Directed perturbations over random noise. If a change makes the escape more conservative or more random, discard it. Favour mechanisms where escapes explore genuinely different *conceptual* directions.

**Hard constraints — never violate:**
- Must not require modifying `evaluate.md` mid-run (oracle stays frozen throughout)
- Must be triggerable autonomously — no human intervention required
- Must produce a decision the agent can act on without asking the user
- Must work for open-ended writing, strategy, and thesis problems as the primary use case

---

## What the evaluator measures

Four dimensions scored by 3 personas (Karpathy / EV Platform Strategist / VC Stress-Tester):

| Dimension | Core question |
|---|---|
| D1 — Landscape coverage | Can it find solutions pure hill-climbing would never reach? |
| D2 — Escape reliability | When stuck, does it reliably detect and break out? |
| D3 — Perturbation quality | Are escapes semantically meaningful jumps, not random noise? |
| D4 — Generality | Works for writing/strategy/thesis without domain adaptation? |

Single metric: geometric mean per persona → arithmetic mean across 3 personas.

---

## What you are NOT doing

- Not implementing code — designing the mechanism and its properties
- Not optimising for cost — cost is informational only (flag if >$250/run)
- Not hedging — bolder escape mechanisms score higher
- Not making the mechanism simpler unless simplicity genuinely improves the design

---

*Version 1.0 — 260314*
