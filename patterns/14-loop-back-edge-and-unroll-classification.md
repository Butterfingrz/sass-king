# PATTERN-14: Loop back-edge and unroll classification

**Category**: control_flow
**Evidence**: [OBS] Chapter 20 contains the controlled SM120 loop-lowering, unroll-pragma, dynamic-loop, template-instantiation, and repeated-body probes for this pattern; [OBS] Chapter 18 contains a supporting tensor-core K-loop case with an explicit `#pragma unroll 1` back-edge.
**Confidence**: [INF] C2 for Chapter 20 controlled source-to-SASS loop and unroll variants; C1 for static recognition in future production dumps unless source mapping, runtime validation, profiling, or compiler-version comparison is added.

**Plain English**

[INF] Loop back-edge classification means deciding whether source-level repetition is still a real loop in SASS or whether ptxas expanded it into straight-line instructions. The visual test is simple: a preserved SASS loop has a branch whose target goes backward to an earlier instruction in the useful body.

[INF] A fully unrolled loop has no useful-body back-edge. Instead, the repeated work appears as repeated static instruction sites: multiple `FFMA`, `FADD`, `HMMA`, stores, or other body instructions laid out in order.

[INF] A partially unrolled loop can show both signals at once: several copies of the body appear before a backward branch. For example, an unroll-by-4 loop has a 4-wide body and then a branch back to the loop header.

[INF] This pattern helps production audits avoid a common mistake: counting source loops rather than SASS sites. The dump tells you what the GPU will execute statically; it does not prove the original source trip count unless source mapping or controlled variants are available.

**SASS signature**

[OBS] Chapter 20 defines a back-edge mechanically as a `BRA` or `BRA.U` whose target offset is lower than the branch instruction offset.

[OBS] Variant 20d observes a preserved constant loop from `#pragma unroll 1` with `BRA.U UP0, 0xe0` at offset `0x0130`.

[OBS] Variant 20n observes a partially unrolled constant loop from `#pragma unroll 4` with a 4-iteration body and a backward `BRA.U UP0, 0xe0` at offset `0x01f0`.

[OBS] Dynamic-loop variants 20c, 20p, and 20q observe multi-path loop structures with three back-edges each.

[OBS] Fully unrolled constant-loop variants 20a, 20b, 20e, 20f, 20g, 20h, 20m, 20s, 20u, and 20v observe no useful-body back-edge.

[OBS] Chapter 20 observes a terminal self-trap `BRA` after `EXIT`.

[RES] That terminal trap branch is not a loop back-edge and must not be counted as useful loop structure.

[OBS] Variant 20k observes `BSSY.RECONVERGENT` and `BSYNC.RECONVERGENT` around a `break`-capable loop body.

[RES] Ordinary preserved loops do not require BSSY/BSYNC in the tested variants.

**Variants**

[OBS] Variants 20a and 20b show default full unroll for constant scalar trip counts 4 and 16.

[OBS] Variants 20e and 20f show default full unroll for nested constant scalar loops with 4 x 2 and 8 x 2 shapes.

[OBS] Variants 20g and 20h show default full unroll for nested constant HMMA loops with 4 x 2 and 8 x 2 shapes; 20h emits 16 visible `HMMA.16816.F32` instructions.

[OBS] Variant 20m shows explicit full unroll matching the default full-unroll shape of 20b.

[OBS] Variant 20o shows a dynamic loop with `#pragma unroll 4` lowering to a 4-wide body plus tail structure and two back-edges.

[OBS] Variants 20i, 20j, and 20l show conditional loop bodies using predicated arithmetic or predicated branches without BSSY/BSYNC.

[OBS] Variant 20r shows a dynamic loop with volatile global stores through `STG.E.STRONG.SYS`.

[OBS] Variant 20t shows that one dump can contain two separate SASS functions for two template instantiations.

[OBS] Variant 20u shows unique per-iteration stores survive as 16 `STG.E` instructions, while 20v shows an identical repeated scalar body remains 16 `FADD` instructions in the tested source.

[RES] Chapter 20 resolves that, in the tested variants, no SASS-level loop without a useful-body back-edge was observed.

**Interpretation**

[INF] In a production dump, first ignore the terminal trap/padding region after `EXIT`, then search useful code for backward `BRA` or `BRA.U` targets.

[INF] If the useful body has no back-edge but contains repeated MMA/arithmetic/store sites, the safe conclusion is full unroll of the visible SASS body, not a hidden loop.

[INF] If the useful body contains repeated sites followed by a back-edge, classify it as partial unroll or a dynamic-loop cascade rather than a single scalar source iteration.

[INF] Dynamic trip counts can produce multi-path cascades instead of one compact loop, so multiple back-edges do not necessarily mean multiple source loops.

[INF] Template instantiations must be audited per SASS function. A source file can produce multiple specialized functions with different static instruction counts.

[INF] BSSY/BSYNC around a loop region is evidence for reconvergence handling around a non-structured or divergent region, not evidence that ordinary loops are encoded through BSSY/BSYNC.

**Anti-patterns**

[INF] Do not count a self-targeting terminal `BRA` after `EXIT` as a loop.

[INF] Do not infer the source trip count from repeated instruction count alone unless the source, template parameters, or controlled variants are available.

[INF] Do not assume a source loop exists just because a dump contains repeated arithmetic or repeated MMA instructions; the loop may be fully unrolled.

[INF] Do not assume absence of a back-edge proves no source loop existed; it proves only that the tested SASS body has no preserved loop in that region.

[INF] Do not treat BSSY/BSYNC as the normal encoding for ordinary loops in the tested SM120 variants.

**Open gaps**

[GAP] Runtime numeric validation of Chapter 20 variants remains blocked by the unavailable NVIDIA driver in that environment.

[GAP] Exact ptxas thresholds and heuristics for choosing full unroll, partial unroll, and dynamic-loop cascades are not fully decoded.

[GAP] The original production FP4 attention 2-QMMA observation remains unresolved without rebuilding and comparing the original source/binary pair.

[GAP] Compiler-version and cross-architecture stability of these loop-lowering heuristics is not established.

---

Source: `knowledge/FINDINGS.md`.
