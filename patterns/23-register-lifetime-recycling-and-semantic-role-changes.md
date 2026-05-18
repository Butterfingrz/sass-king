# PATTERN-23: Register lifetime recycling and semantic role changes

**Category**: register_allocation
**Evidence**: [OBS] Kernels 02, 05, 12, 13, and 17 contain controlled SM120 observations for dead-register recycling, context-dependent register allocation, register role changes, and zero-copy operand placement.
**Confidence**: [INF] C2 for the controlled source-to-SASS examples where a visible register changes role after its previous value is dead; C1 for static recognition in future production dumps unless source mapping or liveness tracing proves the previous role is no longer live.

**Plain English**

[INF] A SASS register name is a physical storage location in the compiled kernel, not a permanent source-level variable name. Once the old value is dead, ptxas can reuse that register for a completely different purpose.

[INF] This pattern explains why `R0` might mean `threadIdx.x` early in a function and later become a floating-point scratch register, or why a pointer register can later hold a store value or address for a different array.

[INF] The practical audit rule is to track live ranges, not semantic labels. A register's meaning is bounded by its producers and consumers; after the last consumer, the same register can safely be reassigned.

[INF] Seeing this pattern identifies register allocation and liveness behavior. It does not by itself prove the original source variable, absence of bugs, register pressure, or performance optimality.

**SASS signature**

[OBS] Kernel 02 observes `R0` first holding `threadIdx.x` from `S2R R0, SR_TID.X`, then later being reused as the destination of `FADD R0, R2, R5` after the index computation has consumed the thread ID.

[OBS] Kernel 02 observes the second `FADD` consuming that recycled `R0` and writing the final value to `R9`.

[OBS] Kernel 05 observes a chain where most FFMA instructions write the recurrence through `R0`, but the final FFMA writes the store value to `R3`.

[OBS] Kernel 05 records that `R3` was previously the high half of the source pointer pair `R2:R3` before the source `LDG`; after that load, the pointer value is no longer needed in the observed code.

[OBS] Chapter 13 observes register recycling across semantic boundaries in 13d: `R4` first holds `&b[tid*2]`, then after the `b` loads complete, ptxas overwrites `R4` to become `&d[tid*4]`.

[OBS] Kernel 12 records register-file reuse and loop restructuring in its anti-spill discussion. [INF] Those are compiler strategies used before local-memory spill becomes necessary in the observed pressure cases.

[OBS] Chapter 17 observes LDSM destination register bases also appearing as the consuming HMMA source register bases. [INF] This is a zero-copy operand placement pattern in that observed LDSM-to-HMMA handoff.

**Variants**

[OBS] Arithmetic scratch reuse: Kernel 02 reuses dead `R0` for an intermediate `FADD`.

[OBS] Chain-exit register change: Kernel 05 writes the last FFMA result to a different dead register before `STG`.

[OBS] Address role reuse: Chapter 13 reuses an address register for a later unrelated address.

[OBS] Operand-placement reuse: Chapter 17 places LDSM outputs directly where HMMA reads them.

[OBS] Spill-avoidance context: Kernel 12 records register-file reuse and loop restructuring in the pressure discussion. [INF] In that chapter, these are treated as alternatives ptxas uses before relying on local-memory spill.

**Interpretation**

[INF] When a register appears to change meaning, the first audit step is to find the last consumer of the old value and the producer of the new value.

[INF] Register role changes are expected in optimized SASS and are not evidence of source-level aliasing or incorrect code by themselves.

[INF] Register allocation is global enough that a small source change can affect register names in nearby or downstream code.

[INF] In tensor-core kernels, register placement can also be a dependency optimization: placing LDSM output where HMMA expects its source avoids intermediate moves.

[INF] A register name is reliable only within a live range. Audits should annotate values as `R4 as pointer`, `R4 as fragment`, or `R4 as store address` rather than assigning one meaning for the whole function.

**Anti-patterns**

[INF] Do not assume `R0` always means `threadIdx.x` just because it was produced by `S2R` earlier.

[INF] Do not track source variables by register name alone across the whole function.

[INF] Do not classify a register overwrite as a clobber unless the old value has a later visible consumer.

[INF] Do not infer increased register pressure from an extra temporary if the temporary reuses a dead register.

[INF] Do not treat zero-copy operand placement as proof of value layout correctness; it is register allocation evidence, not numeric validation.

**Open gaps**

[GAP] The exact ptxas allocation heuristic for choosing one dead register over another is not modeled.

[GAP] Cross-kernel register name stability is not guaranteed and is not established beyond controlled examples.

[GAP] The interaction between register recycling, reuse-cache hints, and scoreboard scheduling is not fully decoded.

[GAP] Register allocation behavior under different optimization levels or compiler versions is not tested.

[GAP] Full automated live-range reconstruction from SASS remains future tooling work.

---

Source: `knowledge/FINDINGS.md`.
