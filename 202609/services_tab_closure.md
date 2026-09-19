---
tier: tale
title: Close the remaining Services tab contract
goal:
  Finish the already-landed Services tab so host chrome, service-health footer, gear
  exclusion, idle refresh, quit copy, and epic-symbol ownership match the sase-11y.7
  phase contract, then close only that phase.
size: medium
proposed_by: bbugyi200.athena.sase-11y.7
bead: sase-11y.7
create_time: 2026-09-19 06:25:37
status: wip
---

- **PARENT:**
  [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:**
  [sase-11y.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.7.md)

# Plan: Close the remaining Services tab contract

## Outcome

`sase-11y.7` already has a landed first pass (`plan:202609/services_tab.md`, commit
`c2befdb`): the tab label is Services, `--tab services` aliases canonical `axe`,
`--no-service`/`--restart-service` exist, configured daemon procs render with the
scheduler's routine/job tree nested under it, and CLI/TUI share
`src/sase/service/actions.py`. This tale does **not** rebuild that surface. It closes
the phase by finishing the remaining user-visible contract, the idle-refresh probe, the
gear miscount, and the leftover `sase-11y.7(...)` Symvision whitelist.

When this tale is done, with `service_host` enabled:

- The Services top chrome always names the host (running with uptime/platform unit, or
  down with the `!x` start hint). Host chrome is not a selectable node.
- The footer health pill is `SVC <running>/<desired>` when healthy and a persistent loud
  `SVC !` when the host or any desired proc is unhealthy. Disabled procs do not inflate
  the denominator. Failed procs stay visually loud without a toast every countdown tick.
- Gear chips exclude monitor rows **and** rows carrying the wire `service` marker, so a
  live sessionless Telegram/service receiver cannot inflate the session gear.
- Disabled and unavailable service-proc rows show provenance / reason inline
  (`disabled here`, layer summary, `unavailable: …`), using `ServiceEnablement` as a
  real typed consumer.
- The `Q` quit modal says "stop Scheduler", routes that option through
  `stop_service_proc("scheduler", ...)`, and never stops the service host.
- Idle auto-refresh wakes when `~/.sase/service/status.json` or `state.json` change.
- `sase bead epic-symbols sase-11y.7` reports no leftovers, and only `sase-11y.7` is
  closed.

When `service_host` is disabled, keep the current AXE runtime, footer RUNNING/STOPPED
grammar, and quit-and-stop-axe path so sunset (`sase-11y.10`) still has one Off branch
to delete.

## Boundaries

- Keep internal tab id `axe`. Do not regenerate the ~667 snapshots that a tab-id rename
  would force; that belongs to `sase-11y.10`.
- Do not migrate `!` background-command slots. Keep the existing `── commands ──`
  section; `sase-11y.8` owns durable oneshots.
- Do not add or remove feature flags. Cover both `service_host` branches in tests.
- Do not reconstruct desired state in Textual. `ServiceStatusSnapshot` remains the TUI
  read model. Render, navigation, and footer paths stay free of sync disk, JSON, config,
  and subprocess work (`tui_perf.md`).
- Do not import leftover public facades solely to satisfy Symvision. Consume
  `ServiceEnablement` by displaying it; consume `service_dir` / `service_state_path` by
  probing them; re-key everything else.
- Do not close `sase-11y` or any ancestor. Record out-of-scope discoveries as
  `PROPOSED FOLLOW-UP:` notes on `sase-11y.7`.

## Already landed (do not redo)

Treat these as given and extend them:

- Tab label `Services` in `src/sase/ace/tui/widgets/tab_bar.py`; `services` alias in
  `tab_order.py`; `--no-service` / `--restart-service` in `parser_ace.py`.
- `ServiceProcItem` tree and scheduler nesting in `widgets/bgcmd_list.py` and
  `actions/axe_display/_loader_items.py`.
- Off-thread snapshot + selected-proc log tail in `_data.py` / `_loader_refresh.py`.
- Shared adapter `src/sase/service/actions.py` used by CLI and TUI.
- Keys: `x` toggles the selected proc, `!x` toggles the host, `!e` toggles machine
  enablement (`default_config.yml` bang_mode `toggle_service_enablement: "e"`).
- Detail pane already prints `proc.enablement.summary` and `unavailable_reason`
  (`widgets/_axe_dashboard_output.py`).
- `ace.procs.default_query` defaults to `-service`; `ProcsSessionState.query` seeds from
  that on first construction.
- Proc query dialect already has `service` / `svc:<name>` (`_proc_query.py`).

## Implementation

### 1. Host chrome on AxeInfoPanel

Evolve `src/sase/ace/tui/widgets/axe_info_panel.py` so the Services top bar **always**
renders host chrome when `_service_host_enabled` is true, independent of the selected
row:

- running: `Services · host ● running <uptime> · <platform_unit or "detached">`
- down: `Services · host ○ stopped · press !x to start` (use the configured `!x`
  display, not a hardcoded chord if the keymap helper is already in reach)
- loading remains `Services …`

Selected-row identity (Scheduler / proc / routine / job / bgcmd) may follow on the same
line; it must not replace the host clause. Pass `ServiceStatusHost` from the cached
snapshot already on the mixin. Keep formatting pure and disk-free.

### 2. Replace the footer AXE pill with service health

Today `widgets/_keybinding_status.py` still paints `SVC` + `RUNNING`/`STOPPED` and a
separate `[⚙n/m]` badge, and `_get_service_proc_counts()` in `_render.py` uses
`len(snapshot.procs)` as the denominator (disabled procs inflate it).

When the flag is on, derive health from the cached snapshot only:

- denominator = configured procs that are enabled and desired-running (or
  enabled-but-unavailable)
- numerator = those currently `state == "running"`
- host not running, or any counted proc failed/stopped/unavailable → loud `SVC !` plus a
  concise summary
- otherwise `SVC <running>/<desired>`
- disabled procs are omitted from both counts
- keep the startup stopwatch and bgcmd `*`/`✓` badges
- notify on health/change-token **transitions** only (store last signature on the mixin;
  countdown ticks must not toast)

Keep the current RUNNING/STOPPED grammar in the flag-off branch. Update
`tests/test_keybinding_footer_status.py` and
`tests/ace/tui/widgets/test_keybinding_footer_stopwatch.py` for healthy, host-down,
failed-proc, disabled-omitted, and transition-dedup cases.

Tab bar color for `axe` is still `#FF5F5F` (AXE failure-red). Change it to the existing
service teal `#00D7AF` used by service-proc rows.

### 3. Gear exclusion and Procs query persistence

`ObservedProc.service` already exists. Gear counting still does
`active_count - active_monitor_count` in `_proc_action_observer.py` and
`actions/lifecycle.py`, which is the sessionless-receiver miscount.

Add `is_gear_eligible_row` / `gear_eligible_count` on
`src/sase/ace/tui/_proc_observer_models.py`: an active row counts only when it is
neither a monitor shell nor `row.service is not None`. Use that helper from both gear
callers. Leave the Procs pane inventory and full projection unchanged.

Add the regression: a live sessionless Telegram/service receiver next to an ordinary
proc and a monitor does not increment the blue gear.

For `ProcsSessionState`, keep seeding `query` from `ace.procs.default_query` on first
construction. Add an explicit `query_initialized` (or equivalent) marker so a committed
empty string stays empty across pane close/reopen, a non-string config value becomes
`""` without crashing, and an explicit empty configured default stays empty. Cover this
in `tests/ace/tui/test_procs_pane_filter.py`. Do not reimplement `_proc_query.py`.

### 4. Inline provenance and a real ServiceEnablement consumer

Tree chips in `bgcmd_list.py` currently say `disabled` / `unavailable` without
provenance. Extract a tiny disk-free formatter that takes `ServiceEnablement` (import
the class from `sase.service.status`) and returns the chip/label the sketch uses —
`disabled here`, the enablement `summary`, or the unavailable reason. Use it from the
tree chip and keep the detail pane on `enablement.summary`. That import is the non-test
consumer that lets us drop `--epic-symbol "sase-11y.7(ServiceEnablement)"`. Do not
import `ServiceFieldProvenance`, `compose_service_config`, or
`resolve_service_enablement` here; the snapshot already carries effective enablement.

### 5. Quit modal stops Scheduler, never the host

`QuitOptionsModal` still says "Quit & Stop axe" and `AxeMixin._stop_axe_and_quit`
currently no-ops the daemon stop when the flag is on (so it stops neither host nor
scheduler).

When the flag is on:

- user-facing copy becomes "stop Scheduler" (help, modal title, footer hint)
- option `quit_stop_axe` (keep the internal choice id) calls
  `stop_service_proc("scheduler", actor="tui", reason="ace quit")` then exits
- it must not call `stop_service_host`

When the flag is off, keep the current axe-daemon stop. Extend
`tests/ace/tui/actions/test_axe_stop_quit.py` and
`tests/ace/tui/actions/test_service_host_keys.py` for both branches. Keep service
mutations on the existing Textual thread-worker path already used by host start/stop;
recapture the selected service key after completion; do not add a second dispatcher.

### 6. Idle token probe includes service status/state

`actions/event_refresh/_surface_tokens.py` still probes only `axe_state_dir()`. Host
reconciliation writes `~/.sase/service/status.json` (and state mutations write
`state.json`), so an idle TUI can miss them.

Extend `SurfaceTokenRoots` / `live_surface_token_roots()` / `_probe_axe_token` (or an
adjacent helper on the same axe surface) to stat:

- `service_dir()` (so the directory's appearance is observable)
- `service_state_path()`
- `service_status_path()` (already used elsewhere; include it in the same probe)

Stat metadata only — no JSON reads on the probe path. This is the real non-test consumer
of `service_dir` and `service_state_path`. Add a token-probe unit test that flips those
files and reports drift, and a flag-off test that still functions if the service dir is
absent.

### 7. Re-key leftover epic-symbols; do not invent consumers

After steps 4 and 6, drop from `Justfile`:

- `sase-11y.7(ServiceEnablement)`
- `sase-11y.7(service_dir)`
- `sase-11y.7(service_state_path)`

Re-key the platform/env symbols this phase does not own to the still-open platform-units
phase `sase-11y.5`:

- `CapturedServiceEnvironment`, `ServiceEnvironmentError`, `read_service_environment`
- `NativeInspection`, `NativeServiceDefinition`, `ServicePlatformApplyResult`
- `build_native_definition`, `inspect_native_service`, `readiness_warnings`
- `service_platform_supported`

Re-key the remaining public facades that the TUI must not call (snapshot is the read
model; enable/disable already goes through `set_service_enablement`) to the parent epic
`sase-11y`:

- `ServiceFieldProvenance` — field-level config provenance, not tab chrome
- `compose_service_config` — `load_service_config()` is the public loader
- `resolve_service_enablement` — snapshot already carries effective enablement
- `clear_service_enablement` — reset-to-config-default is not a Services-tab key

Document the retarget in the Justfile diff (short comment above the re-keyed block is
enough). Do not privatize symbols that `sase-11y.5` is currently landing; that collides
with the in-progress platform-units worker.

Immediately before close, `sase bead epic-symbols sase-11y.7` must be empty. If
Symvision then reports a newly unused public name this phase created, follow
`sase/memory/symvision.md`: delete, privatize, or give it a real non-test consumer —
never a test-only import.

## Verification and closure

1. Focused tests: service health footer (including disabled-omitted and transition
   dedup), host chrome, gear exclusion with a sessionless service receiver, Procs
   default-query / cleared-query, token-probe drift, quit-with-Scheduler vs flag-off axe
   stop, enablement chip formatter, and existing service action/key tests.
2. `just fix-tui-screenshots` for the Services-label / tree / footer / host-chrome
   goldens. Inspect actual/expected/diff before accepting; do not change the canonical
   tab id in snapshots. `just check` does not run PNG snapshots.
3. Exercise both flag states: `--tab services` and `--tab axe`, both startup-flag
   spellings, host-down chrome, disabled/unavailable procs, Scheduler child
   folding/actions, quit-with-Scheduler, footer degradation, user-cleared Procs query.
4. Navigation benchmark under a restart-storm of status snapshots (`SASE_TUI_PERF=1` /
   `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` as needed). Record p95 < 16 ms on
   the Services/axe tab. Confirm render/nav callbacks do no sync disk/config/ subprocess
   work.
5. `just fix`, then `just check` (monitor if long). `just install` first if the
   workspace venv is stale. Do not run `just check-full` unless scoped selection
   escalates; this is one phase, not the epic land.
6. `sase bead epic-symbols sase-11y.7` empty. Out-of-scope discoveries only as
   `PROPOSED FOLLOW-UP:` notes on `sase-11y.7`.
7. Close only this phase:

   ```text
   sase bead close sase-11y.7 --note "<tests, visual inspection, p95, just check, empty epic-symbols>"
   ```
