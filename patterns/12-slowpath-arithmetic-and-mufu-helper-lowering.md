# PATTERN-12: Slowpath arithmetic and MUFU helper lowering

**Category**: arithmetic
**Evidence**: [OBS] Kernel 11 contains the controlled SM120 division, math-library, MUFU, subnormal-handling, local-CALL, and Payne-Hanek observations for this pattern; Kernel 06 contains supporting external u16 helper-call context; Kernel 12 contains supporting local-CALL-without-spill context.
**Confidence**: [INF] C2 for Kernel 11 controlled arithmetic-source variants; C1 for static recognition in future production dumps unless source mapping, runtime validation, numeric checking, or profiling is added.

**Plain English**

[INF] Slowpath arithmetic means a simple-looking source operation expands into a larger SASS sequence because the compiler has to handle hard numeric cases: division, subnormal floats, NaN/Inf, large-angle trig reduction, or integer width corner cases.

[INF] MUFU instructions are hardware approximation helpers for functions such as reciprocal, reciprocal square root, log2, and exp2. They are fast building blocks, but ptxas often surrounds them with range checks, scaling, correction arithmetic, or fallback paths.

[INF] A local slowpath is code inside the same SASS function that is reached only for selected input cases. It can appear after the main `EXIT` and be called with `CALL.REL.NOINC`, or it can appear as a branch-controlled inline region.

[INF] Seeing this pattern is evidence for nontrivial compiler-generated arithmetic lowering. It does not by itself prove numerical accuracy, source-level intent, or runtime hotness of the slowpath without runtime inputs and profiling.

**SASS signature**

[OBS] Kernel 11 observes hardware MUFU forms `MUFU.RCP`, `MUFU.LG2`, `MUFU.EX2`, and `MUFU.RSQ`.

[OBS] Kernel 11 observes integer division lowering through a reciprocal-multiply skeleton that includes conversion to float, `MUFU.RCP`, float-to-integer conversion, and integer correction arithmetic.

[OBS] Kernel 11 observes direct intrinsic-style math forms: `__log2f(x)` emits `MUFU.LG2` plus subnormal handling, `rsqrtf(x)` emits `MUFU.RSQ` plus subnormal handling, and `__fdividef(a,b)` emits `MUFU.RCP` plus `FMUL` and subnormal handling.

[OBS] Kernel 11 observes standard math forms that are not just one MUFU: `log2f(x)` emits an inline polynomial, `expf(x)` emits range reduction plus `MUFU.EX2`, `sinf(x)` emits a fast path plus a Payne-Hanek slowpath, and `sqrtf(x)` emits a fast path plus a local slowpath.

[OBS] Kernel 11 observes the canonical FP32 subnormal guard skeleton around MUFU or polynomial work: `FSETP.GEU` against FLT_MIN, predicated multiply by `16777216`, arithmetic work, then predicated correction.

[OBS] Kernel 11 observes local subroutine slowpaths using `CALL.REL.NOINC <local_address>` and `RET.REL.NODEC Rn 0x0`.

[OBS] Kernel 11 observes the caller materializing a return address in a register before local `CALL.REL.NOINC`.

[OBS] Kernel 11 observes local slowpath bodies placed after the main `EXIT` in the same `.text` section.

[OBS] Kernel 11 observes Payne-Hanek large-argument `sinf` reduction using `LDG.E.CONSTANT` table loads, a uniform loop, integer accumulation, FP64 `DMUL`, and `F2F.F32.F64`.

**Variants**

[OBS] Kernel 11a `u32` runtime division is fully inline and has no `CALL`.

[OBS] Kernel 11b `u64` runtime division uses an inline fast path plus `CALL.REL.NOINC 0x2e0` to a local slowpath after the main `EXIT`.

[OBS] Kernel 11c `s32` runtime division uses integer absolute value plus the unsigned division skeleton and sign fix-up.

