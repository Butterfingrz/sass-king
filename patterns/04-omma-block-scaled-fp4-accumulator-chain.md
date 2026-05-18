# PATTERN-04: OMMA block-scaled FP4 accumulator chain

**Category**: tensor_core
**Evidence**: [OBS] Chapter 16 contains the controlled dense block-scaled OMMA observations for this pattern; Chapters 19, 23, 24, and 25 contain supporting sparse, scale-vector, production-like, and epilogue contexts where `OMMA.SF` forms appear.
**Confidence**: [INF] C2 for Chapter 16 controlled block-scaled OMMA variants; C3 for the Chapter 16 clock64 latency microbenchmark; C1 for static recognition in future production dumps unless source, runtime, or profile evidence is added.

**Plain English**

[INF] An OMMA block-scaled FP4 accumulator chain is the Blackwell FP4 peak-path version of the tensor-core accumulator chain. The kernel repeatedly applies FP4 matrix multiply-accumulate work, with explicit scale factors, to the same output fragment.

[INF] In source-level terms, this is the SASS shape behind a block-scaled FP4 GEMM tile accumulating partial sums across K. Each OMMA reads the current accumulator as C, multiplies packed FP4 A and B fragments with scale factors, adds the result, and writes the updated accumulator back to the same registers.

[INF] The important difference from dense QMMA is the scaling channel. OMMA carries scale operands in the SASS instruction, and the mnemonic records scale dtype and scale-vector mode, such as `.E8`, `.UE4M3`, and `.4X`.

[INF] When this pattern appears in a production dump, the safe conclusion is that a dense block-scaled FP4 tensor-core accumulator chain is present. It does not by itself prove packed FP4 value correctness, scale-value semantics beyond the tested forms, tensor-core throughput, or the final numeric result without additional evidence.

**SASS signature**

[OBS] The dense SM120 OMMA forms observed in Chapter 16 include `OMMA.SF.16864.F32.E2M1.E2M1.E8` and `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X`.

[INF] In the observed OMMA mnemonic, `.16864` identifies m16n8k64 and `.SF` identifies scale-factor mode.

[OBS] Chapter 16 observes that the tested `kind::mxf4nvf4` forms emit the block-scaled OMMA path with low opcode byte `0x7f`, distinct from HMMA low byte `0x3c` and QMMA low byte `0x7a`.

[OBS] The tensor-core cross-chapter summary classifies dense OMMA as a warp-level instruction in the studied SM120 dumps.

[INF] One visible `OMMA.SF.16864.*` instruction therefore represents the 32 lanes cooperating on one tensor-core tile fragment, not 32 independent scalar instructions.

[OBS] Dense block-scaled OMMA uses seven visible SASS operands in the observed forms: D base, A base, B base, C base, SFA, SFB, and `URZ` for the bid/tid operand.

[OBS] Chapter 16 observes `OMMA.SF.16864.F32.E2M1.E2M1.E8` for the mxf4nvf4 2X ue8m0 path and `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X` for the mxf4nvf4 4X ue4m3 path.

[OBS] Chapter 16 observes identical opcode bytes between the 2X ue8m0 and 4X ue4m3 dense OMMA variants when operand bases are unchanged.

[INF] Scale mode selection lives in the control code rather than in a distinct opcode byte sequence, because the tested dense OMMA scale variants keep opcode bytes stable while their control codes and displayed scale suffixes change.

[OBS] Chapter 16 observes chained OMMA forms where the first instruction can use `C=RZ` and subsequent OMMAs use the previous D accumulator register as C.

[OBS] Chapter 16 observes `.reuse` on the B and SFB operands for non-terminal OMMAs in the 16d chain: examples include `R2.reuse` for B and `R8.reuse` for the scale operand.

[OBS] Chapter 16 observes that the last OMMA in the 16d chain drops `.reuse` on both B and SFB: `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X R12, R4, R2, R12, R8, R8, URZ`.

[OBS] Chapter 16 observes chain-specific control-code transitions in 16d: the first OMMA uses `0x084ff60000043eff`, middle OMMAs use `0x080ff60000043e0c`, and the last OMMA uses `0x000ff00000043e0c`.

[INF] The control-code transition is consistent with scoreboard-coordinated waiting on the previous accumulator result, matching the Chapter 16 interpretation and the earlier MMA-chain model.

**Variants**

[OBS] Chapter 16 observes the dense QMMA.SF baseline for `kind::mxf8f6f4`: `QMMA.SF.16832.F32.E4M3.E4M3.E8`.

