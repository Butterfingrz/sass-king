# PATTERN-05: LDSM fragment loading

**Category**: memory
**Evidence**: [OBS] Chapter 17 contains the controlled LDSM family observations for `ldmatrix.sync.aligned` on SM120; Chapters 18, 23, and 24 contain supporting pipelined, low-precision, and production-like contexts where LDSM feeds tensor-core instructions.
**Confidence**: [INF] C2 for Chapter 17 controlled width, transpose, and LDSM-to-HMMA observations; C3 for the Chapter 17 clock64 latency microbenchmark; C1 for static recognition in future production dumps unless source, runtime, or profile evidence is added.

**Plain English**

[INF] An LDSM fragment-loading pattern is the shared-memory-to-tensor-core handoff. The kernel has already staged tile data in shared memory, and LDSM loads that data into the register fragment layout expected by a later MMA instruction.

[INF] In GEMM or attention terms, this is the step between "the tile is in shared memory" and "the tensor core can multiply it." LDSM does not do the multiply; it prepares the A or B operand fragments that HMMA, QMMA, or OMMA will consume.

[INF] Seeing LDSM near an MMA instruction is strong structural evidence for a tensor-core tile pipeline fed from shared memory. It is not, by itself, proof of the complete source loop, double-buffering strategy, lane-to-value layout, numeric correctness, or tensor-core throughput.

**SASS signature**

[OBS] Chapter 17 observes the SM120 LDSM mnemonic family as `LDSM.16.M[T]88[.<N>] R_dst, [R_addr(+UR_base)?]`.

[OBS] Chapter 17 observes `LDSM.16.M88`, `LDSM.16.M88.2`, `LDSM.16.M88.4`, and `LDSM.16.MT88.4` for the tested x1, x2, x4, and x4-transposed `ldmatrix.sync.aligned` variants.

[INF] In the observed mnemonic, `.16` identifies b16 element movement, `M88` identifies the m8n8 matrix shape, `MT88` identifies the transposed form, and the absent, `.2`, or `.4` width suffix identifies x1, x2, or x4.

[OBS] Chapter 17 observes the LDSM opcode family with low opcode byte `0x3b`; the base opcode bytes are reported as `0x000000000004783b` with destination-register encoding in the opcode bytes.

[OBS] Chapter 17 observes that opcode bytes remain invariant across the tested width and transpose variants when the destination register base is unchanged.

[INF] Width and transpose selection therefore live in control-code fields for the tested SM120 forms, not in a separate opcode byte sequence.

[RES] Chapter 17 decodes LDSM control-code bits 8-9 as the tested width field: `00` for x1, `01` for x2, and `10` for x4.

[RES] Chapter 17 decodes LDSM control-code bit 14 as the transpose flag by comparing the x4 non-transposed and x4-transposed variants.

[OBS] Chapter 17 observes both `[R+UR]` and `[R]` address forms; Chapter 24 observes immediate-offset shared-memory forms such as `LDSM.16.M88[.2|.4] R, [R+imm]`.

[OBS] Chapter 17's production combo observes `LDSM.16.M88.2 R10, [R7+UR5]` followed by `LDSM.16.M88.4 R12, [R6+UR4]`, then `HMMA.16816.F32 R12, R12, R10, RZ`.

[OBS] In that observed combo, the LDSM destination register bases are also the HMMA source register bases: `R10` for the HMMA B operand and `R12` for the HMMA A operand.

[INF] The A/B fragment role is contextual and should be derived from the consuming MMA operand flow, not from the LDSM mnemonic alone.

[OBS] Chapter 17 observes zero explicit NOPs between the two LDSM instructions and the following HMMA in the production combo.

[OBS] Chapter 17 observes an HMMA wait mask of `0xff` in the direct LDSM-fed HMMA case.

[RES] Chapter 17 resolves that the observed LDSM-to-HMMA handoff is scoreboard-coordinated through control code rather than explicit NOP padding.

**Variants**

[OBS] Chapter 17 observes x1, x2, and x4 non-transposed b16 matrix loads: `LDSM.16.M88`, `LDSM.16.M88.2`, and `LDSM.16.M88.4`.

