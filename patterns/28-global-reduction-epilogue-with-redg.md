# PATTERN-28: Global reduction epilogue with REDG

**Category**: epilogue
**Evidence**: [OBS] Chapter 24 contains a controlled SM120a production mini-GEMM probe, variant 24z, that emits `REDG.E.ADD.F32.FTZ.RN.STRONG.GPU` in a split-K or multi-CTA reduction stub.
**Confidence**: [INF] C2 for recognizing the observed `REDG.E.ADD.F32...` global reduction form; C1 for labeling a future production kernel as split-K or multi-CTA without launch geometry, source, or inter-CTA accumulation context.

**Plain English**

[INF] A normal GEMM epilogue writes each final output element once with stores such as `STG.E.128`. A reduction epilogue is different: it adds a partial result into an existing global-memory destination, which is the shape needed when multiple CTAs or K-slices contribute to the same output.

[INF] In the observed Chapter 24 stub, that global accumulation is visible as `REDG.E.ADD.F32.FTZ.RN.STRONG.GPU`. In plain English, the kernel is not just storing an output tile; it is reducing a partial value into global memory.

[INF] Seeing `REDG.E.ADD...` is strong evidence for a global reduction writeback path. It does not by itself prove the full source-level split-K schedule, grid decomposition, numerical associativity, or final reduction ordering.

**SASS signature**

[OBS] Chapter 24 variant 24z emits `REDG.E.ADD.F32.FTZ.RN.STRONG.GPU desc[UR4][R2.64], R8`.

[OBS] Chapter 24 records 24z as a split-K or multi-CTA reduction stub containing HMMA compute, the `REDG.E.ADD.F32...` reduction operation, and normal `STG.E.128` store traffic.

[OBS] The observed `REDG` destination uses the same descriptor-addressed global-memory shape as other global operations: `desc[UR][R.64]`.

[OBS] The observed source value is a register operand, `R8`, added into the global destination.

**Variants**

[OBS] Observed variant: F32 add reduction with `.FTZ.RN.STRONG.GPU` suffixes in 24z.

[GAP] Integer, half, vector, min/max, scope, semantic-strength, and system-scope `REDG` variants are not covered by the current local evidence.

[GAP] `UREDGR`, `USTGR`, and related newer reduction/writeback instructions remain watchlist items rather than established patterns in this file.

**Interpretation**

[INF] `REDG.E.ADD...` should be classified as a global reduction or atomic accumulation epilogue, not as a plain global store.

[INF] In a production audit, `REDG` marks an epilogue region that may coexist with normal stores. The matcher should separate compute (`HMMA`/`QMMA`/`OMMA`), local storeback (`STSM`/`LDS`), plain output stores (`STG`), and global reductions (`REDG`) instead of collapsing them into one generic epilogue bucket.

[INF] The `.ADD.F32` portion identifies the observed arithmetic operation and datatype. The remaining suffixes are part of the observed instruction spelling, but their precise encoding and full memory-model semantics are not decoded here.

[INF] A source-level `atomicAdd` or split-K reduction stub is a plausible producer for this pattern in Chapter 24, but future dumps should be checked against source or launch context before making a whole-kernel split-K claim.

**Anti-patterns**

[INF] Do not count `REDG.E.ADD...` as a vectorized `STG.E.128` store.

[INF] Do not infer split-K solely from HMMA plus STG; the reduction signal in the observed stub is the `REDG` writeback.

[INF] Do not claim final numerical determinism, reduction order, or correctness from the SASS opcode alone.

[INF] Do not generalize the observed F32 add form to all reduction datatypes or scopes.

[INF] Do not merge this pattern with warp-level `REDUX`; `REDUX` reduces values within a warp, while `REDG` writes an accumulated value to global memory.

**Open gaps**

[GAP] REDG latency, scoreboard behavior, and memory-ordering effects are not measured.

[GAP] Full suffix semantics for `.STRONG.GPU`, `.FTZ`, and `.RN` are not decoded beyond the observed spelling.

[GAP] Non-F32 and non-ADD REDG variants are not locally established.

[GAP] Interaction with CTA clusters, system scope, and newer UREDGR/UTC reduction instructions remains untested.

[GAP] Production-library split-K examples beyond the controlled 24z stub remain to be audited.

---

Source: `knowledge/FINDINGS.md`.
