# PATTERN-22: FFMA fusion and arithmetic chain lowering

**Category**: arithmetic
**Evidence**: [OBS] Kernels 02, 03, and 05 contain controlled SM120 observations for non-fused addition chains, fused multiply-add lowering, and straight-line FFMA chains after compile-time loop unrolling.
**Confidence**: [INF] C2 for the controlled source-to-SASS cases that distinguish `a+b+1.0f`, `a*b+c`, and fixed-trip-count repeated multiply-add; C1 for static recognition in future production dumps unless source mapping or numeric validation proves the exact expression.

**Plain English**

[INF] FFMA is the SASS instruction for a fused floating-point multiply-add in the observed FP32 cases: it computes `a * b + c` as one operation. When ptxas sees the right source expression shape in the tested kernels, it lowers multiply-add work to one `FFMA` instead of separate multiply and add instructions.

[INF] The key point is that fusion is not the same as "any three-term arithmetic expression". A source expression like `a + b + 1.0f` is still two additions and stays as two `FADD` instructions in the tested kernel, while `a * b + c` becomes one `FFMA`.

[INF] In fixed-trip-count loops, repeated source-level multiply-add steps can become a straight-line chain of `FFMA` instructions after unrolling. That chain shows the recurrence dependency directly in registers.

[INF] Seeing this pattern identifies arithmetic lowering and dependency shape. It does not by itself prove fast-math policy, source-level algebraic equivalence for all edge cases, final numeric correctness, or throughput without additional evidence.

**SASS signature**

[OBS] Kernel 02 observes that `a[i] + b[i] + 1.0f` lowers to two `FADD` instructions rather than one fused instruction.

[OBS] Kernel 02 observes the first `FADD` writing an intermediate to recycled register `R0`, followed by a second `FADD` that adds the immediate `1`.

[OBS] Kernel 03 observes `a[i] * b[i] + c[i]` lowering to `FFMA R11, R2, R5, R7`.

[OBS] Kernel 03 records the FFMA operand order as destination, multiply source A, multiply source B, and addend source C. [INF] In the observed example, `FFMA R11, R2, R5, R7` corresponds to `R2 * R5 + R7`.

[OBS] Kernel 03 observes three co-consumed global-load producers sharing `SB4`, followed by the `FFMA` consumer with `wait={SB4}`.

[OBS] Kernel 05 observes a compile-time fixed loop count lowering to eight straight-line `FFMA` instructions and no backward branch or loop-control machinery.

[OBS] Kernel 05 observes the repeated chain shape `FFMA R0, R0, R9, 0.5` for middle iterations, with the final FFMA writing the store value to `R3`.

**Variants**

[OBS] Non-fused addition chain: Kernel 02 emits `FADD` then `FADD` for `a+b+1.0f`.

[OBS] Single fused expression: Kernel 03 emits one `FFMA` for `a*b+c`.

[OBS] Fixed-count recurrence: Kernel 05 emits a straight-line FFMA chain for eight compile-time iterations of `x = x * 1.001f + 0.5f`.

[OBS] Kernel 05 combines the FFMA chain with a separately materialized multiplier constant in `R9`. [INF] The constant-loading mechanism is a separate pattern from FFMA fusion itself.

**Interpretation**

[INF] A single `FFMA` with three source operands is evidence for multiply-add lowering, not merely an add chain.

[INF] Two adjacent `FADD` instructions can be the correct lowering for a three-term addition expression; their presence is evidence that the source did not match the multiply-add template in the tested case.

[INF] Consecutive FFMA instructions that feed the previous destination back as the next multiplicand identify a serial arithmetic recurrence or unrolled loop body.

[INF] When an FFMA waits on a scoreboard shared by several loads, the arithmetic instruction is also the point where the loaded operands become jointly consumed.

[INF] Register reuse inside this pattern should be audited separately from arithmetic fusion, because ptxas may recycle dead registers for intermediates.

**Anti-patterns**

[INF] Do not infer FFMA fusion from "three values are involved" unless the SASS actually contains `FFMA` or equivalent fused arithmetic.

[INF] Do not treat two `FADD` instructions as missed optimization by default. Kernel 02 shows two `FADD`s are the expected result for `a+b+1.0f` in the controlled case.

[INF] Do not infer a source loop from a straight-line FFMA chain alone. The same SASS shape could come from explicit repeated source statements or unrolled code.

[INF] Do not use the presence of `FFMA` alone as a performance claim. Dependency depth, issue throughput, register pressure, load waits, and scheduling still need separate evidence.

[INF] Do not fold constant materialization into this pattern. `HFMA2` constant loading before an FFMA chain is related arithmetic setup, but it has its own evidence and gaps.

**Open gaps**

[GAP] Fast-math, contraction flags, and source-level controls over FFMA fusion are not exhaustively tested.

[GAP] Denormal, rounding, NaN, and signed-zero behavior are not validated for the fused versus non-fused cases.

[GAP] Throughput of independent FFMA streams is not measured by these controlled kernels.

[GAP] Cross-architecture stability of the exact fusion and scheduling choices is not established.

[GAP] Interaction with compiler flags, PTX-level `fma.rn`, and explicit no-contraction source patterns remains future work.

---

Source: `knowledge/FINDINGS.md`.
