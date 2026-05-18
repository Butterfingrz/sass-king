# PATTERN-06: Async global-to-shared tile pipeline

**Category**: memory
**Evidence**: [OBS] Chapter 18 contains the controlled `cp.async` pipeline observations for SM120; Chapter 24 contains production-like mini-GEMM contexts where `LDGSTS`, `LDGDEPBAR`, `DEPBAR`, `BAR`, `LDSM`, MMA, and `STG` appear together.
**Confidence**: [INF] C2 for Chapter 18 controlled 2-stage, looped, and 3-stage async-copy variants; C3 for Chapter 24 runtime smoke execution of production-like mini-GEMM probes; C1 for static recognition in future production dumps unless source, runtime, correctness, or profile evidence is added.

**Plain English**

[INF] An async global-to-shared tile pipeline is the SASS shape behind prefetching the next GEMM or attention tile from global memory into shared memory while tensor cores work on an earlier tile.

[INF] In source-level terms, this is the `cp.async` mainloop idea: copy tile data into shared memory, commit that copy group, wait only when the tile is needed, synchronize the CTA, then use LDSM and MMA to consume the staged tile.

[INF] The goal of the pattern is latency hiding. The copy path and the compute path are deliberately overlapped so global-memory latency is less visible to the tensor-core work.

[INF] Seeing this pattern in a dump is evidence for an async shared-memory staging pipeline. It does not by itself prove the kernel is bandwidth optimal, numerically correct, double-buffered in the source, or equivalent to a production library implementation.

**SASS signature**

[OBS] Chapter 18 observes PTX `cp.async.ca.shared.global.L2::128B` lowering to `LDGSTS.E.LTC128B.128 [R_smem], desc[UR_desc][R_gmem.64]`.

[OBS] Chapter 18 observes `LDGSTS.E.LTC128B.128` with low opcode byte `0xae`, distinct from the observed LDG, LDS, and STS opcode families.

[INF] In the observed mnemonic, `LDGSTS` is the global-to-shared async transfer, `.E` is the extended-addressing form, `.LTC128B` is the observed L2 cache hint, and `.128` is the observed 128-bit per-thread transfer width.

[OBS] Chapter 18 observes PTX `cp.async.commit_group` lowering to operandless `LDGDEPBAR`.

[OBS] Chapter 18 observes `LDGDEPBAR` opcode bytes `0x00000000000079af`.

[OBS] Chapter 18 observes PTX `cp.async.wait_group N` lowering to `DEPBAR.LE SB0, N`.

[OBS] Chapter 18 observes `DEPBAR.LE SB0, 0x0`, `DEPBAR.LE SB0, 0x1`, and `DEPBAR.LE SB0, 0x2` in the tested 2-stage and 3-stage variants.

[RES] Chapter 18 decodes the tested `DEPBAR.LE SB0, N` argument in control-code bits 38-39: `00` for N=0, `01` for N=1, and `10` for N=2.

[INF] The unused `11` value predicts N=3 for the same 2-bit wait-group field, but Chapter 18 does not compile an N=3 case.

[OBS] Chapter 18 observes all tested async-copy waits targeting `SB0`.

[INF] For the tested SM120 `cp.async` forms, SB0 is the observed async-copy dependency bank. Other scoreboard banks are not established by this corpus.

[OBS] Chapter 18 observes the recurring pre-LDGSTS triplet `@!PT LDS RZ, [RZ]` before LDGSTS groups.

[GAP] The purpose of the `@!PT LDS RZ, [RZ]` triplet is not decoded; Chapter 18 treats it as a scheduling, resource, or alignment marker candidate rather than a functional shared-memory load.

[OBS] Chapter 24 observes production-like single-stage async structure in 24e: `LDGSTS.E.LTC128B.128`, `LDGDEPBAR`, `DEPBAR.LE SB0, 0x0`, `LDSM.16.M88.2`, `LDSM.16.M88.4`, then `HMMA.16816.F32`.

[OBS] Chapter 24 observes production-like double-buffer async structure in 24f and 24t: two `LDGSTS` groups with `LDGDEPBAR` commits before `DEPBAR.LE SB0, 0x0`, followed by LDSM and HMMA.

