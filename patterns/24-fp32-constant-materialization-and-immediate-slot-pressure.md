# PATTERN-24: FP32 constant materialization and immediate-slot pressure

**Category**: arithmetic
**Evidence**: [OBS] Kernels 04, 05, 09, 10, and 11 contain SM120 observations for `HFMA2` constant loading, `MOV` immediate loading, FFMA addend immediates, identity constants, and math-helper polynomial constants.
**Confidence**: [INF] C2 for Kernel 05 controlled constant variants; C1 for static recognition in future dumps unless source mapping or bit-level output checks prove the exact intended constant.

**Plain English**

[INF] Constants in SASS are not always loaded with `MOV`. Ptxas can materialize a 32-bit value by using an instruction that looks like arithmetic, especially `HFMA2` with zero multiply inputs and two FP16 immediates.

[INF] The trick works because two 16-bit half-precision results can be packed into one 32-bit register. That 32-bit bit pattern can later be consumed as an FP32 value by `FFMA` or another instruction.

[INF] This pattern matters because FFMA has limited immediate placement in the observed kernels: the addend can appear as an immediate, while multiplier constants are materialized into registers first.

[INF] Seeing this pattern identifies constant setup, not real half-precision computation. It does not by itself prove the compiler heuristic, throughput benefit, or numeric intent without checking the downstream consumer.

**SASS signature**

[OBS] Kernel 04 observes `HFMA2 R2, -RZ, RZ, 1.875, 0.00931549072265625` before FFMA chains that use `R2` as an operand. [INF] The chapter explains this as materializing the FP32 bit pattern for `1.001f`.

[OBS] Kernel 05 observes `HFMA2 R9, -RZ, RZ, 1.875, 0.00931549` before eight FFMA instructions that use `R9` as the multiplier operand.

[OBS] Kernel 05 tests six multiplier-constant variants and observes `HFMA2` materialization for all six, with FP16 immediate halves changing to match the target FP32 bit pattern.

[OBS] Kernel 05 observes `0.5` encoded directly as the FFMA addend immediate in `FFMA R*, R*, R9, 0.5`.

[OBS] Kernel 05 observes that `0.5f` used as a multiplier is still materialized through `HFMA2`, while `0.5` used as the addend is immediate-encoded in FFMA.

[OBS] Chapter 10 observes reduction identity constants loaded with either `HFMA2` or `MOV` depending on the identity value.

[OBS] Chapter 11 observes FP edge values and polynomial constants in math-helper paths, including immediate `+INF` or `-INF` operands and `HFMA2` packed constants consumed by later arithmetic.

**Variants**

[OBS] Packed FP32 constant via HFMA2: `HFMA2 R, -RZ, RZ, half_hi, half_lo`.

[OBS] Plain immediate materialization: `MOV R, 0x...` appears for constants that ptxas does not choose to encode through HFMA2 in the observed contexts.

[OBS] FFMA addend immediate: Kernel 05 uses an immediate in the `src3` addend slot.

[OBS] Identity setup: Chapters 09 and 10 observe HFMA2 for identities representable through packed-half bit patterns and MOV for identities such as `0x7fffffff` or `0xffffffff`.

[OBS] Math-helper constants: Chapter 11 observes constants embedded in slowpath arithmetic and polynomial reconstruction code.

**Interpretation**

[INF] `HFMA2` with `-RZ` and `RZ` operands can be a bit-pattern constant load rather than meaningful FP16 math.

[INF] The downstream consumer determines how to interpret the produced bits. A register produced by HFMA2 can be read later as FP32, integer bits, or another packed representation depending on the consuming instruction.

[INF] In the Kernel 05 FFMA variants, multiplier constants must be in registers in the observed SASS, while the addend constant can be an immediate.

[INF] Constant materialization cost can be amortized when one materialized register feeds many later uses.

[INF] Different constant-loading choices can reflect scheduling and encoding heuristics, not source-level intent.

**Anti-patterns**

[INF] Do not treat every `HFMA2` as productive half-precision arithmetic; first check whether both multiply operands are zero-like and whether a later instruction consumes the bit pattern as another type.

[INF] Do not assume a numeric-looking FP16 immediate is the source-level constant. In this pattern, the two FP16 immediates are pieces of a 32-bit payload.

[INF] Do not infer that all constants are free immediates. Kernel 05 shows multiplier constants paying a materialization instruction in the observed FFMA shape.

[INF] Do not infer pipeline benefit from HFMA2 materialization without a benchmark; the reason for choosing HFMA2 over MOV remains partly heuristic.

[INF] Do not collapse kernel-argument constants loaded by `LDC` with literal materialization through `HFMA2` or `MOV`; they are different channels.

**Open gaps**

[GAP] The exact ptxas heuristic for choosing `HFMA2`, `MOV`, `IMAD`, or another materialization form is not modeled.

[GAP] Whether FFMA can use immediates in multiplier source slots in any other SM120 context remains unproven.

[GAP] Throughput and scheduling impact of HFMA2 versus MOV constant loading are not microbenchmarked.

[GAP] Cross-architecture stability of the HFMA2 materialization idiom is not established.

[GAP] Compiler-flag and CUDA-version sensitivity of the materialization heuristic remains open.

---

Source: `knowledge/FINDINGS.md`.
