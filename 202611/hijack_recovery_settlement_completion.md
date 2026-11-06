---
tier: tale
title: Finish hijack recovery settlement and close the keep-work epic
goal:
  Every keep-work recovery reaches a settled predecessor without user action,
  interrupted settlement is repaired, proven-absent launches return to held through Rust
  policy, and epic sase-1qz.6.1 plus its parent phase sase-1qz.6 close with verified
  evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1qz.6.1.land
proposed_with: claude/opus
bead: sase-1qz.6.1
status: done
---

- **PARENT:**
  [202611/hijack_keep_work_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/hijack_keep_work_recovery.md)
- **BEAD:**
  [sase-1qz.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qz/sase-1qz.6.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1qz.6.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1qz.6.1.land.md)
- **COMMITS:**
  - [71011e1](https://github.com/sase-org/sase/commit/71011e1e10771c589e2c6dd8219dd6a3d32a1040)
    — feat(hijack-recovery): finish keep-work settlement without user action

# Plan: Finish hijack recovery settlement and close the keep-work epic

## Context

This tale is the landing remainder of epic **sase-1qz.6.1** ("Complete keep-work
recovery for hijacked agents"). Its authoritative contract is
`plan:202611/hijack_keep_work_recovery.md`; read it with
`sase artifact read plan:202611/hijack_keep_work_recovery.md "<why>"`. In particular,
read the "Invariants and shared interfaces" list and the `reconcile_wait` phase. Also
run `sase bead read sase-1qz.6.1 -r "<why>"` and read the land notes at the end. They
record the verification evidence and the follow-up triage, which is already finished. Do
not create task beads for the items below. They are epic work.

All seven phases are closed. 290 focused recovery tests pass, and the CLI
(`sase agent recover`, `sase agent persist-recover`) is in place. The landing review
confirmed, by reading the code, three gaps in the crash and settlement paths. Together
they mean a normal **queued** recovery never settles automatically. The predecessor
stays `resolving`, its `done.json` keeps `recovery.state: pending`, and anyone waiting
on it stays parked until the user reruns `sase agent recover`. Fix them without
reopening the plan's decisions (Enter = Escalate, fork-by-default context).

Rust owns state transitions and reconciliation decisions (invariant 7). Python gathers
facts and performs effects. Do not add a parallel Python state machine.

## Work

### 1. Queued and crashed recoveries must settle without user action

Current code:

- `src/sase/agent/hijack/custody.py` `matching_active_record` matches a claim only when
  its pid equals `expected_claim_owner_pid`, which is the predecessor's claim pid.
- Once `_recovery_admission_dispatcher.py` spawns the successor with
  `transfer_from_pid`, the workspace claim belongs to the successor's pid. Stale cleanup
  (`src/sase/ace/scheduler/stale_running_cleanup.py`, `_apply_hijack_custody` →
  `_reconcile_resolving_record`) therefore never reaches the record.
- `confirm_and_settle` is only called synchronously from
  `_recover_dispatch.dispatch_held_recovery`. The deferred dispatcher used for queued
  admission spawns the successor and never settles.

Required changes:

- **Sweep every resolving record.** Make the stale-cleanup pass reconcile every active
  `resolving` hijack record once per sweep, whoever currently holds the claim. Active
  records are few, so iterating `ledger.list_active_records()` is acceptable; scanning
  `settled/` is not. Keep it fail-closed and best-effort, as
  `_reconcile_resolving_record` already is.
- **Protect the workspace while resolving.** A claim in a workspace whose hijack record
  is still `resolving` must never be released by the generic stale-release path or by
  persist-cleanup. That holds whether the claim is held by the predecessor's pid, the
  reserved successor's pid, or a dead successor. Match by project plus workspace, and
  verify the successor through the record's launch identity, successor pid, or reserved
  successor name. Keep `requested`/`held` matching exact so unrelated later occupants of
  a released workspace are unaffected.
- **Settle automatically.** Once the resolver is dead and the 60 s age threshold has
  passed, the reconciliation pass must adopt the proven live or completed successor and
  settle it. The proof is the bootstrap ack plus the launch identity, which is the
  existing Rust `adopt` path. You may also have the deferred dispatcher confirm and
  settle directly after a successful spawn when it can observe the bootstrap ack.
  Reconciliation remains the guaranteed path either way. Never settle on name-only
  evidence.

### 2. Interrupted settlement must be repaired

Current code:

- `settle_recovery` (`recover_settle.py`) CASes the record to `resolved`, which moves it
  into `settled/`, and only then writes the predecessor `done.json`.
- The normal confirm path (`_recover_confirm_flow.py`) never calls `add_pending_repair`
  before that CAS.
- `recover.py` returns `already_resolved` for a resolved record without repairing it. At
  around line 321, `_reconcile_before_resume` imports `repair_settled_projection` and
  then discards it.
- `repair_pending_settlements` runs only inside a matched resolving record.

Required changes:

- **Register before the CAS.** Register the pending repair before every settle CAS: the
  confirm path, reconcile adoption, and any new settle path from step 1. Clear it after
  the `done.json` write succeeds. The simplest place is inside `settle_recovery` itself.
- **Drain every sweep.** Drain a bounded batch of pending repairs once per stale-cleanup
  sweep, whether or not any resolving record exists.
- **Repair on the resolved entry path.** When `recover_hijacked_agent` finds a
  `resolved` record, it runs `repair_settled_projection` before returning
  `already_resolved`.
- **Make the pending store safe.** `_recover_reconcile_pending.py` uses
  read-modify-write. Guard it with a file lock and write atomically (reuse
  `sase.notification_gates.durability` `file_lock`/`atomic_write_json`) so concurrent
  cleanup processes cannot drop entries.

### 3. Proven-absent launches return to held through Rust

Current code:

- `begin_resolution` always records `launch_identity` and `admission_request_id`.
- Rust `hijack_recovery_reconcile` (sase-core
  `crates/sase_core/src/agent_hijack/reconcile.rs`) therefore always takes the
  `has_recorded_launch_evidence` → retain branch for an `absent` observation.
- `_recover_reconcile_observation._successor_kind` maps a missing admission request with
  recorded launch evidence to `uncertain`.
- The result: a resolver crash before admission submission, or a cancelled or terminally
  rejected admission, stays `resolving` forever.
- Separately, `recover.py` `_resume_existing_resolution` decides
  `definitely_not_dispatched` (cancelled, or `launch_error` with receipt
  `failed`/`rejected`) in Python.

Required changes:

- **Rust.** Open the linked repo with `/sase_repo` (`sase repo open sase-core`) and read
  its `AGENTS.md`. Add a successor observation kind such as `not_dispatched`. It means
  the admission journal proves the reserved request is missing, cancelled, or terminally
  failed or rejected, and no ack, child metadata, or successor artifacts exist. Return
  it to held in `hijack_recovery_reconcile` only after the existing dead-resolver and
  minimum-age gates pass. A new `kind` string needs no wire-field change, and older
  cores keep failing closed through the unknown-kind arm. If you do change a schema
  constant, follow the repo's wire-schema recipe. Add Rust unit tests and a binding
  round-trip test. Run `sase tool run check` in sase-core.
- **Python observer.** Emit that kind from the observer only on positive proof from the
  admission journal (`observe_recovery_admission`). An unreadable or uncertain journal
  stays `uncertain`.
- **One rule for "not dispatched".** Make `_resume_existing_resolution` use the same
  observation helper, so there is a single definition of "not dispatched". The
  return-to-held decision must come from Rust: route it through
  `hijack_recovery_reconcile`, or through a Rust predicate if the immediate-return
  semantics of the resume path must be preserved. Do not keep the standalone Python
  predicate.
- **Surface retain messages.** `retain_with_error` and `uncertain` messages must reach
  the user. Stale cleanup logs them through `log_fn`, at most once per record per sweep.
  The recover entry includes the reconcile code and message in the result detail instead
  of discarding them.
- **Core pin.** Python callers depend on the new core behavior. Host finalization owns
  commits and the `sase-core-revision.txt` pin. Never write a fabricated future revision
  into that file. Rebuild the local `sase_core_rs` extension so sase tests exercise the
  new Rust code; see `docs/rust_backend.md`.

### 4. Wait following and result accuracy

- **Expired hijacks stay parked.** The landing review reported that
  `terminal_blocking_artifacts_for_name` treats `outcome: hijacked` as a failure and
  only parks `pending`/`deferred`/`launching`. That would make an **expired** hijack
  fail its waiters. The plan requires pending and expired hijacks to stay parked. Write
  a test that asserts the waiter is neither resolved nor failed, confirm the defect, and
  fix it. Strengthen the existing `test_expired_hijack_stays_parked`, which checks only
  `.resolved`.
- **Bounded successor lookup.** `_index_candidate_queries.py`
  `_resolved_hijack_successor_for_name` (around line 444–460) falls back to a loop over
  every artifact on each hop. When names collide it takes the first match in dict order.
  Use the index's per-name lookup and prefer the newest matching artifact.
- **Exit code after resume.** `_resume_existing_resolution` returns `already_resolving`,
  which `cli_recover.recovery_exit_code` maps to exit 2 ("refused"), even when it
  actually resumed and dispatched the reserved request. Return `queued` or `launched`
  (exit 0) when the resume dispatched or remains queued. Keep exit 2 only for a genuine
  refusal. Make JSON output and persist-recover agree.
- **Lost CAS in return-to-held.** `recover_reconcile._return_to_held` reports
  `returned_to_held` even when `fail_to_held` lost its CAS. Reload the record and report
  the real outcome.

### 5. Tests for the plan's acceptance matrix

Extend `tests/test_agent_hijack_recover_reconcile.py` and the
`tests/test_agent_hijack_recovery_transaction_*` modules. Reuse
`tests/_agent_hijack_recovery_transaction_support.py` and
`tests/_agent_hijack_recover_helpers.py`. Never spawn real providers or touch live
workspaces.

- **Crash matrix through reconciliation.** Inject a crash before admission, while
  queued, after child creation, after claim transfer, after the ack, and between the
  settle CAS and the `done.json` write. In each case prove there is no duplicate launch
  and no premature release, and that the eventual outcome is correct: adopt, repair, or
  return to held.
- **Stale-cleanup integration.** A queued recovery whose deferred dispatch spawns the
  successor after the CLI exits is settled by `cleanup_stale_running_entries`. The
  predecessor's `done.json` recovery reads `resolved` and the ESCALATED/CONTINUED status
  label is correct. A resolving workspace is never released by stale cleanup. Two
  concurrent cleanup passes do not double-settle.
- **Repair paths.**
  - `repair_settled_projection` through the recover entry.
  - Pending-repair draining when no resolving record exists.
  - Malformed metadata keeps custody with an actionable message. Tighten the existing
    test, which currently accepts any outcome.
- **CLI.**
  - Escalate without `-m`, so the previewed default target is used.
  - A failed-inheritance case and a queued case through the CLI or persist-recover with
    the fake provider.
- **Waits.** Session waits that pass through solo-agent promotion, and a chain that
  includes a second hijack.

Run the focused suites, then `sase tool run check` in sase. Do not run
`just check-full`. If a failure reproduces identically on the clean base, record it on
the epic with `sase bead note sase-1qz.6.1 "..."` and check for an existing tracking
bead. Such a failure does not block closeout.

### 6. Closeout (final step; do this in the same turn as the code)

1. Run `sase bead epic-symbols sase-1qz.6.1` and `sase bead epic-symbols sase-1qz.6`.
   For every listed entry, either resolve the symbol or, only when a still-open later
   bead genuinely needs the exemption, re-key it to that bead. To resolve it, wire up a
   real caller, privatize it, or delete it, following the Symvision epic-whitelist
   policy in memory `symvision.md`. New symbols from this tale must have real production
   callers. The three existing
   `sase-1qz.7(AgentHijackConfig|normalize_hijack_rung|strip_hijack_launch_directives)`
   rows belong to the still-open restart/dismiss phase sase-1qz.7; leave them.
2. Close the epic:
   `sase bead close sase-1qz.6.1 --note "<what was implemented and verified: same-workspace recovery, required inheritance, concurrent/crash tests, automatic settlement of queued and crashed recoveries, settlement repair, wait following, CLI/ops parity, check result>"`.
   Never use `--force` merely to make the close succeed.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read sase-1qz.6.1`, i.e. `202611/hijack_keep_work_recovery.md` in
   the plans repository.
5. The epic's `parent_bead` is the **phase** `sase-1qz.6` ("Keep-the-work recovery
   (Escalate, Switch, Continue, Instruct)"). Verify that its description is fully
   delivered by this child epic:
   - the exactly-once resolution transaction;
   - strict same-workspace successor launches with required inheritance;
   - the hijack fork frame;
   - predecessor ESCALATED/CONTINUED settlement;
   - crash reconciliation;
   - the recovery preview API;
   - `sase agent recover` for escalate and continue.

   Then close only that phase:
   `sase bead close sase-1qz.6 --note "<what you verified>"`. Do not close the
   containing epic `sase-1qz` or any other ancestor; its own land agent owns it.
