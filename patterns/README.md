# SASS Pattern Library

This directory contains the formal Phase 3 pattern library extracted from `knowledge/FINDINGS.md`. Each page is a reusable audit signature for recognizing a SASS structure in a new dump while preserving the evidence limits from the research log.

`knowledge/FINDINGS.md` remains the authoritative research log. The files here are the practical per-pattern entry points for production audits.

## How to Use

1. Identify a candidate instruction sequence or region in a SASS dump.
2. Open the closest pattern page below.
3. Match the `SASS signature` and `Variants` sections before citing the pattern.
4. Carry over the `Confidence`, `Anti-patterns`, and `Open gaps` sections into any audit conclusion.

## Claim Tags

- `[OBS]`: verified observation from a SASS dump, log, runtime output, or profile
- `[INF]`: inference from one or more observations; the evidence chain must be stated
- `[HYP]`: open hypothesis, to be tested
- `[RES]`: resolved hypothesis, rejected or confirmed
- `[GAP]`: open question not answered by the current evidence

## Confidence Levels

- `C0`: section identified
- `C1`: SASS pattern observed
- `C2`: controlled variant confirmed
- `C3`: runtime smoke validated
- `C4`: numeric correctness validated
- `C5`: profile/performance validated
- `C6`: cross-context confirmed

## Index

- [PATTERN-01: Warp reduction](01-warp-reduction.md) - `collective`
- [PATTERN-02: HMMA accumulator chain](02-hmma-accumulator-chain.md) - `tensor_core`
- [PATTERN-03: QMMA accumulator chain](03-qmma-accumulator-chain.md) - `tensor_core`
- [PATTERN-04: OMMA block-scaled FP4 accumulator chain](04-omma-block-scaled-fp4-accumulator-chain.md) - `tensor_core`
- [PATTERN-05: LDSM fragment loading](05-ldsm-fragment-loading.md) - `memory`
- [PATTERN-06: Async global-to-shared tile pipeline](06-async-global-to-shared-tile-pipeline.md) - `memory`
- [PATTERN-07: STSM matrix-store epilogue](07-stsm-matrix-store-epilogue.md) - `memory`
- [PATTERN-08: Divergence and reconvergence control](08-divergence-and-reconvergence-control.md) - `control_flow`
- [PATTERN-09: Register spill and local-memory frame](09-register-spill-and-local-memory-frame.md) - `memory`
- [PATTERN-10: Sparse MMA metadata operand channel](10-sparse-mma-metadata-operand-channel.md) - `tensor_core`
- [PATTERN-11: Vectorized global memory access](11-vectorized-global-memory-access.md) - `memory`
- [PATTERN-12: Slowpath arithmetic and MUFU helper lowering](12-slowpath-arithmetic-and-mufu-helper-lowering.md) - `arithmetic`
- [PATTERN-13: Production mini-GEMM audit segmentation](13-production-mini-gemm-audit-segmentation.md) - `audit_method`
- [PATTERN-14: Loop back-edge and unroll classification](14-loop-back-edge-and-unroll-classification.md) - `control_flow`
- [PATTERN-15: Template-specialized SASS function separation](15-template-specialized-sass-function-separation.md) - `audit_method`
- [PATTERN-16: Predicated bounds, early-exit, and tail-store guard](16-predicated-bounds-early-exit-and-tail-store-guard.md) - `control_flow`
- [PATTERN-17: Scalar shared-memory staging and CTA barrier](17-scalar-shared-memory-staging-and-cta-barrier.md) - `memory`
- [PATTERN-18: Scoreboard grouping and variable-latency waits](18-scoreboard-grouping-and-variable-latency-waits.md) - `scheduling`
- [PATTERN-19: Uniform-register dataflow and cross-file transfers](19-uniform-register-dataflow-and-cross-file-transfers.md) - `register_dataflow`
- [PATTERN-20: Kernel argument and global descriptor addressing spine](20-kernel-argument-and-global-descriptor-addressing-spine.md) - `memory_addressing`
- [PATTERN-21: Narrow-fragment operand channels](21-narrow-fragment-operand-channels.md) - `tensor_core`
- [PATTERN-22: FFMA fusion and arithmetic chain lowering](22-ffma-fusion-and-arithmetic-chain-lowering.md) - `arithmetic`
- [PATTERN-23: Register lifetime recycling and semantic role changes](23-register-lifetime-recycling-and-semantic-role-changes.md) - `register_allocation`
- [PATTERN-24: FP32 constant materialization and immediate-slot pressure](24-fp32-constant-materialization-and-immediate-slot-pressure.md) - `arithmetic`
- [PATTERN-25: Warp shuffle, vote, match, and sync primitives](25-warp-shuffle-vote-match-and-sync-primitives.md) - `collective`
- [PATTERN-26: Local CALL and helper ABI shape](26-local-call-and-helper-abi-shape.md) - `call_abi`
- [PATTERN-27: Read-only global constant-cache load path](27-read-only-global-constant-cache-load-path.md) - `memory`
- [PATTERN-28: Global reduction epilogue with REDG](28-global-reduction-epilogue-with-redg.md) - `epilogue`
- [PATTERN-29: Cold error and trap path](29-cold-error-and-trap-path.md) - `control_flow`

