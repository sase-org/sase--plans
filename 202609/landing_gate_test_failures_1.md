---
tier: tale
title: Fix the three test-cost failures that blocked sase-11l.11 landing
goal:
  The three test-cost nodes that failed monitor pz319b38sapt pass under isolated pytest
  and just check, with x vs !x matching the Services-tab contract, import-budget
  CPU-stable, and real-gateway hello diagnosable under load.
size: medium
proposed_by: bbugyi200.athena.sase-11l.11.5.land.f0
create_time: 2026-09-19 09:53:07
status: wip
---

# Fix the three test-cost failures that blocked sase-11l.11 landing

## Why this is a tale

One coding agent can implement all three repairs. They share a verification gate
(`just check`, not `just check-full`) and a landing constraint (do not close
`sase-11l.11` or `sase-11y.7`), but they are not a multi-phase product epic. An epic
would only slow the unblock.

Do **not** parent this plan under `sase-11l.11`. The hold-repair land agent already
classified these failures as unrelated to hold/pin work. Do **not** parent it under
`sase-11y` either: phase `sase-11y.7` remains in progress for the rest of the Services
tab.

## Out of scope

- `just check-full`, `just test-cost`, and the sase-core `just check` gate. The user
  forbade `just check-full`; verify with `just check` plus focused pytest.
- `pyproject.toml` / `uv.lock`. The published `sase-core-rs` floor stays on `sase-10d` /
  `sase-12y.4`.
- Memory files.
- The leak-detector `SASE_*_DISABLE` poisoning from the same monitor. Origin
  `8989d0a724` already reuses the isolation ignore list, and this tree includes that
  commit.
- Closing `sase-11l.11`, `sase-11l`, or `sase-11y.7`.

## Evidence (current tree is `48d0a69287`)

Isolated pytest of the three identified nodes plus the rest of
`test_service_host_keys.py` is **5 passed in 9.55s** on this workspace. That does not
mean the landing-gate failures are gone:

