# PATTERN-15: Template-specialized SASS function separation

**Category**: audit_method
**Evidence**: [OBS] Chapter 20 contains controlled template-instantiation probes for this pattern, including one-instantiation and two-instantiation dumps.
**Confidence**: [INF] C2 for Chapter 20 controlled template-instantiation visibility; C1 for static recognition in future production dumps unless source mapping, symbol demangling, build metadata, or compiler-version comparison is added.

**Plain English**

[INF] Template-specialized function separation means that a single CUDA source pattern can appear as more than one compiled SASS function. Each template instantiation, kernel variant, or specialization can have its own function body, instruction count, loop shape, and repeated instruction sites.

[INF] In practical audit terms, the unit of counting is the SASS function body, not the source file and not the whole dump. If a dump contains two `Function :` sections, each section must be classified separately before drawing conclusions about loops, MMA counts, stores, or slowpaths.

[INF] This matters for production kernels because template constants often control tile size, head dimension, dtype, stage count, or epilogue shape. Two specializations from the same source can legitimately produce different SASS without implying that one of them is stale or incorrect.

[INF] Seeing multiple specialized functions is evidence that the binary contains multiple compiled bodies. It does not by itself prove which one was launched, which source template parameters produced it, or which production workload path used it.

**SASS signature**

[OBS] Chapter 20 observes that multiple `Function :` sections in one SASS dump can indicate template specializations or multiple kernels.

[OBS] Variant 20s emits one SASS function for `template_nested_kernel<8,2>` with 64 instructions, no back-edge, 16 `FFMA`, and 16 `FADD`.

[OBS] Variant 20t emits two SASS functions in one dump: `template_nested_kernel<4,2>` and `template_nested_kernel<8,2>`.

[INF] Because the two observed template bodies have different template parameters, audits should not assume they have the same static loop, instruction-count, or repeated-body shape.

[OBS] In Chapter 20, the template-instantiation evidence is separate from runtime launch evidence because runtime numeric validation is blocked in that environment.

**Variants**

[OBS] Variant 20s isolates the single-instantiation case, so the dump has one relevant template-specialized function body for that source form.

[OBS] Variant 20t isolates the multi-instantiation case, so the dump has two relevant template-specialized function bodies generated from one controlled source family.

[OBS] The production-audit gap records an earlier template `<int HEAD_DIM>` case with two visible instantiations, 64 and 128, in the SASS dump.

[RES] Chapter 20 partially resolves the template-specialization visibility gap by showing that controlled template instantiations can be distinguished as separate SASS functions in one dump.

**Interpretation**

[INF] A production audit should first split the dump by `Function :` boundaries, then apply pattern matching inside each function body independently.

[INF] Instruction counts, MMA counts, back-edge counts, spill counts, and epilogue signatures should be reported per function. Aggregating across all functions can mix different specializations and produce misleading totals.

[INF] If two functions come from related template names but have different static shapes, the safe conclusion is that the binary contains multiple specialized bodies. The source-level parameter values require symbol names, demangling, build metadata, or controlled rebuilds.

[INF] A mismatch between expected source repetition and observed MMA count should be triaged against the specific function body that was launched, not against another specialization in the same dump.

[INF] Template specialization can explain why two SASS functions from one source family differ, but it should not be used as a catch-all explanation without matching the function symbol and source/build context.

**Anti-patterns**

[INF] Do not count instructions across multiple `Function :` sections as if they belonged to one kernel body.

[INF] Do not compare an observed MMA count against source loops until the audit has identified the matching SASS function.

[INF] Do not assume the first function in a dump is the launched production path.

[INF] Do not infer exact template parameter values from instruction count alone.

[INF] Do not treat a template-specialized function body as numerically validated just because its SASS shape is recognized.

**Open gaps**

[GAP] Chapter 20 does not prove which function body a production runtime dispatch selected.

[GAP] The effect of production template constants on complete attention/GEMM loop structure remains open until a production-style templated kernel is rebuilt and audited end-to-end.

[GAP] Symbol demangling and source-to-function mapping are not formalized as a complete workflow in this pattern.

[GAP] Compiler-version and cross-architecture stability of template specialization visibility is not established.

---

Source: `knowledge/FINDINGS.md`.
