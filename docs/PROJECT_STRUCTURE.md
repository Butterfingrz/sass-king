# Project Structure

SASS King is organized as a research evidence pipeline, not as a traditional application repository.

The shortest mental model is:

```text
corpus/      produces evidence
knowledge/   consolidates evidence
patterns/    turns evidence into reusable audit signatures
production/  applies signatures to real kernels
```

## Top-Level Directories

| Path | Role | Put content here when... |
|---|---|---|
| `corpus/` | Controlled CUDA kernels, SASS dumps, and chapter writeups. | You are adding or reproducing an experiment. |
| `knowledge/` | Project-wide findings, instruction inventory, and encoding notes. | A fact is stable enough to be reused outside one chapter. |
| `patterns/` | Phase 3 pattern library. | A repeated SASS structure can be cited during audits. |
| `production/` | Phase 4 production-kernel audits. | A real library kernel is being segmented and interpreted. |
| `docs/` | Reader onboarding and project navigation. | The content explains how to use the repository rather than SASS itself. |
| `assets/` | Project images and static presentation assets. | The file supports README or documentation presentation. |
| `.github/` | GitHub issue, PR, and automation metadata. | The file changes GitHub workflow behavior. |
| `guide/` | External SASS reading guide submodule. | Do not place core project evidence here. |

## Content Flow

1. A controlled experiment starts in `corpus/`.
2. Chapter-local observations are written in the chapter `conclusion*.md`.
3. Reusable facts are promoted to `knowledge/FINDINGS.md`.
4. Stable instruction-family notes are summarized in `knowledge/SASS_INSTRUCTIONS_SM120.md` or `knowledge/encoding/`.
5. Repeated structures become named `PATTERN-NN` pages under `patterns/`.
6. Production audits cite those patterns from `production/`.

This flow is intentional. It keeps raw evidence, project-wide claims, reusable signatures, and applied audits separate.

## Reader Paths

| Reader | Best path |
|---|---|
| New reader | `README.md` -> `docs/START_HERE.md` -> this file |
| CUDA/SASS learner | `docs/START_HERE.md` -> `corpus/basics/` -> `corpus/warp_collectives/` |
| Tensor-core reader | `corpus/tensor_cores/README.md` -> chapters 13-25 -> `patterns/02-*` to `patterns/07-*` |
| Audit reader | `patterns/README.md` -> matching pattern page -> `knowledge/FINDINGS.md` |
| Contributor | `CONTRIBUTING.md` -> closest existing chapter or pattern |

## Documentation Indexes

Every major entry directory should have a `README.md`. If a directory is linked from the root README, it should explain its role before listing files.

Current entry indexes:

- `docs/README.md`
- `corpus/README.md`
- `knowledge/README.md`
- `knowledge/encoding/README.md`
- `patterns/README.md`
- `production/README.md`

## What Not To Mix

- Do not put raw chapter evidence directly in `patterns/`; patterns summarize reusable signatures.
- Do not put production audit conclusions in `knowledge/FINDINGS.md` until they become project-wide facts.
- Do not promote an inference from a pattern page to `[OBS]`; direct observations stay tied to dumps, logs, runtime output, or profiles.
- Do not use `docs/` as a dumping ground for research evidence; it is for navigation and process.

## Naming Rules

- Chapter directories keep their numeric prefix and descriptive name.
- Pattern pages use `NN-short-description.md` and keep the `PATTERN-NN` title inside the page.
- Production audits should use a library or kernel-oriented directory name once real audits are added.
- Project-wide documents use explicit names over abbreviations.

## Current Phase Boundary

Phase 3 is complete at the initial pattern-library level. Phase 4 begins in `production/` and should start with manual real-kernel audits before any audit CLI is built.