[OBS] Kernel 11d standard `log2f(x)` emits a fully inline polynomial approximation and no `MUFU.LG2`.

[OBS] Kernel 11e `__log2f(x)` emits `MUFU.LG2` with subnormal handling.

[OBS] Kernel 11f standard `expf(x)` emits range reduction, `MUFU.EX2`, and final scaling, without a local `CALL`.

[OBS] Kernel 11g standard `sinf(x)` emits a fast path for smaller inputs and a branch-controlled Payne-Hanek slowpath for large inputs.

[OBS] Kernel 11h standard `sqrtf(x)` emits `MUFU.RSQ`, one Newton-Raphson refinement in the fast path, and `CALL.REL.NOINC 0x1d0` for edge cases.

[OBS] Kernel 11i `rsqrtf(x)` emits `MUFU.RSQ` with subnormal handling.

[OBS] Kernel 11j `__fdividef(a,b)` emits `MUFU.RCP`, `FMUL`, and subnormal handling on both operands.

[OBS] Kernel 06 observes an external named helper for sub-word modulo, `__cuda_sm20_rem_u16`, while Kernel 11 observes no external named helper for tested u32, u64, s32, or math-library variants.

[INF] In the tested CUDA 13.2 SM120 corpus, ptxas prefers inline lowering or local in-kernel slowpaths for u32-and-wider division and tested math functions, while the observed external helper path is limited to the earlier u16 case.

**Interpretation**

[INF] A MUFU mnemonic by itself identifies an approximate hardware math building block, not the full source-level operation. The surrounding guards, scaling, and correction instructions determine whether it is reciprocal, reciprocal sqrt, log2, exp2, divide, sqrt, or part of a larger standard function.

[INF] A `CALL.REL.NOINC` to a local numeric address inside the same kernel should be treated as a local slowpath candidate, especially when the body is after the main `EXIT` and returns with `RET.REL.NODEC`.

[INF] A `CALL.REL.NOINC` to a named helper such as `__cuda_sm20_rem_u16` is a different shape from the local slowpath pattern and should be audited separately.

[INF] `LDG.E.CONSTANT` inside a uniform counted loop with integer accumulation is a strong signature for a table-driven arithmetic reduction, as observed in the `sinf` Payne-Hanek path.

[INF] Standard C math functions and CUDA intrinsic forms can lower very differently: `log2f` and `__log2f` are distinct in Kernel 11, so audits should not infer source spelling from one mnemonic alone.

[INF] Static SASS can identify the existence and structure of slowpaths, but runtime hotness depends on input values. A large slowpath body may be cold for normal inputs.

**Anti-patterns**

[INF] One `MUFU` instruction is not proof of a complete standard library call; it can be one stage inside a larger inline lowering.

[INF] Absence of an external named helper does not mean arithmetic is cheap. Kernel 11 shows inline division and standard math can still expand into many instructions.

[INF] A local `CALL.REL.NOINC` is not automatically register spilling or stack argument passing; Kernel 12 separately shows local calls can pass many arguments in registers without `STL`/`LDL`.

[INF] `LDG.E.CONSTANT` is not ordinary vectorized global memory access. In the observed arithmetic slowpath it is a read-only table path used for range reduction.

[INF] A body placed after `EXIT` is not necessarily dead code; Kernel 11 shows post-`EXIT` local slowpath bodies reached by `CALL.REL.NOINC`.

**Open gaps**

[GAP] Runtime numeric accuracy of the reconstructed standard math sequences is not independently validated against CUDA library specifications.

[GAP] Runtime frequency and performance cost of local slowpaths are not profiled.

[GAP] Exact derivation of integer-division magic constants and correction bounds is not fully proven from first principles.

[GAP] Exact Payne-Hanek accumulator reconstruction from the observed `sinf` slowpath remains incomplete.

[GAP] Whether the same external-helper versus local-slowpath split holds across CUDA versions and other architectures is not established.

---

Source: `knowledge/FINDINGS.md`.
