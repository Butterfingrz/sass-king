# PATTERN-20: Kernel argument and global descriptor addressing spine

**Category**: memory_addressing
**Evidence**: [OBS] Kernels 01, 03, 05, 08, and 24 contain controlled SM120 observations for kernel argument loads, global descriptors, address arithmetic, and descriptor-based global memory operations.
**Confidence**: [INF] C2 for the controlled basics kernels that map simple CUDA pointer expressions to SASS; C1 for static recognition in future production dumps unless source variants or runtime validation prove the exact source-level expression.

**Plain English**

[INF] This pattern is the reusable SASS shape observed when a kernel gets from CUDA kernel arguments to a real global-memory access. The kernel first loads pointer-like arguments from constant memory, builds a per-thread byte address, then uses a uniform global descriptor plus that address to load or store data.

[INF] In source terms, this is the SASS shape behind expressions like `a[i]`, `b[i]`, and `c[i]`, including vector forms such as `float4` where the stride and access width are larger.

[INF] The useful audit move is to separate three roles: the argument pointer loaded from `c[0x0][...]`, the per-thread index-to-byte conversion done by `IMAD.WIDE`, and the descriptor operand `desc[UR][R.64]` used by `LDG`, `STG`, `LDGSTS`, or related global-memory instructions.

[INF] Seeing this pattern identifies global-memory addressing structure. It does not by itself prove coalescing, cache behavior, alignment correctness, aliasing properties, or source-level array type without width, stride, and surrounding evidence.

**SASS signature**

[OBS] Kernel 01 observes the baseline identity and bounds prologue: `LDC R1, c[0x0][0x37c]`, `S2R R0, SR_TID.X`, `S2UR UR4, SR_CTAID.X`, scalar argument loads from `c[0x0]`, index construction with `IMAD`, bounds check with `ISETP`, and `@P0 EXIT`.

[OBS] Kernel 01 observes pointer arguments loaded from constant memory with `LDC.64 R2, c[0x0][0x380]`, `LDC.64 R4, c[0x0][0x388]`, and `LDC.64 R6, c[0x0][0x390]`.

[OBS] Kernel 01 observes the global descriptor loaded as `LDCU.64 UR4, c[0x0][0x358]`.

[OBS] Kernel 01 observes per-thread byte addresses built with `IMAD.WIDE R*, R_index, 0x4, R_pointer` before scalar FP32 global accesses.

[OBS] Kernel 01 observes descriptor-based global accesses in the form `LDG.E R2, desc[UR4][R2.64]`, `LDG.E R5, desc[UR4][R4.64]`, and `STG.E desc[UR4][R6.64], R9`.

[OBS] Kernel 08 observes the vector-width variant: `IMAD.WIDE R*, R_index, 0x10, R_pointer`, `LDG.E.128 ... desc[UR4][R*.64]`, and `STG.E.128 desc[UR4][R*.64], ...` for `float4`-like accesses.

[OBS] Chapter 24 production mini-GEMM probes repeatedly observe the same descriptor form in tensor-core contexts, including `STG.E.128 desc[UR][R.64]`, `LDGSTS.E.LTC128B.128 [R], desc[UR][R.64]`, and `LDG.E.CONSTANT R, desc[UR][R.64]`.

**Variants**

[OBS] The scalar elementwise examples observe `IMAD.WIDE` with an element-size immediate such as `0x4` before scalar `LDG.E` or `STG.E`.

[OBS] The vectorized global-memory examples observe larger stride immediates such as `0x10` and matching wide opcodes such as `LDG.E.128` and `STG.E.128`.

[OBS] Async global-to-shared copies in Chapter 24 use the same descriptor-address source form through `LDGSTS.E.LTC128B.128 [shared], desc[UR][R.64]`.

[OBS] Read-only production probes in Chapter 24 observe `LDG.E.CONSTANT R, desc[UR][R.64]`.

[INF] This is a global descriptor-addressed load form and should not be confused with `LDC`/`LDCU` argument loads from constant memory.

[OBS] Kernel 03 observes independent pointer or descriptor `LDC`/`LDCU` loads assigned distinct scoreboards before downstream `IMAD.WIDE` address computations.

**Interpretation**

[INF] `c[0x0][...]` loads in this pattern are the kernel argument channel, not loads from the user arrays themselves.

[INF] The `desc[UR][R.64]` operand separates the global descriptor in the uniform register file from the lane-varying 64-bit address in the per-thread register file.

[INF] The `IMAD.WIDE` immediate is useful evidence for byte stride. Combined with `LDG/STG` width, it can help identify scalar versus vectorized addressing.

[INF] Distinct scoreboards on nearby pointer loads can allow downstream address computations to proceed independently, while later global data loads may be grouped if they feed one consumer.

[INF] In production tensor-core dumps, this spine often frames the interesting compute region: descriptor-address loads bring data in, tensor-core instructions transform it, and descriptor-address stores write results out.

**Anti-patterns**

[INF] Do not confuse `LDC`/`LDCU` from `c[0x0][...]` with a global data load; those instructions fetch launch parameters, pointers, descriptors, or constants.

[INF] Do not infer array element type from `LDG/STG` alone. Width and stride show bytes moved, while arithmetic and source context are needed for semantic type.

[INF] Do not infer coalescing or alignment correctness only from `LDG.E.128` or `STG.E.128`; alignment assumptions require source, ABI, or runtime evidence.

[INF] Do not assume all global-memory users follow the simple scalar ordering from Kernel 01. Production kernels can defer pointer loads, interleave address arithmetic, use async copies, or use read-only `LDG.E.CONSTANT` paths.

[INF] Do not treat descriptor-based addressing as a complete memory-dependency model; scoreboards, waits, barriers, and cache modifiers still need separate audit.

**Open gaps**

[GAP] The exact descriptor layout loaded by `LDCU.64` is not decoded.

[GAP] Cross-architecture stability of the constant-memory offsets for descriptors and launch parameters is not established.

[GAP] Cache modifier semantics for `LDG.E`, `LDG.E.CONSTANT`, `LDGSTS`, and store variants are not fully profiled.

[GAP] Alignment and coalescing behavior are not proven by this static pattern.

[GAP] The full set of descriptor-addressing forms used by production libraries remains future audit scope.

---

Source: `knowledge/FINDINGS.md`.
