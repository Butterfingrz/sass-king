# PATTERN-18: Scoreboard grouping and variable-latency waits

**Category**: scheduling
**Evidence**: [OBS] Kernels 01 and 03 contain the controlled SM120 scoreboard observations for pointer loads, global loads, fixed-latency stalls, and co-consumed memory producers.
**Confidence**: [INF] C2 for the controlled Kernel 01 and 03 source-to-SASS examples; C1 for static recognition in future production dumps unless control-code parsing, profiling, or latency microbenchmarks are added.

**Plain English**

[INF] Scoreboards are how SASS waits for variable-latency work. A load, special-register read, barrier, MUFU, or other variable-latency instruction can signal a scoreboard when its result is ready; a later consumer waits on that scoreboard instead of relying only on a fixed stall count.

[INF] The key audit question is not just "which instruction loaded the data", but "which scoreboard did it signal, and which later instruction waited for that scoreboard".

[INF] ptxas can group multiple producers onto one scoreboard when they are consumed together, or assign distinct scoreboards when their consumers can proceed independently.

[INF] Seeing this pattern identifies dependency scheduling structure. It does not by itself prove cache latency, memory throughput, occupancy behavior, or whether the chosen grouping is optimal.

**SASS signature**

[OBS] Kernel 03 records six scoreboards available per warp, labeled SB0 through SB5 in the chapter analysis.

[OBS] Kernel 03 records the basic mechanism: an instruction with `SBS=N` signals scoreboard N, and an instruction with `wait={N}` blocks until that scoreboard is clear.

[OBS] Kernel 03 observes fixed-latency operations such as FFMA using stall count in the chapter scheduling model. [INF] For those fixed-latency dependencies, the stall count rather than a scoreboard wait is the local synchronization mechanism described by the chapter.

[OBS] Kernel 03 records `LDG`, `LDS`, `S2R`, `MUFU`, and `BAR` as variable-latency operation classes in the chapter scoreboard model.

[INF] These classes should be audited as scoreboard-coordinated producers when their results feed later consumers.

[OBS] Kernel 01 observes two co-consumed global loads on the same scoreboard: `LDG.E R2, ...` with `SBS=4` and `LDG.E R5, ...` with `SBS=4`, followed by `FADD R9, R2, R5` with `wait={SB4}`.

[OBS] Kernel 03 extends the co-consumed pattern to three global loads on `SB4`, followed by `FFMA R11, R2, R5, R7` with `wait={SB4}`.

[OBS] Kernel 03 observes four pointer/descriptor constant loads using distinct scoreboards: `LDC.64 R2` with `SBS=0`, `LDCU.64 UR4` with `SBS=1`, `LDC.64 R4` with `SBS=2`, and `LDC.64 R6` with `SBS=3`.

[OBS] Kernel 03 observes those distinct pointer-load scoreboards being consumed by downstream address or bounds computations. [INF] The separate scoreboards preserve readiness separately for those downstream computations.

[OBS] Kernel 03 observes `yield` on instructions that wait on scoreboards in the studied dump.

**Variants**

[OBS] Kernel 01 demonstrates two LDG producers grouped on `SB4`, followed by one downstream `FADD` that consumes both loaded values.

[OBS] Kernel 03 demonstrates three LDG producers grouped on `SB4`, followed by one downstream `FFMA` that consumes all three loaded values.

[OBS] Kernel 03 demonstrates separate LDC/LDCU scoreboards before separate downstream `IMAD.WIDE` address computations.

[OBS] Kernel 03 observes address computations interleaved between global-load emissions: `IMAD.WIDE`, `LDG.E`, `IMAD.WIDE`, `LDG.E`, then later consumer wait.

[OBS] Kernel 01 observes global stores as fire-and-forget in the tested elementwise shape, with no downstream readback in the function.

**Interpretation**

[INF] When several loads share the same `SBS` and one later instruction waits on that same scoreboard, the safe local conclusion is that the producers are synchronized as a group for that consumer.

[INF] Distinct scoreboards on nearby constant or pointer loads can indicate that ptxas wants dependent address computations to become ready independently.

[INF] A small stall count on a variable-latency producer is not a full latency model. The scoreboard wait on the consumer is the dependency-correctness mechanism.

[INF] Interleaved address arithmetic between `LDG` instructions is evidence that ptxas is filling the latency window with independent work.

[INF] Yield on a scoreboard wait tells the scheduler it can run another warp if the current warp blocks, but it does not quantify how long the wait will last.

**Anti-patterns**

[INF] Do not infer one scoreboard per load. Kernel 01 and Kernel 03 show co-consumed global loads can share a scoreboard.

[INF] Do not infer that same-scoreboard loads are independent-ready for separate consumers; grouping means the later wait observes the group.

[INF] Do not treat stall count alone as the data-ready mechanism for global loads or other variable-latency producers.

[INF] Do not infer memory latency or bandwidth from the presence of `SBS` and `wait` without profiling or microbenchmark evidence.

[INF] Do not treat a global store with no local consumer as requiring a visible scoreboard wait in the same function.

**Open gaps**

[GAP] The exact control-code bit placement for stall, yield, scoreboard signal, and wait masks remains partially decoded rather than fully mapped.

[GAP] Throughput and latency differences between grouped and distinct scoreboard choices are not measured.

[GAP] Whether the observed six-scoreboard budget is universal across related architectures and compiler versions is not established.

[GAP] The scheduler policy behind choosing grouped versus distinct scoreboards is inferred from local dependency shape, not proven by profiling.

---

Source: `knowledge/FINDINGS.md`.