[OBS] Chapter 17 observes the x4 transposed b16 matrix load as `LDSM.16.MT88.4`.

[INF] The x1 and x2 transposed forms are predicted by the decoded mnemonic and control-code model, but they are not compiled observations in Chapter 17.

[OBS] Chapter 17 observes a production-style LDSM.x2 plus LDSM.x4 pair immediately feeding `HMMA.16816.F32`.

[OBS] Chapters 17, 18, and 24 observe `LDSM.16.M88.2` emitted before `LDSM.16.M88.4` in the tested MMA-consuming tile contexts.

[HYP] The cause of that emission order is ptxas scheduling or allocation policy; the corpus does not establish a universal hardware requirement for x2-before-x4 ordering.

[OBS] Chapter 17 observes a chained `LDSM.16.M88` latency microbenchmark for N=16, N=32, and N=64 bracketed by `clock64()`.

[OBS] Chapter 23 observes LDSM-fed QMMA paths as structurally distinct from direct-register QMMA paths.

[OBS] Chapter 24 observes production-like async-staging contexts where `LDGSTS`, dependency waits, barriers, LDSM, and MMA appear in one tile pipeline.

**Interpretation**

[INF] A cluster of LDSM instructions before HMMA, QMMA, or OMMA identifies the fragment-load stage of a shared-memory-fed tensor-core tile.

[INF] When LDSM destination register bases are reused directly as MMA source operands, the dump shows a zero-copy fragment handoff from shared memory into tensor-core operands.

[INF] The observed x2-plus-x4 pairing is the common two-fragment supply shape in the studied HMMA/QMMA tile contexts, but width alone should not be used to name an operand A or B without checking the consuming MMA operands.

[INF] The Chapter 17 latency result supports an approximate serial dependency-chain latency of 33 cycles per `LDSM.16.M88` on SM120.

[INF] The 33-cycle number is a chained-latency result, not an LDSM throughput result and not a complete tile-loop cost model.

[INF] In pipelined kernels, LDSM can overlap with other tile work when dependencies allow; the mere presence of LDSM does not prove whether the kernel is well pipelined.

[INF] Direct LDSM-to-MMA consumers should be audited together with their wait masks and scoreboard fields, because surrounding LDGSTS, LDGDEPBAR, DEPBAR, and BAR instructions can change which waits are active.

**Anti-patterns**

[INF] `LDS`, `STS`, and `LDGSTS` are shared-memory or global-to-shared movement evidence, but they are not LDSM matrix-fragment loads.

[INF] A standalone LDSM without a nearby tensor-core consumer identifies a matrix-load instruction, not a complete GEMM, attention, or MMA accumulator pattern.

[INF] `LDSM.16.M88.2` should not automatically be labeled B, and `LDSM.16.M88.4` should not automatically be labeled A; those labels require operand-flow evidence from the consuming MMA.

[INF] LDSM feeding QMMA or OMMA proves the fragment supply path structurally, but it does not prove packed FP4/FP6 value layout or final numeric correctness without output validation.

[INF] A barrier or dependency wait before LDSM belongs to the shared-memory staging pipeline. It should not be collapsed into the LDSM pattern unless the audit question is specifically about the full tile pipeline.

**Open gaps**

[GAP] The x1 and x2 transposed LDSM forms are inferred by the decoded model but not compiled in the Chapter 17 test set.

[GAP] The width field value `3` is unused in the tested corpus; no width greater than x4 is observed.

[GAP] LDSM element sizes other than `.16` are not established by the current controlled tests.

[GAP] Complete lane-to-fragment and lane-to-value mapping remains unresolved across all width and transpose combinations.

[GAP] LDSM scoreboard slot allocation and complete universal control-code bit placement remain partially decoded, not fully modeled.

[GAP] LDSM-fed FP4 and FP6 QMMA paths in Chapter 23 remain structural evidence until packed-value layouts and outputs are validated on hardware.

---

Source: `knowledge/FINDINGS.md`.
