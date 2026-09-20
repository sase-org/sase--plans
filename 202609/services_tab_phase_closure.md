---
tier: tale
title: Finish and close the Services tab phase
size: medium
bead: sase-11y.7
goal: Finish the remaining sase-11y.7 Services-tab contract — always-on host chrome,
  a real service-health footer pill, service-aware gear counting, inline enablement
  provenance, a quit modal that stops Scheduler instead of the host, a service-aware
  idle refresh probe — then clear the phase's Symvision whitelist and close only sase-11y.7.
proposed_by: bbugyi200.athena.sase-11y.7
status: done
---

- **BEAD:**
  [sase-11y.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.7.md)

# Plan: Finish and close the Services tab phase

## Outcome

`sase-11y.7` ("Services tab in the TUI") has a landed first pass from
`plan:202609/services_tab.md` (commit `c2befdb`) and a landing-gate key-routing repair
from `plan:202609/landing_gate_test_failures_1.md` (commit `7b4cd80f`). A follow-up
tale, `plan:202609/services_tab_closure.md`, was approved but never implemented — its
worker did not land any of its seven steps. This tale re-states that remaining work
against the tree as it actually stands today, corrects the parts of that plan that went
stale, finishes the phase contract, and closes `sase-11y.7`.

When this tale is done, with the `service_host` flag enabled:

- The Services top bar **always** names the host — running with uptime and platform
  unit, or down with the configured `!x` start hint — independent of which row is
  selected. Host chrome is chrome, never a selectable node.
- The footer pill reads `SVC <running>/<desired>` when healthy and a persistent loud
  `SVC !` when the host or any counted proc is unhealthy. Disabled procs inflate neither
  number. A failed proc stays loud without emitting a toast on every countdown tick.
- The blue proc gear excludes monitor rows **and** rows carrying the wire `service`
  marker, so a live sessionless Telegram/service receiver can no longer inflate the
  session gear in every session.
- Disabled and unavailable service rows carry provenance/reason inline in the tree
  (`disabled here`, the enablement layer summary, `unavailable: <reason>`), with
  `ServiceEnablement` as a real typed consumer.
- The `Q` quit modal offers "stop Scheduler", routes it through
  `stop_service_proc("scheduler", ...)`, and never stops the service host.
- An idle TUI wakes when `~/.sase/service/status.json` or `state.json` change.
- `sase bead epic-symbols sase-11y.7` reports no entries, and only `sase-11y.7` closes.

With `service_host` disabled, the legacy AXE runtime, `RUNNING`/`STOPPED` footer
grammar, and quit-and-stop-axe path stay exactly as they are, so `sase-11y.10` (sunset)
still has one explicit Off branch to delete.

## Verified current state (2026-09-20, master `e65cc22e`)

Confirmed by reading the tree, not assumed:

**Already landed — do not rebuild.**

- `_TAB_DISPLAY_NAMES["axe"] == "Services"` in `src/sase/ace/tui/widgets/tab_bar.py`;
  `services` aliases canonical tab id `axe`; `--no-service` / `--restart-service` exist
  in `src/sase/main/parser_ace.py` and `src/sase/main/ace_handler.py`.
- `ServiceProcItem` rows and Scheduler nesting exist in
  `src/sase/ace/tui/widgets/bgcmd_list.py` and
  `src/sase/ace/tui/actions/axe_display/_loader_items.py`, with
  `_format_service_proc_option`, `_service_proc_marker`, and `_service_proc_chip`.
- Off-thread snapshot collection plus the selected-proc log tail live in
  `actions/axe_display/_data.py` and `_loader_refresh.py`; `_service_status` /
  `_service_host_enabled` are cached on the mixin (`_loader_state.py`).
- `src/sase/service/actions.py` is the shared CLI/TUI adapter (`start_service_proc`,
  `stop_service_proc`, `restart_service_proc`, `enable_service_proc`,
  `disable_service_proc`).
- Key routing is correct and test-covered: bare `x` toggles a selected service proc and
  no-ops on nested Scheduler rows / empty selection / host chrome
  (`_toggle_or_kill_axe_view`); `!x` always toggles the host (`_toggle_axe_global`);
  `!e` is `bang_mode.toggle_service_enablement: "e"` in `src/sase/default_config.yml`.
  `tests/ace/tui/actions/test_service_host_keys.py` covers these.
