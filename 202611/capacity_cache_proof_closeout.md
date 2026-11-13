---
tier: tale
title:
  Prove capacity-cache concurrent writes and root rejection, then finish the landing
goal:
  The promised cache safety tests exercise real publication and independent root
  rejection, and sase-1sl.1.1 plus parent phase sase-1sl.1 close normally after
  verification.
size: small
proposed_by: bbugyi200.athena.sase-1sl.1.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1sl.1.1
status: done
---

- **PARENT:**
  [202611/capacity_scan_core.md](https://github.com/sase-org/sase--plans/blob/main/202611/capacity_scan_core.md)
- **BEAD:**
  [sase-1sl.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1sl/sase-1sl.1.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1sl.1.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1sl.1.1.land.md)
- **COMMITS:**
  - [1b3aef9](https://github.com/sase-org/sase-core/commit/1b3aef9464b929dafbce125638367596041ece5b)
    — test(agent-scan): prove capacity cache write safety

# Finish capacity-cache safety proof and land sase-1sl.1.1

## Scope and authority

Finish the two concrete verification gaps found while landing `sase-1sl.1.1` (plan
`plan:202611/capacity_scan_core.md`), then complete that epic's landing and its parent
phase `sase-1sl.1` in this same implementation turn. One agent can implement this
bounded test repair directly. There is no separate tale land agent and no later
automatic return to the interrupted lander.

The approved decisions remain `verdict_cache = persist` and
`live_tui_restart = separate_instance`. No architecture change, feature flag, memory
edit, live TUI operation, or global installation is needed. The outer epic `sase-1sl`
owns Python adoption, `sase-core-revision.txt`, real-archive measurement and the TUI
soak; leave those to its existing workers.

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. All source paths below are
relative to that returned checkout. Use audited bead and artifact reads for the existing
scope and notes, including the landing audit on `sase-1sl.1.1` and all three child
phases.

## Already verified

The three implementation commits are `1ceb6a8f` (projection and parity), `fa5adc2b`
(persistent cache), and `bab5978195d9a52afa828b18cdba779a1b930ee2` (benchmark and
warm-path tuning). The last phase's note incorrectly names `fa5adc2b` as its final
commit; the containing adoption revision is `bab59781` or a later commit containing it.
The current derived format is `.capacity_scan_cache_v2.json`, not the earlier v1 format
described in phase 2's historical note.

The only proposed follow-up, phase 3 note 1's sccache connection-reset report, was
independently reproduced with healthy storage and a failing cache-server stub. It is the
distinct medium bug task `sase-1sp`, with evidence
`file:explicit:1182493b8b8ec8ca08782193` and related links to `sase-1p0` and `sase-165`.
It predates this scanner epic and does not block this landing.

The lander read production projection, retention, cache, scanner, context, legacy oracle
and test code. Both collection APIs preserve full parsing for retained records,
pre-retention clan context, conservative marker/PID behavior, and independent admission.
It fetched both base branches: core at `bab59781` and sase at `4a4ee98c65`, with no
unintegrated remote commits. Core's intervening `3416e4e2` process-identity projection
is preserved by retained-row hydration. Sase's restart lookup, GC policy, attachment
visibility and edit-lock/prompt changes do not require scanner changes. Refresh this
drift check before closing.

Core check ToolRun `4bb448250ae0457058ff41f4f6c4ed04` succeeded with exit 0, verdict
pass. The lander compared all five recorded dirty-file SHA-256 hashes to the files
committed in `bab59781`; all match. The release benchmark recorded on phase 3 passed
without lowered budgets: 20,000 historical rows plus 2,500 done rows, 175 retained,
4,096 clan generations, three repeats; baseline 42,316.0 ms, cold 1,519.8 ms (27.84x),
warm 876.6 ms, and 40k warm 1,679.6 ms. Fresh-process reuse and
record/context/capacity-decision equality are asserted by that benchmark. A test-only
repair does not require repeating this expensive benchmark unless a production change or
new concern invalidates its evidence.

## Remaining implementation

Repair `crates/sase_core/src/agent_scan/capacity_cache_tests.rs` without weakening the
production cache invariants or adding public APIs.

1. Make `concurrent_full_and_subset_writers_stay_valid` exercise actual overlapping
   writes. Today `cold_scan` prepopulates every entry; all workers only hit unchanged
   cache entries, so `store_capacity_cache` returns without writing. The test also
   discards every worker result. Use a disposable root and an absent or intentionally
   stale cache, coordinate overlapping full, exact-subset and filtered scans, and assert
   each worker's returned records, clan context and capacity decisions against its own
   independent legacy oracle. Include live and retained-by-closure rows so the
   assertions are meaningful. Prove the observed run actually exercises cache
   publication, with a narrow test-only seam if scheduling alone cannot establish it.
   Inspect the persisted file before any repairing full scan; it must parse as a
   complete current-version cache. Lost hits are allowed by the lock-free contract, so
   do not require a union of all writers. Confirm subsequent scans still match the
   oracle and warm reuse remains effective. Avoid sleeps, timing assertions and
   process-global environment mutation.
2. Make root-mismatch rejection independent in `cold_warm_and_corrupt_cache_agree` (or a
   focused new test). Its current fixture uses version 1 and an entry `{}` with no
   required `meta`, so deserialization fails before root validation. Start with a real
   valid current-version cache, change only its root identity, prove loading rejects it,
   then prove the scanner performs fresh classification and matches the independent
   oracle. Keep corrupt-data and unknown-version coverage as separate cases. Ensure
   removing the root-identity check would fail this test.

If these tests expose a production defect in the epic's cache implementation, fix that
defect within this tale and keep the same public behavior. Add the minimal regression
coverage and rerun the archive budget proof only when the production change affects the
measured scan path. Do not broaden the task to unrelated test infrastructure.

## Verification

Use `just fmt`, the targeted `just test -p sase_core` cache/parity tests, and the
required core `sase tool run check` gate (`just check` through its guarded wrapper). No
bare cargo and no `just check-full`. Hand a long check to `/sase_monitor` when required;
its successor must carry this tale's closeout instructions. A targeted run is not a
substitute for the core gate. If any tracked sase file changes, read `lint_and_test.md`
through `/sase_memory_read` and run its `just check` workflow as well. Do not weaken
checks for unrelated infrastructure failures.

## Final step: finish the interrupted landing

This step runs after verification, before the final code's commit, push or CI result.
Never create an ordering dependency on this turn's future commit SHA. The host finalizer
commits the completed work after the normal closeout.

1. Re-read `sase-1sl.1.1` and children `sase-1sl.1.1.1`, `sase-1sl.1.1.2`,
   `sase-1sl.1.1.3`, including every note; confirm this tale resolves both audit gaps
   and that no new issue or descendant blocks completion. Review new commits since the
   lander's two tips above. The only existing proposed follow-up (sccache failure, phase
   3 note 1) was triaged by the lander; copy its recorded outcome from the epic audit
   note, without filing a duplicate.
2. Run `sase bead epic-symbols sase-1sl.1.1`. Resolve every listed exemption by wiring,
   privatizing, deleting or applying a legitimate non-test pragma under the Symvision
   policy; re-key only to an open later bead that still needs it. The previous audit
   found none. Then close normally:
   `sase bead close sase-1sl.1.1 --note "<verified implementation, repaired cache tests, checks, drift review, benchmark evidence, corrected containing revision, and follow-up outcome>"`.
   If rejected, resolve the actual blocker and retry. Never force a successful nested
   landing; unfinished phases must be finished or reopened. Force is permitted only for
   a deliberately canceled or superseded scope with an explicit reason, never to make
   this close succeed.
3. Run `just symvision` in sase after closing. If its cleanup changes source, verify
   those changes. Open the plans repository with `/sase_repo`, use
   `sase artifact read plan:202611/capacity_scan_core.md` for the audited read, and set
   `status: done` in that plan's frontmatter at the path resolved in this workspace. Do
   not mark the outer plan done.
4. Run `sase bead read sase-1sl.1.1 -r "Need the parent link"` and confirm the parent is
   phase `sase-1sl.1`. Re-read that phase and the audited parent plan
   `plan:202611/capacity_scan_tui_runaway.md`; confirm this child completes its Rust
   scanner/cache/parity/budget requirements. Run `sase bead epic-symbols sase-1sl.1` and
   resolve or legitimately re-key every entry, then close only that phase normally with
   `sase bead close sase-1sl.1 --note "<Rust behavior, repaired verification, budgets, core gate and containing revision verified>"`.
   Run `just symvision` again after that closure. Leave containing epic `sase-1sl` to
   its already-waiting land agent. If the phase is unexpectedly incomplete or its parent
   link is ambiguous, record a blocker note and stop ancestor closure; do not force it.
5. Finish through `/sase_final`, declaring every modified repository for commit. Report
   the normal epic and parent-phase closeout, or any precise remaining blocker. The
   finalizer, not this tale, creates the commits.
