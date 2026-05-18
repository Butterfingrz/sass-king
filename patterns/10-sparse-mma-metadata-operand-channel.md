# PATTERN-10: Sparse MMA metadata operand channel

**Category**: tensor_core
**Evidence**: [OBS] Chapter 19 contains the controlled SM120 sparse MMA observations for this pattern; Chapters 23 and 24 contain supporting fragment-layout and production-like contexts where sparse metadata and scale operands appear.
**Confidence**: [INF] C2 for Chapter 19 controlled sparse MMA opcode, metadata, selector, and scale variants; C3 for Chapter 24 runtime smoke execution of production-like sparse probes; C1 for static recognition in future production dumps unless source mapping, numeric correctness, or profiling is added.

**Plain English**

[INF] Sparse MMA means the tensor-core instruction is told that one input matrix has structured zeros. The instruction therefore needs normal A/B fragments plus a compact metadata value that describes which elements are present.

[INF] In SASS, that metadata is not hidden inside the opcode. It is a visible register operand consumed by sparse `QMMA` or `OMMA` instructions.

[INF] For block-scaled sparse forms, there is also a separate scale operand channel. Metadata tells the tensor core which sparse elements to use; scale values tell it how to rescale low-precision blocks.

[INF] Seeing this pattern identifies a sparse tensor-core compute region. It does not by itself prove that the metadata is numerically correct, that the source matrix satisfies the expected sparsity contract, or that the sparse path is faster than dense compute.

**SASS signature**

[OBS] Chapter 19 observes non-scaled sparse `kind::f8f6f4` lowering to `QMMA.SP.16864.F32.E4M3.E4M3 R4, R4, R16, R20, R0, 0x0`.

[OBS] The non-scaled sparse `QMMA.SP` form uses six visible SASS operands: D base, A base, B base, C base, metadata register, and selector immediate.

[OBS] Chapter 19 observes metadata materialized as a normal register value before the sparse MMA, for example `MOV R0, 0xaaaaaaaa`.

[OBS] Changing metadata constants from `0xaaaaaaaa` to `0x55555555` or `0xffffffff` changes the metadata producer instruction but leaves the tested `QMMA.SP` opcode bytes and control code unchanged.

[INF] Metadata is therefore a runtime register operand at the SASS level for the tested sparse forms, not an opcode field or a control-code field.

[OBS] Sparse non-scaled `kind::f8f6f4` adds the `.SP` mnemonic modifier and uses low opcode byte `0x7a`, the same low byte observed for dense QMMA in Chapters 14 and 15.

[INF] For the tested non-scaled sparse form, `QMMA.SP` remains in the QMMA opcode family rather than becoming a separate sparse-only opcode family.

[OBS] Sparse non-scaled shape is printed as `.16864`, while the related dense QMMA forms in Chapters 14 and 15 are printed as `.16832`.

[INF] For the tested non-scaled sparse QMMA family, the printed sparse K field doubles relative to the dense low-precision QMMA shape while staying in the QMMA opcode family.

[OBS] Chapter 19 observes selector `0x0` as the final immediate operand in accepted sparse non-scaled forms.

[OBS] Chapter 19 observes selector `1` rejected by ptxas for the tested `kind::f8f6f4` warp-level sparse form.

[OBS] Chapter 19 observes sparse block-scaled `kind::mxf8f6f4` lowering to `QMMA.SF.SP`.

[OBS] Chapter 19 observes sparse block-scaled `kind::mxf4nvf4` lowering to `OMMA.SF.SP`.

[OBS] Sparse block-scaled SASS operand order after C is metadata register, scale register, `URZ`, selector immediate.

[INF] The metadata and scale operands should be tracked as separate dependency channels in audits, because Chapter 19 and Chapter 24 show they are loaded or materialized independently before the MMA consumes them.

**Variants**

[OBS] Chapter 19 observes non-scaled sparse `kind::f8f6f4` as `QMMA.SP.16864`.

[OBS] Chapter 19 observes sparse `kind::mxf8f6f4` block-scale forms as `QMMA.SF.SP.16864`.