- `ace.procs.default_query` defaults to `-service` in `src/sase/default_config.yml`;
  `ProcsSessionState.query` seeds from it via `_default_procs_query()`.
- The proc query dialect already carries `service` and `svc:<name>`
  (`src/sase/ace/tui/_proc_query.py`); `ObservedProc.service` exists.
- The detail pane already prints `proc.enablement.summary` and `unavailable_reason`.

**Still missing — this tale's work.**

- `AxeInfoPanel` (`src/sase/ace/tui/widgets/axe_info_panel.py`) has no host clause at
  all. It renders a selected-row label or nothing; `ServiceStatusHost` is never passed
  in.
- `src/sase/ace/tui/widgets/_keybinding_status.py` still paints `SVC` plus
  `RUNNING`/`STOPPED` and a separate `[⚙n/m]` badge. `_get_service_proc_counts()` in
  `actions/axe_display/_render.py` uses `len(snapshot.procs)` as the denominator, so
  disabled procs inflate it.
- `_TAB_COLORS["axe"]` is still AXE failure-red `#FF5F5F`.
- Gear counting is still `projection.active_count - projection.active_monitor_count` in
  both `actions/_proc_action_observer.py:137` and `actions/lifecycle.py:217`.
  `_proc_observer_models.py` has no service-aware helper.
- `_service_proc_chip()` returns bare `"disabled"` / `"unavailable"` with no provenance
  or reason. Nothing imports `ServiceEnablement`.
- `QuitOptionsModal` still says "Quit & Stop axe"; `AxeMixin._stop_axe_and_quit`
  explicitly skips the daemon stop when the flag is on, so it stops _neither_ the host
  nor the scheduler.
- `actions/event_refresh/_surface_tokens.py` probes only `axe_state_dir()`; nothing in
  `SurfaceTokenRoots` or `live_surface_token_roots()` touches the service directory.
- All sixteen `sase-11y.7(...)` `--epic-symbol` entries remain in the `Justfile` (lines
  359–374).

**Stale in the prior closure plan — corrected here.** That plan told the worker to
re-key the platform/env symbols to `sase-11y.5`. `sase-11y.5` has since **closed**
(commit `23c740a9`), so re-keying there would produce an immediately-stale entry that
turns other agents' `just check` red. They go to the still-open parent epic `sase-11y`
instead. Its warning about colliding with an in-flight platform-units worker is also
obsolete.

## Boundaries

- Internal tab id stays `axe`. Do not rename it; that would regenerate ~667 visual
  snapshots and belongs to `sase-11y.10`.
- Do not migrate `!` background-command slots or add an oneshot section. `sase-11y.8`
  owns durable oneshot service procs.
- Do not add or remove feature flags. Cover both `service_host` branches in tests.
- `ServiceStatusSnapshot` remains the sole TUI read model. Do not reconstruct desired
  state, enablement merge rules, or health classification in Textual code.
- Per `tui_perf.md`: render, navigation, and footer paths perform no synchronous disk
  I/O, JSON parsing, config loads, or subprocess calls. New formatting helpers are pure
  and take already-cached data. Slow bodies go through the existing pump-free task path;
  re-capture the selected service key after any await.
- Do not manufacture Symvision consumers. `ServiceEnablement` earns its keep by being
  displayed; `service_dir` / `service_state_path` by being probed. Everything else is
  re-keyed, never imported from a test to satisfy the linter.
- Do not close `sase-11y` or any ancestor. Record out-of-scope discoveries only as
  `PROPOSED FOLLOW-UP:` notes on `sase-11y.7`.

## Implementation

### 1. Always-on host chrome in `AxeInfoPanel`

Teach `src/sase/ace/tui/widgets/axe_info_panel.py` a host clause that renders whenever
the service host is enabled, regardless of the selected row.

- Add a setter (e.g. `update_host_chrome(host, *, enabled, start_hint)`) that stores a
  `ServiceStatusHost | None` plus the display string for `!x`. Keep the import under
  `TYPE_CHECKING` as the module already does for `ServiceStatusProc`.
