---
tier: tale
title: Finish failed-turn handoff settlement and land sase-1vn.3.1
goal:
  Make failed-stage cleanup retryable, preserve recovery evidence, settle after terminal
  publication, and close the child epic and parent phase.
size: medium
proposed_by: bbugyi200.athena.sase-1vn.3.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1vn.3.1
create_time: 2026-11-27 19:25:32
status: wip
---

- **PARENT:**
  [202611/failed_turn_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202611/failed_turn_handoffs.md)
- **BEAD:**
  [sase-1vn.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vn/sase-1vn.3.1.md)

# Finish failed-turn handoff settlement and land sase-1vn.3.1

Complete the remaining work in epic **sase-1vn.3.1**, whose approved plan is
`plan:202611/failed_turn_handoffs.md`. Its direct parent is phase **sase-1vn.3**
(`failed-turn-handoffs`) of containing epic **sase-1vn**. This tale has no land agent:
the coder must finish the nested closeout below in the same turn as the final code,
after verification. Closeout does not depend on this tale's future commit SHA, push, or
CI result. Do not wait for another land agent to resume this child epic.

## Verified baseline and scope

The lander read both child beads and every note. `sase-1vn.3.1.1` delivered the Rust
policy in sase-core commit `34461b7fd8c03405c330985b8fd658b469e564c1`; `sase-1vn.3.1.2`
delivered the host wiring in sase commit `c783119ffdf81215cbeaa93bddff6f4b82334e49`. The
current `sase-core-revision.txt` already includes that policy. Their recorded `check`
runs `ec922cf4702b03e5c523acd94b495502` and `07116f5e14ad994965fdaade68afc801` both
succeeded. The lander independently ran 25 inactive-host tests and 61 adjacent
stranded/recovery/reconcile/contracts/reclaim tests, all passing.

Those tests miss three acceptance gaps. Repair only these gaps and the integration
coverage needed to prove the original contract; retain the delivered Rust ownership
policy and its conservative treatment of unknown evidence.

Governing decisions:

- `decisions:rust-core-required`: shared eligibility/admission policy remains in
  sase-core; Python gathers facts and performs host I/O. If a fix changes shared policy,
  use `/sase_repo`, change and test the Rust API/binding, and keep sase's pin
  coordinated through the host finalizer. No Python policy fallback.
- `decisions:handoffs-end-through-host-completion`: inactive settlement never authorizes
  execution. Automatic reconciliation must not adopt a held stage. Explicit readoption
  still needs the existing authenticated finalizer proof.
- `decisions:host-owned-completion`: commits remain host-owned through `/sase_final`.
- `decisions:check-full-is-explicit`: run guarded `sase tool run check`; this tale does
  not authorize `just check-full`.

The parent's final decisions remain `cost_budgets = rebaseline` and
`zombie_recovery = relaunch`; neither is changed here. Do not change memory, test-cost
policy, zombie-epic relaunch policy, or TUI presentation.

## 1. Make incomplete resource cleanup retryable

In `src/sase/axe/run_agent_exec_staged_inactive.py`, `_settle_monitor_resources`
swallows import, member-resolution, cleanup, and owner-index errors.
`_settle_inactive_staged_handoff_locked` nevertheless writes the final
`.sase_handoff_settled_inactive.json` marker. The next call sees that marker and skips
cleanup permanently. The existing cleanup API also has best-effort false/skipped
outcomes, so absence of an exception alone is insufficient proof of completion.

Confirmed reproduction on the baseline: construct a failed monitor origin and member
using the helpers in `tests/test_staged_inactive_host.py`, including the owner index and
frozen member payload. Make `cleanup_staged_monitor` raise `OSError`. Call
`settle_inactive_staged_handoff` twice. The first result says `settled=True`, writes the
marker and deletes the owner index. The second says `already_settled=True`; the cleanup
function was called only once. No cleanup succeeded.

Make the result distinguish completed cleanup, already-completed cleanup, and incomplete
or unknown cleanup. Confirm every required owned reservation was settled/released, and
that a skipped cleanup is safe, before certifying settlement. A stage with no such
resource may settle without inventing an obligation. Preserve sufficient identity and
member-location evidence to retry after any partial effect. Keep an incomplete attempt
retryable; do not erase its only locator or make a completed marker suppress unfinished
work. Safely converge from an existing prematurely written marker as well, if unfinished
resources are still observable; do not blindly trust it as proof.

