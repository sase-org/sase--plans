---
tier: tale
title: Make gate_shell_reclaim report its result, and make the chop contract enforceable
goal:
  The hourly gate_shell_reclaim chop exits 0 and writes a structured result, a reclaim
  sweep that hits per-record errors escalates instead of hiding them, and a registry
  guard stops any future builtin chop from shipping the same contract violation.
size: medium
proposed_by: bbugyi200.athena.0fi
create_time: 2026-08-28 10:07:04
status: wip
---

# Make `gate_shell_reclaim` report its result, and make the chop contract enforceable

## Problem

The hourly `housekeeping` / `gate_shell_reclaim` chop has failed on **every tick since
it shipped**. It has never once succeeded.

### Evidence

`~/.sase/axe/recent_errors.json` holds the 100 most recent axe errors.
`('housekeeping', 'gate_shell_reclaim')` is 46 of those 100 — one per hourly tick with
no success in between, and by far the largest single entry (next is
`('housekeeping', 'artifact_link_backfill')` at 13). The oldest occurrence is
`2026-08-26T18:23:59-04:00`, the first housekeeping tick after commit `1cb772d9c`
("feat(gate): add gate shell lifecycle", sase-ud.3) introduced the chop.

The digest entry the user sees carries no diagnosis:

```
  Lumberjack: housekeeping
  Job:        gate_shell_reclaim
  Error:      exit code 1
  Traceback:  <no python traceback: subprocess error>
```

The real error is in `~/.sase/axe/lumberjacks/housekeeping/logs/output.log`, most
recently at `08:48:41` and again at `09:48:02`:

```
File ".../src/sase/scripts/sase_chop_gate_shell_reclaim.py", line 18, in main
    run_builtin_chop("gate_shell_reclaim")
RuntimeError: builtin chop 'gate_shell_reclaim' did not emit its summary
```

### Root cause

`run_builtin_chop` (`src/sase/chops/builtin.py:121-141`) accepts three handler shapes:
`BuiltinResult = ChopResultBuilder | Mapping[str, Any] | None`. Returning `None` is an
_implicit promise_ — "I already emitted my summary line through `runtime.log`" — and the
runner falls back to `_result_from_summary(name, runtime.log.last_summary)`.

`src/sase/scripts/sase_chop_gate_shell_reclaim.py:11-14` returns `None` and never
touches `runtime.log`:

```python
@builtin_chop("gate_shell_reclaim")
def _run(runtime: BuiltinChopRuntime) -> None:
    del runtime
    summary = reclaim_pending_gate_shells()
    print(json.dumps(summary.to_dict(), sort_keys=True))
```

It `del`s the runtime and writes raw JSON with a bare `print`. Only `ChopLogger.info`
feeds `parse_summary` into `ChopLogger._summaries` (`src/sase/chops/sdk.py:111-117`), so
`last_summary` stays `None` and `_result_from_summary` raises. The JSON line could never
have helped: `parse_summary` partitions on the first `:` and then requires `key=value`
tokens, and `{"answered"` / `0,` have no `=`.

Reproduced deterministically in a clean workspace, driving the real chop through a temp
context file with the reclaim helper stubbed to an all-zero summary:

```
{"answered": 0, "errors": 0, "lost": 0, "scanned": 0, "stopped": 0, "timed_out": 0}
REPRO FAILURE: RuntimeError builtin chop 'gate_shell_reclaim' did not emit its summary
```

Note that this reproduces with `scanned=0`. The failure does not depend on there being
any gate shell to reclaim, which is why the failure rate is 100%.

### Blast radius: reporting only, not gate-shell correctness

`reclaim_pending_gate_shells()` runs to completion _before_ the crash, so pending gate
shells have been settled correctly the whole time. What is lost is downstream of the
raise: `write_chop_result` never runs, so the chop has never written a structured result
document and `sase axe status` has never recorded a successful `gate_shell_reclaim` run.
The user-visible cost is a false hourly failure that crowds real errors out of a
100-entry ring buffer it currently occupies 46% of.

### Why it shipped, and why nothing has caught it

Two independent gaps:

