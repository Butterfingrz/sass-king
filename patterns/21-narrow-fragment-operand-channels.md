# PATTERN-21: Narrow-fragment operand channels

**Category**: tensor_core
**Evidence**: [OBS] Chapters 15, 23, and 24 contain controlled SM120 observations for FP4/FP6 dense fragments, LDSM-fed versus direct-register QMMA inputs, scale vectors, sparse metadata, and production-like narrow MMA channels.
**Confidence**: [INF] C2 for the controlled SASS variants that distinguish dense dtype forms, LDSM-fed paths, direct-register paths, scale-factor operands, and sparse metadata operands; C1 for static recognition in future production dumps unless runtime decode, numeric validation, or source variants prove the exact value layout.

**Plain English**

[INF] Narrow tensor-core operands such as FP4 and FP6 are packed fragments, not ordinary scalar values. A production audit has to track which registers carry fragment data, which registers carry scale factors, and which registers carry sparse metadata.

[INF] This pattern is about operand channels around the MMA instruction. Related `QMMA` or `OMMA` family forms can be fed by shared-memory matrix loads, by direct register setup, by scale-vector operands, or by sparse metadata operands.

[INF] The safe audit question is "which channel does this operand belong to?" before asking what exact lane value it represents. Chapter 23 shows the channel structure clearly, while the exact lane-to-value and bit-packing decode remains open.

[INF] Seeing this pattern identifies narrow-fragment organization around tensor-core compute. It does not by itself prove the numeric interpretation of every packed nibble, the full lane mapping, or correctness of the MMA result.

**SASS signature**

[OBS] Chapter 23 observes dense FP4/FP6 `QMMA.16832` variants for `E2M1.E2M1`, `E3M2.E3M2`, `E2M3.E2M3`, `E3M2.E2M3`, and `E2M3.E3M2`.

[OBS] Chapter 23 observes that variants 23a through 23e share visible instruction word `0x0000000e0408727a`, while dtype selection is visible in the mnemonic and encoded outside the low opcode byte.

[OBS] Chapter 15 observes mixed-FP6 directions `QMMA.16832.F32.E3M2.E2M3` and `QMMA.16832.F32.E2M3.E3M2`, both remaining in the QMMA low-byte family.

[OBS] Chapter 23 observes an LDSM-fed path where `LDSM.16.M88.2 R24, [R0+0x800]` and `LDSM.16.M88.4 R4, [R0]` precede `QMMA.16832.F32.E2M1.E2M1 R8, R4, R24, RZ`.

[OBS] Chapter 23 observes a direct-register path where `QMMA.16832.F32.E2M1.E2M1` appears without preceding LDSM in the probe.

[OBS] Chapter 23 observes a block-scaled dense channel with `QMMA.SF.16832.F32.E4M3.E4M3.E8 R8, R4, R14, RZ, R17, R0, URZ`.

[OBS] Chapter 23 observes an OMMA scale-vector channel with `OMMA.SF.16864.F32.E2M1.E2M1.UE4M3.4X R8, R4, R14, RZ, R19, R19, URZ`.

[OBS] Chapter 23 observes a sparse block-scaled channel with `QMMA.SF.SP.16864.F32.E3M2.E2M1.E8 R12, R4, R8, RZ, R20, R19, URZ, 0x0`.

[OBS] Chapter 24 records that scale and metadata operands survive as visible SASS dependencies in controlled mini-GEMM probes.

[INF] Production audits should therefore trace those channels separately.

**Variants**

[OBS] Dense narrow fragments appear as `QMMA.16832.F32.<A_dtype>.<B_dtype>` without `.SF` or `.SP` in the observed FP4/FP6 dense probes.

[OBS] LDSM-fed narrow fragments have visible shared-memory matrix loads before the QMMA instruction; direct-register probes do not show that LDSM setup.

[OBS] The observed block-scaled dense forms add `.SF` and visible scale operands after the D, A, B, and C operands.

[OBS] The observed sparse block-scaled forms add both `.SF.SP` in the mnemonic and visible metadata or selector operands beyond the scale/data operands.

[OBS] Chapter 23 K-boundary, register-pair-boundary, special-value, and shared-alignment probes compile.

[GAP] Those probes do not fully decode the value mapping.

**Interpretation**

[INF] The practical audit rule is to trace operand channels separately: fragment data registers, scale registers, metadata registers, and accumulator registers should not be collapsed into one generic "MMA operand" bucket.

[INF] LDSM presence indicates a shared-memory fragment supply path; its absence in the direct-register probe means the fragment registers were prepared without visible matrix shared-memory loads in that SASS region.

[INF] Dtype suffixes identify the declared narrow input formats, but not the exact bit position of every lane value inside each source register.

[INF] `.SF` and `.SP` are strong local signs that the instruction consumes extra channels beyond dense fragment data.

[INF] In production dumps, this pattern helps prevent mislabeling scale vectors or sparse metadata as ordinary A/B data fragments.

**Anti-patterns**

[INF] Do not infer full FP4/FP6 lane-to-value layout from the QMMA mnemonic alone.

[INF] Do not treat LDSM-fed and direct-register QMMA paths as equivalent when auditing data provenance; they prove different fragment supply paths.

[INF] Do not collapse scale operands into A/B fragment data. Chapter 23 and Chapter 24 keep scale channels visible in the SASS operand list.

[INF] Do not collapse sparse metadata into fragment data. Sparse forms expose metadata as a separate channel that must be traced independently.

[INF] Do not treat successful compilation or runtime smoke execution as numeric correctness for packed FP4/FP6 layout.

**Open gaps**

[GAP] Full FP4/FP6 lane-to-value mapping remains undecoded from the current runtime outputs.

[GAP] E3M2 and E2M3 exact bit packing within each 32-bit source register remains unresolved.

[GAP] Special-value interpretation for zero, NaN-like, Inf-like, and sign-bit patterns remains unresolved.

[GAP] LDSM-fed QMMA correctness for packed FP4/FP6 values remains structural until outputs are checked against a numeric reference.

[GAP] Cross-architecture stability of the narrow-fragment channel layout is not established.

---

Source: `knowledge/FINDINGS.md`.
