# Coverage Mechanism — v0 (null baseline)
*Landscape coverage mechanism — starting state*
*260314*

---

## Survey design
None. The 3 parallel threads are seeded from generic axis labels:
- Thread 1: central frame / primary claim
- Thread 2: causal mechanism / evidence type
- Thread 3: epistemic register / resolution mode

These labels are chosen by the agent without any landscape survey. They may cluster in the same conceptual region.

## Gate mechanism
Uniform threshold: all candidates must score ≥ current-best × 0.80 to be accepted. No adjustment for whether the candidate comes from an explored or unexplored region. The gate is blind to landscape position.

## Integration
No survey phase, no region tracking. The probe seed store (from axis orthogonality verification) exists but is not used for gate adjustment.

## Cost estimate
No overhead vs current v11.

## Domain generality
N/A — no mechanism to be domain-general.

---

## Why this is the baseline problem

The uniform gate treats a candidate from an unexplored region identically to a candidate from an exhausted region. The harness has no incentive to explore — it will always exploit the current best peak. The thread seeding relies on human-readable axis labels that may map to the same conceptual region in practice.

The VC Stress-Tester scores D1 at 7/10 on v11 because of this: the harness does explore (via escape mechanisms) but cannot structurally guarantee that it's covering the landscape. 

---

*Loop: landscape-coverage-autoresearch | Started: 260314*
