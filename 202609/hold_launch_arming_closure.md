---
tier: tale
size: medium
title: Complete and close the hold launch arming epic
goal:
  Verify and finish launch-time hold arming, then close sase-11l.5.1.2.1 and its parent
  phase sase-11l.5.1.2 with concrete completion evidence.
proposed_by: bbugyi200.athena.0mi
create_time: 2026-09-17 15:23:41
status: wip
---

# Complete hold launch arming and close its tracking beads

## Outcome and scope

Complete the remaining integration work for **sase-11l.5.1.2.1**, then close that epic
and its immediate parent phase **sase-11l.5.1.2**, in that order, using the normal
`sase bead close` completion path. The user's explicit request authorizes both closes
and supersedes the old plan's instruction to close only the phase. Leave ancestors
`sase-11l.5.1`, `sase-11l.5`, and `sase-11l` open.

Use one medium tale: the four implementation phases are already closed, the Rust
implementation is published and pinned, and the remaining work is bounded Python handoff
hardening, integration coverage, closure-check correctness, and combined verification.
This does not need another phased epic.

## Evidence and required context

Read these designs through `sase artifact read`:

- `plan:202609/hold_launch_arming.md`, especially sections 2, 6, 7, and 8.
- `plan:202609/hold_directive_surface.md`, especially phase `arm-runtime`.

Read current state with
`sase bead show sase-11l.5.1.2.1.. sase-11l.5.1.2 --format json --no-links`. At planning
time, all four child phases were closed with resolution `done`; the epic and parent
phase were `in_progress`. The epic's latest note prefers a tale. The bead artifact page
was unavailable, so bead state was read through the CLI; do not regenerate unrelated
published pages merely to perform this work.

Use `/sase_memory_read` for `sase_beads.md`, `lint_and_test.md`, `symvision.md`,
`xprompts.md`, and `sase_flags.md`. Read applicable repository instructions. Open Rust
sources only through `/sase_repo` and
`sase repo open sase-core -r "Verify hold launch arming completion"`.

Confirmed implementation and remaining gaps:

- `src/sase/agent/launch_hold.py` implements pre-arm, rollback, coordinator and runner
  re-anchor, bootstrap rebind, and terminal release. Admission, proc dispatch, runner
  bootstrap, implied priority, and tri-state marker enrichment are wired in production.
- In both `reanchor_pending_unit_holds` and `reanchor_dispatched_agent_hold`,
  `_hold_record_without_liveness` is called **outside** the exception handler. A store
  read/binding error can escape a best-effort handoff: before the coordinator's startup
  acknowledgement, or after agent spawn but before its launched receipt is recorded. The
  approved design requires these failures to be logged without aborting the handoff.
- `tests/test_launch_admission_dispatch.py` covers pre-arm, rollback, flag-off, dispatch
  env, terminal release, self-exclusion, and priority. Its runner re-anchor assertion
  uses the no-liveness store reader. Existing hold tests do not exercise the coordinator
  acknowledgement ordering or the entire `%proc(...) %q:1 %hold(pending, future)`
  barrier-and-release sequence.
- `tests/test_run_agent_runner_bootstrap_baseline.py` checks bootstrap ordering with a
  mocked arming function; add an integration assertion using the real store at the
  dependency-wait boundary.
- The published Rust feature commit is `f93ed13` (launch armer core support). The
  existing pin `e210d18a80a3d16066ff81dd2866c68ab7a24786` is its descendant and is
  reachable from `origin/master`. No pin bump is presently required.
- The old follow-up on phase `.2` about `tests/test_config.py` is already addressed:
  `tests/test_proc_env_isolation.py` now references
  `tests/test_config_merge.py::test_deep_merge_list_concatenation`.
- The current checkout's `Justfile` has no exemptions for this epic. However, ID-scoped
  `sase bead epic-symbols` reports eight old entries from the project's primary
  checkout. `cli_epic_symbols._symbol_scan_start` and
  `cli_crud_lifecycle._owner_symbol_start` unconditionally choose the routed primary
  workspace. The same mismatch can block closing a bead after its exemptions have
  already been resolved in the implementing checkout.

## Implementation

### 1. Make the two handoff re-anchors consistently best-effort

