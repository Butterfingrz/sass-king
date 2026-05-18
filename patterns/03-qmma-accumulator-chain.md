# PATTERN-03: QMMA accumulator chain

**Category**: tensor_core
**Evidence**: [OBS] Chapter 14 contains the controlled dense QMMA observations for this pattern; Chapters 19, 23, 24, and 25 contain supporting sparse, fragment-layout, production-like, and epilogue contexts where `QMMA` forms appear.
**Confidence**: [INF] C2 for Chapter 14 controlled accumulator-chaining and dtype variants; C3 for the Chapter 14 clock64 latency microbenchmark; C1 for static recognition in future production dumps unless source, runtime, or profile evidence is added.

**Plain English**

[INF] A QMMA accumulator chain is the low-precision tensor-core version of the same "keep accumulating here" structure documented for HMMA. The kernel repeatedly applies FP8, FP6, or FP4 matrix multiply-accumulate work to the same output fragment.

[INF] In source-level terms, this is the SASS shape behind a low-precision GEMM tile accumulating partial sums across K. Each QMMA reads the current accumulator as C, adds another A x B product, and writes the updated accumulator back to the same registers.

[INF] The main difference from HMMA is the input format space. QMMA names both input dtypes explicitly, such as `E4M3`, `E5M2`, `E3M2`, `E2M3`, or `E2M1`, and those dtype choices are encoded in the control code rather than by changing the base opcode bytes.

[INF] When this pattern appears in a production dump, the safe conclusion is that a dense low-precision tensor-core accumulator chain is present. It does not by itself prove the exact source template, the packed-value correctness for FP4/FP6, tensor-core throughput, or the final numeric result without additional evidence.

**SASS signature**

[OBS] The dense SM120 QMMA form observed in Chapter 14 is `QMMA.16832.<acc>.<inputA>.<inputB>`, with `.16832` identifying m16n8k32 and both input dtypes explicit in the mnemonic.

[OBS] Chapter 14 observes that `kind::f8f6f4` requires `-arch=compute_120a -code=sm_120a`; plain `-arch=sm_120` rejects the feature.

[OBS] The dense QMMA family uses low opcode byte `0x7a` in the tested SM120 dumps, matching the tensor-core family summary.

[OBS] Dense QMMA is warp-level in the studied SM120 dumps: one visible `QMMA.16832.*` instruction represents the 32 lanes cooperating on one tensor-core tile fragment, not 32 independent scalar instructions.

[OBS] Dense QMMA uses four visible SASS operands in the order D base, A base, B base, C base; fragment spans are implicit in the shape and accumulator dtype.

[OBS] Chapter 14 observes opcode bytes invariant across the tested dense input and accumulator dtypes when operand bases are unchanged.

[INF] Dtype selection lives in the control code rather than in a distinct opcode byte sequence, because the tested dtype variants keep opcode bytes stable while their control codes and displayed dtype suffixes change.

[OBS] The observed dense QMMA dtype model uses bits 14, 18, and 19 for operand A and bits 15, 20, and 21 for operand B; Chapter 14 validates the model with the asymmetric 14i prediction for `E4M3 x E2M1`.

[OBS] Chapter 14 observes two chained `QMMA.16832.F32.E4M3.E4M3` instructions in 14e with D and C colocated on the same register base.

[INF] D/C colocation identifies an in-place accumulator update under the same MMA-family allocation rule established by Chapters 13 and 14.

[OBS] Chapter 14 observes `.reuse` on the B operand of every QMMA in a chain except the last one.

[INF] The QMMA `.reuse` placement matches the HMMA chain rule from Chapter 13.

[OBS] Chapter 14 observes chain-specific control-code transitions: chain-start uses a `0x084ff6...` high-byte pattern, mid-chain uses `0x080ff6...`, and the last QMMA uses `0x000ff0...`.

**Variants**

[OBS] Chapter 14 observes dense single-QMMA variants for `E4M3.E4M3`, `E4M3.E5M2`, `E5M2.E5M2`, `E2M1.E2M1`, `E3M2.E3M2`, `E2M3.E2M3`, and mixed `E4M3.E2M1` inputs.

[OBS] Chapter 14 observes accumulator dtype variants: `QMMA.16832.F32...` and `QMMA.16832.F16...`.

