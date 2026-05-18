# PATTERN-08: Divergence and reconvergence control

**Category**: control_flow
**Evidence**: [OBS] Chapter 20 contains controlled loop, predication, `break`, and back-edge observations; Chapter 21 contains controlled lane-divergence, reconvergence, vote, guarded HMMA, local-call, and tail-store observations; Chapter 24 contains production-like guarded HMMA and cold-error-path contexts.
**Confidence**: [INF] C2 for Chapter 20 and Chapter 21 controlled source-shape variants; C1 for static recognition in future production dumps unless source mapping or runtime validation is added. Runtime behavior of the Chapter 21 probes remains a gap because the driver was unavailable in that environment.

**Plain English**

[INF] A divergence and reconvergence control pattern is how SASS represents lanes of a warp taking different logical paths and later returning to a common execution point.

[INF] The important rule is that lane-dependent source code does not always become a visible `BSSY`/`BSYNC` region. ptxas can choose predicated arithmetic, select instructions, predicated exits, predicated stores, ordinary forward branches, explicit reconvergence scopes, vote-based control, or `WARPSYNC.ALL` depending on the code shape.

[INF] In practical audits, this pattern tells the reader how to avoid over-reading control flow. A predicated instruction may be a cheap local choice, while `BSSY.RECONVERGENT` plus `BSYNC.RECONVERGENT` marks a branch-kept region that ptxas decided needs explicit reconvergence tracking.

[INF] Seeing this pattern is evidence for control-flow lowering structure. It does not by itself prove runtime mask behavior, branch probability, divergence cost, or correctness of partially predicated tensor-core execution.

**SASS signature**

[OBS] Chapter 20 observes preserved loops through backward `BRA` or `BRA.U` targets.

[RES] Ordinary preserved loops do not require `BSSY`/`BSYNC` in the tested variants.

[OBS] Chapter 20 observes `BSSY.RECONVERGENT B0, 0x1c0` and `BSYNC.RECONVERGENT B0` in the tested dynamic `break` loop, while short conditional bodies, larger conditional bodies, and `continue` variants emit no BSSY/BSYNC.

[OBS] Chapter 21 observes BSSY/BSYNC in exactly six of twenty tested divergence variants: 21c, 21e, 21f, 21l, 21o, and 21r.

[OBS] Chapter 21 observes lane-dependent control without BSSY/BSYNC in 21a, 21d, 21h, 21i, 21j, 21k, 21m, 21n, 21p, 21q, 21s, and 21t.

[RES] Chapter 21 rejects the hypothesis that lane divergence alone always forces visible BSSY/BSYNC.

[OBS] Chapter 21 observes the branch-kept divergent arithmetic form in 21c: `BSSY.RECONVERGENT B0, 0x1e0`, a predicated forward `BRA`, divergent arithmetic, and `BSYNC.RECONVERGENT B0`.

[OBS] Chapter 21 observes nested divergence in 21e with one BSSY/BSYNC pair and forward branches inside the reconvergence region.

[OBS] Chapter 21 observes long divergent-body lowering in 21l with one BSSY/BSYNC pair around the long divergent body.

[OBS] Chapter 21 observes lane-dependent `break` in 21f and lane-dependent trip count in 21o using BSSY/BSYNC plus useful loop back-edges.

[OBS] Chapter 21 observes divergent noinline call lowering in 21r with `BSSY.RECONVERGENT B0, 0x170`, a predicated branch, `CALL.REL.NOINC`, `BSYNC.RECONVERGENT B0`, and a callee ending in `RET.REL.NODEC`.

[OBS] Chapter 21 observes predicated datapath lowering without BSSY/BSYNC: 21a uses predicated `FADD`, 21d uses predicated arithmetic on both paths, and 21k uses `FSEL`.

[OBS] Chapter 21 observes predicated exit and tail-store lowering without BSSY/BSYNC in 21h, 21p, and 21q.