[OBS] Chapter 19 observes sparse `kind::mxf4nvf4` block-scale forms as `OMMA.SF.SP.168128`.

[RES] Chapter 19 resolves the tested SM120 sparse MMA opcode question at SASS level: non-scaled sparse `kind::f8f6f4` emits `QMMA.SP`, sparse `kind::mxf8f6f4` emits `QMMA.SF.SP`, and sparse `kind::mxf4nvf4` emits `OMMA.SF.SP`.

[RES] Chapter 19 resolves sparse metadata placement at SASS operand level for the tested forms: metadata is an explicit register operand before the selector immediate, or before the scale register in the block-scaled operand tail.

[OBS] Chapter 19 observes 19m as a 16-instruction sparse QMMA chain: the first instruction writes `R12` with C=`RZ`, middle instructions write `R12` with C=`R12`, and B carries `.reuse` until the last instruction.

[INF] This matches the dense QMMA accumulator-chain shape while preserving the extra sparse metadata operand.

[INF] Sparse tensor-core chaining should still be audited with the same accumulator-chain rules as dense QMMA or OMMA, while preserving the extra sparse metadata and scale operands.

[OBS] Chapter 23 observes sparse metadata separation again in fragment-layout probes, including `QMMA.SF.SP.16864.F32.E3M2.E2M1.E8`.

[OBS] Chapter 24 observes production-like sparse probes: 24r emits `QMMA.SP.16864.F32.E4M3.E4M3`, 24s emits `OMMA.SF.SP.168128.F32.E2M1.E2M1.UE4M3.4X`, and 24ab loads sparse metadata with `LDG.E.CONSTANT` before `QMMA.SP`.

**Interpretation**

[INF] A sparse MMA signature is the `.SP` modifier plus the extra metadata operand, not merely a dense `QMMA` or `OMMA` mnemonic in a low-precision kernel.

[INF] In a production dump, a sparse tensor-core region should be read as at least three dependency channels: fragment data, sparse metadata, and, for `.SF.SP`, scale values.

[INF] The metadata producer may be an immediate `MOV` in a controlled microkernel or a memory load such as `LDG.E.CONSTANT` in a production-like kernel; the key audit question is whether that producer reaches the sparse MMA metadata operand.

[INF] `QMMA.SP` and `QMMA.SF.SP` are sparse QMMA-family forms, while `OMMA.SF.SP` is a sparse block-scaled OMMA-family form. They should not be collapsed into one dense MMA bucket during audit.

[INF] The sparse MMA instruction proves that the sparse tensor-core path is selected. It does not prove that the metadata values match the actual packed A/B data, because runtime numeric validation of the sparse metadata patterns remains open.

**Anti-patterns**

[INF] A low-precision dense `QMMA.16832` or `OMMA.SF.16864` without `.SP` is not this sparse metadata pattern.

[INF] A standalone metadata-looking constant such as `0xaaaaaaaa` is not sufficient evidence for sparse MMA unless a later `.SP` instruction consumes it as the metadata operand.

[INF] Scale loads alone are not sparse metadata. Dense block-scaled `OMMA.SF` also uses scale operands without sparse `.SP` metadata.

[INF] The selector immediate is not the metadata value. In the observed non-scaled sparse form, metadata is the register operand before the selector immediate.

[INF] Sparse SASS structure is not a performance claim. Sparse MMA latency, metadata-load overhead, and effective speedup require separate measurement.

**Open gaps**

[GAP] Runtime validity of Chapter 19 metadata patterns `0xaaaaaaaa`, `0x55555555`, and `0xffffffff` is not measured.

[GAP] Sparse QMMA and sparse OMMA latency are not measured.

[GAP] Sparse metadata load overhead in production-like paths is not profiled.

[GAP] Exact bit-level meaning of the sparse `.SP` modifier inside QMMA and OMMA opcode/control fields remains incomplete.

[GAP] Selector semantics across all warp-level sparse MMA families are not fully mapped.

---

Source: `knowledge/FINDINGS.md`.