[OBS] Chapter 14 observes a two-QMMA dense chain in 14e with D/C colocated, B `.reuse` on the first QMMA, and control code `0x084ff60000002c10` on the first QMMA and `0x000ff60000002c10` on the last.

[OBS] Chapter 14 observes N-QMMA serial chains in 14f for N=16, N=32, and N=64, bracketed by `clock64()` for latency measurement.

[OBS] Chapter 19 observes sparse QMMA chain scheduling in 19m that follows the Chapter 14 dense chain pattern, but sparse `QMMA.SP.16864` adds metadata operands and is supporting evidence rather than the dense `QMMA.16832` signature itself.

[OBS] Chapter 24 observes production-like dense QMMA contexts, including 24h with four chained `QMMA.16832.F32.E4M3.E4M3` instructions.

**Interpretation**

[INF] Consecutive dense QMMAs with D and C on the same accumulator register base identify a low-precision tensor-core accumulator dependency chain because the later QMMA reads the earlier QMMA's output as C and writes the updated accumulator in place.

[INF] The explicit A and B dtype suffixes identify the low-precision element formats used by the tensor-core instruction. The control-code dtype bits let the reader distinguish format changes from operand-register changes.

[INF] The `sm_120a` target requirement is a compile-target constraint for these dense low-precision QMMA forms. It is not itself a runtime pattern, but it explains why an otherwise similar `sm_120` build may not contain this SASS family.

[INF] `.reuse` on B identifies ptxas preparing for a later QMMA to re-read the same B fragment. The absence of `.reuse` on the last observed QMMA is consistent with no later QMMA consumer of that B operand.

[INF] The control-code high-byte transition identifies the same MMA-family scoreboard chaining scheme observed for HMMA. Chapter 14 resolves the dense QMMA serial dependency-chain latency at approximately 35 cycles per `QMMA.16832.F32.E4M3.E4M3` on SM120.

[INF] The 35-cycle number is a dependency-chain latency result, not a throughput result. It applies when each QMMA must wait on the previous QMMA's accumulator output.

[INF] The Chapter 14 result implies higher work per serial-latency slot than HMMA for the tested dense FP8 form, because `QMMA.16832` covers k=32 while `HMMA.16816` covers k=16, but the measured marginal chain latency is approximately the same.

**Anti-patterns**

[INF] A single `QMMA.16832.*` instruction is dense low-precision tensor-core compute evidence, but it is not an accumulator-chain pattern unless a subsequent QMMA reads the previous accumulator through the C operand or the surrounding SASS shows the dependency.

[INF] Consecutive QMMAs using different C/D accumulator bases may be independent accumulators scheduled near each other, not a serial accumulator chain.

[INF] `QMMA.SF`, `QMMA.SP`, and `QMMA.SF.SP` are related QMMA-family forms with additional scale or sparse metadata operands. They should not be collapsed into the dense `QMMA.16832` chain signature without preserving their extra operands and gaps.

[INF] A dense QMMA followed by `STG`, `STSM`, or scalar arithmetic is an accumulator consumer path, not a QMMA chain unless a later QMMA reuses the accumulator as C.

**Open gaps**

[GAP] FP4 E2M1 fragment layout remains decode-unresolved from Chapter 14 and later reduced but not fully closed by Chapter 23; the SASS signature is valid, but productive FP4 value interpretation remains open.

[GAP] FP6 layout and exact lane-to-value mappings remain incomplete for the tested dense QMMA formats.

[HYP] QMMA throughput in independent, non-chained configurations is not established by the Chapter 14 latency chain; the measured 35 cycles is serial dependency-chain latency, not tensor-core throughput.

[HYP] QMMA shapes other than tested dense m16n8k32 are not established by Chapter 14.

[HYP] The approximately 510-cycle fixed overhead in the Chapter 14 latency model is not decomposed; Chapter 14 hypothesizes deeper FP8 pipeline startup or related overhead but does not resolve it.

[GAP] Single-QMMA versus chain-last control-code deltas beyond bits 26 and 27 remain partially decoded only.

[GAP] NCU validation of the Chapter 14 latency and pipeline-utilization interpretation was not performed.

---

Source: `knowledge/FINDINGS.md`.
