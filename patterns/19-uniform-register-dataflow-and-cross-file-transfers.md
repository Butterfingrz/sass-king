# PATTERN-19: Uniform-register dataflow and cross-file transfers

**Category**: register_dataflow
**Evidence**: [OBS] Kernels 01, 03, 06, 07, 12, 13, and 24 contain controlled SM120 observations for uniform loads, special-register reads, shared-memory base construction, mixed R/UR arithmetic, and cross-file transfers.
**Confidence**: [INF] C2 for the controlled source-to-SASS cases that distinguish uniform and per-thread values; C1 for static recognition in future production dumps unless a controlled variant proves the source-level cause.

**Plain English**

[INF] Uniform registers are the "one value per warp" register file. If every lane needs the same value, ptxas can place it in `UR` instead of duplicating it in 32 per-thread `R` registers.

[INF] This pattern tells the reader which values ptxas proved uniform: block IDs, scalar kernel arguments, global-memory descriptors, shared-memory bases, clock samples, and some address bases.

[INF] A production audit should follow where values cross between the per-thread and uniform register files. A uniform value can feed per-thread arithmetic directly, but some producer or consumer shapes need an explicit transfer such as `R2UR`, `MOV R, UR`, `S2UR`, or `CS2UR`.

[INF] Seeing this pattern identifies compiler uniformity analysis and register-file placement. It does not by itself prove that an address is warp-uniform at the source level, that the uniform path is faster in the measured kernel, or that the same allocation will hold across compiler versions.

**SASS signature**

[OBS] Kernel 03 documents the two visible register files: per-thread `R0` through `R255` and uniform `UR` registers. [INF] The surrounding examples show ptxas placing values in `UR` when the value is uniform across the warp.

[OBS] Kernels 01 and 03 observe `S2UR UR4, SR_CTAID.X` for `blockIdx.x` and `LDCU` or `LDCU.64` for uniform scalar arguments and global descriptors.

[OBS] Kernel 03 observes mixed-source arithmetic where an instruction reads both register files, including `IMAD R11, R11, UR4, R0`.

[OBS] Kernel 06 observes shared-memory base construction with `S2UR UR5, SR_CgaCtaId`, `UMOV UR4, 0x400`, and `ULEA UR4, UR5, UR4, 0x18`.

[OBS] Kernel 07 observes that `UMOV UR4, 0x400` appears with shared-memory declarations and is absent in the no-shared Kernel 01 control.

[OBS] Kernel 12 observes `R2UR UR7, R1` after stack-pointer adjustment in the local-array spill case.

[OBS] Kernel 13 observes `CS2UR UR6, SR_CLOCKLO` in the HMMA latency timing harness, followed by mixed arithmetic such as `IADD.64 R2, R2, -UR6`.

[OBS] Chapter 10 observes the opposite consumer direction after REDUX: `REDUX.SUM.S32 UR7, R2` produces a uniform result, and downstream per-thread code uses `MOV R9, UR7`.

[OBS] Kernel 24v observes `S2UR`, `LDCU.64`, `IADD.64 R, R, UR`, tensor-core compute, and global storeback in one production-audit probe. [INF] This is the same uniform-dataflow shape appearing inside a tensor-core kernel segment.

**Variants**

[OBS] The observed uniform special-register reads include `S2UR` for launch identifiers such as `SR_CTAID.X` and `SR_CgaCtaId`.

[OBS] Uniform constant-memory loads appear as `LDCU` or `LDCU.64` in the observed examples. [INF] The `U` destination marks values ptxas classified for the uniform register file.

[OBS] The observed uniform-immediate materialization uses `UMOV`, including the shared-memory-addressing constant `0x400` observed in Kernels 06 and 07.

[OBS] Kernel 06 observes `ULEA` in the shared-memory base setup.

[OBS] Cross-file transfers are visible in both directions in the corpus: `R2UR` in Kernel 12 and `MOV R, UR` after REDUX in Chapter 10.

[OBS] Kernel 13 timing probes use `CS2UR` for the start-clock sample. [INF] That is the uniform-register form of the special-register read in the observed timing harness.

**Interpretation**

[INF] `LDCU` and `S2UR` are strong local signs that ptxas proved a value uniform enough to keep it in the uniform register file.

[INF] Mixed `R`/`UR` operands usually mean a per-thread computation is combining a lane-varying component with a warp-uniform base, stride, descriptor, or constant.

[INF] The shared-memory setup sequence is a compact marker that a kernel uses shared memory, but the exact architectural meaning of the `0x400` constant remains unresolved.

[INF] `R2UR` should be treated as a cross-file placement decision. In Kernel 12 the observed use is stack/local addressing; the exact addressing-mode reason remains a hypothesis, not a resolved rule.

[INF] A `MOV R, UR` after a uniform-producing instruction marks the point where a value re-enters per-thread code.

**Anti-patterns**

[INF] Do not assume every kernel argument is loaded into `UR`; ptxas can choose `LDC` when the downstream use is per-thread or when addressing form requires it.

[INF] Do not treat `UR` as a scalar CPU register. It is warp-uniform GPU state and still participates in SASS scheduling, scoreboards, and operand restrictions.

[INF] Do not infer that a source expression was syntactically uniform only because the final SASS uses `UR`; ptxas may have transformed or hoisted the computation.

[INF] Do not promote the exact meaning of `UMOV 0x400` beyond the observed shared-memory marker without additional architecture evidence.

[INF] Do not treat `R2UR` as generally safe or always legal for arbitrary per-thread values; the corpus only observes it in a stack/local-addressing context.

**Open gaps**

[GAP] The exact architectural meaning of the shared-memory `UMOV 0x400` constant is still unresolved.

[GAP] The full instruction set that accepts mixed `R` and `UR` operands is not mapped.

[GAP] The precise compiler rule for choosing `LDC` versus `LDCU` is inferred from observed uniformity, not formally specified.

[GAP] The reason Kernel 12 uses `R2UR` for stack/local addressing remains a hypothesis pending addressing-mode tests.

[GAP] Cross-architecture stability of the observed `UR` register counts and allocation choices is not established.

---

Source: `knowledge/FINDINGS.md`.