Reuse existing cleanup rather than creating another resource lifecycle. Re-observe
eligibility under the existing lock ordering, and preserve submitted procs, joined runs,
workspace retention and saved recovery evidence. A failed proc-store read at any cleanup
boundary must not become confirmed absence. Ensure a mismatched/replaced owner-index row
cannot be deleted by stale settlement. Add failure-injection regressions for cleanup and
owner-index failures, partial success followed by retry, and a skipped or unreadable
resource probe. Assert the durable resources, not only boolean helper results.

## 2. Preserve the failed origin's saved stage during admission

`src/sase/agent/_staged_handoff_admission.py::occupying_handoff_names` now hides an
identity-matched settled stage. `write_staged_handoff` uses that same admission result
and atomically replaces `.sase_handoff_staged.json`. A confirmed reproduction settles a
failed monitor stage, then calls `write_staged_handoff` with a questions record and the
same origin directory: the write succeeds and the saved monitor stage disappears while
its old receipt remains.

Separate inactivity for live ownership from authorization to mutate a turn's saved
stage. Preserve the original stage, receipt, response, frozen launch payload and
recovery context. A new handoff targeting that original turn must refuse instead of
replacing evidence; subsequent work uses its own turn artifacts. Keep the original stage
available for explicit readoption. Enforce the rule under the existing
staging/declaration locks for both CLI and host staging, using the Rust admission
boundary where policy is needed.

Add a regression through `write_staged_handoff` that proves a settled origin cannot be
overwritten, with original bytes/identities intact. Exercise the actual later-turn
questions entry/admission path in the same session, rather than merely writing a record
into an unrelated empty directory. Verify that the failed monitor no longer blocks that
fresh turn's lane and that live origins still retain their one-handoff protection.

## 3. Settle new failures after durable terminal publication

Both calls to `_maybe_settle_inactive_stage` in `src/sase/axe/run_agent_exec.py` run
immediately after a refused dispatch and before `_finalize_loop`. They require a failed
`done.json`, so ordinary first-time failures cannot settle there.
`src/sase/axe/run_agent_exec_finalize.py` publishes the terminal marker through
`write_done_marker_and_update_index` but has no subsequent inactive-settlement call. A
local probe confirmed no settlement marker either before or after publishing the failed
done marker through that function. Historical reconciliation is wired, but the normal
terminal-origin boundary required by the approved plan is absent.

Invoke the same idempotent settlement implementation at the host's appropriate boundary
after durable failed-terminal evidence exists. Do not synthesize terminal evidence
before finalization or weaken eligibility. Retain nonblocking/best-effort behavior so a
busy adoption lock leaves work for the existing bounded reconciler. Keep successful
turns, live/finalizing origins, submitted execution, active recovery and authenticated
adoption protected. Test the actual finalize/failed-dispatch sequence: no premature
settlement, then settlement after failure publication without waiting for a periodic
sweep, with no command or successor launched.

## 4. Complete integration evidence and verification

Extend the existing integration fixtures to run failure, reconciliation, resource
settlement, later-turn admission, and explicit recovery together. In particular,
`test_readopt_invalidates_settlement_exactly_once` currently calls
`write_adopted_locked` directly: it does not prove authenticated readoption or
reservation reacquisition. Exercise the real recovery service after repairing finalizer
proof, including any released join/prepared reservation and settled never-run ToolRun;
use the existing `_start_stage_adopt_flow.py` reacquisition path. Assert one activation,
marker invalidation, and no duplicate activation on retry. Keep unsafe/external process
effects stubbed at the process launch boundary while retaining the real policy, locks,
persisted resources and recovery checks.

Preserve and rerun coverage for failed-origin `stranded/failed`, blocked ancestors,
settlement grace, member ordering, successful pending adoption, live successors, adopted
receipts, malformed/mismatched evidence and unknown stores. Use the original approved
plan's full acceptance contract as the checklist.

The lander fetched both origins and found no unseen commits. Since this epic was
created, the non-epic sase commits were `51fc0d8cb2`, `1cde333808`, `5b184c837c`,
`f6dd22d4dd`, `dcd6c726c3` and `61d955445c`; the non-epic core change was `7e857edd`.
These concern bead ID collision/publication repair and test organization and require no
handoff-policy integration. Recheck subsequent drift before close and integrate any
newly relevant consumers or conflicts.

