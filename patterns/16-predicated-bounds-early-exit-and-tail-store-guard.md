# PATTERN-16: Predicated bounds, early-exit, and tail-store guard

**Category**: control_flow
**Evidence**: [OBS] Kernel 01 contains the baseline SM120 bounds-check form; [OBS] Chapter 21 contains controlled early-return, epilogue-bounds, and masked-store tail variants; [OBS] Chapter 25 contains a predicated tail storeback context after STSM.
**Confidence**: [INF] C2 for Chapter 21 controlled source-shape variants; C1 for static recognition in future production dumps unless source mapping, runtime validation, numeric tail checks, or profiling is added.

**Plain English**

[INF] A predicated bounds or tail guard is the SASS shape used when only some lanes should continue to a store, epilogue, or early return. Instead of always building a full branch/reconvergence region, ptxas can make the sensitive instruction conditional with an `@P` predicate.

[INF] The common baseline is a comparison that sets a predicate, followed by `@P EXIT` for lanes that are out of range. In-bounds lanes fall through into the useful body; out-of-bounds lanes leave the function.

[INF] Tail storeback uses the same idea later in the kernel: the address calculation and/or `STG` can be predicated so only valid output lanes write memory.

[INF] Seeing this pattern identifies guard lowering for bounds, early exits, or tails. It does not by itself prove the source-level bounds expression, runtime branch cost, final output correctness, or that every possible tail case was tested.

**SASS signature**

[OBS] Kernel 01 observes the baseline bounds-check sequence `ISETP.GE.AND P0, PT, R11, UR5, PT` followed by `@P0 EXIT`.

[OBS] Kernel 01 observes this bounds check as predication rather than an explicit `BRA` around the useful body.

[OBS] Kernel 03 observes the same `ISETP` to `@P EXIT` bounds-check shape and records `stall=13` on the predicate producer before the control-flow consumer.

[OBS] Chapter 20 observes standard prologues with bounds `ISETP` and early `@P0 EXIT` across the controlled loop-lowering variants.

[OBS] Chapter 21 variant 21h emits an additional `@P0 EXIT` for lane-dependent early return and no BSSY/BSYNC.

[OBS] Chapter 21 variant 21p emits an additional `@P0 EXIT` for an epilogue bounds check and no BSSY/BSYNC.

[OBS] Chapter 21 variant 21q emits predicated `LDC.64`, predicated `IMAD.WIDE`, predicated `STG.E`, and an additional `@P1 EXIT`, with no BSSY/BSYNC.

[OBS] Chapter 25 variant 25k covers predicated tail storeback after STSM.

**Variants**

[OBS] Baseline elementwise bounds guards appear in the early prologue of Kernel 01 and recur through the basic kernels as `ISETP` feeding `@P EXIT`.

[OBS] Early-return lowering appears in Chapter 21 variant 21h as a second predicated exit inside the useful body.

[OBS] Epilogue-bounds lowering appears in Chapter 21 variant 21p as an additional predicated exit guarding a later output path.

[OBS] Masked-store tail lowering appears in Chapter 21 variant 21q as predicated store setup plus predicated global store.

[OBS] Matrix-store epilogue tail guarding appears in Chapter 25 variant 25k after the STSM/storeback path.

[RES] Chapter 21 resolves that the tested early-exit and masked-store tail forms can lower without visible BSSY/BSYNC.

**Interpretation**

[INF] In a production dump, `ISETP` or another predicate producer followed by `@P EXIT` is a strong first-pass locator for bounds checks, early returns, or tail guards.

[INF] Predicated `STG` and predicated address setup near an epilogue should be read as masked output writeback, not as unconditional global-store behavior.

[INF] A predicated tail guard is narrower than a full divergence/reconvergence region. Unless `BSSY.RECONVERGENT` and `BSYNC.RECONVERGENT` are visible, the safer classification is predicated guard lowering.

[INF] Tail correctness still requires source mapping or runtime/numeric validation. SASS shape alone shows which lanes are guarded, not whether the predicate is the intended one for every edge case.

[INF] Bounds-check prologue guards should be separated from later epilogue/tail guards, because they protect different regions of the function.

**Anti-patterns**

[INF] Do not classify every `@P EXIT` as an error path; it can be normal bounds or tail control.

[INF] Do not classify a predicated `STG` as an unconditional vectorized store.

[INF] Do not infer BSSY/BSYNC-style reconvergence from predicated exits or predicated stores alone.

[INF] Do not claim numeric tail correctness from a visible predicate without checking output values or a source-level reference.

[INF] Do not count the early prologue bounds guard as evidence that later epilogue stores are also tail-safe; later stores need their own predicates or source/runtime evidence.

**Open gaps**

[GAP] Runtime validation of the Chapter 21 early-exit and masked-store variants remains blocked by unavailable driver evidence in that chapter.

[GAP] Predicate producer-to-`EXIT` scheduling cost is not microbenchmarked; the observed large stall remains a control-code/scheduling clue, not a measured latency model.

[GAP] Full numeric validation for predicated epilogue tail cases remains future work.

[GAP] Compiler-version and cross-architecture stability of these predicated guard choices is not established.

---

Source: `knowledge/FINDINGS.md`.
