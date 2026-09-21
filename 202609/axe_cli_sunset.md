---
tier: epic
title: Retire the AXE watchdogs and alias sase axe to sase scheduler
goal: 'The sase service host is the only thing that starts, restarts, or stops the
  scheduler from a CLI path: the ensure watchdog and its systemd timer are gone, no
  agent wait heals axe, the axe-start systemd scope wrapper and its doctor check are
  gone, `sase update` / `sase flag` / `sase plugin` restart the `scheduler` service
  proc instead of the AXE daemon, `sase axe start|stop|restart|status` is a documented
  alias of `sase scheduler`, and the direct AXE restart machinery that those paths
  kept alive is deleted.

  '
phases:
- id: ensure-watchdog
  title: Delete the axe ensure watchdog and its healing notifications
  depends_on: []
  size: medium
  description: 'ensure-watchdog: delete `sase axe ensure`, the `ensure.py` / `_ensure_timer.py`
    / `_ensure_runtime.py` modules, the ensure lock the stop path waits on, the opportunistic
    heal on agent waits, and the three healing notification senders that lose their
    only producer.'
- id: systemd-scope
  title: Delete the axe-start systemd scope wrapper and its evidence
  depends_on:
  - ensure-watchdog
  size: medium
  description: 'systemd-scope: delete `sase/axe/systemd_scope.py`, the `_allow_systemd_scope`
    wrapping and unwrapped-retry branch in `_process_start.py`, the `axe.systemd_scope`
    doctor check, and the `orchestrator_session_scope` issue the status collector
    appends.'