Run focused tests including `tests/test_staged_inactive_host.py`,
`tests/test_stranded_staged_handoff.py`, `tests/test_staged_recovery_service.py`,
`tests/test_staged_handoff_reconcile.py`, `tests/test_staged_monitor_contracts.py`,
`tests/agent/test_staged_handoff_reclaim.py`, and affected finalize/questions/resource
tests. Run `just fix` and guarded `sase tool run check` in each repository changed. For
changed Rust, use its `just test` wrapper and `check`, never bare cargo; confirm the
source pin through the supported host coordination. Use `/sase_monitor` for long checks
with a follow-up explicitly responsible for this tale's closeout; do not use prepared
completion that would skip the nested closeout below. A future commit, push or CI result
is not a prerequisite for closeout.

## 5. Finish this epic and its parent phase

The lander already completed follow-up triage and recorded it on sase-1vn.3.1. Neither
child has a `PROPOSED FOLLOW-UP:` entry. There is no genuinely distinct unrelated task
to file or corroborate. The three defects above belong to this epic.

The note on sase-1vn.3.1.2 recommends live reconciliation of incident monitor
`3w69pb9ezmfy` and closure of sase-1sh. That is this epic's verification, not a new
task. The lander read sase-1sh and observed that monitor still held with no elapsed
execution. The historical phase sase-1r2.6 was manually closed already; do not reopen it
or claim its current status can flip to stranded. Use an isolated open-phase fixture for
that assertion. After the fixes, inspect the incident's persisted failed-origin evidence
and exercise the bounded automatic recovery path without explicit adoption or running
its saved command. If current evidence cannot safely settle it, record the exact reason
and prove the intended historical shape in the isolated fixture. Following the approved
child plan's final paragraph, leave the sase-1sh resolution recommendation to the
containing sase-1vn land agent. Separate existing scopes sase-1r4 (owned by sase-1ra),
sase-1si and sase-1sj remain untouched. Include these triage outcomes and why no
separate follow-up was filed in the epic close note.

Perform these closeout operations after implementation and checks, in this order:

1. Re-read `sase bead read sase-1vn.3.1 -r "Need final child and note readiness"` and
   every descendant scope/note, including any new descendants. Verify they are complete
   and the linked plan's acceptance criteria are met. Do not force past an unfinished
   phase; finish or deliberately reopen it. Both original children are already closed,
   so no new phase close is normally necessary.
2. Run `sase bead epic-symbols sase-1vn.3.1`. Resolve each entry by wiring it,
   privatizing it, adding a justified non-test pragma, or deleting it under the
   Symvision policy; re-key only to a still-open later bead that truly needs it. The
   lander's audit found no entries, but recheck. Never leave an exemption keyed to a
   closing bead.
3. Close normally with `sase bead close sase-1vn.3.1 --note "<verification>"`. Write a
   concrete note naming the reviewed commits/pin, fixed partial-cleanup,
   evidence-preservation and terminal-boundary gaps, relevant drift review, actual check
   results, incident recommendation and follow-up triage. Never use `--force` merely to
   succeed or advance this successful nested landing. If symbol validation refuses,
   finish cleanup and retry normally.
4. Run `just symvision` and require a clean whitelist. Open the plans sidecar with
   `/sase_repo` (`sase repo open plans -r "Mark verified child epic plan complete"`) and
   set `status: done` in the frontmatter of **202611/failed_turn_handoffs.md**, the plan
   resolved by `plan:202611/failed_turn_handoffs.md`. Use that command's printed
   checkout path, and include the plan change in the final declaration.
5. Run `sase bead read sase-1vn.3.1 -r "Need the parent link"` and
   `sase bead read sase-1vn.3 -r "Verify the delegated phase is complete"`. The expected
   parent is phase **sase-1vn.3**, not a plan bead. Confirm this child completed its
   failed-turn-handoffs scope, including the final reviewer decisions and incident
   evidence. Audit `sase bead epic-symbols sase-1vn.3` and retire/re-key any entries
   before closing. Then close only that phase normally with
   `sase bead close sase-1vn.3 --note "<verified failed-turn scope and child closeout>"`.
   Re-closing an already auto-completed phase is a safe no-op; add a note for later
   evidence as needed. Leave containing epic **sase-1vn** and
   `plan:202611/backlog_epic_1.md` to its already-waiting land agent. If the parent
   remains incomplete or its link/scope is ambiguous, record the blocker on sase-1vn.3
   and stop the ancestor walk rather than force closing it.

Finish through `/sase_final` with all changed repositories declared. The host commits
the final code and plan status after this turn; that normal ordering must not postpone
the closes above.
