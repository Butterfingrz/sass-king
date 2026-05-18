# PATTERN-13: Production mini-GEMM audit segmentation

**Category**: audit_method
**Evidence**: [OBS] Chapter 24 contains the controlled SM120 production-like mini-GEMM audit probes for this pattern; [INF] the prior component patterns provide the classification vocabulary for individual regions.
**Confidence**: [INF] C2 for Chapter 24 controlled source-to-SASS structure; C3 for Chapter 24 runtime smoke execution of all 30 accepted probes; C1 for static recognition in future production dumps unless source mapping, numeric correctness, profiling, or production-library comparison is added.

**Plain English**

[INF] Production audit segmentation means reading a large SASS kernel as a set of regions instead of as one flat instruction list. A mini-GEMM-like dump can contain copy, wait, shared-load, tensor-core compute, epilogue, reduction, and cold/error regions.

[INF] The purpose of this pattern is not to introduce a new instruction. It tells the auditor how to assemble existing patterns into an end-to-end story without overclaiming what the source code or performance must have been.

[INF] In practical terms, first identify the region boundaries, then classify the instructions inside each region using the narrower patterns: async copy, LDSM, MMA chain, STSM/storeback, vectorized global store, sparse metadata, or slowpath/cold control.

[INF] Seeing a complete pipeline-shaped sequence is evidence for a production-like tensor-core kernel structure. It does not by itself prove full GEMM correctness, source template identity, CUTLASS equivalence, or bandwidth/throughput efficiency.

**SASS signature**

[OBS] Chapter 24 observes production-like async staging regions in 24e, 24f, 24t, and 24x through `LDGSTS`, `LDGDEPBAR`, `DEPBAR.LE`, `BAR`, `LDSM`, MMA, and `STG` appearing together.

[OBS] Chapter 24 observes shared matrix-load regions with `LDSM.16.M88.2` and `LDSM.16.M88.4` feeding HMMA in 24e, 24f, 24g, 24q, 24t, and 24x.

[OBS] Chapter 24 observes tensor-core compute regions spanning dense HMMA, dense QMMA, dense OMMA, sparse QMMA, and sparse OMMA-family forms.

[OBS] Chapter 24 observes shared epilogue regions in 24j and 24k through `STSM.16.M88.4`, synchronization, and global output.

[OBS] Chapter 24 observes global epilogue regions where most variants end in `STG.E.128`.

[OBS] Chapter 24 observes split-K or multi-CTA style reduction separately from normal stores in 24z through `REDG.E.ADD.F32.FTZ.RN.STRONG.GPU`.

[OBS] Chapter 24 observes cold/error-path structure in 24ad through `BSSY.RECONVERGENT`, `BSYNC.RECONVERGENT`, and `BPT.TRAP`.

[OBS] Chapter 24 observes scale-load and metadata-load dependency channels in 24aa and 24ab through parameter `LDG.E.CONSTANT` loads feeding OMMA scale operands and QMMA.SP metadata operands.

**Variants**

[OBS] Variants 24a through 24d isolate minimal tensor-core tiles ending in `STG.E.128`: HMMA, QMMA E4M3, QMMA E2M1, and scaled OMMA FP4.

[OBS] Variant 24e isolates a single-stage async pipeline and 24f isolates a double-buffer async pipeline.

[OBS] Variant 24h isolates accumulator-chain depth with four chained `QMMA.16832.F32.E4M3.E4M3` instructions.

[OBS] Variant 24i is a store-only epilogue baseline with `STG.E.128` and no MMA.

[OBS] Variants 24m and 24n contrast preserved dynamic tile-loop structure with fixed-loop unrolled HMMA structure.

[OBS] Variant 24o isolates predicated guarded HMMA plus `WARPSYNC.ALL`.

[OBS] Variants 24r and 24s isolate sparse tensor-core tiles: `QMMA.SP.16864.F32.E4M3.E4M3` and `OMMA.SF.SP.168128.F32.E2M1.E2M1.UE4M3.4X`.

[OBS] Variant 24u confirms the tested SM120 warp-level mini-GEMM path emits `HMMA` and no `WGMMA` or `TCGEN` mnemonics.

[OBS] Variants 24v, 24w, and 24ac expose uniform-register, descriptor, parameter-load, and nontrivial stride/address arithmetic paths feeding tensor-core work.

[OBS] Variant 24z isolates the reduction epilogue by combining HMMA, `REDG.E.ADD.F32...`, and `STG.E.128`.

[RES] Chapter 24 resolves that a first-pass SM120 mini-GEMM audit should segment the dump into global-to-shared copy, async dependency wait, shared matrix load, tensor-core compute, epilogue, optional reduction, and cold/error paths.

**Interpretation**

[INF] A production audit should classify each region with the most specific existing pattern before making a whole-kernel claim. For example, `LDGSTS` belongs to the async-copy region, `LDSM` to fragment loading, `HMMA/QMMA/OMMA` to tensor-core compute, and `STG.E.128` or `REDG.E.ADD` to the epilogue path.

[INF] `STG.E.128` at the end of a tensor-core probe is an output-store signature, not proof that the preceding compute was numerically correct.

[INF] `REDG.E.ADD...` separates split-K or multi-CTA accumulation from normal vectorized stores and should not be collapsed into the `STG.E.128` epilogue pattern.

[INF] Scale values and sparse metadata must be traced as independent dependency channels from their producers into the MMA operands; they are not interchangeable with A/B fragment data.

[INF] Cold/error paths can coexist with a normal compute pipeline. They should be segmented separately so `BPT.TRAP` or defensive branches do not distort the mainloop interpretation.

[INF] The audit confidence framework matters here: Chapter 24 supports structural recognition and runtime launchability, but not full numeric GEMM correctness or production-library equivalence.

**Anti-patterns**

[INF] A single MMA instruction does not prove a complete GEMM pipeline; it may be an isolated probe, a guarded region, or a partial tile.

[INF] A pipeline-shaped SASS sequence does not prove the exact source loop nest, template specialization, or K-iteration count without source mapping or controlled variants.

[INF] Absence of `WGMMA` or `TCGEN` in the tested SM120 probes should not be generalized to architectures outside the tested consumer Blackwell scope.

[INF] A store-only baseline such as 24i prevents over-reading `STG.E.128`: the store can exist without MMA.

[INF] Runtime smoke execution is not the same as full numeric validation. Chapter 24 intentionally includes structural probes whose output can be zero or incomplete while still validating SASS shape.

**Open gaps**

[GAP] Full numeric GEMM correctness is not claimed by Chapter 24.

[GAP] Production library comparison against CUTLASS, FlashAttention, or vendor kernels remains future Phase 4 work.

[GAP] Full audit-confidence scoring on real production kernels remains future work beyond the mini-GEMM probes.

[GAP] STSM lane-to-shared layout and accumulator storeback semantics are covered separately by Chapter 25 and remain incomplete at the full semantic layout level.

[GAP] Cross-architecture stability of this segmentation grammar is not established beyond the checked SM120/SM120a corpus.

---

Source: `knowledge/FINDINGS.md`.
