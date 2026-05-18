# PATTERN-11: Vectorized global memory access

**Category**: memory
**Evidence**: [OBS] Kernel 08 contains the controlled SM120 vectorized global load/store observations for this pattern; Kernel 24 contains supporting production-like epilogue contexts where `STG.E.128` appears after tensor-core compute.
**Confidence**: [INF] C2 for Kernel 08 controlled vector-width, scalar-indexing, and alignment variants; C3 for Kernel 24 runtime smoke execution of production-like `STG.E.128` epilogues; C1 for static recognition in future production dumps unless source mapping, runtime validation, or profiling is added.

**Plain English**

[INF] Vectorized global memory access means one SASS load or store moves several adjacent scalar elements at once. For example, `LDG.E.128` moves 16 bytes and fills four 32-bit registers.

[INF] This is a memory-traffic pattern, not a vector arithmetic pattern. Kernel 08 shows `float4 + float4` still becomes four scalar `FADD` instructions; the vectorization is in the global load/store instructions.

[INF] In source-level terms, ptxas needs a vector type or an equivalent aligned reinterpretation to emit the wide global access. Merely writing four adjacent scalar loads is not enough in the tested SM120 cases.

[INF] Seeing this pattern is evidence for wide global-memory transactions and consecutive-register data movement. It does not by itself prove coalescing efficiency, cache behavior, or end-to-end memory bandwidth without runtime profiling.

**SASS signature**

[OBS] Kernel 08 observes scalar 32-bit global loads as `LDG.E`, 64-bit loads as `LDG.E.64`, 128-bit loads as `LDG.E.128`, and 256-bit loads as `LDG.E.ENL2.256`.

[OBS] Kernel 08 observes matching global store forms including `STG.E.128` and `STG.E.ENL2.256`.

[OBS] `LDG.E.128` loads 128 bits into four consecutive registers from a single destination base register.

[OBS] `STG.E.128` stores 128 bits from four consecutive registers from a single source base register.

[OBS] `LDG.E.ENL2.256` loads 256 bits into eight consecutive registers named through two destination base registers in the visible SASS.

[OBS] `STG.E.ENL2.256` mirrors the 256-bit load pattern for global stores.

[OBS] Kernel 08 observes `.ENL2` only at the tested 256-bit width.

[HYP] In Kernel 08, `.ENL2` likely marks an enlarged encoding that uses two register base fields because eight consecutive registers cannot be represented by one visible register base field.

[OBS] Kernel 08 observes the global addressing form `desc[UR_desc][R_addr.64]` for vectorized global loads and stores.

[OBS] Kernel 08 observes the `IMAD.WIDE` stride immediate matching `sizeof(T)`: `0x4` for `float`, `0x8` for `float2`, `0x10` for `float4`, `0x20` for `float8` and `double4_32a`, and `0x40` for `float16`.

**Variants**

[OBS] Kernel 08a `float4` emits `LDG.E.128` and `STG.E.128`.

[OBS] Kernel 08c `float2` emits `LDG.E.64`, filling the width slot between scalar `LDG.E` and `LDG.E.128`.

[OBS] Kernel 08d 32-byte-aligned `float8` emits `LDG.E.ENL2.256` and `STG.E.ENL2.256`.

[OBS] Kernel 08e 64-byte-aligned `float16` emits two 256-bit global loads per array, including a second `LDG.E.ENL2.256` at immediate offset `+0x20`, rather than a wider 512-bit instruction.

[RES] Kernel 08 resolves that the tested SM120 global LDG width caps at 256 bits.

[OBS] Kernel 08f deprecated `double4` emits four `LDG.E.128` operations for the same 32-byte data size where `float8` uses 256-bit loads.

[OBS] Kernel 08g `double4_32a` emits `LDG.E.ENL2.256`, matching the 32-byte-aligned `float8` memory path.

[RES] Kernel 08 resolves that alignment, not element type alone, determines whether the tested 32-byte transfer can use `LDG.E.ENL2.256`.

[OBS] Kernel 08b scalar `float*` manual indexing emits eight scalar `LDG.E` instructions instead of two `LDG.E.128` instructions, despite contiguous aligned scalar accesses.

[RES] Kernel 08 resolves that ptxas does not auto-vectorize the tested scalar contiguous access pattern into wide global loads.

**Interpretation**

[INF] The practical audit signature is the mnemonic width suffix plus consecutive-register operands: `.64`, `.128`, or `.ENL2.256` on `LDG.E`/`STG.E`.

[INF] `LDG.E.128` or `STG.E.128` in an epilogue identifies wide global movement, not necessarily tensor-core output by itself. The surrounding compute and storeback path must be checked separately.

[INF] In production-like GEMM dumps, vectorized `STG.E.128` often belongs to the output epilogue, while tensor-core compute is identified separately through HMMA/QMMA/OMMA patterns.

[INF] A 256-bit global access is a stronger alignment/type signal than a 128-bit access in the tested SM120 corpus, because Kernel 08 only emits `.ENL2.256` when the source type carries a 32-byte alignment guarantee.

[INF] The static instruction width is not the same thing as achieved memory throughput. Runtime coalescing, cache state, occupancy, and address distribution still need profiling.

**Anti-patterns**

[INF] Four adjacent scalar `LDG.E` instructions are not the same as one `LDG.E.128`; Kernel 08b shows ptxas can leave scalar contiguous source code scalar.

[INF] `LDG.E.CONSTANT` is a constant-cache global load path, not this normal vectorized global load pattern unless it also carries a vector width form in the observed SASS.

[INF] `LDS`, `STS`, `LDSM`, `STSM`, `LDGSTS`, `LDL`, and `STL` are different memory spaces or matrix/local-memory families; they should not be collapsed into global `LDG.E`/`STG.E` vectorization.

[INF] `REDG.E.ADD...` is an atomic/reduction global operation, not a normal vectorized `STG.E.128` epilogue store.

[INF] Vectorized memory access does not imply packed arithmetic. Kernel 08 shows vector data still consumed by scalar FP32 or FP64 arithmetic instructions.

**Open gaps**

[GAP] `.ENL2` semantics beyond the observed 256-bit register encoding are not microbenchmarked; cache, ordering, or latency effects remain unknown.

[GAP] Wider global modes such as a hypothetical 512-bit form are not observed in Kernel 08.

[GAP] Cross-architecture stability of the 256-bit `.ENL2` form is not established for SM80, SM86, SM89, SM90a, or SM100a.

[GAP] The effect of `__restrict__` on scalar auto-vectorization is not tested.

[GAP] Runtime throughput and coalescing behavior for each vector width are not profiled.

---

Source: `knowledge/FINDINGS.md`.
