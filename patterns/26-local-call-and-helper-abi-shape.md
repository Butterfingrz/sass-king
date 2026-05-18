# PATTERN-26: Local CALL and helper ABI shape

**Category**: call_abi
**Evidence**: [OBS] Chapters 06, 11, 12, and 21 contain controlled SM120 observations for named helper calls, local slowpath calls, manual return-address setup, register argument passing, and divergent noinline calls.
**Confidence**: [INF] C2 for the controlled same-compilation-unit examples; C1 for future dump recognition unless separate compilation, callee ownership, and runtime control flow are validated.

**Plain English**

[INF] A SASS `CALL` is a visible control-flow transfer to helper code, but in the observed SM120 same-compilation-unit cases it does not look like a conventional stack-based CPU call. The caller writes a return address into a normal register, executes `CALL.REL.NOINC`, and the callee returns with `RET.REL.NODEC` using that register.

[INF] This pattern explains the call frame shape itself. A call target may be a named external helper such as `__cuda_sm20_rem_u16`, a local hex-address slowpath placed after the main `EXIT`, or a noinline local function body selected by ptxas.

[INF] Seeing this pattern identifies out-of-line helper control flow. It does not by itself prove a source-level function boundary, stack spilling, an external library call, or the ABI used by separate compilation.

**SASS signature**

[OBS] Chapter 06 observes runtime modulo lowering to `CALL.REL.NOINC __cuda_sm20_rem_u16`.

[OBS] Chapter 06 observes the u16 helper ABI using `R8` as the dividend input, `R9` as divisor input and remainder output, and a caller-selected return-address register such as `R7` or `R10`.

[OBS] Chapter 06 observes u16 argument masking with `LOP3.LUT` before the helper call.

[OBS] Chapter 11 observes local slowpath calls with `BSSY.RECONVERGENT`, a predicate or branch selecting the slowpath, `CALL.REL.NOINC <hex_address>`, `BSYNC.RECONVERGENT`, and a slowpath body placed after the main kernel `EXIT`.

[OBS] Chapter 11 observes the u64 division slowpath setup `MOV R4, 0x290` before `CALL.REL.NOINC 0x2e0`, with the slowpath returning through `RET.REL.NODEC R4 0x0`.

[OBS] Chapter 11 observes the sqrtf slowpath using the same local-call structure with a caller-written return address, `CALL.REL.NOINC`, and `RET.REL.NODEC`.

[OBS] Chapter 12 observes Kernel 12j setting `MOV R12, 0x1d0` before `CALL.REL.NOINC 0x220`.

[OBS] Chapter 12 observes Kernel 12j passing 16 float arguments entirely in registers and reports no `STL` or `LDL` for argument passing.

[OBS] Chapter 21 observes a divergent noinline local call using `BSSY.RECONVERGENT B0, 0x170`, `CALL.REL.NOINC 0x1b0`, `BSYNC.RECONVERGENT B0`, and a local callee ending in `RET.REL.NODEC R2 0x0`.

**Variants**

[OBS] Named helper call: Chapter 06 uses `CALL.REL.NOINC __cuda_sm20_rem_u16` for runtime u16 modulo.

[OBS] Local arithmetic slowpath: Chapter 11 uses hex-address `CALL.REL.NOINC` targets for rare division or math-library cases.

[OBS] Local noinline call: Chapters 12 and 21 observe local callee bodies selected by ptxas from source shape, with register-passed inputs.

[OBS] Divergent guarded call: Chapters 11 and 21 place local calls inside `BSSY`/`BSYNC` reconvergence structure when only some lanes may need the callee.

**Interpretation**

[INF] The strongest signature is the return-address `MOV` immediately before a `CALL.REL.NOINC`, paired with a callee-side `RET.REL.NODEC` that consumes the selected register.

[INF] The `.NOINC` and `.NODEC` suffixes are consistent with the observed calls avoiding visible stack pointer increment/decrement in these same-compilation-unit cases.

[INF] A hex-address `CALL.REL.NOINC` is best read as a local subroutine or slowpath until a symbol, relocation, or separate object boundary proves otherwise.

[INF] The Chapter 12 16-argument noinline case shows that a local call can still be register allocated globally by ptxas; the call is not spill evidence by itself.

[INF] The chosen return-address register is local to the compiled shape. Chapters 06, 11, 12, and 21 show different registers, so future audits should identify it from the nearby `MOV` and `RET` rather than hard-code one ABI register.

**Anti-patterns**

[INF] Do not treat every `CALL` as a stack frame, spill, or caller-saved register boundary.

[INF] Do not treat every hex-address call as an external library call; Chapter 11 uses local slowpaths with numeric targets.

[INF] Do not assume absence of `CALL` means absence of slowpath work; Chapter 11 also contains inline branch slowpaths and MUFU-based lowering.

[INF] Do not infer the separate-compilation ABI from same-compilation-unit local calls.

[INF] Do not classify a named runtime helper and a local noinline callee as the same evidence class; their control-flow shape may match while ownership and ABI guarantees differ.

**Open gaps**

[GAP] Separate compilation and `-rdc=true` behavior are not tested.

[GAP] CALL/RET latency and scoreboard behavior are not measured.

[GAP] The ptxas threshold for choosing local CALL versus inline slowpath versus named helper remains unknown.

[GAP] Return-address register selection is observed but not modeled.

[GAP] Cross-architecture stability of `CALL.REL.NOINC` and `RET.REL.NODEC` forms is not established.

---

Source: `knowledge/FINDINGS.md`.
