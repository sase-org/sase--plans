---
tier: tale
title: Recover typed launch hold arming and finish its integration
goal:
  Typed launches arm, carry, rebind, and release durable holds end to end on current
  master with complete acceptance coverage and no epic symbol exemptions.
size: medium
proposed_by: bbugyi200.athena.sase-11l.5.1.2.1.land
bead: sase-11l.5.1.2.1
status: done
---

- **BEAD:**
  [sase-11l.5.1.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.5.1.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-11l.5.1.2.1.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-11l.5.1.2.1.land.md)
- **COMMITS:**
  - [ff08843](https://github.com/sase-org/sase/commit/ff088437985ffa0f679ae59406280202f58b9279)
    — feat(agent): pre-arm typed launch holds

# Recover typed launch hold arming and finish its integration

## Context

Epic `sase-11l.5.1.2.1` cannot land yet. Its typed-arm phase `sase-11l.5.1.2.1.3` was
closed after producing commit `ca8a419d9e` (rebased as `0db1201999`), but that commit
was later orphaned by an aborted pull/rebase and a reset to `origin/master`. The current
tree therefore contains the Rust hold support, Python facade, and runner-bootstrap work
from the other phases, but not the typed-plan call sites. All source hunks from
`0db1201999` still apply cleanly to current master except the `Justfile`, whose
surrounding Symvision list changed.

The implementation must recover the intended behavior, not merely make the orphaned
commit reachable. Preserve all changes that landed after the epic started, especially
the routine-job evidence refactor in `run_agent_directives.py` and later `Justfile`
recipes. The published core pin `e210d18a80a3d16066ff81dd2866c68ab7a24786` already
contains Rust commit `f93ed139f4b1295d7c508a90cf822be9f229e75c`; no Rust change is
expected.

The one `PROPOSED FOLLOW-UP` from phase `.2` is not part of this tale: commit
`88175f34fc` already changed the stale proc-environment test-family entry from
`tests/test_config.py` to `tests/test_config_merge.py`.

## Implementation

1. Recover the typed-arm source delta from `0db1201999` onto current master. If that
   local object is unavailable, implement the equivalent behavior from this checklist
   and the approved parent plan `plan:202609/hold_launch_arming.md`:
   - Pre-arm every hold-carrying typed unit under the admission lock after writing the
     sidecar and before any dispatch. Write idempotent `units/<logical_id>.hold.json`
     markers, resolve agent/proc projects consistently with dispatch, and roll back
     every record and marker armed by a failed submission.
   - Ignore `*.hold.json` in receipt reconciliation. Re-anchor bundle records to a
     detached coordinator before its `started.json`, and re-anchor a successful local
     agent dispatch to the spawned runner without changing hold timing.
   - Carry `SASE_LAUNCH_HOLD_KEY` through the default, approved-request, and AXE chop
     local-agent dispatch paths. Release the launch key for guarded remote dispatches
     and for every terminal unit that never dispatches.
   - Rebind a dispatched proc hold to `proc:<proc_id>`, release it when rebinding fails
     or the proc is already terminal, and set the pending proc candidate's `armer_key`
     so its own `future` hold cannot block it.
   - Use `HOLD_ARMER_WAIT_PRIORITY` for hold-carrying procs without an authored priority
     while preserving an authored priority. Keep this inside the existing authored
     queue-capacity path.
   - Add the facade read needed to inspect records without applying launch-armer
     liveness while re-anchoring. Preserve the existing flag-off behavior and all
     `%hold: ` error prefixes.

2. Resolve every Symvision exemption keyed to `sase-11l.5.1.2.1` as part of the code
   integration; do not re-key any entry to a later bead.
   - The recovered cross-file consumers make `LaunchHoldError`, `rebind_hold`,
     `release_hold_best_effort`, and `unit_hold_key` live. Remove their four
     `--epic-symbol` entries.
   - `arm_hold_for_fields`, `hold_fields_for`, `launch_unit_armer`, and
     `runner_anchor_armer` remain helpers used only inside `launch_hold.py`. Rename them
     as private helpers, update their in-module callers and focused tests, remove them
     from `__all__`, and delete their four `--epic-symbol` entries rather than retaining
     a stale whitelist.

3. Close the acceptance gaps that allowed the phase to appear complete without its
   commit on master. Reuse existing launch-admission fixtures and avoid duplicating
   coverage already present for the Rust store, facade, or bootstrap primitives. Add
   focused tests for the missing externally observable behavior:
   - agent and proc pre-arm fields (scope, TTL, project, pending capture), marker
     idempotency, flag-off no-op, empty request ID, and rollback for both kin and
     over-cap TTL failures;
   - coordinator and runner re-anchor, including preservation of `created_at`, receipt
     completion, the runner-wins race, and missing spawned-result fallback;
   - launch-key transport through default, approved, and AXE chop dispatch, plus the
     remote-dispatch release guard;
   - proc rebind, settlement release, failed-rebind cleanup, self-exclusion, and
     authored-versus-implied capacity priority;
   - release on `skipped`, `condition_error`, `launch_error`, and `cancelled`, and the
     composed `%proc(...) %q:1 %hold(pending, future)` drain/block/admit flow;
   - any still-missing bootstrap-level assertion needed to prove a fresh hold is armed
     before dependency waiting, while refresh/retry/flag-off paths do not re-arm and an
     authored priority still wins.

   Fix implementation defects exposed by those tests within this tale's launch-hold
   scope.

## Verification

- Confirm `sase bead epic-symbols sase-11l.5.1.2.1` reports no entries before handing
  control back to the parent land agent.
- Run the focused hold, launch-admission, proc-capacity, AXE chop, bootstrap,
  runner-slot, scan-wire, and TUI-enrichment tests affected by the recovered paths.
- Run `tools/check_sase_core_rs_bindings` and validate the installed binding against the
  sanctioned linked `sase-core` checkout; do not edit the Rust repository unless a real
  binding incompatibility is found.
- Run `just fix`, then the repository-required `just check`. Leave the exhaustive
  `just check-full`, epic close, post-close `just symvision`, and linked-plan status
  update to the resumed land agent.
