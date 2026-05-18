# PATTERN-02: HMMA accumulator chain

**Category**: tensor_core
**Evidence**: [OBS] Chapter 13 contains the controlled HMMA observations for this pattern; Chapters 17, 18, 20, 21, 22, 24, and 25 contain supporting contexts where `HMMA.16816.F32` appears in load, control-flow, epilogue, and production-like kernels.
**Confidence**: [INF] C2 for Chapter 13 controlled accumulator-chaining variants; C3 for the Chapter 13 clock64 latency microbenchmark; C1 for static recognition in future production dumps unless source, runtime, or profile evidence is added.

**Plain English**

[INF] An HMMA accumulator chain means that tensor-core matrix multiply instructions are stacked so each instruction adds into the same running accumulator. In source-level terms, this is the SASS shape behind doing several small matrix multiply-accumulate steps for the same output tile.

[INF] The important idea is "keep accumulating here." One HMMA writes an accumulator fragment, and the next HMMA reads that same fragment as its C input, adds another A x B product, and writes the updated accumulator back to the same registers.

[INF] In a GEMM-like kernel, this is the compute spine of the tile: load fragments, run one or more HMMAs, keep the partial sums live in registers, then eventually store or transform the accumulator. Seeing this pattern identifies an FP16/BF16 tensor-core accumulation region, not a complete GEMM by itself.

[INF] When this pattern appears in a production dump, the safe conclusion is that consecutive HMMA instructions are dependency-chained through the accumulator. It does not by itself prove the original loop nest, the number of source-level K iterations, tensor-core throughput, or final numeric correctness without additional evidence.

**SASS signature**

[OBS] The SM120 HMMA form observed in Chapter 13 is `HMMA.16816.<acc_dtype>[.<input_dtype>]`, with `.16816` identifying the m16n8k16 shape, `.F16` or `.F32` identifying accumulator dtype, and `.BF16` appearing only for BF16 input.

[OBS] HMMA is warp-level in the studied SM120 dumps: one visible `HMMA.16816.*` instruction represents the 32 lanes cooperating on one tensor-core tile fragment, not 32 independent scalar instructions.

[OBS] Dense HMMA uses four visible SASS operands in the order D base, A base, B base, C base; the per-thread fragment spans are implicit in the opcode suffix and dtype.

[OBS] The chained form colocates D and C on the same register base. Chapter 13 observes `HMMA R16, R12, R10, R16` in 13d and `HMMA R16, R4, R2, R16` across the 13e latency chain.

[INF] D/C colocation is the in-place accumulator update form for the observed HMMA chain.

[OBS] In the observed chained form, A and B stay on distinct register bases.

[INF] A and B remain distinct from D because later HMMAs re-read those fragments, while D and C can be colocated for the in-place accumulator under the Chapter 13 resolved allocation rule.

[OBS] Chapter 13 observes `.reuse` on the B operand of every HMMA in a chain except the last one: 13d uses `R10.reuse` on the first HMMA and no `.reuse` on the last; 13e uses `R2.reuse` on HMMAs #1-15 and no `.reuse` on HMMA #16.

[OBS] Chapter 13 observes chain-specific control-code transitions: 13e HMMA #1 uses control code `0x084ff60000001810`, HMMAs #2-15 use `0x080ff60000001810`, and HMMA #16 uses `0x000ff00000001810`.

**Variants**

[OBS] Chapter 13 observes a two-HMMA chain in 13d with FP16 input and FP32 accumulator: 2 x `HMMA.16816.F32`, D/C colocated, B `.reuse` on the first HMMA, and output `d[0] = 32.0`.

[OBS] Chapter 13 observes N-HMMA serial chains in 13e for N=16, N=32, and N=64, bracketed by `clock64()` for latency measurement.

[OBS] Chapter 13 observes single-HMMA FP16 accumulator and FP32 accumulator variants as non-chain baselines: 13a emits `HMMA.16816.F16`, 13b emits `HMMA.16816.F32`, and 13c emits `HMMA.16816.F32.BF16`.

[OBS] Chapter 21 observes a predicated guarded HMMA form, `@P0 HMMA.16816.F32`, after `@P0 WARPSYNC.ALL`.

[INF] This is HMMA control-flow evidence, but it is not by itself an accumulator-chain signature.

[OBS] Chapters 22 and 25 observe HMMA feeding STSM epilogue forms; those contexts show accumulator storeback uses but are separate from the chain signature unless consecutive D/C-colocated HMMAs are visible.

**Interpretation**

[INF] Consecutive HMMAs with D and C on the same accumulator register base identify an accumulator dependency chain because the later HMMA reads the earlier HMMA's output as its C operand and writes the updated accumulator back in place.

[INF] `.reuse` on the B operand identifies ptxas preparing for a later HMMA to re-read the same B fragment. The absence of `.reuse` on the last HMMA is consistent with no later HMMA consumer of that B operand in the observed chain.

[INF] The control-code high-bit transition identifies scoreboard-coordinated variable-latency chaining. Chapter 13 resolves HMMA as scoreboard-using and measures the serial dependency-chain latency at approximately 35 cycles per `HMMA.16816.F32` on SM120.

[INF] The 35-cycle number is a dependency-chain latency result, not a throughput result. It applies when each HMMA must wait on the previous HMMA's accumulator output.

[INF] The 2 x `@!UPT UIADD3 URZ` NOP pad after an HMMA is a scheduling fallback when the next consumer depends on the HMMA D result and no useful independent work is available. It is not intrinsic to every HMMA, because Chapter 13 observes zero NOPs after the final HMMA when the next instruction is an independent `CS2R`.

**Anti-patterns**

[INF] A single `HMMA.16816.*` instruction is tensor-core compute evidence, but it is not an accumulator-chain pattern unless a subsequent HMMA reads the previous accumulator through the C operand or the surrounding SASS shows the dependency.

[INF] Consecutive HMMA instructions using different C/D accumulator bases may be independent accumulators scheduled near each other, not a serial accumulator chain.

[INF] HMMA followed by `STG`, `STSM`, or scalar arithmetic is an accumulator consumer path, not an HMMA chain unless a later HMMA reuses the accumulator as C.

[INF] `@P0 HMMA` guarded by `WARPSYNC.ALL` is a predicated tensor-core execution form. It should not be classified as an accumulator chain without the D/C register-base recurrence.

**Open gaps**

[HYP] HMMA latency for FP16 accumulator and BF16 input variants is not measured; Chapter 13 measures the serial latency for `HMMA.16816.F32` only.

[HYP] HMMA throughput in independent, non-chained configurations is not established by the Chapter 13 latency chain; the measured 35 cycles is serial dependency-chain latency, not tensor-core throughput.

[HYP] HMMA shapes other than m16n8k16 are not tested in Chapter 13; the pattern is currently scoped to `HMMA.16816.*` on SM120.

[HYP] The approximately 310-cycle fixed overhead in the Chapter 13 latency model is not decomposed; possible contributors include timer reads, first-HMMA startup, final visibility, and scoreboard setup.

[GAP] The exact scoreboard slot ID used by HMMA is not decoded.

[GAP] Low-order scheduling bits of the HMMA control code remain partially decoded only.

[GAP] NCU validation of the Chapter 13 latency and pipeline-utilization interpretation was not performed.

---

Source: `knowledge/FINDINGS.md`.