- Render, in `_update_display()`, before the existing selected-row clause:
  - running: `Services · host ● running <uptime> · <platform_unit>`, falling back to
    `detached` when `host.platform_unit` is `None`. Derive uptime from `host.started_at`
    with a small pure duration formatter (`4d`, `3h`, `12m`, `45s`).
  - not running: `Services · host ○ stopped · press !x to start`, using the configured
    bang-mode chord display rather than a hard-coded `!x` when the keymap helper is
    already reachable from the render call site; hard-code only as a fallback.
  - `set_loading(True)` keeps rendering `Services …` unchanged.
- The selected-row identity (Scheduler / proc / routine / job / bgcmd) follows the host
  clause on the same line; it must not replace it.
- Feed it from both call sites in `actions/axe_display/_render.py` — the main display
  update (~line 90) and `_update_axe_info_panel()` (~line 370) — passing
  `self._service_status.host` from the already-cached snapshot. When the flag is off,
  pass `enabled=False` and keep today's copy verbatim.
- Formatting stays pure and disk-free; unit-test the duration formatter and each chrome
  variant (running with unit, running detached, stopped, loading, flag-off) directly on
  the widget.

### 2. Service-health footer pill

Replace the AXE status pill in `src/sase/ace/tui/widgets/_keybinding_status.py` when the
flag is on.

- Add a pure health derivation next to `_get_service_proc_counts()` in
  `actions/axe_display/_render.py` (or a small sibling module imported by both the
  render path and tests) that takes the cached `ServiceStatusSnapshot` and returns a
  typed health result: `(running, desired, healthy, summary)`.
  - denominator: configured procs that are enabled **and** either desired-running or
    enabled-but-unavailable;
  - numerator: those counted procs whose `state == "running"`;
  - disabled procs (`not proc.enablement.enabled`) are omitted from both counts;
  - unhealthy when the host is not running, or any counted proc is failed/errored,
    unavailable, or desired-running while not running;
  - `summary` is a short human clause naming the first offender
    (`telegram_receiver failed`, `host stopped`).
- Push that result to the footer through the existing
  `set_service_proc_count(...)`-shaped setter (extend or replace it — keep one setter,
  not two) and render:
  - healthy: `SVC <running>/<desired>` in the service teal;
  - unhealthy: loud `SVC !` in the existing red/emphasis styling.
- Keep the startup stopwatch branch and the bgcmd `[*n]` / `[✓n]` badges exactly as they
  are. Drop the separate `[⚙n/m]` badge — the pill now carries those counts.
- Include the health tuple in `_status_signature()` so an unchanged health does not
  repaint, and store a last-notified signature so a toast fires only on a
  health/`change_token` **transition** — countdown ticks must never toast.
- When the flag is off, keep the current `RESTARTING`/`STARTING`/`STOPPING`/
  `RUNNING`/`STOPPED` grammar untouched.
- Change `_TAB_COLORS["axe"]` in `widgets/tab_bar.py` from `#FF5F5F` to the service teal
  `#00D7AF` already used for service rows.
- Update `tests/test_keybinding_footer_status.py` and
  `tests/ace/tui/widgets/test_keybinding_footer_stopwatch.py` for healthy, host-down,
  failed-proc, disabled-omitted, unavailable-counted, transition-dedup, and flag-off
  cases.

### 3. Service-aware gear counting and Procs query persistence

- In `src/sase/ace/tui/_proc_observer_models.py` add `is_gear_eligible_row(row)` and
  `gear_eligible_count(projection, *, all_sessions=False)`: an active row counts only
  when it is neither `is_monitor_shell_row(row)` nor `row.service is not None`. Leave
  `active_rows`, `active_count`, `active_monitor_count`, and the full projection alone —
  the Procs pane inventory must not change.
- Use the helper from both gear callers, replacing the subtraction:
  `actions/_proc_action_observer.py:137` and `actions/lifecycle.py:217`. Do not infer
  ownership from `session_id is None` anywhere.
- Regression test: a projection holding an ordinary session proc, a monitor shell, and a
  live **sessionless** service receiver yields a gear count of 1, while the monitor
  indicator and Procs inventory are unchanged.