[OBS] Chapter 24 observes wait-depth variants in 24x with `DEPBAR.LE SB0, 0x2`, `DEPBAR.LE SB0, 0x1`, and `DEPBAR.LE SB0, 0x0`.

**Variants**

[OBS] Chapter 18 observes an unrolled 2-stage pipeline in 18a.

[OBS] Chapter 18 observes a looped 2-stage K-loop in 18b where the SASS contains a real counter update, predicate compare, and `BRA` back-edge.

[OBS] Chapter 18 observes a 3-stage pipeline in 18c specifically to isolate `DEPBAR.LE` wait-group values N=2, N=1, and N=0.

[OBS] Chapter 24 observes single-stage, double-buffer, full-audit, and barrier/wait mini-GEMM variants that reuse the Chapter 18 async-copy instruction family.

[OBS] Chapter 18 observes ptxas moving the LDSM for tile 1 before the HMMA for tile 0 in the fully unrolled 18a schedule.

[INF] That schedule is software-pipelined at the SASS level: later tile fragment loading can overlap with current tile tensor-core compute when dependencies allow.

**Interpretation**

[INF] The minimal async-copy staging signature is `LDGSTS` for the transfer, `LDGDEPBAR` for the commit group, and `DEPBAR.LE SB0, N` for the wait group.

[INF] A full tensor-core tile pipeline then adds the already documented LDSM fragment-load pattern and an MMA compute pattern after the wait and CTA synchronization.

[INF] `DEPBAR.LE SB0, 0x1` in a 2-stage mainloop means the code waits until the older tile is ready while allowing one newer copy group to remain in flight.

[INF] `DEPBAR.LE SB0, 0x0` is the observed wait-all form and commonly appears before consuming the final staged tile or in single-stage probes.

[INF] The observed `LDGSTS`/`LDGDEPBAR`/`DEPBAR` sequence identifies async staging, but performance conclusions require profiling or timing evidence because global-memory latency depends on cache state, bandwidth, occupancy, and overlap.

[INF] The presence of a loop back-edge around one prefetch plus one consume step identifies a mainloop structure, while an unrolled sequence may represent the same staging strategy without an explicit SASS loop.

[INF] HMMA wait masks can differ between direct LDSM-fed cases and async-pipeline cases, so an audit should not assume the direct `0xff` wait mask from `PATTERN-05` applies inside an LDGSTS/DEPBAR pipeline.

**Anti-patterns**

[INF] Plain `LDG` followed later by `STS` is not this async-copy pattern unless the dump also shows the `LDGSTS`/`LDGDEPBAR`/`DEPBAR.LE` family.

[INF] LDSM plus MMA without preceding `LDGSTS` and async dependency waits is the fragment-loading pattern, not the async global-to-shared pipeline.

[INF] A single `LDGSTS` proves an async global-to-shared transfer instruction is present, but it does not prove a correctly staged GEMM mainloop without commit, wait, synchronization, and consumer context.

[INF] `DEPBAR.LE SB0, N` should not be interpreted as a generic CTA barrier. CTA synchronization remains visible separately through `BAR.SYNC` forms.

[INF] The pre-LDGSTS `@!PT LDS RZ, [RZ]` triplet should not be treated as functional shared-memory traffic until its scheduling role is decoded.

**Open gaps**

[GAP] The `@!PT LDS RZ, [RZ]` triplet before LDGSTS remains unresolved.

[GAP] Only the observed `cp.async.ca.shared.global.L2::128B` path is covered; other PTX cache hints or transfer shapes may lower differently.

[GAP] The predicted N=3 `DEPBAR.LE SB0, 0x3` form is not compiled in the current corpus.

[GAP] LDGSTS latency and throughput are not microbenchmarked; the corpus does not provide a cycles-per-LDGSTS model.

[GAP] Scoreboard banks other than SB0 are not observed for the tested SM120 async-copy path.

[GAP] Full production-library comparison against CUTLASS, FlashAttention, or vendor kernels remains future Phase 4 work.

---

Source: `knowledge/FINDINGS.md`.