[OBS] Chapter 21 observes vote and warp-level control forms without BSSY/BSYNC: `VOTE.ANY R5, PT, P0` in 21j and `VOTE.ANY P0, P0` before a branch guarding `SHFL.IDX` in 21t.

[OBS] Chapter 21 observes guarded tensor-core execution in 21n as predicated setup, `@P0 WARPSYNC.ALL`, and `@P0 HMMA.16816.F32`, with no BSSY/BSYNC.

[OBS] Chapter 24 observes supporting production-like forms: 24o emits predicated guarded HMMA plus `WARPSYNC.ALL`, and 24ad emits `BSSY.RECONVERGENT`, `BSYNC.RECONVERGENT`, and `BPT.TRAP` around a cold path.

**Variants**

[OBS] Predicated arithmetic variants include 21a and 21d, where lane-dependent choices are represented by predicated math and no explicit reconvergence scope.

[OBS] Select variants include `FSEL` in 21i and 21k, and `UFSEL` in the uniform-branch 21b case.

[OBS] Predicated exit variants include early return and bounds/tail checks in 21h, 21p, and 21q.

[OBS] Explicit reconvergence variants include branch-kept divergent arithmetic, nested divergent control, lane-dependent break, lane-dependent trip count, and divergent local calls.

[OBS] Guarded warp-level variants include `VOTE.ANY`, `SHFL.IDX`, `WARPSYNC.ALL`, and predicated HMMA contexts in Chapter 21.

**Interpretation**

[INF] `BSSY.RECONVERGENT Bn, target` opens a reconvergence scope and `BSYNC.RECONVERGENT Bn` closes it for the same barrier register in the observed patterns.

[INF] The branch target on BSSY is the reconvergence point to inspect first, but full stack semantics and barrier-register allocation beyond the observed B0 cases remain unresolved.

[INF] In the tested variants, BSSY/BSYNC is associated with divergent regions that ptxas keeps as explicit control regions or local calls, not with every lane-derived predicate.

[INF] Predicated arithmetic, `FSEL`, `UFSEL`, predicated `EXIT`, and predicated `STG` should be treated as control-flow lowering forms, even when there is no explicit branch region.

[INF] `@P0 WARPSYNC.ALL` before `@P0 HMMA` identifies guarded warp-level tensor-core execution in the tested source, but it does not establish full `WARPSYNC.ALL` semantics or safe behavior for all partial-lane MMA patterns.

[INF] A backward `BRA`/`BRA.U` identifies a preserved SASS loop, while BSSY/BSYNC identifies reconvergence machinery. These are related control-flow features but should not be collapsed into one loop signature.

**Anti-patterns**

[INF] A lane-derived predicate alone is not enough to claim explicit reconvergence; Chapter 21 shows many lane-dependent variants without BSSY/BSYNC.

[INF] A backward `BRA.U` alone identifies a loop back-edge, not a divergence/reconvergence scope.

[INF] `BAR.SYNC` is CTA synchronization, not a substitute name for warp reconvergence through BSSY/BSYNC.

[INF] `WARPSYNC.ALL` by itself is not a reduction signature and not proof of BSSY/BSYNC-style reconvergence.

[INF] A predicated `EXIT` or predicated `STG` in a tail path should not be classified as a full branch-kept reconvergence region unless the BSSY/BSYNC region is visible.

**Open gaps**

[GAP] Runtime validation of Chapter 21 variants remains blocked by unavailable driver evidence in that chapter.

[GAP] Exact ptxas threshold for predication/select versus explicit BSSY branch region remains unresolved.

[GAP] `WARPSYNC.ALL` control-code bits, predicate behavior, and interaction with HMMA, LDSM, and SHFL remain unresolved.

[GAP] Non-full warp masks after genuine divergence remain under-tested.

[GAP] Exact BSSY/BSYNC stack semantics and barrier-register allocation beyond B0 remain unresolved.

---

Source: `knowledge/FINDINGS.md`.
