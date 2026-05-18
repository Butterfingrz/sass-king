# Production Audits

This directory is the Phase 4 entry point for applying the Phase 3 pattern library to real production kernels.

No production audit is accepted here until it preserves the same evidence discipline as the controlled corpus:

- identify the binary, library, version, GPU target, CUDA toolkit, driver, and dump command;
- segment the SASS into regions before assigning meanings;
- cite matching `PATTERN-NN` pages from `patterns/`;
- carry over each pattern's confidence level, anti-patterns, and open gaps;
- record unexplained regions explicitly instead of forcing them into a known pattern;
- keep optimization conclusions separate from direct SASS observations.

The first deliverable for this phase should be a manual audit report for one representative real kernel. The audit tool comes later, after the manual report format is stable enough to automate.