| Node                                                                | At SHA `423316a051` (monitor)                                   | On this tree                                                                                                                                             |
| ------------------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test_x_does_not_toggle_the_host_on_nested_scheduler_rows`          | Deterministic fail: `['start-host']` vs `[]`                    | Passes because `91b67672b4` (ToolRun, not Services-tab work) added nested-row early returns and taught the test to set chop/lumberjack                   |
| `test_tui_app_import_stays_under_startup_budget`                    | Full-lane 32.7s vs 5.0s wall; isolated 2.75s                    | Still wall-clock `perf_counter` with 3 retries (`8989d0a724`). Isolated 2.80s. Retries cannot save a 32s measurement under a 6-worker 43k-test cost lane |
| `test_bootstrap_issue_enroll_hello_round_trip_through_real_gateway` | `status.state == "error"` after enroll succeeded; isolated pass | Isolated pass (call 0.51s). Fixture teardown still burns **5.01s** waiting for SIGTERM (`_SHUTDOWN_TIMEOUT_SECONDS`)                                     |

## 1. Complete Services-tab `x` vs `!x` routing

`plan:202609/service_host_1.md` (services-tab phase): **`x` start/stop the selected
proc**, **`!x` host start/stop**. The host status line is chrome, not a node. Nested
Scheduler rows are not service procs.

`x` on the Services tab is `kill_agent` → `_toggle_or_kill_axe_view`. `!x` is bang-mode
`toggle_axe` → `_toggle_axe_global`. Today `_toggle_axe_global` on the axe/Services tab
**delegates to `_toggle_or_kill_axe_view`**, so `x` and `!x` are the same key on that
tab.

`91b67672b4` made nested chop/lumberjack rows no-op inside `_toggle_or_kill_axe_view`,
which is why the identified test now passes. It left the empty-selection fallback as
host start/stop, so:

- `x` with no selected service proc still starts the host (the original `485a6082e1`
  test body, before that commit narrowed it).
- `!x` while a service proc is selected toggles the **proc**, not the host.

### Production change (`src/sase/ace/tui/actions/axe.py`)

- `_toggle_or_kill_axe_view` (`x` on Services, and bgcmd kill):
  - view `"axe"` + `_axe_service_selection` set → `_toggle_selected_service_proc`
  - view `"axe"` otherwise (nested Scheduler, empty selection, host chrome) → **return
    without starting or stopping the host**
  - numeric bgcmd view → existing `_confirm_kill_bgcmd`
- `_toggle_axe_global` (`!x`):
  - on the axe/Services tab, **always** start/stop the service host (or legacy axe when
    `_service_host_enabled` is false), even if a proc or nested Scheduler row is
    selected
  - other tabs keep the existing start/stop/selector logic

The nested chop/lumberjack `getattr` early returns become redundant once non-proc
`"axe"` views no-op; delete them rather than leaving a second policy.

### Tests (`tests/ace/tui/actions/test_service_host_keys.py`)

Keep the existing three tests and add the cases the original contract needed:

- `x` with `_axe_service_selection is None` and no chop/lumberjack → `[]` (restore the
  `485a6082e1` body as its own test).
- `x` on nested Scheduler rows still `[]`.
- `!x` with a selected service proc still starts/stops the **host**, not the proc.
- `!x` on nested Scheduler rows still starts/stops the host.

## 2. Make the TUI import-budget timing load-stable (`sase-136`)

`tests/ace/tui/test_app_import_budget.py` already hard-asserts `deferred_modules == []`
and `module_count < 3290` inside the subprocess helper. Those are the real import-graph
contract from `264eedc6c4`.

The flake is the **wall-clock** `time.perf_counter()` budget of 5.0s. Under
`just test-cost` (6 xdist workers, 43383 items) the same subprocess took 32.7s; isolated
it is ~2.8s. Three retries (`8989d0a724`) still fail when every attempt is scheduled
against a pegged machine. Widening the wall-clock budget would hide a real import-graph
regression; skipping the elapsed check under xdist would disable it in the only lane
that runs the full suite.

### Change

Inside the measurement subprocess, record `time.process_time()` (process CPU seconds,
excluding sleep and scheduler delay) instead of `perf_counter()`. Keep the 5.0s CPU
budget, the module-count cap, and the deferred-module denylist. One retry is enough; do
not add sleeps. Do not raise `_MAX_MODULE_COUNT`.

`process_time()` is process-wide CPU of the import subprocess, so a machine that is
merely busy no longer fails the canary, while an import graph that actually does several
extra CPU-seconds of work still fails.

## 3. Real-gateway enroll/hello flake (`sase-13h`)

Enrollment succeeded (`quarantined is False`, credential present) and then
`assert status.state == "ok"` failed with `"error"`. The assertion did not print
`status.message`, so the parallel-lane cause is still one of:

- `hello failed: transport_failed` / timeout (`FleetGatewayClient` default 5.0s) under
  CPU contention
- `machine alias is not configured` / `local credential ref is missing` (config
  isolation; fail immediately, do not retry)

Isolated teardown of this fixture is 5.01s: `real_gateway` uses
`start_new_session=True`, `stdout/stderr=DEVNULL`, `terminate()`, then waits
`_SHUTDOWN_TIMEOUT_SECONDS = 5.0` before `kill()`. `sase_gateway` is the
`sase_core_rs.gateway` binding and does not exit on SIGTERM in time. That is
independently broken and will pile processes during a 6-worker cost lane.

### Fixture (`tests/dispatch/real_gateway_fixture.py`)

- On teardown, kill the process group (`os.killpg(proc.pid, signal.SIGKILL)` with
  `ProcessLookupError` ignored), then `wait`. Do not spend 5s on SIGTERM first.
- Capture gateway stdout/stderr to files under the fixture `tmp_path` so a failed hello
  can dump them.

### Test (`tests/dispatch/test_machine_bootstrap_real_gateway.py`)

- Always include `status.message` in the assertion.
- Retry **only** when `state == "error"` and `message` starts with `hello failed:`
  (transport/timeout after a successful enroll). Bound the retry window (~2s, short
  sleeps tagged `# sase-test-wait: ...`).
- Do **not** retry alias-missing or credential-missing errors; those are isolation bugs
  and must fail with the message.
- Do not change product `MachineService.status` retry policy. Isolated pass shows
  enrollment/hello work when the machine is idle.

## Verification (mandatory)

Do **not** run `just check-full`.

1. Focused pytest (serial):

   ```bash
   .venv/bin/python -m pytest \
     tests/ace/tui/actions/test_service_host_keys.py \
     tests/ace/tui/test_app_import_budget.py \
     tests/dispatch/test_machine_bootstrap_real_gateway.py \
     -q --tb=short
   ```

2. Contention smoke **without** the full suite: run the import-budget and real-gateway
   nodes under xdist (`-n 4` or `tools/run_pytest` if it accepts an explicit file list).
   Do not take this as a substitute for (1) or (3).

3. `just check` (scoped lane). Expected to select the files above. If scoped gear
   escalates, do not follow it into `check-full`; re-run the focused pytest and report
   the escalation.

4. After those pass:
   - Close `sase-136` and `sase-13h` with notes pointing at the commits and the
     verification commands.
   - Note on `sase-11y.7` (and epic `sase-11y` if you already left a DISCOVERED ISSUE
     there) that the `x` vs `!x` routing issue is fixed; **leave the phase open**.
   - Leave `sase-11l.11` in progress for its land agent.

## Non-goals

Do not "fix" the nested-row test by further narrowing it. Do not mark these nodes `slow`
or exclude them from the cost lane. Do not raise the 5.0s import CPU budget or the 3290
module cap to paper over a real graph regression.