In `src/sase/agent/launch_hold.py`, include record lookup and re-anchor preparation
inside each helper's guarded operation. For coordinator handoff, isolate failures per
unit so one failed lookup or rebind does not prevent remaining units from being
attempted or prevent `started.json` from being written. For dispatched agents, a lookup
or rebind failure must not prevent the already-spawned unit's receipt and `launched`
state from being persisted. Log the unit/key and cause.

Keep initial pre-arm fail-closed with rollback and zero dispatch on failure. Keep
bootstrap rebind failure behavior, missing-key behavior, TTL, original `created_at`,
captured pending targets, and the existing fail-open liveness backstop. Do not re-arm a
missing/expired/released hold or extend its lifetime.

Add regression tests that inject a raw store lookup error and a rebind error in both
handoff paths. Assert the coordinator acknowledges and the dispatched agent is recorded
once; assert the error is observable in logs. Exercise the production admission entry
points where feasible, rather than merely asserting helper calls.

### 2. Cover the lifecycle boundaries required by the original design

Use the real Rust-backed store and predicates in an isolated temporary `SASE_HOME`. Fake
external launches, runner records, clocks, and process-liveness facts where needed;
never create a hold in the user's live store or launch real agents. Reuse
`tests/_launch_admission_helpers.py` and runner-slot fixtures. Put new focused tests in
small modules such as `tests/test_launch_hold_handoffs.py` and
`tests/test_launch_hold_integration.py` instead of enlarging oversized suites.

Verify these observable behaviors:

1. **Coordinator acknowledgement:** pre-arm a blocked typed plan, enter
   `run_coordinator_in_bundle`, and observe that its bundle-anchored hold has the
   coordinator PID before the real startup marker is written. Preserve key, timing,
   scope, and pending capture. Include a record already anchored to a runner and prove
   it is left alone. Complete/incomplete receipt and dead-PID liveness cases must agree
   with existing service tests.
2. **Submission through runner bootstrap:** complete an inline agent admission, then use
   the active, liveness-aware reader to prove the hold survives its completed bundle
   receipt. Rebind through bootstrap and assert a single agent hold retains both
   timestamps and selectors. A non-kin candidate created between submission and rebind
   must still match `future`. Cover the reverse ordering, where bootstrap binds first
   and the engine's re-anchor is a no-op.
3. **Dependency wait:** bootstrap `%wait:dependency %hold(pending)` and assert the real
   hold is already present at the wait/claim boundary. Pending capture occurs once;
   refresh/retry does not recapture; the launch-key env variable is scrubbed. Reuse
   existing focused tests for skip and flag-off behavior.
4. **Proc composition:** exercise a typed proc carrying queue capacity 1 and
   `pending, future`. While existing work occupies capacity, its hold is already active
   and the proc remains pending without holding itself. After capacity drains, dispatch
   and rebind to `proc:<id>`; a later non-kin runner must park with `hold-barrier`.
   Settle through the production proc-settlement path and prove that runner can then
   claim a slot. Assert authored priority wins and the implied priority remains
   non-explicit, using existing tests where they already prove those facts.

Fix any failures directly attributable to these contracts. Shared selector, identity,
timing, and admission semantics remain in Rust; do not duplicate them in Python. If a
regression requires a Rust change, update core and bindings first, run the core
repository's required checks, and use the sanctioned publication/pin workflow. Pure
process plumbing stays in Python.

### 3. Make the closure guard inspect the appropriate checkout

Unify the scan-root choice used by the ID-scoped `epic-symbols` display and the
close-time exemption guard. When the invocation checkout belongs to the routed bead's
project, inspect that checkout's `Justfile`; a foreign-project target must continue to
inspect its owner's checkout. Calls from outside a recognized local checkout should
retain the owner fallback. Use existing project/store ownership resolution rather than
path-name heuristics or assuming every invocation cwd is the owner. Keep shared
ownership policy in Rust if new policy is needed; prefer reusing the existing Rust
routing results with thin Python filesystem glue.

Relevant files are `src/sase/bead/cli_epic_symbols.py`,
`src/sase/bead/cli_crud_lifecycle.py`, `src/sase/bead/epic_symbols.py`, and
`src/sase/bead/operation_context.py`. Add same-project isolated-checkout regressions in
`tests/test_bead/test_cli_epic_symbols.py` and
`tests/test_bead/test_cli_close_epic_symbols.py`:

- stale primary exemptions do not block a clean implementing checkout;
- an exemption in the implementing checkout still blocks closure;
- foreign-project targets ignore caller decoys and still validate the owner;
- invocation from a checkout subdirectory works, and no usable local checkout falls back
  to the owner without disabling the guard.

Do not edit another checkout to erase the discrepancy, bypass the guard, or re-key dead
exemptions to an unrelated open bead. Recheck current state before editing because
another change may already have fixed the routing issue.

## Verification and continuation

Reconfirm the pinned revision contains `f93ed13` and is published. Validate the
installed binding with `tools/check_sase_core_rs_bindings` and
`tools/validate_sase_core_rs`, supplying the opened core path as required. Use
`just install` if the local test environment is stale. Do not move the pin to newer
unrelated core work merely because a newer release exists.

Run focused pytest coverage for the new tests and these affected suites:

- `tests/test_launch_hold.py`, `tests/test_agent_hold_service.py`,
  `tests/test_launch_admission_dispatch.py`, `tests/test_launch_proc_runtime.py`;
- `tests/test_run_agent_runner_bootstrap_baseline.py`,
  `tests/test_run_agent_runner_slot_capacity.py`,
  `tests/test_run_agent_runner_slot_priority.py`;
- `tests/test_enrich_agent_waiting_queue.py`, `tests/test_core_agent_scan_options.py`,
  the affected bead-symbol suites, and the previously reported
  `test_sase_ml_file_families_ignore_inherited_live_proc_env` regression.

Run Rust tests as separate valid cargo commands (cargo accepts one test filter per
invocation): `cargo test -p sase_core agent_hold`,
`cargo test -p sase_core agent_launch`, `cargo test -p sase_core agent_scan`,
`cargo test -p sase_core --test agent_scan_parity`, and `cargo test -p sase_core_py`. If
core changes, its `just check` is also required.

Run `just fix` or at least `just fmt` before verification. Run `just check` after
changes. The original epic requires **`just check-full` on the combined tree**; run it
only through `/sase_monitor` with the `verify` profile (`TESTING`/`TESTED`). The monitor
must include a follow-up instruction to inspect results, fix any in-scope failures,
rerun affected checks as warranted, then perform the ordered bead closure below.
Starting a monitor does not complete this plan.

Record actual results, tested source revisions, and the binding/pin relationship. Do not
claim that tests run against a newer local core certify the pinned build; use a binding
built from the pin or explicitly verify relevant source identity and disclose any
remaining difference. Unrelated failures must be identified accurately and handled under
the existing discovered-work rules; do not call a red combined run green or close on
incomplete required evidence.

## Ordered completion

After required checks pass:

1. Reread the epic and parent, confirm every descendant is closed, and inspect their
   current notes. Confirm the historical config-test follow-up is resolved; record its
   existing fix instead of filing a duplicate. Use `/sase_new_task` only if genuinely
   new follow-up work needs tracking.
2. Run `sase bead epic-symbols sase-11l.5.1.2.1` and
   `sase bead epic-symbols sase-11l.5.1.2`; both must be empty and must inspect the
   intended checkout. Resolve any real remaining entries per Symvision rules.
3. Close `sase-11l.5.1.2.1` with resolution `done` and a substantive `--note`
   summarizing the fixes, lifecycle tests, full verification result, Rust pin, and
   disposition of follow-ups. Do not use force or hand-edit bead state.
4. Close `sase-11l.5.1.2` with resolution `done` and a note recording that its child
   epic is complete. Preserve the original design refinements in this evidence: only
   complete receipts end launch holds; coordinator and runner handoffs preserve timing;
   the env key is scrubbed; future-cycle checks apply; hold markers are not receipts;
   condition errors release holds; explicit false priority survives the scan wire.
5. Read both beads again and verify `closed`/`done`, correct notes, and unchanged
   ancestor statuses. Finish using `/sase_final` so the host handles any required
   commits. If a manual response is needed after a monitor, it still requires that final
   declaration.

If this tale's execution itself creates an open descendant under a target bead, settle
that descendant through its normal completion mechanism first. Never force-close the
target around unfinished work or reopen completed child phases solely to repeat their
implementation.

Previews, confirmation UI, directive syntax changes, global deadlock detection, flag
removal, and ancestor-epic completion are outside this tale. No memory edits are
required.