- In `src/sase/ace/tui/modals/config_center_session.py`, give `ProcsSessionState` an
  explicit initialization marker (e.g. `query_initialized: bool`) alongside the existing
  `_default_procs_query()` seed, and have every commit path — including a commit of the
  empty string — set it. A reopened pane must reuse the committed query rather than
  reseeding; a user-cleared query stays cleared. Harden `_default_procs_query()` so a
  non-string configured value yields `""` without raising, and an explicitly empty
  configured default stays empty. Cover these in
  `tests/ace/tui/test_procs_pane_filter.py`. Do not touch `_proc_query.py`; assert its
  existing `service` / `svc:<name>` support instead of reimplementing it.

### 4. Inline enablement provenance (and the real `ServiceEnablement` consumer)

- Add a small pure formatter — in `widgets/bgcmd_list.py` or a tiny sibling module it
  imports — that takes a `ServiceEnablement` (imported by name from
  `sase.service.status`, not `getattr`-duck-typed) and returns the chip text/style:
  - locally disabled → `disabled here`;
  - disabled by a layer → the enablement `summary` (e.g. `disabled by sase_apollo.yml`),
    truncated to a tree-safe width;
  - enabled → `None`.
- Use it from `_service_proc_chip()`; when the proc is unavailable, render
  `unavailable: <reason>` from `unavailable_reason` (falling back to bare `unavailable`)
  instead of today's bare chip. Keep the existing running/restarts/state chips.
- Keep the detail pane on `enablement.summary`; do not duplicate the text there.
- That real import is what lets `sase-11y.7(ServiceEnablement)` leave the `Justfile`. Do
  **not** import `ServiceFieldProvenance`, `compose_service_config`, or
  `resolve_service_enablement` — the snapshot already carries effective enablement.
- Unit-test the formatter and the chip for enabled, locally disabled, layer-disabled,
  unavailable-with-reason, and unavailable-without-reason.

### 5. Quit modal stops Scheduler, never the host

- In `src/sase/ace/tui/modals/quit_options_modal.py`, make the user-facing copy
  flag-aware: "Quit & Stop Scheduler" (and the `1/s quit & stop scheduler` hint) when
  the service host is enabled, today's "Quit & Stop axe" when it is not. Keep the
  internal `QuitOption` id `quit_stop_axe`.
- In `AxeMixin._stop_axe_and_quit` (`actions/axe.py`), replace the flag-on no-op with
  `stop_service_proc("scheduler", actor="tui", reason="ace quit")` executed off the
  event loop on the existing worker path, then exit through the same
  `_begin_controlled_exit` / `_do_quit` `finally:` branch. Swallow adapter errors the
  way the legacy branch does — a failed stop must never block the quit.
- It must never call `stop_service_host`.
- Update the flag-aware copy in `widgets/_keybinding_modes.py` (bang-mode
  `start/stop axe` → `start/stop host` when the flag is on) and the help-modal strings
  in `modals/help_modal/axe_bindings.py`, `agents_bindings.py`, and
  `patches_bindings.py` so the Services vocabulary is consistent.
- Extend `tests/ace/tui/actions/test_axe_stop_quit.py` and
  `tests/ace/tui/actions/test_service_host_keys.py`: flag-on quit calls
  `stop_service_proc("scheduler", ...)` and not `stop_service_host`; flag-off quit still
  calls the legacy axe daemon stop; both still quit when the stop raises.

### 6. Idle token probe covers the service directory

- Extend `actions/event_refresh/_surface_tokens.py`:
  - add service paths to `SurfaceTokenRoots` (e.g. `service_dir`, `service_state_path`,
    `service_status_path`, all optional so existing constructions keep working);
  - populate them in `live_surface_token_roots()` from
    `sase.service.paths.service_dir()`, `service_state_path()`, and
    `service_status_path()`;
  - stat them inside the existing `axe` surface probe (`_probe_axe_token`, or an
    adjacent helper folded into the same token) so host reconciliation marks the
    Services surface dirty.
- Metadata only: existence, mtime, size. No JSON reads, no directory walks beyond the
  existing bounded pattern. A missing service directory must produce a stable token, not
  an indeterminate one, so a flag-off TUI does not refresh every tick.
- This is the real non-test consumer of `service_dir` and `service_state_path`.
- Add cases to `tests/ace/tui/test_surface_tokens.py`: touching `status.json` and
  `state.json` drifts the `axe` token; an absent service directory yields a stable,
  determinate token across probes.

### 7. Clear the phase's Symvision whitelist

Run `just install`, then `just _lint-symvision`. In the `Justfile` (lines 359–374):

