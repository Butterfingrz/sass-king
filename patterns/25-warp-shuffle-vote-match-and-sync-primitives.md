# PATTERN-25: Warp shuffle, vote, match, and sync primitives

**Category**: collective
**Evidence**: [OBS] Chapters 09 and 21 contain controlled SM120 observations for non-reduction warp collectives: `SHFL.IDX`, `SHFL.UP`, `SHFL.DOWN`, `VOTE`, `MATCH`, `__activemask`, `__syncwarp`, and `WARPSYNC.ALL`.
**Confidence**: [INF] C2 for the controlled Chapter 09 and 21 source-to-SASS variants; C1 for static recognition in future dumps unless runtime masks, divergence behavior, or source mapping are validated.

**Plain English**

[INF] Warp collectives are instructions where lanes in the same warp communicate or agree on a predicate. They are not all reductions: some broadcast a value, shift values between lanes, compute a ballot mask, find lanes with matching values, or synchronize guarded warp-level work.

[INF] This pattern tells the reader which kind of warp-level communication is present. `SHFL` moves register values between lanes, `VOTE` aggregates predicates, `MATCH` builds masks of lanes with equal values, and `WARPSYNC.ALL` marks explicit warp synchronization in observed guarded tensor-core code.

[INF] Seeing one of these instructions identifies warp-level coordination. It does not by itself prove a reduction, full-mask correctness, safe behavior under arbitrary divergence, or the original CUDA intrinsic without surrounding evidence.

**SASS signature**

[OBS] Chapter 09 observes four SHFL variants on SM120: `SHFL.IDX`, `SHFL.BFLY`, `SHFL.UP`, and `SHFL.DOWN`.

[OBS] Chapter 09 observes `SHFL.IDX` for lane broadcast, with format `Pdst, Rdst, Rsrc, lane_source, 0x1f`.

[OBS] Chapter 09 observes `SHFL.UP PT, R9, R2, 0x4, RZ`; the fifth operand is `RZ` in the observed UP form.

[OBS] Chapter 09 observes `SHFL.DOWN PT, R9, R2, 0x4, 0x1f`; the fifth operand matches the segment-mask form used by IDX and BFLY in the observed dumps.

[OBS] Chapter 09 observes 64-bit shuffle lowering as two `SHFL.IDX` instructions, one per 32-bit half.

[OBS] Chapter 09 observes register-output ballot form `VOTE.ANY R5, PT, P0`.

[OBS] Chapter 09 observes `__activemask()` as `VOTE.ANY R5, PT, PT`.

[OBS] Chapter 09 observes `MATCH.ANY R0, R2` and `MATCH.ALL PT, R5, R2`.

[OBS] Chapter 21 observes `VOTE.ANY R5, PT, P0` for `__ballot_sync`, `VOTE.ANY P0, P0` before a branch guarding `SHFL.IDX`, and `@P0 WARPSYNC.ALL` before `@P0 HMMA.16816.F32`.

**Variants**

[OBS] Broadcast: `SHFL.IDX` moves a value from a selected lane to other lanes.

[OBS] Directional exchange: `SHFL.UP` and `SHFL.DOWN` move values from neighboring lanes by a delta.

[OBS] Butterfly exchange: `SHFL.BFLY` exchanges values by XOR masks and is also used inside manual reduction patterns.

[OBS] Predicate aggregation: `VOTE.ANY` and `VOTE.ALL` appear in predicate or register-output forms depending on whether the source operation needs a boolean or a ballot mask.

[OBS] Equality grouping: `MATCH.ANY` and `MATCH.ALL` produce lane masks for matching values, with `MATCH.ALL` also producing a predicate in the observed form.

[OBS] Sync helper lowering: Chapter 09 observes `__syncwarp()` lowering to NOP padding or elimination in tested forms, while Chapter 21 later observes explicit `WARPSYNC.ALL` in guarded HMMA code.

**Interpretation**

[INF] A lone `SHFL.IDX`, `SHFL.UP`, or `SHFL.DOWN` identifies value movement, not a reduction, unless a repeated combine chain follows.

[INF] `VOTE.ANY R, PT, P` is a ballot-style register mask, while `VOTE.ANY P, P` is a predicate-output form in the observed Chapter 21 branch pattern.

[INF] `VOTE.ANY R, PT, PT` is the observed `__activemask()` idiom because the predicate input is always true.

[INF] Two adjacent SHFLs over neighboring registers can indicate a 64-bit warp value transfer split into two 32-bit halves.

[INF] `WARPSYNC.ALL` should be read as explicit warp-level synchronization/control evidence, not as a reduction signature.

**Anti-patterns**

[INF] Do not classify every SHFL as a warp reduction. Reductions require a repeated exchange plus arithmetic/logical combine chain or a `REDUX`.

[INF] Do not assume `__syncwarp()` always emits `WARPSYNC.ALL`; Chapter 09 observed NOP padding or elimination, while Chapter 21 observed `WARPSYNC.ALL` in a different guarded-HMMA context.

[INF] Do not infer non-full mask behavior from full-mask SHFL/VOTE examples.

[INF] Do not treat MATCH masks as sparse MMA metadata or reduction masks; they are warp-lane equality masks in the observed collectives.

[INF] Do not infer correctness under genuine divergent control without checking the surrounding reconvergence, vote-converged branch, or `WARPSYNC` context.

**Open gaps**

[GAP] Full SHFL/VOTE/MATCH control-code bit placement is not decoded.

[GAP] SHFL.UP fifth-operand semantics are not fully explained beyond the observed `RZ` form.

[GAP] Runtime behavior under non-full masks and genuine divergence remains under-tested.

[GAP] `WARPSYNC.ALL` predicate behavior, encoding, and interaction with SHFL/LDSM/HMMA remain incomplete.

[GAP] Cross-architecture stability of these collective forms is not established.

---

Source: `knowledge/FINDINGS.md`.