- id: update-restart
  title: Route the post-update restart through the scheduler service proc
  depends_on: []
  size: medium
  description: 'update-restart: make `restart_after_update` call `restart_service_proc("scheduler",
    ...)`, rename the axe-shaped injection types, drop the AXE-only `pid` / `attempts`
    / `verified` fields from `RestartInfo`, and update every `sase update`, `sase
    flag`, and `sase plugin` caller.'
- id: axe-alias
  title: Make sase axe lifecycle verbs an alias of sase scheduler
  depends_on:
  - ensure-watchdog
  - systemd-scope
  - update-restart
  size: medium
  description: 'axe-alias: retarget `sase axe start|stop|restart|status` at `handle_scheduler_command`,
    share the scheduler parser''s option builders, retarget the detached daemon command
    at `sase scheduler run`, and say the alias out loud in both commands'' help.'
- id: dead-supervisors
  title: Delete the AXE restart machinery the alias orphaned
  depends_on:
  - axe-alias
  size: medium
  description: 'dead-supervisors: delete `_process_restart.py`, `_restart_events.py`,
    `restart_render.py`, and `status_render.py` once the alias removes their last
    callers, and prune the `sase.axe.process` / `sase.axe` re-export surface that
    kept a second supervisor reachable.'
proposed_by: bbugyi200.athena.sase-11y.10.1.3
parent_bead: sase-11y.10.1.3
create_time: 2026-09-20 21:17:49
status: done
bead_id: sase-11y.10.1.3.1
---

- **PROMPT:** [prompts/202609/axe_cli_sunset.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/axe_cli_sunset.md)
- **PARENT:** [202609/service_host_sunset.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_sunset.md)
- **BEAD:** [sase-11y.10.1.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.10.1.3.1.md)

# Plan: Retire the AXE watchdogs and alias sase axe to sase scheduler

This is the `axe-cli` phase of the service-host sunset epic, broken into phases of its
own. `flag-removal` (sase-11y.10.1.2) has landed: the `service_host` flag is gone,
`handle_scheduler_command` dispatches straight to `_handle_service_scheduler`, and the
TUI's direct start/stop path is deleted. What is left are the CLI paths that still
start, restart, or stop the AXE orchestrator behind the service host's back.

Four still exist today, and this plan deletes all four:

1. `sase axe ensure` — a second healer with an optional systemd timer, plus an
   opportunistic `ensure_axe()` on every agent wait
   (`src/sase/axe/run_agent_wait.py:258`).
2. `wrap_axe_start_in_systemd_scope` — a cgroup-escape wrapper that exists only because
   a TUI-launched orchestrator used to need to outlive its pane.
3. `restart_after_update` — `sase update`, `sase flag enable|disable`,
   `sase plugin install|uninstall|update`, and the PluginsRequired gate all funnel
   through it, and it calls `restart_axe_daemon_result` directly.
4. `sase axe start|stop|restart|status` — still drives `sase/axe/process.py`.

Once those land, `restart_axe_daemon_result` and the live restart renderer have no
caller left, so a fifth phase deletes them rather than leaving a second supervisor
reachable from `sase.axe`.

Nothing here adds a feature and no phase creates a feature flag: every branch removed is
already dead on both live machines.

## Evidence this plan is built on

Consumer evidence gathered on the current tree (`98bf83a38`). It is the reason each
deletion below is safe and each surviving symbol is named as surviving:

| symbol                                                                        | surviving non-test consumer                                                                                                                          | verdict                                                                      |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `canonical_axe_start_command`                                                 | `src/sase/service/executable.py:141` (`ExecStart` resolution)                                                                                        | **keep**                                                                     |
| `stop_axe_daemon_result`                                                      | `src/sase/service/host_support.py:189`                                                                                                               | **keep**                                                                     |
| `start_axe_daemon_result`                                                     | `_process_start.py` self-recursion; `start_axe_daemon` ← `src/sase/integrations/chat_install.py:377`                                                 | **keep**                                                                     |
| `collect_axe_status_snapshot`                                                 | `src/sase/doctor/checks_deep_axe.py:29`                                                                                                              | **keep**                                                                     |
| `should_reexec_axe_start_from_canonical`                                      | only `axe_handler._handle_start`                                                                                                                     | delete in `axe-alias`                                                        |
| `restart_axe_daemon_result`                                                   | `update_handler`, `feature_flags/cli_set`, `plugins/cli_{install,uninstall,update}`, `plugins/_required_gate_actions`, `axe_handler._handle_restart` | all removed by `update-restart` + `axe-alias` → delete in `dead-supervisors` |
| `render_axe_status_human` / `render_axe_status_json`                          | only `axe_handler._handle_status`                                                                                                                    | delete in `dead-supervisors`                                                 |
| `restart_render.py` (whole module)                                            | only `axe_handler._handle_restart`                                                                                                                   | delete in `dead-supervisors`                                                 |
| `notify_axe_healed` / `notify_axe_ensure_failed` / `notify_axe_restart_storm` | only `sase/axe/ensure.py` + `_ensure_runtime.py`                                                                                                     | delete in `ensure-watchdog`                                                  |
| `notify_axe_lock_recovered`                                                   | `_process_start._notify_wedged_lock_recovery`                                                                                                        | **keep**                                                                     |
| `acquire_axe_ensure_lock` / `release_axe_ensure_lock`                         | `ensure_axe` and `_process_stop.py:75` only                                                                                                          | delete in `ensure-watchdog`                                                  |

`sase/repos/linked/` holds only `sase-core` (Rust); no linked Python repo imports the
`sase.axe` process surface, and no symvision URI pragma points at one of these symbols.

## Two decisions this plan makes, and why

### `sase axe status` becomes `sase scheduler status`

The parent plan lists `axe_handler.py:205-218` — `_handle_status` — in the retarget set,
so all four verbs delegate. `flag-removal` already decided what `sase scheduler status`
prints: the legacy branch that rendered the whole-system AXE snapshot (`_legacy_status`)
was deleted with the flag, and the surviving arm is `handle_service_proc_show`. An alias
whose `status` behaves differently from the command it aliases is not an alias, so
`sase axe status` adopts the service-proc view and `src/sase/axe/status_render.py` loses
its last consumer.

That deletes the only CLI surface for the rich snapshot (routine tree, runner occupancy,
maintenance, lifecycle events, classified issues). The _data_ survives — `sase doctor`
still collects it through `checks_deep_axe.py` — but the view does not.
`dead-supervisors` records a `PROPOSED FOLLOW-UP:` on the phase bead so the epic's land
agent can decide whether to reinstate it (for example as
`sase scheduler status --full`). Reinstating it is out of scope here: this plan deletes,
it does not add.

### The detached daemon command becomes `sase scheduler run`

`_build_axe_start_command` (`src/sase/axe/_process_start.py:381`) spawns
`sase axe start`, and `sase axe start` is today the _foreground_ orchestrator. The
moment `sase axe start` delegates to `start_service_proc`, that spawn would ask the host
to start a proc and return immediately instead of becoming the orchestrator, and
`start_axe_daemon_result` would hang until its 15s PID-file timeout.

`sase scheduler run` is already the foreground orchestrator — it is exactly what the
service host execs (`src/sase/service/host_support.py:36`) — so `axe-alias` retargets
the spawn at it. That keeps the one remaining direct-start consumer (`chat_install.py`,
which is out of this epic's stated scope) working unchanged.

## Cross-cutting rules for every phase

- Never leave two owners for one child. If a phase deletes the last caller of a
  supervision helper, delete the helper and its tests too rather than leaving it
  reachable.
- **Docs are out of scope.** `docs/axe.md`, `docs/cli.md`, `docs/configuration.md`, and
  `docs/remote_dispatch.md` still describe `sase axe ensure` and the scope check; the
  epic's `docs` phase (sase-11y.10.1.5) owns them and is blocked on this bead. No lint
  gate cross-checks docs against the CLI, so leaving them stale does not redden
  `just check`. Do not pre-empt that phase.
- Follow the `cli_rules` memory for every CLI change: alphabetical subcommands and
  options, short aliases, no required options, excellent `-h`.
- Symvision deletions follow the `symvision` memory's hierarchy: delete dead symbols and
  their tests; do not whitelist. Note that `sase/axe/process.py` and
  `sase/axe/__init__.py` re-export several symbols, which keeps symvision quiet about
  them — so a symbol being green under symvision is **not** evidence it has a live
  consumer. Use the evidence table above, then re-grep.
- No phase of this plan needs an `--epic-symbol` entry: no phase deletes a public symbol
  that a later phase still consumes. If one turns out to be needed, key it to
  `sase-11y.10.1` (the parent epic bead) and remove it in the phase that lands the
  consumer — never key it to `sase-11y.10.1.3`, which closes first.
- Do **not** touch `LEGACY_SYSTEMD_UNITS` in `src/sase/service/platform.py:48-52`;
  `sase service init` must keep detecting an already-installed `sase-axe-ensure.timer`.
  A stale timer invoking a deleted subcommand fails loudly and harmlessly.
- Keep `src/sase/detach_scope.py`, `src/sase/procs/spawn.py:143`, and
  `src/sase/monitor/spawn.py:139` exactly as they are.
- No phase changes rendered TUI output, so no phase should need
  `just fix-tui-screenshots`. `update-restart` changes a Rich panel printed by the _CLI_
  (`render_restart_info`), not a Textual widget. If a golden does move, inspect the
  report before applying.
- Run `sase tool run check` before finishing. Do **not** run `just check-full` unless
  explicitly instructed.
- Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on
  `sase-11y.10.1.3`; they do not create beads.

## Phase detail

### ensure-watchdog — Delete the axe ensure watchdog and its healing notifications

With the host reconciling roughly every second and restarting a crashed scheduler
itself, a second healer is the exact race this epic exists to remove.

**Delete outright**

- `src/sase/axe/ensure.py`, `src/sase/axe/_ensure_timer.py`, and
  `src/sase/axe/_ensure_runtime.py`. Every symbol in `_ensure_runtime.py` (the rate
  limit marker, the failure-notification marker, the restart-storm damper, the downtime
  estimate, `published_orchestrator_running`, and the ensure lock) exists only for
  `ensure_axe`.
- The `sase axe ensure` subparser (`src/sase/main/parser_ace.py:184-212`) and its
  `install` / `uninstall` children. Drop `ensure` from the `axe_subparsers` `metavar` at
  `parser_ace.py:157`.
- `_handle_ensure` (`src/sase/main/axe_handler.py:107-143`), the
  `elif axe_sub == "ensure"` arm at `:35-36`, and `ensure` in the usage string at
  `:50-52`.
- `_opportunistic_ensure_axe` (`src/sase/axe/run_agent_wait.py:55-66`) and its call site
  at `:258`. An agent wait must not start a supervisor. Fix the adjacent comment that
  still explains "Fallback resolution and axe healing both allocate"; the
  `release_idle_memory()` trim below it stays.
- `notify_axe_healed`, `notify_axe_ensure_failed`, and `notify_axe_restart_storm`
  (`src/sase/notifications/senders.py:243,292,311`), plus the `notify_axe_healed` /
  `notify_axe_restart_storm` entries in `src/sase/notifications/__init__.py` (import at
  `:44,46` and `__all__` at `:126,128`). Keep `notify_axe_lock_recovered` immediately
  between them — `_process_start` still calls it.
- `tests/test_axe_ensure.py`.

**The ensure lock in the stop path.** `src/sase/axe/_process_stop.py:73-83` acquires
`acquire_axe_ensure_lock(blocking=True)` around `write_desired_state("stopped", ...)` so
an in-flight ensure start cannot overwrite the operator's stop. `ensure_axe` was the
only other holder of that lock, so with it gone the acquisition can never block and
guards nothing. Delete the import, the acquire/release, and the comment; keep the
unconditional `write_desired_state("stopped", source=desired_state_source)` inside the
`if record_desired_state:` block.

**Doctor next-steps.** `src/sase/doctor/checks_axe.py:63` tells the operator to run
`sase axe ensure` when axe is desired-running but down. Replace it with `("Run `sase
scheduler start`.",)`. Leave the rest of `_check_axe_health` alone.

**Strings in soon-to-die modules.** `src/sase/axe/restart_render.py:254` and `:387`
print `sase axe ensure` as recovery advice. `dead-supervisors` deletes that module, but
this phase must leave the tree green and honest, so retarget both strings at
`sase scheduler restart` / `sase scheduler status`.

**Tests to audit.** Grep for `ensure_axe`, `ensure_timer`, `axe_ensure`, and
`"axe ensure"` under `tests/` and fix each hit; the known set is
`tests/main/test_parser_command_help.py:85` (asserts `sase axe ensure install`),
`tests/test_axe_outage_incident_regressions.py`, `tests/test_axe_restart_recovery.py`,
`tests/test_run_agent_wait.py`, `tests/test_run_agent_wait_fallback.py`,
`tests/test_axe_process_start.py`, `tests/test_axe_restart_render.py`,
`tests/test_axe_status_cli.py`, and `tests/doctor/test_checks_axe.py`. Also grep
`tests/` for the deleted notification senders. Delete the cases that assert healing
happens; keep the cases that assert a wait blocks and resumes without it.

Verify with `sase tool run check`.

### systemd-scope — Delete the axe-start systemd scope wrapper and its evidence

`wrap_axe_start_in_systemd_scope` exists so a TUI-launched orchestrator survived its
launching session. The host now owns the scheduler, its cgroup is `sase.service` rather
than a `.scope`, and `src/sase/detach_scope.py` is the sanctioned cgroup-escape helper.
The check cannot fire any more — it is dead, not merely quiet.

- Delete `src/sase/axe/systemd_scope.py`.
- `src/sase/axe/_process_start.py`: drop the import at `:29`, the `_allow_systemd_scope`
  parameter at `:99` and every pass-through (`:150`, `:250`), the wrap at `:185-189`,
  and the whole `if wrapped_in_systemd_scope:` unwrapped-retry branch at `:228-236`
  (including the `record_desired_state=False` recursion at `:232`). After the branch
  goes, the `exit_code is not None` arm returns the plain "exited before publishing a
  PID" failure. Confirm `_wait_for_daemon_start` and the wedged-lock recovery paths are
  untouched.
- `src/sase/doctor/checks_axe.py`: delete the `axe.systemd_scope` `CheckSpec`
  (`:40-45`), `_check_axe_systemd_scope` (`:96-122`), and the import at `:15`. The
  `axe.health` and `axe.jobs` specs stay.
- `src/sase/axe/status_collector.py`: delete the import at `:42` and
  `_with_systemd_scope_issue` (`:131`+), and make `collect_axe_status_snapshot` return
  `classify_axe_status(request)` directly at `:128`. Check whether `AxeStatusIssue`
  becomes an unused import in that module. `orchestrator_session_scope` is composed
  entirely in Python — no Rust wire change is needed in `../sase-core`.
- Tests: `tests/test_axe_process_start.py` (the `disable_systemd_scope_by_default`
  fixture at `:38`, the wrap tests at `:72-108`, and
  `test_retries_unwrapped_when_systemd_scope_exits_before_pid_publish` at `:111`),
  `tests/doctor/test_checks_axe.py:14-21,147-198`,
  `tests/test_axe_status_collector.py:116,293-300`,
  `tests/test_axe_cli_chop_run_contract.py:342`, and
  `tests/main/test_doctor_command.py:228`. Grep for any other doctor check-id inventory
  that lists `axe.systemd_scope`.

Verify with `sase tool run check`.

### update-restart — Route the post-update restart through the scheduler service proc

`restart_after_update` (`src/sase/main/update_restart.py:29`) is the single seam every
post-change restart funnels through: `sase update` (live and mode-switch),
`sase flag enable|disable`, `sase plugin install|uninstall|update|restart`, and the
PluginsRequired gate host effect. All of them inject `restart_axe_daemon_result`, which
restarts the orchestrator while the host also notices it died.

**The seam.** Make `restart_after_update` call
`restart_service_proc("scheduler", actor="cli", reason=source)` from
`sase.service.actions`. That call raises `ServiceProcActionError` or
`ServiceConfigError` rather than returning a failure result, so the `try/except` maps
those to `status="failed"` with the exception text; otherwise the outcome is
`status="restarted"` carrying `outcome.message`. The old success condition
(`result.succeeded and result.pid is not None`) has no analogue and goes away.

**Types.** In `src/sase/main/update_types.py`, rename `AxeRunningFn` →
`SchedulerRunningFn` and `RestartAxeFn` → `RestartSchedulerFn`. `SchedulerRunningFn`
keeps its meaning and its `is_axe_running` default — the scheduler still publishes the
orchestrator PID file. `RestartSchedulerFn` returns the service-proc outcome, not
`AxeStartResult`. That outcome class is `_ServiceProcActionOutcome`
(`src/sase/service/actions.py:26`), and symvision forbids importing a private symbol
across files, so rename it to `ServiceProcActionOutcome`, add it to that module's
`__all__`, and type the alias against it. `update_types.py` is then its cross-file
consumer.

**`RestartInfo`.** Drop `pid`, `attempts`, and `verified`
(`src/sase/main/update_types.py:60,63,64`); a service-proc restart is a request to the
host and has none of them. `attempted`, `status`, `message`, and `reason` stay. Then:

- `src/sase/main/update_json.py:257` — drop the `pid` key from `restart_info_json`.
- `src/sase/dev_update/journal.py:122-142` — drop `pid`, `verified`, and the whole
  `attempts` list from `_restart_record`.
- Bump `UPDATE_JSON_SCHEMA_VERSION` (`src/sase/main/update_types.py:19`) from 3 to 4;
  its docstring says to bump when the `-j` payload shape changes incompatibly.
- Read `src/sase/mode_switch/render.py:78` and every `render_restart_info` caller and
  fix anything that read a removed field.

**Wording.** `restart_skipped`'s `"axe is not running"` reason, the
`"Axe restarted (pid N)"` / `"Failed to restart axe"` messages, the `"Axe Restart"`
panel title and `"Axe restart failed."` fallback in `render_restart_info`
(`update_restart.py:106-118`), and `_AXE_NOT_RUNNING_MESSAGE`
(`src/sase/feature_flags/cli_set.py:39`) all name the scheduler now. Grep `tests/` for
each literal before changing it.

**`_call_restart_axe`.** Its `inspect.signature` probe exists only to tolerate
zero-argument test fakes. The new callee is a local adapter with a known signature, so
delete the helper and call the injected function directly with the attribution keyword;
update the fakes in the tests instead.

**Callers to rewire** (keyword names, parameter types, and defaults):
`src/sase/main/update_handler.py:24-27,69-70`, `update_handler_live.py:192`,
`update_handler_mode_switch.py:118`, `src/sase/feature_flags/cli_set.py:13,51-52,67`,
`src/sase/plugins/cli_install.py:29,83-84`, `cli_uninstall.py:29,71-72`,
`cli_update.py:31,88-89`, `cli_restart.py:5-23`, and
`src/sase/plugins/_required_gate_actions.py:18-25` (whose module docstring also says
"restarts axe"). Each must keep compiling and keep rendering a sensible
`render_restart_info` panel.

**Tests.** `tests/main/test_update_command_{upgrade,dev,mode_switch,completion}.py`,
`tests/feature_flags/test_cli_set.py`, `test_cli_journeys.py`, `test_boundaries.py`,
`tests/test_plugin_cli_{install,uninstall,update}.py`,
`tests/test_plugins_required_gate_actions.py`,
`tests/completion/_update_refresh_harness.py`, `tests/dev_update/test_journal.py`, and
the ACE update tests under `tests/ace/tui/`. Add a case proving a successful
`sase update` requests a `scheduler` service-proc restart and never calls
`restart_axe_daemon_result`.

Verify with `sase tool run check`.

### axe-alias — Make sase axe lifecycle verbs an alias of sase scheduler

`sase axe {job,routine,maintenance}` (with the hidden `chop` / `lumberjack` aliases at
`axe_handler.py:33,37`) manage the scheduler's routine/job tree, not its process
lifetime, and stay. `sase axe bgcmd-launch` is an internal durable entrypoint and stays.
The other four verbs delegate.

**Handler.** In `src/sase/main/axe_handler.py`, replace the `restart` / `start` /
`status` / `stop` arms (`:41-48`) with one arm that sets
`args.scheduler_subcommand = axe_sub` and calls
`sase.main.scheduler_handler.handle_scheduler_command(args)` — which is `NoReturn`, so
it terminates the process exactly as the old arms did. Delete `_handle_start`
(`:254-293`), `_handle_restart` (`:296-351`), `_handle_stop` (`:354-374`), and
`_handle_status` (`:205-218`), together with the module-level
`from sase.axe.process import restart_axe_daemon_result` (`:11`) and
`from sase.main.update_types import RestartAxeFn` (`:12`) imports. Keep
`load_axe_config_with_overrides` — `scheduler_handler._run_foreground_scheduler` calls
it. Update the usage string at `:50-52` to
`sase axe {job,maintenance,restart,routine,start,status,stop}`.

**Parser.** In `src/sase/main/parser_ace.py`, the four verbs must accept exactly what
`sase scheduler` accepts, so import and reuse `_add_scheduler_overrides` and
`_add_json_flag` from `src/sase/main/parser_scheduler.py` instead of re-declaring the
options; promote them to public names there so the two parsers cannot drift. Concretely:

- `axe stop` loses `-f/--force` (`:333-338`) — the legacy orphan sweep no longer runs.
- `axe restart` loses `-t/--verify-timeout` (`:269-276`) — there is no heartbeat
  verification on the service path.
- `axe restart` and `axe start` take the shared overrides plus, for `restart`, the
  shared `-j/--json`; `axe status` takes the shared `-j/--json`.
- Keep every option and subcommand alphabetical.

**Help text.** The `axe` parser's help becomes "Alias of `sase scheduler`, plus the
routine and job tree" (`parser_ace.py:141-143`), and `register_scheduler_parser`'s
description (`src/sase/main/parser_scheduler.py:13-17`) names `sase axe` as the accepted
former name. `--no-service`/`--no-axe` and `--restart-service`/`--restart-axe`
(`parser_ace.py:79-85,124-130`) already carry both spellings — leave the aliases and the
`no_axe` / `restart_axe` dests alone; only confirm their help names the service host.

**The detached daemon command.** Retarget `_build_axe_start_command`
(`src/sase/axe/_process_start.py:381-394`) from `sase axe start` to
`sase scheduler run`, keeping the same `--max-hook-runners` / `--max-agent-runners` /
`--zombie-timeout` / `-q` spellings, which `_add_scheduler_overrides` already accepts.
This is required, not cosmetic: see "Two decisions this plan makes" above. Desired-state
recording is unaffected — `start_axe_daemon_result` already writes
`write_desired_state("running", ...)` in the parent at `:118-119`, and the host-run
scheduler never wrote it.

**Now-dead start helpers.** `should_reexec_axe_start_from_canonical`
(`_process_start.py:361-366`) had exactly one caller, `_handle_start`. Delete it and
remove it from `src/sase/axe/process.py:11,64`. Keep `_running_from_ephemeral_workspace`
— `_build_axe_start_command` still uses it — and keep `canonical_axe_start_command`,
which `src/sase/service/executable.py:141` needs for `ExecStart`. `AXE_START_SOURCE_ENV`
(`_process_start.py:32`) is set into the daemon env at `:66` and read only by the
deleted `_handle_start` (`axe_handler.py:259,277`); confirm with a fresh grep, then
delete the constant and its use in `_compose_axe_daemon_env`.

**Tests.** `tests/test_axe_restart_cli.py`, `tests/test_axe_status_cli.py`,
`tests/main/test_parser_command_help.py`, `tests/main/test_parser_machine.py`,
`tests/main/test_parser_namespace_migrations.py`, `tests/test_axe_process_start.py`, and
`tests/test_axe_cli_chop_run_contract.py`. Add a test asserting that
`sase axe start|stop|restart|status` and the matching `sase scheduler` verb reach the
same handler with the same namespace, and one asserting the detached daemon argv is
`... scheduler run ...`.

Verify with `sase tool run check`.

### dead-supervisors — Delete the AXE restart machinery the alias orphaned

Re-run the consumer greps first; only delete what the greps confirm. Remember that
`process.py` and `sase/axe/__init__.py` re-exports keep symvision quiet, so symvision
passing is not evidence of a live consumer.

- `src/sase/axe/restart_render.py` and `tests/test_axe_restart_render.py` — the live and
  plain restart renderers existed only for `sase axe restart`.
- `src/sase/axe/status_render.py` — `render_axe_status_human` / `render_axe_status_json`
  existed only for `sase axe status`. Update `tests/test_axe_status_cli.py` and the
  `sase.axe.status_render` import in `tests/test_axe_cli_chop_run_contract.py:37`.
  `status_collector.py` and `collect_axe_status_snapshot` **stay**;
  `src/sase/doctor/checks_deep_axe.py:29` consumes the snapshot.
- `src/sase/axe/_process_restart.py` and `src/sase/axe/_restart_events.py` —
  `restart_axe_daemon_result` and `restart_axe_daemon` have no caller left outside the
  re-export chain. Leaving them reachable is exactly the "second supervisor" this epic
  removes.
- Prune the re-export surface: `restart_axe_daemon`, `restart_axe_daemon_result`, and
  every `_restart_events` name in `src/sase/axe/process.py:8,26-38,42,46-56,62-63`; the
  `restart_axe_daemon` lazy-export entry, `__all__` entry, and `TYPE_CHECKING` import in
  `src/sase/axe/__init__.py:54,158,251`. Check whether `AxeStartAttempt`
  (`_process_types.py:17`) and `AxeStartResult.attempts` / `.verified` still have a
  producer once `_process_restart.py` is gone; `update-restart` already removed their
  last non-AXE consumer. Delete only what the grep confirms dead, and keep
  `AxeStartResult`, `AxeStopResult`, `StartStatus`, and `AxeOrchestratorProbe`, which
  the surviving start/stop/probe path uses.
- Run `just _lint-symvision` on its own after each deletion group and re-check every
  reported symbol individually before removing it.

Finally, record on `sase-11y.10.1.3`:

```
sase bead note sase-11y.10.1.3 'PROPOSED FOLLOW-UP: the whole-system AXE status view lost its CLI home — sase axe status now aliases sase scheduler status (service-proc show), so render_axe_status_human/json were deleted; collect_axe_status_snapshot still feeds sase doctor checks_deep_axe, so consider reinstating the rich view as sase scheduler status --full.'
sase bead note sase-11y.10.1.3 'PROPOSED FOLLOW-UP: sase/integrations/chat_install.py still starts the AXE orchestrator directly via start_axe_daemon after a chat-driven update — a sixth direct-start path the service_host_sunset plan did not enumerate; route it through start_service_proc("scheduler") or delete it.'
```

Verify with `sase tool run check`.

## Risks

- **Over-deletion in `dead-supervisors`.** `sase/axe/process.py` and `_process_start.py`
  hold logic the service host still needs. A deletion that breaks
  `canonical_axe_start_command` breaks `ExecStart` resolution and therefore boot on both
  live machines. The evidence table above names every symbol that must survive; re-grep
  before each deletion rather than trusting the table.
- **`_build_axe_start_command` retarget.** If `axe-alias` delegates `sase axe start`
  without retargeting the spawn, `start_axe_daemon_result` silently regresses to a 15s
  timeout and `chat_install`'s post-update restart stops working. The two changes must
  land together in that phase.
- **The ensure lock.** `_process_stop.py` takes it _blocking_. Deleting the acquire
  while leaving a caller that still takes it would deadlock a stop. Delete both sides in
  the same phase.
- **Test fakes.** `update-restart` changes the injected callable's signature and return
  type. Zero-argument fakes currently survive on `_call_restart_axe`'s introspection;
  once that is gone they fail loudly. That is the intent — fix each fake rather than
  reintroducing the probe.