- **Drop** — now really consumed by steps 4 and 6:
  - `sase-11y.7(ServiceEnablement)`
  - `sase-11y.7(service_dir)`
  - `sase-11y.7(service_state_path)`
- **Re-key to the parent epic `sase-11y`** — platform/init symbols whose owning phase
  `sase-11y.5` is closed, so they cannot be keyed there, and which this phase must not
  consume:
  - `CapturedServiceEnvironment`, `read_service_environment`
  - `NativeInspection`, `NativeServiceDefinition`, `ServicePlatformApplyResult`
  - `build_native_definition`, `inspect_native_service`, `readiness_warnings`
  - `service_platform_supported`
- **Re-key to the parent epic `sase-11y`** — service facades the TUI must not call,
  because the snapshot is the read model and enable/disable already goes through
  `set_service_enablement`:
  - `ServiceFieldProvenance` (field-level config provenance, not tab chrome)
  - `compose_service_config` (`load_service_config()` is the public loader)
  - `resolve_service_enablement` (the snapshot carries effective enablement)
  - `clear_service_enablement` (reset-to-config-default is not a Services-tab key)

Keep the list `keep-sorted`-clean and add a short comment above the re-keyed block
recording why each group moved (owning phase closed / deliberately unconsumed by the
Services tab). Then record one note for the land agent:

```text
sase bead note sase-11y.7 'PROPOSED FOLLOW-UP: sase-11y platform/config service facades have no non-test consumer — at sunset, privatize or delete CapturedServiceEnvironment, read_service_environment, NativeInspection, NativeServiceDefinition, ServicePlatformApplyResult, build_native_definition, inspect_native_service, readiness_warnings, service_platform_supported, ServiceFieldProvenance, compose_service_config, resolve_service_enablement, clear_service_enablement instead of carrying epic-symbol entries.'
```

If Symvision then flags a newly unused public name this tale created, follow
`sase/memory/symvision.md`: delete it, make it private, or give it a real non-test
consumer — never a test-only import, never a new `--epic-symbol` entry.

## Verification and closure

1. Focused tests: host chrome and duration formatter, footer health (healthy, host-down,
   failed proc, disabled-omitted, unavailable-counted, transition dedup, flag-off
   grammar), gear exclusion with a sessionless service receiver, Procs default-query and
   cleared-query persistence, surface-token drift on `status.json`/`state.json`,
   quit-with-Scheduler vs flag-off axe stop, the enablement chip formatter, and the
   existing `tests/service/` and `tests/ace/tui/actions/test_service_host_keys.py`
   suites.
2. `just fix-tui-screenshots` for the Services host-chrome / tree-chip / footer /
   tab-color goldens. Inspect each creation, removal, and update group in
   `.pytest_cache/sase-visual/latest-report.json` before accepting; generation is not
   approval. Do not change the canonical internal tab id in snapshots. Run it through
   `/sase_monitor` with the `TESTING` / `TESTED` pair if it outruns a comfortable inline
   wait.
3. Exercise both flag states by hand or in tests: `sase tui --tab services` and
   `--tab axe`, `--no-service`/`--no-axe`, `--restart-service`/`--restart-axe`,
   host-down chrome, a disabled proc, an unavailable proc, Scheduler child folding and
   job actions, quit-with-Scheduler, footer degradation, and a user-cleared Procs query.
4. Navigation benchmark with status snapshots churning in a restart-storm loop:
   `SASE_TUI_PERF=1`, `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`. Record p95 < 16
   ms on the Services tab and confirm render/navigation/footer callbacks do no
   synchronous disk, config, or subprocess work.
5. `just fix` inline, then `just check` (hand it to `/sase_monitor` if it runs long).
   Run `just install` first — this workspace may have drifted deps. Do **not** run
   `just check-full`; this is one phase, not the epic landing.
6. `sase bead epic-symbols sase-11y.7` must print no entries. Record any genuinely
   out-of-scope discovery as a `PROPOSED FOLLOW-UP:` note on `sase-11y.7` via
   `sase bead note`; do not create beads.
7. Close only this phase, naming the tests, the visual inspection, the p95 number, the
   `just check` result, and the empty epic-symbol audit:

   ```text
   sase bead close sase-11y.7 --note "<what was verified>"
   ```

   Do not close `sase-11y` or any other ancestor.
