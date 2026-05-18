# PATTERN-17: Scalar shared-memory staging and CTA barrier

**Category**: memory
**Evidence**: [OBS] Kernel 06 contains the controlled SM120 scalar shared-memory round-trip and multi-buffer variants for this pattern; [OBS] Kernel 07 contains supporting `UMOV 0x400` presence/absence context for shared-memory use.
**Confidence**: [INF] C2 for Kernel 06 controlled shared-size, block-size, modulo, and multi-buffer variants; C1 for static recognition in future production dumps unless source mapping, runtime validation, bank-conflict profiling, or cross-architecture comparison is added.

**Plain English**

[INF] Scalar shared-memory staging is the ordinary `__shared__` path: a kernel writes per-thread values into shared memory with `STS`, synchronizes the CTA, then reads values back with `LDS`.

[INF] This is different from tensor-core matrix shared-memory instructions. `LDS` and `STS` move ordinary scalar or vector data through shared memory, while `LDSM` and `STSM` move matrix fragments for tensor-core layouts.

[INF] In practical audits, this pattern identifies a block-level shared scratchpad, not just a global-memory cache effect. The key visual shape is shared-base setup, per-thread address formation, `STS`, usually a CTA barrier, then `LDS`.

[INF] Seeing this pattern does not by itself prove bank-conflict behavior, cross-thread correctness, shared-memory occupancy, or whether the barrier placement is optimal.

**SASS signature**

[OBS] Kernel 06 observes shared-base construction with `S2UR UR5, SR_CgaCtaId`, `UMOV UR4, 0x400`, and `ULEA UR4, UR5, UR4, 0x18`.

[OBS] Kernel 06 decodes the printed ULEA expression as `shared_base = (CgaCtaId << 24) + 0x400`.

[OBS] Kernel 06 observes per-thread shared address formation with `LEA R7, R0, UR4, 0x2` for `&smem[tid] = UR4 + tid * 4`.

[OBS] Kernel 06 observes scalar shared writeback as `STS [R7], R2`.

[OBS] Kernel 06 observes `BAR.SYNC.DEFER_BLOCKING 0x0` for the tested `__syncthreads()` point.

[OBS] Kernel 06 observes scalar shared reload as `LDS R7, [R0]` before global storeback.

[OBS] Kernel 06 observes shared-memory accesses using direct `[R]` or mixed `[R+UR]` addressing, unlike global memory's `desc[UR][R.64]` addressing.

[OBS] Kernel 06g observes two shared buffers using one shared-base construction plus `UIADD3 UR5, UPT, UPT, UR4, 0x400, URZ` to derive the second buffer base.

**Variants**

[OBS] Variants 06b, 06e, and 06f keep `UMOV UR4, 0x400` across tested shared sizes 1024 B, 512 B, and 2048 B, and across tested block sizes 256, 128, and 512.

[RES] Kernel 06 rejects the tested hypotheses that `0x400` directly encodes shared size, block size, or launch configuration.

[OBS] Kernel 06g observes `LDS R8, [R6+UR4]` and `LDS R11, [R6+UR5]`. [INF] This uses one per-thread offset with two uniform shared-buffer bases.

[OBS] Kernel 06g observes consecutive shared stores followed by one `BAR.SYNC`. [INF] In that tested source, the single barrier covers the preceding adjacent stores.

[OBS] Kernel 07 observes `0x400` appearing conditionally with the presence of `__shared__` memory in the tested kernels.

**Interpretation**

[INF] `UMOV 0x400` plus `ULEA ..., 0x18` is a strong first-pass locator for scalar shared-memory base setup in the tested SM120 corpus.

[INF] `STS` and `LDS` should be classified as scalar/vector shared-memory traffic, not global memory and not tensor-core matrix memory.

[INF] `BAR.SYNC.DEFER_BLOCKING 0x0` is CTA-level synchronization for the tested `__syncthreads()` path, not warp reconvergence and not a tensor-core dependency wait.

[INF] Multiple shared buffers can be audited by tracing one uniform base plus derived uniform offsets; do not assume every buffer requires a separate full setup sequence.

[INF] The decoded address expression is a SASS-level addressing formula. The architectural meaning of the `0x400` base and `CgaCtaId << 24` mapping remains unresolved.

**Anti-patterns**

[INF] Do not confuse `LDS`/`STS` with `LDSM`/`STSM`; the latter are matrix-fragment instructions with different operand layouts and audit implications.

[INF] Do not treat `[R]` or `[R+UR]` shared-memory addressing as global descriptor addressing.

[INF] Do not infer bank-conflict freedom from the presence of `LDS`, `STS`, or `BAR.SYNC`; profiling or layout analysis is still required.

[INF] Do not generalize the exact `0x400` and shift-24 interpretation to physical shared-memory layout without architectural validation.

[INF] Do not read `BAR.SYNC` as BSSY/BSYNC-style reconvergence; they solve different synchronization problems.

**Open gaps**

[GAP] The exact architectural meaning of `UMOV UR4, 0x400` plus `ULEA` shift 24 remains unresolved beyond the decoded SASS expression.

[GAP] The behavior of `SR_CgaCtaId` in non-cluster kernels remains partially hypothetical.

[GAP] The `.DEFER_BLOCKING` modifier on `BAR.SYNC` is not fully decoded beyond being the observed default form for tested SM120 `__syncthreads()`.

[GAP] Shared-memory bank conflicts, latency, and throughput are not measured by Kernel 06.

[GAP] Cross-architecture stability of the scalar shared-memory setup sequence is not established.

---

Source: `knowledge/FINDINGS.md`.