## By Category

### arithmetic

- [PATTERN-12: Slowpath arithmetic and MUFU helper lowering](12-slowpath-arithmetic-and-mufu-helper-lowering.md)
- [PATTERN-22: FFMA fusion and arithmetic chain lowering](22-ffma-fusion-and-arithmetic-chain-lowering.md)
- [PATTERN-24: FP32 constant materialization and immediate-slot pressure](24-fp32-constant-materialization-and-immediate-slot-pressure.md)

### audit_method

- [PATTERN-13: Production mini-GEMM audit segmentation](13-production-mini-gemm-audit-segmentation.md)
- [PATTERN-15: Template-specialized SASS function separation](15-template-specialized-sass-function-separation.md)

### call_abi

- [PATTERN-26: Local CALL and helper ABI shape](26-local-call-and-helper-abi-shape.md)

### collective

- [PATTERN-01: Warp reduction](01-warp-reduction.md)
- [PATTERN-25: Warp shuffle, vote, match, and sync primitives](25-warp-shuffle-vote-match-and-sync-primitives.md)

### control_flow

- [PATTERN-08: Divergence and reconvergence control](08-divergence-and-reconvergence-control.md)
- [PATTERN-14: Loop back-edge and unroll classification](14-loop-back-edge-and-unroll-classification.md)
- [PATTERN-16: Predicated bounds, early-exit, and tail-store guard](16-predicated-bounds-early-exit-and-tail-store-guard.md)
- [PATTERN-29: Cold error and trap path](29-cold-error-and-trap-path.md)

### epilogue

- [PATTERN-28: Global reduction epilogue with REDG](28-global-reduction-epilogue-with-redg.md)

### memory

- [PATTERN-05: LDSM fragment loading](05-ldsm-fragment-loading.md)
- [PATTERN-06: Async global-to-shared tile pipeline](06-async-global-to-shared-tile-pipeline.md)
- [PATTERN-07: STSM matrix-store epilogue](07-stsm-matrix-store-epilogue.md)
- [PATTERN-09: Register spill and local-memory frame](09-register-spill-and-local-memory-frame.md)
- [PATTERN-11: Vectorized global memory access](11-vectorized-global-memory-access.md)
- [PATTERN-17: Scalar shared-memory staging and CTA barrier](17-scalar-shared-memory-staging-and-cta-barrier.md)
- [PATTERN-27: Read-only global constant-cache load path](27-read-only-global-constant-cache-load-path.md)

### memory_addressing

- [PATTERN-20: Kernel argument and global descriptor addressing spine](20-kernel-argument-and-global-descriptor-addressing-spine.md)

### register_allocation

- [PATTERN-23: Register lifetime recycling and semantic role changes](23-register-lifetime-recycling-and-semantic-role-changes.md)

### register_dataflow

- [PATTERN-19: Uniform-register dataflow and cross-file transfers](19-uniform-register-dataflow-and-cross-file-transfers.md)

### scheduling

- [PATTERN-18: Scoreboard grouping and variable-latency waits](18-scoreboard-grouping-and-variable-latency-waits.md)

### tensor_core

- [PATTERN-02: HMMA accumulator chain](02-hmma-accumulator-chain.md)
- [PATTERN-03: QMMA accumulator chain](03-qmma-accumulator-chain.md)
- [PATTERN-04: OMMA block-scaled FP4 accumulator chain](04-omma-block-scaled-fp4-accumulator-chain.md)
- [PATTERN-10: Sparse MMA metadata operand channel](10-sparse-mma-metadata-operand-channel.md)
- [PATTERN-21: Narrow-fragment operand channels](21-narrow-fragment-operand-channels.md)

## Maintenance

When changing a pattern, update the source evidence in `knowledge/FINDINGS.md` first, then mirror the same bounded claim into the matching page here. Do not promote `[INF]` or `[HYP]` text to `[OBS]` during formatting.
