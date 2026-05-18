# Release Notes

## v0.2.0-pattern-library

This release marks the completion of the initial Phase 3 pattern library for SASS King. The project now has 29 reusable SM120 / SM120a SASS pattern pages, an updated evidence trail, and a clearer path into Phase 4 production-kernel audits.

It does not ship a standalone disassembler, assembler, Ghidra plugin, or audit CLI.

## Highlights

- Added the formal Phase 3 pattern library under `patterns/`.
- Added 29 reusable audit signatures for tensor-core compute, matrix memory, control flow, register/dataflow behavior, arithmetic lowering, scheduling, and warp collectives.
- Expanded `knowledge/FINDINGS.md` with the Phase 3 pattern evidence and closeout.
- Updated denvdis integration notes after the SM120 cross-validation pass.
- Added documentation indexes for `docs/` and `knowledge/encoding/`.
- Added `production/README.md` as the Phase 4 production-audit entry point.
- Updated the root README to explain the current repository structure and Phase 4 next step.

## Scope

Included in the v0.2 boundary:

- controlled CUDA kernel corpus through chapters 01-25;
- reorganized `corpus/` layout with section indexes;
- project-wide `knowledge/FINDINGS.md` with navigation index and claim discipline;
- initial Phase 3 pattern library with 29 reusable audit signatures under `patterns/`;
- SM120 / SM120a instruction glossary;
- pilot encoding pages for `LDSM`, `STSM`, `QMMA`, and partial control-code modeling;
- denvdis integration notes and representative cross-validation results;
- contribution rules for evidence tagging and dump metadata;
- Phase 4 audit entry point under `production/`.

Not included yet:

- standalone SASS disassembler;
- standalone SASS assembler;
- Ghidra plugin;
- one-command audit CLI;
- completed cross-architecture replay;
- production-library audit suite.

## Reproducible today

The executable part of v0.2 remains the controlled CUDA corpus. A reader can compile a kernel, dump NVIDIA SASS with `cuobjdump`, and compare the result with the chapter conclusion.

Example:

```bash
cd corpus/basics/01_vector_add
nvcc -arch=sm_120 kernel1.cu -o vector_add
cuobjdump --dump-sass vector_add > sm_120.sass
```

## Main artifacts

| Artifact | Purpose |
|---|---|
| `README.md` | Project overview, roadmap, and related work. |
| `docs/README.md` | Documentation index. |
| `docs/START_HERE.md` | Minimal onboarding path. |
| `docs/PROJECT_STRUCTURE.md` | Repository navigation model and content ownership rules. |
| `corpus/README.md` | Corpus section map and reproduction model. |
| `knowledge/FINDINGS.md` | Primary source of truth for observations, hypotheses, resolutions, and gaps. |
| `knowledge/SASS_INSTRUCTIONS_SM120.md` | Evidence-backed instruction-family inventory. |
| `knowledge/encoding/README.md` | Index for reusable instruction-family notes. |
| `knowledge/DENVDIS_INTEGRATION.md` | denvdis validation status and policy. |
| `patterns/README.md` | Phase 3 pattern index for audit-facing SASS signatures. |
| `production/README.md` | Phase 4 entry point for upcoming manual production audits. |
