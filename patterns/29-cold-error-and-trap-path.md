# PATTERN-29: Cold error and trap path

**Category**: control_flow
**Evidence**: [OBS] Chapter 21 variant 21s observes `BPT.TRAP 0x1` behind a predicated branch with no `BSSY`/`BSYNC`; Chapter 24 variant 24ad observes `BSSY.RECONVERGENT`, `BSYNC.RECONVERGENT`, and `BPT.TRAP` around a production-like cold error/assert path.
**Confidence**: [INF] C2 for recognizing observed `BPT.TRAP 0x1` cold/error-path structure; C1 for inferring source-level assert, runtime hotness, or error semantics without source and branch-condition context.

**Plain English**

[INF] A cold error path is code that exists in the binary but is expected to run only for exceptional conditions, such as an assert, explicit trap, defensive check, or impossible-state guard. In SASS, this can appear as a branch-protected region ending in `BPT.TRAP`.

[INF] This pattern tells an auditor not to mistake defensive error code for the main compute loop. A production-like tensor-core kernel can contain a normal HMMA/epilogue pipeline and also carry a side path that traps if a rare condition is met.

[INF] Seeing `BPT.TRAP` identifies a trap-capable path. It does not by itself prove the original source construct was `assert`, that the path is impossible, or that the trap can be ignored for correctness.

**SASS signature**

[OBS] Chapter 21 variant 21s emits `BPT.TRAP 0x1` behind a predicated forward branch.

[OBS] Chapter 21 records 21s with 32 instructions, no `BSSY`, and no `BSYNC`.

[OBS] Chapter 24 variant 24ad emits `BSSY.RECONVERGENT`, `BSYNC.RECONVERGENT`, and `BPT.TRAP` around a cold path.

[OBS] Chapter 24 records 24ad as a cold error/assert path in a production mini-GEMM style probe that also contains normal tensor-core and store traffic.

[OBS] Chapter 20 observes terminal self-trap `BRA` regions after `EXIT`; those are separate safety/padding structures and are not the same as an in-body `BPT.TRAP` cold path.

**Variants**

[OBS] Simple branch-protected trap: Chapter 21 21s uses a predicated forward branch and `BPT.TRAP 0x1` without explicit reconvergence markers.

[OBS] Reconvergent cold path: Chapter 24 24ad places the trap path in a larger structure that includes `BSSY.RECONVERGENT` and `BSYNC.RECONVERGENT`.

[OBS] Terminal safety trap: Chapter 20 observes a self-branch after `EXIT`.

[INF] That form is a post-exit safety region and should be classified separately from executable cold error code.

**Interpretation**

[INF] In a region audit, `BPT.TRAP` should start a cold/error-path classification unless nearby control flow proves it belongs to the hot path.

[INF] The branch condition and reconvergence context decide whether the trap is lane-divergent, warp-level, or structurally isolated. The opcode alone only proves the trap instruction exists.

[INF] A `BPT.TRAP` path should be segmented before classifying mainloop compute, otherwise cold defensive code can make a small kernel look more branch-heavy than its hot path really is.

[INF] The presence or absence of `BSSY`/`BSYNC` around a trap is source-shape and compiler-shape dependent in the observed chapters; both forms are valid evidence classes.

**Anti-patterns**

[INF] Do not count a post-`EXIT` self-trap `BRA` as a useful loop back-edge or cold assert body.

[INF] Do not treat every predicated `EXIT` as a cold error path; predicated exits are also normal bounds and tail guards.

[INF] Do not infer source-level `assert` solely from `BPT.TRAP`; source, comments, or controlled variant context are needed.

[INF] Do not remove a trap path from correctness reasoning without understanding the predicate that reaches it.

[INF] Do not let a cold trap region dominate the classification of the main tensor-core, copy, or epilogue regions.

**Open gaps**

[GAP] Exact `BPT.TRAP` encoding fields and immediate semantics are not decoded beyond the observed `0x1` form.

[GAP] Runtime behavior, exception reporting, and debugger interaction are not characterized.

[GAP] Source constructs beyond explicit inline `trap` and the controlled cold-path probe are not mapped.

[GAP] Cross-architecture stability of `BPT.TRAP` lowering and reconvergence wrapping is not established.

[GAP] Production-library cold error paths remain to be sampled beyond the Chapter 24 mini-GEMM probe.

---

---

Source: `knowledge/FINDINGS.md`.
