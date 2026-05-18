# PATTERN-09: Register spill and local-memory frame

**Category**: memory
**Evidence**: [OBS] Kernel 12 contains the controlled SM120 register-pressure and local-array observations for this pattern.
**Confidence**: [INF] C2 for Kernel 12 controlled `-maxrregcount`, accumulator-pressure, local-array, and call-argument variants; C1 for static recognition in future production dumps unless source mapping, runtime validation, or profiling is added.

**Plain English**

[INF] A register spill means ptxas could not keep every live value in registers, so it stores some per-thread values in local memory and later reloads them.

[INF] Local memory is not shared memory. It is per-thread storage addressed through the stack pointer path, and it usually lives behind the memory system rather than inside the register file.

[INF] In source-level terms, this can come from too much register pressure, a forced register limit, or an actual local array such as `int arr[64]`. The SASS signature is a stack-frame allocation plus `STL` and `LDL` traffic.

[INF] Seeing this pattern is evidence for local-memory traffic. It does not by itself prove a performance bottleneck, because ptxas may interleave spills with compute, and the runtime cost depends on cache behavior, occupancy, and how frequently the path executes.

**SASS signature**

[OBS] Kernel 12 observes the standard stack pointer register `R1` loaded from `c[0x0][0x37c]` even in kernels that do not spill.

[OBS] Kernel 12 observes an additional stack-frame allocation when local memory is needed: `IADD R1, R1, -0x128` in 12i and `IADD R1, R1, -0x100` in 12k.

[INF] The extra negative `IADD` on `R1` after the standard prologue is the first strong static signal that the kernel has a local-memory frame for spills or local arrays.

[OBS] Kernel 12 observes scalar local-memory stores as `STL [R1+offset], R` and scalar local-memory loads as `LDL R, [R1+offset]`.

[OBS] Kernel 12 observes `LDL.LU R, [R1+offset]` in 12i on local loads whose destination is consumed soon after reload.

[HYP] The `.LU` suffix is a last-use hint for local-memory caching or scheduling; Kernel 12 observes the usage pattern but does not decode the cache semantics.

[OBS] Kernel 12 observes vectorized local stores as `STL.128 [R1+offset], R` in 12k.

[OBS] Kernel 12 observes `R2UR UR7, R1` in 12k, followed by runtime-indexed local loads such as `LDL R24, [R37+UR7]`.

[INF] In 12k, `R2UR UR7, R1` enables an addressing form where a per-thread computed offset register is added to a uniform copy of the local-memory base.

[OBS] Kernel 12 observes the runtime-index mask `LOP3.LUT R, R_idx, 0xfc, RZ, 0xc0, !PT` before local-array `LDL` access.

[INF] For the tested `int arr[64]` case, the `0xfc` mask both aligns to 4-byte elements and keeps the indexed byte offset inside the 64-element array range.

**Variants**

[OBS] Kernel 12 observes no spill in 12a, 12b, 12c, and 12d for the simpler 10-accumulator kernel, even with `-maxrregcount=24`; `-maxrregcount=16` is raised by ptxas to the SM120 24-register floor.

[RES] Kernel 12 resolves that ptxas will not allocate fewer than 24 registers per thread for the tested SM120 profile.

[OBS] Kernel 12 observes no spill in 12e for a 16-accumulator loop without a register cap; ptxas uses up to `R34`.

[OBS] Kernel 12 observes no spill in 12f for the same 16-accumulator loop with `-maxrregcount=24`; ptxas restructures the loop body instead.

[INF] In the tested pressure range, ptxas prefers restructuring live ranges before emitting local-memory spill traffic.

[OBS] Kernel 12 observes massive spill in 12i for a 32-accumulator loop with `-maxrregcount=24`: 170+ `STL`/`LDL` pairs and a kernel body around 550 instructions.

[OBS] Kernel 12 observes local-array allocation in 12k for `int arr[64]`: a 0x100-byte frame, `STL.128` vectorized initialization, and runtime-indexed `LDL` reads.

[OBS] Kernel 12 observes no spill in 12j for a `__noinline__` function with 16 float arguments; arguments are passed in registers rather than through a stack argument area.

[HYP] The no-spill local-call behavior may change under separate compilation or `-rdc=true`; Kernel 12 does not test that ABI boundary.

**Interpretation**

[INF] `STL`/`LDL` traffic with `R1` as base identifies per-thread local-memory traffic, not CTA shared-memory traffic.

[INF] A frame allocation plus many interleaved `STL` and `LDL.LU` instructions around arithmetic identifies a register-pressure spill pattern.

[INF] A frame allocation plus `STL.128` burst initialization and runtime-indexed `LDL [R_offset+UR_base]` identifies a source local-array pattern.

[INF] In the observed 12i rolling-window spill, ptxas interleaves local stores and loads with FFMA work rather than batching all spills at region boundaries.

[INF] In the observed 12i pressure case, ptxas preserves the loop unroll and pays local-memory traffic instead of reducing the unroll factor.

[INF] The static cost signal is instruction-count and local-memory traffic expansion. The runtime cost still requires profiling because local memory can hit in cache and because scheduling can hide some latency.

**Anti-patterns**

[INF] The standard `LDC R1, c[0x0][0x37c]` prologue alone is not a spill. It appears even when no local memory is used.

[INF] A local `CALL.REL.NOINC` by itself is not spill evidence; Kernel 12 observes no `STL`/`LDL` around the 16-argument local call in 12j.

[INF] `STS` and `LDS` are shared-memory instructions, not local-memory spill instructions.

[INF] `STL.128` in a local-array initialization should not be interpreted as global or shared vector store traffic.

[INF] `-maxrregcount` alone is not proof of spill; Kernel 12 shows ptxas can ignore ineffective caps, raise too-low caps to 24, or restructure code without spilling.

**Open gaps**

[GAP] Runtime performance cost of the observed spill patterns is not profiled.

[GAP] Frame alignment is only hypothesized from observed 0x100 and 0x128 frame sizes.

[GAP] `LDL.128` is not observed in Kernel 12.

[GAP] Complete `.LU` semantics are not decoded.

[GAP] Separate-compilation call ABI behavior is not tested.

---

Source: `knowledge/FINDINGS.md`.
