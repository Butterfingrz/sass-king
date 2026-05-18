# PATTERN-01: Warp reduction

**Category**: collective
**Evidence**: [OBS] Chapters 09, 10, and 21 contain the supporting observations for this pattern.
**Confidence**: [INF] C2 for the controlled Chapter 09 and 10 reduction kernels; C1 for static recognition in future production dumps unless a controlled source variant or runtime validation is added.

**Plain English**

[INF] A warp reduction means that the 32 lanes of one warp each start with their own value, then cooperate to produce one result for the whole warp. For example, lane 0 may hold `3`, lane 1 may hold `7`, lane 2 may hold `2`, and the reduction computes one warp-level sum, maximum, minimum, bitwise result, or similar combined value.

[INF] In source-level terms, this is the shape behind operations such as "sum these 32 lane values," "take the maximum across the warp," or "combine one integer flag from each lane." The important idea is that the result belongs to the warp, not to independent per-thread scalar work.

[INF] In SASS, the manual form does the reduction step by step: `SHFL.BFLY` moves values between lanes, then arithmetic instructions combine the exchanged values. The hardware form does the same high-level job with one `REDUX` instruction that writes the combined result into a uniform register.

[INF] The output still has to be used carefully. A `REDUX` result lives in the uniform register file, so per-thread consumers usually need a `MOV R, UR` before a normal per-thread store or arithmetic instruction can use the value.

[INF] When this pattern appears in a production dump, the safe conclusion is that a warp-level reduction structure is present. It does not by itself prove the original source expression, the numeric correctness of the reduction, or the runtime cost without additional evidence.

**SASS signature**

[OBS] The butterfly FP32 reduction form is five `SHFL.BFLY` stages paired with five arithmetic combine instructions over masks `0x10`, `0x08`, `0x04`, `0x02`, and `0x01`; Chapter 09 observes the `SHFL.BFLY` plus `FADD` sequence for the 32-lane warp reduction.

[OBS] The hardware integer reduction form is a single `REDUX` instruction that takes a per-thread `R` input and writes a uniform `UR` output; Chapter 10 observes `REDUX.SUM.S32 UR7, R2` and the other tested `REDUX` variants in the same kernel skeleton.

[OBS] The complete observed REDUX consumer path includes a cross-file transfer when the reduced value is used by per-thread code: Chapter 10 observes `MOV R9, UR7` after `REDUX.SUM.S32 UR7, R2`, before the selected lane stores the value.

[INF] The `MOV R9, UR7` is the REDUX scoreboard consumer in the Chapter 10 scheduling model, because it is the first per-thread use of the uniform REDUX result and its control code shows the wait pattern documented there. [GAP] Full bit-level validation of the REDUX scheduling fields remains pending.

[OBS] A per-warp output path commonly includes the lane-zero predicate idiom `LOP3.LUT P, RZ, R_tid, 0x1f, RZ, 0xc0, !PT`, a predicated exit for non-lane-zero threads, warp-id derivation with `SHF.R.U32.HI`, and one `STG` for the selected lane's result.

[OBS] When a warp-synchronous reduction follows divergent input loading, Chapters 09 and 10 observe identity initialization before the bounds check plus `BSSY.RECONVERGENT` and `BSYNC.RECONVERGENT` before the warp-level reduction operation.

**Variants**

[OBS] Chapter 09 observes the manual FP32 butterfly form: `SHFL.BFLY` plus `FADD` for a warp-wide sum.

[OBS] Chapter 10 observes hardware integer reductions for signed sum, signed min, signed max, unsigned min, unsigned max, bitwise and, bitwise or, and bitwise xor through `REDUX.SUM.S32`, `REDUX.MIN.S32`, `REDUX.MAX.S32`, `REDUX.MIN`, `REDUX.MAX`, plain `REDUX`, `REDUX.OR`, and `REDUX.XOR`.

[OBS] Chapter 10 observes that all tested `REDUX` variants share opcode bytes `0x00000000020773c4`; the operation and signedness are selected by control-code bits rather than by distinct opcode bytes.

[OBS] Chapter 10 observes identity-load variants before the reduction: `HFMA2` for identities representable as packed half constants and `MOV` for identities such as `0x7fffffff` or `0xffffffff`.

[OBS] Chapter 09 observes that `__syncwarp()` itself does not lower to a dedicated `WARPSYNC` instruction in the tested SM120 forms; ptxas emits NOP padding or eliminates redundant calls. Chapter 21 later observes `@P0 WARPSYNC.ALL` before a predicated HMMA, which is supporting warp-level control evidence but not a warp-reduction signature.

[OBS] Chapter 21 observes related warp-level control forms, including `VOTE.ANY R5, PT, P0`, `VOTE.ANY P0, P0`, `SHFL.IDX`, and `@P0 WARPSYNC.ALL`; these are supporting collective-control evidence but are not by themselves the warp-reduction signature.

**Interpretation**

[INF] A five-stage `SHFL.BFLY` plus arithmetic cascade identifies a manual warp reduction because each stage exchanges values across a progressively smaller XOR lane mask and immediately combines the exchanged value into the running accumulator.

[INF] A `REDUX` instruction with `R` input, `UR` output, and a downstream `MOV R, UR` identifies a hardware warp reduction because Chapter 10 controlled variants replace the Chapter 09 butterfly reduction with one uniform-output reduction instruction.

[INF] The lane-zero store path identifies a per-warp result emission strategy, not the reduction operation itself, because the selected lane writes one result per warp after the warp-level value has been produced.

[INF] Identity initialization before a divergent bounds check is part of making all lanes contribute valid values to a warp-synchronous operation. It is supporting context for reductions with guarded loads, not enough to identify the reduction without the SHFL/FADD cascade or REDUX instruction.

**Anti-patterns**

[INF] A single `SHFL.IDX`, `SHFL.UP`, or `SHFL.DOWN` without a repeated combine chain is a warp value-exchange or broadcast pattern, not enough evidence for a warp reduction by itself.

[INF] A `VOTE.ANY`, `VOTE.ALL`, or `MATCH` instruction by itself is a warp collective, not a value reduction, unless the surrounding SASS shows that its output participates in a reduction-like decision or store path.

[INF] A `WARPSYNC.ALL` instruction by itself is a synchronization or guarded warp-level execution marker, not a reduction signature.

**Open gaps**

[GAP] REDUX cycle latency is not measured; Chapter 10 records the instruction-count and stage-count reduction but leaves cycle-accurate latency to a microbenchmark.

[GAP] REDUX unsigned sum is not tested; Chapter 10 leaves the unsigned `__reduce_add_sync` form as a future variant.

[GAP] Full scheduling-bit decoding for REDUX, SHFL, VOTE, and related collective operations remains incomplete; `knowledge/encoding/CONTROL_CODE.md` contains the current partial control-code model.

[GAP] Non-full warp masks and genuine divergence before warp-level collectives remain under-tested; Chapter 09 shows partial `__syncwarp` masks do not change the subsequent SHFL segment encoding, and Chapter 21 leaves non-full masks after real divergence open.

[GAP] Cross-architecture stability is not established for this pattern beyond the checked-in SM120 and SM89 artifacts; Phase 6 replay must test whether the same signatures and suffix conventions hold on SM80, SM86, SM90a, and SM100a.

---

Source: `knowledge/FINDINGS.md`.