1. **Nothing enforces the `-> None` contract.** Survey of all 24 builtin chop scripts in
   `src/sase/scripts/`: 13 return `ChopResultBuilder`; 11 return `None`, and 10 of those
   11 are a single delegating statement to `runtime.hook_runner.run_*` or
   `runtime.check_cycle_runner.run_*`, whose runners log a parseable summary line.
   `gate_shell_reclaim` is the only non-delegating `-> None` handler, and the only
   handler in `src/sase/scripts/` containing `del runtime`. The violation is invisible
   to mypy — the annotation is perfectly legal — and to every existing test.
2. **`src/sase/gate_shell/reclaim.py` has no test module at all.** `tests/gate_shell/`
   has ten modules and none of them covers `reclaim.py`, and
   `tests/test_axe_chop_output_contract.py` covers `error_digest`, `managed_tmp_reap`,
   and `epic_launch_flush` but not this chop.

### A latent second instance of the same defect

`reclaim_pending_gate_shells` counts per-record failures into `counts["errors"]` behind
a bare `except Exception` (`src/sase/gate_shell/reclaim.py:69-71`) and logs nothing at
all. Fixing only the summary contract would round-trip that count into an `ok` result
and leave it just as invisible: a gate shell that can never be settled would be silently
retried every hour forever. That is the same class of failure this epic exists to
eliminate, so the reclaim result needs to carry enough information to escalate.

### Relationship to the already-approved plan `202608/axe_chop_summary_contract.md`

An approved tale plan (`plan:202608/axe_chop_summary_contract.md`, proposed by
`0fc--plan`, approved earlier today) diagnosed the same summary-contract bug and paired
it with an unrelated `stale_running_cleanup` fix. **Its implementation was completed and
then lost before landing.** Per the `0fc` chat transcripts, `0fc--code` implemented and
verified both halves; the run handed off to a `just check-full` monitor, and when
`0fc__1` resumed at 07:54 the only dirty path left in its workspace was
`tests/reproducible_flake_baseline.txt` — the single file its final declaration
committed, as `d929ed82b`.

Confirmed by inspection: workspace `sase_19` is clean at `de491c710` with the broken
chop script intact, and
`git log --all -- src/sase/scripts/sase_chop_gate_shell_reclaim.py` still shows
`1cb772d9c` as the only commit that has ever touched it. The fix exists nowhere in any
branch. The _mechanism_ of the loss is not established by this plan and is listed as a
follow-up.