[INF] The dense QMMA.SF baseline is related block-scaled evidence, but it is not the OMMA pattern itself because the low opcode byte remains `0x7a`.

[OBS] Chapter 16 observes the dense OMMA 2X ue8m0 path in 16b: `OMMA.SF.16864.F32.E2M1.E2M1.E8`.

[OBS] Chapter 16 observes the dense OMMA 4X ue4m3 peak path in 16c: `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X`.

[OBS] Chapter 16 observes N-OMMA serial chains in 16d for N=16, N=32, and N=64, bracketed by `clock64()` for latency measurement.

[OBS] Chapter 19 observes sparse block-scaled `OMMA.SF.SP.168128...` variants that preserve OMMA low byte `0x7f` but add sparse metadata and selector operands.

[INF] Sparse `OMMA.SF.SP.168128...` is supporting OMMA-family evidence, not the dense `OMMA.SF.16864` signature itself.

[OBS] Chapter 24 observes production-like OMMA contexts, including `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X` and sparse `OMMA.SF.SP.168128...` forms.

[OBS] Chapter 25 observes OMMA accumulator paths feeding STSM epilogues, including `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X` before `STSM`.

**Interpretation**

[INF] Consecutive dense OMMAs with the previous D accumulator reused as C identify a dense block-scaled FP4 tensor-core accumulator dependency chain.

[INF] The `.SF` modifier and post-C scale operands identify the scale-factor data path. Scale factors should be tracked as a separate dependency channel from A/B fragment data in production audits.

[INF] The `.UE4M3.4X` suffix identifies the fine-grained FP4 peak path observed in Chapter 16, while `.E8` without `.4X` identifies the tested default 2X ue8m0 dense OMMA path.

[INF] `.reuse` on B and SFB identifies ptxas preparing for later OMMAs to re-read those operands across the chain. The exact reuse-cache placement policy is not fully decoded.

[INF] The absence of `.reuse` on the last OMMA is consistent with no later OMMA consumer of those B and SFB operands in the observed chain.

[INF] The control-code wait transition identifies scoreboard-coordinated variable-latency chaining. Chapter 16 resolves the dense OMMA 4X serial dependency-chain latency at approximately 29 cycles per `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X` on SM120.

[INF] The 29-cycle number is a dependency-chain latency result, not a throughput result. It applies when each OMMA must wait on the previous OMMA's accumulator output.

[INF] The Chapter 16 result shows the tested OMMA chain has lower serial latency and more work per instruction than HMMA and dense QMMA. It does not prove the advertised peak TFLOPS, because peak requires pipelined throughput with multiple independent MMAs in flight.

**Anti-patterns**

[INF] A single `OMMA.SF.16864.*` instruction is block-scaled FP4 tensor-core compute evidence, but it is not an accumulator-chain pattern unless a later OMMA reads the previous accumulator through C or the surrounding SASS shows the dependency.

[INF] `QMMA.SF.16832.*` is block-scaled low-precision tensor-core evidence, but it remains a QMMA-family pattern with low opcode byte `0x7a`, not OMMA.

[INF] `OMMA.SF.SP.168128.*` is a sparse block-scaled OMMA-family form. It should not be collapsed into the dense OMMA chain signature without preserving sparse metadata operands and sparse-specific gaps.

[INF] OMMA followed by `STG`, `STSM`, or scalar arithmetic is an accumulator consumer path, not an OMMA chain unless a later OMMA reuses the accumulator as C.

**Open gaps**

[GAP] Dense mxf4nvf4 4X with ue8m0 is not tested; Chapter 19 sparse evidence constrains this combination for sparse OMMA, but dense OMMA scale-mode encoding remains not fully disambiguated.

[GAP] OMMA throughput across independent, non-chained configurations is not established by the Chapter 16 latency chain; the measured 29 cycles is serial dependency-chain latency, not tensor-core throughput.

[GAP] Scale factor values other than 1.0 are not tested in Chapter 16; real block-scaled kernels use diverse scale values.

[GAP] FP4 E2M1 register and lane-to-value layout remains unresolved for full productive FP4 audits, independent of the OMMA opcode signature.

[GAP] Sparse OMMA latency is not measured; sparse `OMMA.SF.SP` forms are structurally observed but not latency-characterized.

[GAP] NCU validation of the Chapter 16 latency and pipeline-utilization interpretation was not performed.

---

Source: `knowledge/FINDINGS.md`.
