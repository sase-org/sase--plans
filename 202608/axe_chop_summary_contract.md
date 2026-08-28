---
tier: tale
title: Fix chronic axe chop failures in gate_shell_reclaim and stale_running_cleanup
goal:
  The hourly gate_shell_reclaim chop reports its summary correctly and exits 0, and the
  stale_running_cleanup skip_monitor_claims degradation path survives an unimportable
  sase.monitor package.
size: medium
proposed_by: bbugyi200.athena.0fc
create_time: 2026-08-28 06:57:19
status: wip
---

# Fix chronic axe chop failures: `gate_shell_reclaim` and `stale_running_cleanup`

## Problem

Two axe chop jobs have been reporting `exit code 1` in the axe error digest. They have
two independent causes, one chronic and one latent-but-triggered.

### Evidence

`~/.sase/axe/recent_errors.json` (100 most recent axe errors) shows
`('housekeeping', 'gate_shell_reclaim')` as by far the dominant entry — 40 of 100
errors, one per hourly housekeeping tick, with no successful run in between. The first
occurrence is `2026-08-26T18:23:59-04:00`, the first housekeeping tick after commit
`1cb772d9c` ("feat(gate): add gate shell lifecycle", 2026-08-26 16:52) introduced the
chop.

`('hooks', 'stale_running_cleanup')` accounts for only 3 of 100 errors and is otherwise
healthy: `~/.sase/axe/lumberjacks/hooks/logs/output.log` shows it succeeding on its
normal ~8s cadence immediately before and after each failure.

### Root cause 1 — `gate_shell_reclaim` violates the builtin chop result contract

`src/sase/scripts/sase_chop_gate_shell_reclaim.py:11-14` is the only builtin chop that
discards its runtime and writes raw JSON to stdout:

```python
@builtin_chop("gate_shell_reclaim")
def _run(runtime: BuiltinChopRuntime) -> None:
    del runtime
    summary = reclaim_pending_gate_shells()
    print(json.dumps(summary.to_dict(), sort_keys=True))
```

`run_builtin_chop` (`src/sase/chops/builtin.py:136-138`) treats a `None` return as "the
handler logged its summary line through the runtime logger", and falls back to
`_result_from_summary(name, runtime.log.last_summary)`. Because this handler never
touches `runtime.log`, `last_summary` is `None`. The raw JSON line does not help:
`parse_summary` (`src/sase/chops/sdk.py`) rejects it, since after partitioning on the
first `:` the body token `0,` has no `=`.

The result is the traceback recorded in
`~/.sase/axe/lumberjacks/housekeeping/logs/output.log`:

```
File ".../src/sase/scripts/sase_chop_gate_shell_reclaim.py", line 18, in main
    run_builtin_chop("gate_shell_reclaim")
RuntimeError: builtin chop 'gate_shell_reclaim' did not emit its summary
Error in gate_shell_reclaim: exit code 1
```

Important: `reclaim_pending_gate_shells()` runs to completion _before_ the crash, so the
reclaim work itself has been succeeding all along. Only result reporting fails — the
chop writes no structured result document, and the hourly job has been marked failed
every time. This is a pure reporting-contract bug, not a gate-shell correctness bug.

Every other `-> None` handler in `src/sase/scripts/` (9 of them: `comment_checks`,
`comment_zombie_checks`, `hook_checks`, `mentor_checks`, `orphan_cleanup`,
`pending_checks_poll`, `pr_submitted_checks`, `stale_running_cleanup`,
`suffix_transforms`, `workflow_checks`) is a single delegating statement to
`runtime.hook_runner.run_*` or `runtime.check_cycle_runner.run_*`, and those runners log
a parseable summary line. `gate_shell_reclaim` is the sole deviant. Nothing in `tests/`
exercises this chop script, which is why it shipped broken.

### Root cause 2 — the `skip_monitor_claims` degradation path still imports the monitor

`HookJobRunner.run_stale_running_cleanup` (`src/sase/axe/hook_jobs.py:375-381`) is
already written to degrade when the monitor subsystem cannot be loaded:
`_reconcile_dead_monitor_supervisors()` catches the failure, returns `None`, logs a
warning, and the caller passes `skip_monitor_claims=True` so no monitor-workflow claim
is touched on that sweep.

But `cleanup_stale_running_entries`
(`src/sase/ace/scheduler/stale_running_cleanup.py:116`) unconditionally performs the
very import that just failed:

```python
from sase.monitor.start import MONITOR_WORKSPACE_CLAIM_WORKFLOW
```

so the "skip" path crashes anyway. The 2026-08-28T06:44:51 failure is exactly this
sequence — the log shows the intended graceful degradation firing first, then the
unguarded import killing the process:

```
[hooks] monitor reconciliation failed: invalid syntax (_agent_status_apply.py, line 169)
[hooks] Traceback (most recent call last):
  ...
  File ".../src/sase/ace/scheduler/stale_running_cleanup.py", line 116, in cleanup_stale_running_entries
    from sase.monitor.start import MONITOR_WORKSPACE_CLAIM_WORKFLOW
  ...
  File ".../src/sase/ace/tui/models/_agent_status_apply.py", line 169
    <<<<<<< HEAD
    ^^
SyntaxError: invalid syntax
[hooks] Error in stale_running_cleanup: exit code 1
```

The _trigger_ was environmental and has already healed: the axe daemon runs chops out of
the primary checkout `/home/bryan/projects/github/sase-org/sase`, and that tree briefly
held unresolved merge-conflict markers in
`src/sase/ace/tui/models/_agent_status_apply.py`. That file is clean now, and the next
tick (06:44:58) succeeded. Do **not** try to "fix" the conflict — it is gone.

What is worth fixing is the code defect the trigger exposed: a documented degradation
path that cannot actually run because it depends on the thing it is degrading away from.
Any future import-time breakage anywhere in the `sase.monitor` → `sase.shells.member` →
`sase.axe.run_agent_helpers` → `sase.ace.tui.models` chain (a long chain reached from a
leaf cleanup helper) reproduces this.

## Scope

In scope:

1. Make `gate_shell_reclaim` satisfy the builtin chop result contract.
2. Make the `skip_monitor_claims=True` path in `cleanup_stale_running_entries` survive
   an unimportable `sase.monitor`.
3. Add regression tests for both, plus one registry-level guard so a future builtin chop
   cannot ship with this same contract violation.

Out of scope:

- Any change to gate-shell reclaim _semantics_. `reclaim_pending_gate_shells()` works
  correctly; only its reporting is broken.
- Repairing `_agent_status_apply.py` conflict markers (already resolved upstream).
- Shortening the `sase.monitor` import chain, or moving
  `MONITOR_WORKSPACE_CLAIM_WORKFLOW` to a leaf module. That is a defensible larger
  refactor; file it as a follow-up task bead instead of doing it here.

## Step 1 — Emit a real summary from `gate_shell_reclaim`

Rewrite `src/sase/scripts/sase_chop_gate_shell_reclaim.py` to follow the
`ChopResultBuilder`-returning pattern already used by
`src/sase/scripts/sase_chop_managed_tmp_reap.py` (the closest sibling: also
housekeeping, also a standalone helper call rather than a `hook_runner` delegation).

- Drop the `import json`, the `del runtime`, and the bare `print(...)`.
- Change the handler annotation to `-> ChopResultBuilder` and import `ChopResultBuilder`
  from `sase.chops.sdk`.
- Return `runtime.emit_summary({...})` built from the reclaim summary's counters.
  `_GateShellReclaimSummary.to_dict()` (`src/sase/gate_shell/reclaim.py:29-38`) already
  returns exactly the right shape — six `int` values keyed `scanned`, `answered`,
  `stopped`, `timed_out`, `lost`, `errors` — and every value is a valid `SummaryValue`.
  Pass it straight through rather than restating the keys, so the summary cannot drift
  from the dataclass.
- Supply a `reason` so `ChopResultBuilder.from_summary` classifies quiet runs as `no_op`
  rather than `ok`, matching every sibling chop. Use `reason="no_pending_gate_shells"`
  when `scanned == 0`, and `None` otherwise. Note that `scanned` counts only
  non-terminal records (`reclaim.py` skips `record.is_terminal` before incrementing), so
  `scanned == 0` is the correct "nothing to do" signal — a run that scanned shells but
  settled none is still meaningful work and should stay `ok`.

Keep the module docstring and the `main()` / `__main__` block as they are.

## Step 2 — Guard the monitor import behind `skip_monitor_claims`

In `src/sase/ace/scheduler/stale_running_cleanup.py`, make the
`MONITOR_WORKSPACE_CLAIM_WORKFLOW` import non-fatal when the caller has already told the
function not to touch monitor claims.

- Keep `from sase.gate_shell.claims import GATE_WORKSPACE_CLAIM_WORKFLOW` as-is; it is
  not implicated.
- When `skip_monitor_claims` is `True`, do not import `sase.monitor.start` at all. The
  function only needs `MONITOR_WORKSPACE_CLAIM_WORKFLOW` to _identify_ monitor claims,
  and every identified monitor claim is skipped on that path anyway.
- Implementation note: `is_monitor_claim` is consulted in more than one place in the
  loop body (the `skip_monitor_claims` early-`continue` and the
  `_monitor_claim_is_releasable` branch), so do not simply delete the variable. Set the
  workflow constant to a sentinel that no real claim workflow can equal (for example a
  module-level private sentinel string, or `None` with an
  `is_monitor_claim = workflow is not None and claim.workflow == workflow` comparison)
  so `is_monitor_claim` is uniformly `False` and every monitor-claim branch is
  unreachable. Either way, be explicit in a short comment that this is only safe because
  the caller has already committed to leaving monitor claims alone.