**This plan supersedes `202608/axe_chop_summary_contract.md`.** It carries that plan's
`stale_running_cleanup` half forward verbatim in intent (Step 4), so already-approved
work is not orphaned and the two plans cannot collide on the shared guard test. It adds
what that plan explicitly deferred to follow-ups: the public reclaim result contract
(raised independently as sase-ud note #4) and error escalation.

## Scope

In scope:

1. A public, error-aware reclaim result contract.
2. `gate_shell_reclaim` satisfying the builtin chop result contract, including its
   `no_op` and `check_error` cases.
3. The `stale_running_cleanup` `skip_monitor_claims` import guard, carried forward from
   the superseded plan.
4. A registry-level guard so no future builtin chop can ship this violation.
5. The first direct tests for `src/sase/gate_shell/reclaim.py`'s result contract.

Out of scope:

- **Gate-shell reclaim semantics.** Deadline, grace, cancellation, and settlement
  behavior are correct and must not change. Only reporting and error visibility change.
- The other chronic axe entries — `artifact_link_backfill` (13/100) and the smaller
  `hooks` entries — have separate causes. File task beads, do not fix here.
- Moving `MONITOR_WORKSPACE_CLAIM_WORKFLOW` out of the `sase.monitor` import chain
  (follow-up).
- Determining how `0fc--code`'s verified implementation was lost (follow-up).
- sase-ud notes #1, #2, and #3. Those are unrelated gate-shell defects with their own
  plans.

## Step 1 — Make the reclaim result public and error-aware

File: `src/sase/gate_shell/reclaim.py`.

- Rename `_GateShellReclaimSummary` to `GateShellReclaimSummary`.
  `reclaim_pending_gate_shells` is public and already returns it; a public function
  returning a private type is exactly what sase-ud note #4 asked the epic to resolve.
  Add it to the module's `__all__`.
- **Symvision constraint, read this before writing the rename.** A public symbol needs a
  real non-test consumer, and test references never count. The rename is only safe
  because Step 2's chop script imports `GateShellReclaimSummary` by name and uses it in
  its `_reason_for` helper. Do **not** reach for a `# symvision:` pragma or an
  `--epic-symbol` whitelist to paper over a missing consumer. If the implementer finds
  the chop no longer needs the name, the correct alternative is to keep the dataclass
  private and have `reclaim_pending_gate_shells` return a plain `dict[str, int]` — but
  the error-detail field below makes the named public type the better shape.
- Add a bounded detail field to the dataclass: `error_details: tuple[str, ...] = ()`,
  and a module-private `_MAX_ERROR_DETAILS = 5`.
- Replace the bare swallow in `reclaim_pending_gate_shells` with one that records what
  failed, keeping the `continue` so one unreclaimable shell never stops the sweep:

  ```python
  except Exception as error:
      counts["errors"] += 1
      if len(error_details) < _MAX_ERROR_DETAILS:
          error_details.append(
              f"{record.member_agent_name or record.gate_id}: "
              f"{type(error).__name__}: {error}"
          )
      continue
  ```

  `GateShellRecord` (`src/sase/gate_shell/models.py:41-70`) has both `member_agent_name`
  and `gate_id`, so the detail always identifies the shell it came from. The
  `_MAX_ERROR_DETAILS` bound keeps a pathological sweep from producing an unbounded
  summary or log burst.

- **Keep `to_dict()` returning exactly the six int counters.** It is passed straight
  into `runtime.emit_summary`, whose values must be `SummaryValue = int | str | None`,
  and a tuple is not one. Expose details through the dataclass field, never through
  `to_dict()`.

## Step 2 — Rewrite the chop to satisfy the result contract

File: `src/sase/scripts/sase_chop_gate_shell_reclaim.py`.

Follow `src/sase/scripts/sase_chop_managed_tmp_reap.py` — the closest sibling, also
housekeeping and also a plain helper call rather than a `hook_runner` delegation — plus
the `check_error` idiom already established in
`src/sase/scripts/sase_chop_external_issue_mirror.py:24-35,92-93`.

- Drop `import json`, `del runtime`, and the bare `print(...)`.
- Annotate `_run` as `-> ChopResultBuilder`, importing `ChopResultBuilder` from
  `sase.chops.sdk`.
- Add the module-level helper that consumes the newly public type — this is what keeps
  the symbol alive for symvision, and it is a genuine use, not a linter appeasement:

  ```python
  def _reason_for(summary: GateShellReclaimSummary) -> str | None:
      if summary.errors:
          return "reclaim_errors"
      if not summary.scanned:
          return "no_pending_gate_shells"
      return None
  ```

- Handler body:

  ```python
  summary = reclaim_pending_gate_shells()
  for detail in summary.error_details:
      runtime.log.error(f"gate shell reclaim failed: {detail}")
  result = runtime.emit_summary(summary.to_dict(), reason=_reason_for(summary))
  if summary.errors:
      result.status = "check_error"
  return result
  ```

  Pass `to_dict()` straight through rather than restating the six keys, so the emitted
  summary can never drift from the dataclass.

- The three statuses and why each is right:
  - `errors > 0` → **`check_error`**.
    `src/sase/axe/chop_runner_script_result.py:164-190` turns a `check_error` result
    into an axe error whose message is the result's `reason`, so the digest gains an
    actionable `reclaim_errors` entry instead of today's bare `exit code 1`, while
    `runtime.log.error` puts the per-shell detail in the lumberjack log. This
    _intentionally_ keeps a digest entry alive if reclaim is genuinely broken for some
    shell — that is the point. Those failures are invisible today.
  - `scanned == 0` → **`no_op`**, `reason="no_pending_gate_shells"`. `scanned` counts
    only non-terminal records (`reclaim.py` `continue`s on `record.is_terminal` before
    incrementing), so it is the correct "nothing to do" signal.
    `ChopResultBuilder.from_summary` maps any reason to `no_op`, matching every sibling
    chop.
  - otherwise → **`ok`**. A run that scanned shells but settled none is still meaningful
    work and must not be reported as a no-op.

- Keep the module docstring, `main()`, and the `__main__` block as they are.

## Step 3 — Registry-level guard so this cannot ship again

Add to `tests/test_axe_chop_output_contract.py`, reusing its `_write_context` helper and
`_isolate_chop_result_file` fixture.

- Import every `sase.scripts.sase_chop_*` module so `_BUILTIN_CHOPS` in
  `src/sase/chops/builtin.py` is fully populated, then inspect each registered handler's
  source with `inspect.getsource` + `ast`. Assert that each handler either (a) is
  annotated to return `ChopResultBuilder`, or (b) delegates through
  `runtime.hook_runner` / `runtime.check_cycle_runner`; and that no handler contains
  `del runtime`.
- This is exactly the rule that would have caught the shipped bug: `gate_shell_reclaim`
  was annotated `-> None`, did not delegate, and deleted its runtime.
- Do **not** hardcode a chop count — assert the rule per registered chop so adding a
  chop does not require touching the test. For orientation only: today there are 24
  scripts, 13 returning `ChopResultBuilder` and 11 returning `None`; after Step 2 it is
  14 and 10.
- Make the assertion message actionable: name the offending chop and state the two legal
  patterns.

## Step 4 — Carry forward the `stale_running_cleanup` import guard

This step is inherited from the superseded plan; it is unchanged in substance.

`HookJobRunner.run_stale_running_cleanup` (`src/sase/axe/hook_jobs.py:375-381`) is
already written to degrade when the monitor subsystem cannot be loaded:
`_reconcile_dead_monitor_supervisors()` catches the failure, returns `None`, logs a
warning, and the caller passes `skip_monitor_claims=True` so no monitor-workflow claim
is touched that sweep. But `cleanup_stale_running_entries`
(`src/sase/ace/scheduler/stale_running_cleanup.py:116`) then performs the very import
that just failed, so the "skip" path crashes anyway. The `2026-08-28T06:44:51` failure
is exactly that sequence: the intended degradation fires, then the unguarded import
kills the process on a `SyntaxError` raised from deep in the `sase.monitor` →
`sase.shells.member` → `sase.axe.run_agent_helpers` → `sase.ace.tui.models` chain.

In `src/sase/ace/scheduler/stale_running_cleanup.py`:

- Leave `from sase.gate_shell.claims import GATE_WORKSPACE_CLAIM_WORKFLOW` alone; it is
  not implicated.
- When `skip_monitor_claims` is `True`, do not import `sase.monitor.start` at all. The
  function only needs `MONITOR_WORKSPACE_CLAIM_WORKFLOW` to _identify_ monitor claims,
  and every identified monitor claim is skipped on that path anyway.
- `is_monitor_claim` is consulted in two places in the loop body (the
  `skip_monitor_claims` early-`continue` and the `_monitor_claim_is_releasable` branch),
  so do not simply delete the variable. Bind the workflow name to a value no real claim
  workflow can equal — a module-private sentinel string, or `None` with an
  `is_monitor_claim = workflow is not None and claim.workflow == workflow` comparison —
  so `is_monitor_claim` is uniformly `False` and both monitor branches are unreachable.
  Add a short comment stating that this is only safe because the caller has already
  committed to leaving monitor claims alone.
- **Do not** wrap the import in `try/except ImportError`. A `SyntaxError` is not an
  `ImportError`, so it would not have caught the observed failure, and swallowing
  arbitrary import errors on the `skip_monitor_claims=False` path would let a live
  monitor's workspace be handed to another agent — precisely the hazard the existing
  docstring warns about. The fix is to not perform the import, not to tolerate its
  failure.
- Extend the `skip_monitor_claims` docstring paragraph to record that the flag also
  means "the monitor package may be unimportable; do not touch it."

Leave `HookJobRunner.run_stale_running_cleanup` alone — it already does the right thing.

## Step 5 — Tests

**`tests/test_axe_chop_output_contract.py`** — three new tests alongside the guard from
Step 3, monkeypatching the script module's `reclaim_pending_gate_shells`:

- `test_gate_shell_reclaim_emits_noop_summary`: all-zero summary. Assert stdout contains
  `gate_shell_reclaim:`, `scanned=0`, and `reason=no_pending_gate_shells`; assert the
  written result JSON has `schema_version == 1`, `status == "no_op"`,
  `reason == "no_pending_gate_shells"`, and the full six-key `counters` dict.
- `test_gate_shell_reclaim_emits_action_summary`: e.g. `scanned=3, answered=1, lost=1`.
  Assert `status == "ok"`, no `reason`, and that the counters round-trip.
- `test_gate_shell_reclaim_reports_check_error_on_reclaim_errors`: a summary with
  `errors=1` and one `error_details` entry. Assert `status == "check_error"`,
  `reason == "reclaim_errors"`, and that the detail string reaches stderr.

`test_gate_shell_reclaim_emits_noop_summary` **must fail against the current script**
with `RuntimeError: builtin chop 'gate_shell_reclaim' did not emit its summary`. Confirm
that before writing the Step 2 fix.

**`tests/gate_shell/test_reclaim.py`** (new — the module has no tests today). Keep it
focused on the result contract, not on re-testing settlement:

- A sweep over a record whose `_reclaim_one` raises records `errors == 1` and one
  `error_details` entry naming that shell, and still processes the following record.
- `error_details` is capped at `_MAX_ERROR_DETAILS` when more records fail than the cap.
- `to_dict()` returns exactly the six int counters and no detail field, so it stays a
  legal `emit_summary` payload.

**`tests/test_stale_running_cleanup.py`** — two tests, next to the existing
`test_cleanup_skip_monitor_claims_leaves_ace_monitor_claims_untouched`:

- `cleanup_stale_running_entries(..., skip_monitor_claims=True)` completes normally when
  `sase.monitor.start` cannot be imported. Simulate with a `sys.modules` entry or a
  monkeypatched import hook that raises `SyntaxError`, mirroring the real failure mode
  rather than `ImportError`. Assert it returns an `int` and does not raise.
- A companion asserting that with `skip_monitor_claims=False` the monitor import _is_
  still performed, so the guard cannot silently disable monitor-claim handling on the
  normal path.

## Verification

- `just install` first — this is an ephemeral workspace and its deps may have drifted.
- `just lint` must be clean, and specifically `just _lint-symvision`: Step 1 promotes a
  private class to public, so a missing consumer surfaces there as
  `Unused public functions/classes`.
- `just check` must pass. Hand it to `/sase_monitor` if it runs long.
- This touches the axe chop entry-point surface, so also run `just check-full` through
  `/sase_monitor` with the `TESTING` / `TESTED` status pair before landing.
- Manual end-to-end confirmation, from the workspace checkout — do not wait on the live
  daemon, housekeeping only ticks hourly:

  ```bash
  .venv/bin/python -c "
  from sase.gate_shell.reclaim import reclaim_pending_gate_shells
  print(reclaim_pending_gate_shells())
  "
  ```

  then drive the chop end to end against a temp context file (the shape used by
  `_write_context` in `tests/test_axe_chop_output_contract.py`) and confirm exit status
  0 plus a written result document with `schema_version: 1`.

- After landing, the next housekeeping tick should log a structured
  `gate_shell_reclaim: scanned=... reason=no_pending_gate_shells` result line in
  `~/.sase/axe/lumberjacks/housekeeping/logs/output.log`, and
  `('housekeeping', 'gate_shell_reclaim')` should stop accumulating in
  `~/.sase/axe/recent_errors.json`.

## Follow-ups (file as task beads via `/sase_new_task`; do not implement here)

- `0fc--code` implemented and verified the superseded plan, then its changes disappeared
  before the final declaration, which committed only an unrelated flake-baseline file.
  Silent loss of verified work between a monitor handoff and the resuming agent is a
  serious infrastructure defect and deserves its own investigation.
- `MONITOR_WORKSPACE_CLAIM_WORKFLOW` is a bare workflow-name constant, yet reaching it
  drags the whole `sase.monitor` → `sase.shells` → `sase.axe.run_agent_helpers` →
  `sase.ace.tui.models` chain into a leaf scheduler helper. Moving it to a leaf
  constants module would remove this class of failure outright rather than working
  around it.
- `('housekeeping', 'artifact_link_backfill')` is 13 of the current 100 axe errors and
  has a separate, undiagnosed cause.
