# PATTERN-07: STSM matrix-store epilogue

**Category**: memory
**Evidence**: [OBS] Chapter 22 establishes the SM120 b16 `STSM` matrix-store family; Chapter 25 contains the dedicated epilogue and storeback probes, including HMMA, QMMA, OMMA, sparse QMMA, narrowing, reload, and global-store contexts.
**Confidence**: [INF] C2 for Chapter 22 and Chapter 25 controlled STSM width, transpose, fallback, and accumulator-path variants; C3 for Chapter 25 runtime smoke execution of accepted executable probes; C1 for static recognition in future production dumps unless source, numeric, profile, or cross-context evidence is added.

**Plain English**

[INF] An STSM matrix-store epilogue is the storeback side of a tensor-core tile. After MMA instructions produce accumulator fragments in registers, STSM writes matrix-shaped fragments into shared memory so the kernel can reload, rearrange, narrow, or globally store the result.

[INF] In GEMM terms, LDSM is how tile data enters tensor-core registers from shared memory, while STSM is the matching matrix-store tool often seen on the way out of the accumulator path.

[INF] The common epilogue shape is: tensor-core compute produces registers, optional conversion narrows those values, STSM writes them into shared memory, a CTA barrier makes the shared data visible, LDS reloads it in a store-friendly shape, and STG writes global memory.

[INF] Seeing this pattern is evidence for a matrix-storeback epilogue. It does not by itself prove the full lane-to-value layout, final numeric correctness, global-store coalescing quality, or optimal epilogue performance.

**SASS signature**

[OBS] Chapter 22 observes PTX `stmatrix.sync.aligned` lowering to the tested b16 SASS family `STSM.16.M[T]88[.2|.4]` on SM120.

[OBS] Chapter 22 observes `STSM.16.M88`, `STSM.16.M88.2`, `STSM.16.M88.4`, `STSM.16.MT88`, `STSM.16.MT88.2`, and `STSM.16.MT88.4` for x1, x2, x4, and transposed m8n8 b16 forms.

[OBS] Chapter 22 observes the visible low opcode word for tested STSM forms ending in low byte `0x44`.

[INF] In the observed b16 mnemonic, `.16` identifies the b16 matrix-store family, `M88` identifies the m8n8 shape, `MT88` identifies the transposed shape, and absent, `.2`, or `.4` suffixes identify x1, x2, or x4 width.

[OBS] The STSM operand order is shared-memory destination address first, then source register base: examples include `STSM.16.M88.4 [R0], R8` and `STSM.16.MT88.4 [R0], R8`.

[OBS] Chapter 22 observes ordinary shared-store fallback `STS.128 [R0], R8` in variant 22g, and that fallback does not emit `STSM`.

[INF] `STS.128` can be a vectorized shared-store implementation, but it is not the matrix-store STSM family.

[OBS] Chapter 25 observes accepted storeback probes with `BAR.SYNC.DEFER_BLOCKING` followed by `LDS.128` after STSM.

[OBS] Chapter 25 observes global storeback probes emitting scalar `STG.E` stores after the shared-memory reload path.

[OBS] Chapter 25 observes `STSM.16.M88.4` after `HMMA.16816.F32`, `QMMA.16832.F32.E2M1.E2M1`, `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X`, and `QMMA.SP.16864.F32.E4M3.E4M3` accumulator paths.

[RES] Chapter 25 resolves that the tested dense, block-scaled, and sparse accumulator paths all preserve a visible STSM epilogue.

[OBS] Chapter 25 observes `F2F.F16.F32` and `F2F.BF16.F32` conversions before STSM in narrowing probes.

[RES] Chapter 25 resolves that accumulator narrowing is separable from matrix-store recognition: conversion instructions appear before the STSM family rather than replacing the STSM signature.

**Variants**

[OBS] Chapter 22 observes isolated x1, x2, and x4 b16 STSM forms with and without transpose.

[OBS] Chapter 25 observes raw runtime-layout probes for the same b16 STSM width and transpose families.

[OBS] Chapter 25 observes `STSM -> BAR -> LDS -> STG` storeback structure in accepted epilogue probes.

[OBS] Chapter 25 observes a no-barrier contrast in 25n with adjacent `STSM` and `LDS` without intervening `BAR`.

[INF] The no-barrier contrast is useful for SASS matching, but it is not evidence that cross-thread shared-memory visibility is correct without a CTA barrier.

[OBS] Chapter 25 observes split-accumulator storeback in 25o with two `STSM.16.M88.4` stores and two `LDS.128` reloads.

[OBS] Chapter 25 observes register-pressure epilogue variant 25y preserving the `STSM -> BAR -> LDS -> STG` structure.

[OBS] Chapter 25 observes b8 matrix-store forms `STSM.8.MT168`, `STSM.8.MT168.2`, and `STSM.8.MT168.4` for the tested `sm_120a` target.

[OBS] Chapter 25 also captures plain `sm_120` ptxas rejection of the tested `.m16n8` and `stmatrix.b8` PTX forms.

[RES] Claims about b8 STSM support must name the target: rejected for tested plain `sm_120`, accepted for tested `sm_120a`.

**Interpretation**

[INF] A visible STSM following an MMA accumulator path identifies a matrix-shaped shared-memory epilogue stage, not just ordinary scalar shared stores.

[INF] The stronger production-audit signature is STSM followed by a CTA barrier, shared reload, and global stores, because that sequence shows matrix fragments being materialized in shared memory and then moved to the global-output path.

[INF] STSM width and transpose suffixes should be read as storeback layout controls, but the exact lane-to-value semantic mapping still requires the captured runtime outputs and follow-up decode work.

[INF] If conversion instructions appear before STSM, the audit should treat narrowing as a separate epilogue substage before matrix-storeback.

[INF] A split-accumulator epilogue can contain more than one STSM group, so a matcher should not assume exactly one STSM per output tile.

**Anti-patterns**

[INF] `STS`, `STS.128`, and scalar C++ shared stores are shared-memory store evidence, but they are not the STSM matrix-store pattern.

[INF] STSM alone proves a matrix-store instruction is present, but it does not prove complete global storeback without the reload and STG context.

[INF] STSM after HMMA, QMMA, or OMMA should not be cited as final numeric correctness unless the output path and reference comparison are also validated.

[INF] `STSM.8.MT168*` should not be generalized to all SM120 targets; the current evidence is target-qualified to tested `sm_120a`, while tested plain `sm_120` rejects the PTX forms.

[INF] Adjacent STSM and LDS without `BAR.SYNC` should not be treated as safe cross-thread visibility evidence.

**Open gaps**

[GAP] Full lane-to-value STSM layout remains undecoded from the runtime output words.

[GAP] STSM latency is not measured.

[GAP] STSM scoreboard and complete control-code bit placement remain open beyond printed SASS/control annotations and denvdis field names.

[GAP] Global-store coalescing and production epilogue performance are not established by the Chapter 25 smoke probes.

[GAP] Cross-context comparison against production CUTLASS, FlashAttention, or vendor epilogues remains Phase 4 work.

---

Source: `knowledge/FINDINGS.md`.
