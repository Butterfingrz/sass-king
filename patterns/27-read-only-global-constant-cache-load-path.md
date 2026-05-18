# PATTERN-27: Read-only global constant-cache load path

**Category**: memory
**Evidence**: [OBS] Chapters 11, 15, 19, 20, 24, and 25 contain controlled SM120 observations of `LDG.E.CONSTANT` in math table access, tensor operand loads, sparse metadata loads, scale loads, and runtime layout/epilogue reads.
**Confidence**: [INF] C2 for recognizing the instruction form and separating it from `LDC`/`LDCU`; C1 for inferring source qualifiers, cache behavior, or algorithmic role without producer/consumer context.

**Plain English**

[INF] `LDG.E.CONSTANT` is a global memory load using the read-only/constant-cache path. In plain terms, it means the data is being fetched through a read-only global-load route, not from the kernel argument constant bank.

[INF] This pattern is useful because the same instruction form appears in very different roles: a trig lookup table, tensor input fragments, sparse metadata, scale vectors, and runtime layout data. The opcode tells you the memory path, while the surrounding consumers tell you what the value means.

[INF] Seeing `LDG.E.CONSTANT` identifies read-only global dataflow. It does not by itself prove the source was a `__constant__` symbol, a Payne-Hanek table, a tensor operand, or a compiler-guaranteed invariant pointer.

**SASS signature**

[OBS] Chapter 11 observes `LDG.E.CONSTANT R10, desc[UR6][R6.64]` inside the large-argument `sinf` Payne-Hanek slowpath.

[OBS] Chapter 11 documents `.CONSTANT` as an `LDG.E` modifier and distinguishes it from `LDC` and `LDCU`.

[OBS] Chapter 15 observes narrow-fragment probes using `LDG.E.CONSTANT` loads for operand fragments before low-precision MMA instructions.

[OBS] Chapter 19 sparse MMA dumps repeatedly observe `LDG.E.CONSTANT R, desc[UR4][R.64+offset]` forms feeding dense operand fragments and sparse metadata channels.

[OBS] Chapter 20 control-flow probes include `LDG.E.CONSTANT` in loop, branch, and template-specialized tensor-core kernels.

[OBS] Chapter 24 production mini-GEMM probes observe `LDG.E.CONSTANT R, desc[UR][R.64]` for parameter, scale, and metadata load channels.

[OBS] Chapter 25 STSM epilogue probes observe many `LDG.E.CONSTANT` loads for runtime layout and storeback data feeding later matrix-store or global-store paths.

**Variants**

[OBS] Table lookup: Chapter 11 uses `LDG.E.CONSTANT` in a counted integer loop that reads chunks of a `2/pi` table.

[OBS] Operand-fragment load: Chapters 15, 19, 20, and 24 use `LDG.E.CONSTANT` to fetch read-only tensor operand fragments directly into registers.

[OBS] Metadata or scale load: Chapters 19 and 24 use `LDG.E.CONSTANT` values as sparse metadata or block-scale operands for tensor-core instructions.

[OBS] Runtime layout/epilogue read: Chapter 25 uses `LDG.E.CONSTANT` for read-only global values that feed layout decoding, packing, STSM, or storeback preparation.

**Interpretation**

[INF] The reliable first-pass conclusion is memory-path classification: descriptor-addressed global load plus `.CONSTANT` read-only path.

[INF] The consumer decides the semantic role. A later `IMAD.WIDE` accumulation loop points toward table-driven arithmetic; a later `QMMA.SP` metadata operand points toward sparse metadata; a later `OMMA.SF` scale operand points toward block-scaling data.

[INF] `LDG.E.CONSTANT` should be kept separate from `LDC`/`LDCU`: `LDC` and `LDCU` read constant-memory banks such as launch parameters, while `LDG.E.CONSTANT` still uses the global descriptor-addressed form `desc[UR][R.64]`.

[INF] Multiple scalar `LDG.E.CONSTANT` instructions with explicit offsets can be a compiler-selected scalarization of read-only vector or fragment data; vector width should not be assumed unless the SASS opcode carries a width suffix.

[INF] A `const __restrict__` source pointer is a plausible producer in some controlled kernels, but future audits should cite the source signature or surrounding chapter evidence before claiming the qualifier caused the opcode.

**Anti-patterns**

[INF] Do not classify `LDG.E.CONSTANT` as ordinary vectorized global memory access unless the opcode also shows the relevant vector width form.

[INF] Do not classify every `LDG.E.CONSTANT` as a trig table; tensor-core and epilogue probes use the same load path for other read-only data.

[INF] Do not confuse `LDG.E.CONSTANT` with `LDC` or `LDCU` just because the word `CONSTANT` appears in the opcode.

[INF] Do not infer source-level immutability or aliasing guarantees from the opcode alone.

[INF] Do not treat `.CONSTANT` as proof that the data lives in CUDA `__constant__` memory; Chapter 11 explicitly leaves the table-base/global-memory split as an open clarification.

**Open gaps**

[GAP] Exact cache behavior and performance differences among `LDG.E`, `LDG.E.CONSTANT`, `.SYS`, and other modifiers are not profiled.

[GAP] The source-level conditions that force or prevent `.CONSTANT` selection are not fully mapped.

[GAP] The Chapter 11 split between `c[0x4]` table-base materialization and `LDG.E.CONSTANT` global table access remains partially unresolved.

[GAP] Width selection for read-only fragment loads is not fully explained.

[GAP] Cross-architecture stability of the `.CONSTANT` load path is not established.

---

Source: `knowledge/FINDINGS.md`.