- Do **not** wrap the import in a bare `try/except ImportError`. A `SyntaxError` is not
  an `ImportError`, so that would not have caught the observed failure, and silently
  swallowing arbitrary import errors on the `skip_monitor_claims=False` path would let a
  live monitor's workspace be handed to another agent — precisely the hazard the
  existing docstring warns about. The fix is to not perform the import, not to tolerate
  its failure.
- Update the `skip_monitor_claims` docstring paragraph to record that the flag also
  means "the monitor package may be unimportable; do not touch it."

Leave `HookJobRunner.run_stale_running_cleanup` alone — it already does the right thing.

## Step 3 — Tests

Add to `tests/test_axe_chop_output_contract.py`, reusing that module's existing
`_write_context` helper and `_isolate_chop_result_file` fixture:

- `test_gate_shell_reclaim_emits_noop_summary`: monkeypatch the script module's
  `reclaim_pending_gate_shells` to return a summary with all-zero counters; assert
  stdout contains `gate_shell_reclaim:`, `scanned=0`, and
  `reason=no_pending_gate_shells`; assert the written result JSON has
  `schema_version == 1`, `status == "no_op"`, `reason == "no_pending_gate_shells"`, and
  the full six-key `counters` dict.
- `test_gate_shell_reclaim_emits_action_summary`: same shape with a non-zero summary
  (e.g. `scanned=3, answered=1, lost=1`); assert `status == "ok"`, `reason` absent/None,
  and the counters round-trip.

The first of these two must fail against the current `sase_chop_gate_shell_reclaim.py`
with `RuntimeError: builtin chop 'gate_shell_reclaim' did not emit its summary`. Confirm
that before writing the fix.

Add to `tests/test_stale_running_cleanup.py`:

- A test that `cleanup_stale_running_entries(log_fn, skip_monitor_claims=True)`
  completes normally when `sase.monitor.start` cannot be imported. Simulate by
  installing a `sys.modules` entry (or a `monkeypatch`ed import hook) that raises
  `SyntaxError` for `sase.monitor.start`, mirroring the real failure mode rather than
  `ImportError`. Assert the call returns an `int` and does not raise.
- A companion test asserting that with `skip_monitor_claims=False` the monitor import is
  still performed, so the guard cannot silently disable monitor-claim handling on the
  normal path.

Registry-level guard — add a new test (`tests/test_axe_chop_output_contract.py` is a
reasonable home, or a small dedicated module):

- Import every `sase.scripts.sase_chop_*` module so `_BUILTIN_CHOPS` in
  `src/sase/chops/builtin.py` is fully populated, then AST-inspect each registered
  handler's source. Assert that each handler either (a) is annotated to return
  `ChopResultBuilder`, or (b) delegates through `runtime.hook_runner` /
  `runtime.check_cycle_runner`. Assert additionally that no handler contains
  `del runtime`.
- This exact rule catches the shipped bug: `gate_shell_reclaim` was annotated `-> None`,
  did not delegate, and deleted its runtime. All 24 current chop scripts satisfy the
  rule once Step 1 lands — verified by survey, 14 return `ChopResultBuilder` and 10
  delegate.
- Keep the assertion message actionable: name the offending chop and point at the two
  legal patterns.

## Verification

- `just check` must pass. Run it through `/sase_monitor` if it runs long.
- Because this touches the axe chop entry-point surface, also run `just check-full`
  through `/sase_monitor` with the `TESTING` / `TESTED` status pair before landing.
- Manual confirmation that the chronic error is actually gone, run from the workspace
  checkout:

  ```bash
  python -c "
  from sase.gate_shell.reclaim import reclaim_pending_gate_shells
  print(reclaim_pending_gate_shells().to_dict())
  "
  ```

  then drive the chop end to end against a temp context file and confirm exit status 0
  plus a written result document. Do not rely on watching the live daemon: housekeeping
  only ticks hourly.

- After landing, the next housekeeping tick should log
  `Structured chop result: no_op · gate_shell_reclaim: scanned=0 ...` in
  `~/.sase/axe/lumberjacks/housekeeping/logs/output.log`, and
  `('housekeeping', 'gate_shell_reclaim')` should stop accumulating in
  `~/.sase/axe/recent_errors.json`.

## Follow-ups (file as task beads, do not implement here)

- `MONITOR_WORKSPACE_CLAIM_WORKFLOW` is a bare workflow-name constant, yet reaching it
  drags in the whole `sase.monitor` → `sase.shells` → `sase.axe.run_agent_helpers` →
  `sase.ace.tui.models` chain from a leaf scheduler helper. Moving it to a leaf
  constants module would remove this class of failure entirely rather than working
  around it.
- `sase.gate_shell.reclaim.reclaim_pending_gate_shells` is public but returns the
  private `_GateShellReclaimSummary`. Worth a symvision-aware look at whether the
  dataclass should be public.
