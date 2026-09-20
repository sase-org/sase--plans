---
tier: tale
title: Remove the service_host beta flag and its Off branches
goal:
  The `service_host` beta flag no longer exists in SASE. Its registry entry, schema
  block, env injection helper, and every gate that reads it are deleted; the scheduler
  handler, the service control plane, `sase service init`, `sase doctor`, the mobile
  gateway, and the Services tab run the former On branch unconditionally; the legacy AXE
  lifecycle Off branches they guarded are gone; and flag bead sase-12m is closed with
  the removal.
size: medium
proposed_by: bbugyi200.athena.sase-11y.10.1.2
bead: sase-11y.10.1.2
create_time: 2026-09-20 14:38:23
status: wip
---

- **PARENT:**
  [202609/service_host_sunset.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_sunset.md)
- **BEAD:**
  [sase-11y.10.1.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.10.1.2.md)

# Plan: Remove The `service_host` Beta Flag And Its Off Branches

## Context and invariants

This is the `flag-removal` phase of the `service_host_sunset` epic (`sase-11y.10.1`).
Nine earlier phases shipped the replacement: `sase service run` is a platform unit on
athena and apollo, it owns the scheduler, the gateway, and the Telegram receiver, and
the Services tab renders them. The epic's one disqualifying outcome is **two supervisors
owning the same child**, and the flag's Off branches are the first of five code paths
that can still start the AXE orchestrator behind the service host's back.

Per the `sase_flags` memory, removing a flag **deletes the disabled (Off) branch and
makes the enabled (On) branch unconditional**. Nothing here adds a feature, and **no
sunset flag is created** — every Off branch is already dead on both live machines
(rollout verified in sase-11y.9). `bgcmd_legacy_slots` is a separate, still-live sunset
flag from the oneshots phase; do not touch it.

Invariants:

- `_run_foreground_scheduler` stays. It is what the service host execs for the
  `scheduler` service proc; deleting it breaks the host.
- `LEGACY_SYSTEMD_UNITS` in `src/sase/service/platform.py` stays untouched, so
  `sase service init` keeps detecting an already-installed `sase-axe-ensure.timer`.
- `sase axe start|stop|restart|status` still drives `sase/axe/process.py` in this phase.
  Retargeting those verbs is `axe-cli` (`sase-11y.10.1.3`), so
  `start_axe_daemon_result`, `stop_axe_daemon_result`, `restart_axe_daemon_result`,
  `collect_axe_status_snapshot`, `render_axe_status_human`, `render_axe_status_json`,
  and `render_restart_json` all keep live non-test consumers after this change. Verify
  that per symbol rather than assuming it; do **not** delete them here.
- Documentation still describes the flag after this phase. `docs/` is the `docs` phase
  (`sase-11y.10.1.5`), which depends on `axe-cli` and `tab-id`. Leave `docs/`,
  `mkdocs.yml`, and `docs/blog/posts/` alone.
- The TUI tab id stays `"axe"` — that is the `tab-id` phase (`sase-11y.10.1.4`). Do not
  touch `tab_order.py` or any `"axe"` tab-id comparison.
- No memory file changes. The glossary is the `glossary` phase.

### Ordering prerequisite (already satisfied at the repo level, with a deployment caveat)

`sase-telegram`'s `_service_host_owns_receiver()` reads `FeatureFlag.service_host`
inside a bare `except Exception: return False`. Once the enum member is gone that lookup
raises, the helper returns `False`, and `ensure_receiver_running()` re-arms a second
`getUpdates` consumer on a machine where the host already owns the receiver.

The `telegram-rearm` phase (`sase-11y.10.1.1`) is **closed** and its fix is on
`sase-telegram` `origin/master` as commit `f99521c` ("fix(receiver): stop consulting the
service_host flag in the rearm check"), verified in the linked checkout this turn. The
repo-level ordering constraint is therefore met.

**Deployment caveat the implementer must repeat in its close note:** on this machine the
installed plugin is a dev editable checkout at
`/home/bryan/projects/github/sase-org/sase-telegram` pinned at `f05f53b` (release
0.4.19), one commit behind that fix — `sase plugin show telegram` reports
`v0.4.19 → v0.4.19+1.gf99521c8e ↑`. `sase update` (or a fast-forward of that checkout)
must run on athena and apollo **before** this change is deployed there. Do not run
`sase update` from inside this phase; it is a host action outside the workspace. Record
it as a `PROPOSED FOLLOW-UP:` note on `sase-11y.10.1.2` and state it in the close note.

## Step 1 — Delete the registry entry and regenerate the schema

- `src/sase/feature_flags/registry.py`: delete the `service_host = "service_host"` enum
  member (`:35`) and the `FeatureFlag.service_host: FeatureFlagDefinition(...)` block
  (`:151-159`).
- Regenerate the generated JSON Schema block rather than hand-editing it:
  `just sync-feature-flags-schema`. Confirm the `service_host` property is gone from
  `src/sase/config/sase.schema.json` (currently `:5098`) and that no other property
  moved.

## Step 2 — Scheduler handler and parser

`src/sase/main/scheduler_handler.py`:

- Delete the `from sase.feature_flags import FeatureFlag, current_flags` import and the
  `if current_flags().enabled(...)` branch in `handle_scheduler_command`; dispatch
  straight to `_handle_service_scheduler`.
- Delete `_handle_legacy_scheduler`, `_legacy_start`, `_legacy_status`, `_legacy_stop`,
  and `_legacy_restart`. Drop the now-unused `import json` if nothing else needs it.
- Keep `_run_foreground_scheduler` and the `run` short-circuit exactly as they are.
- Update `handle_scheduler_command`'s docstring: it no longer dispatches "in either
  legacy or service-host mode".

`src/sase/main/parser_scheduler.py` (follow the `cli_rules` memory — alphabetical
subcommands and options, short aliases, no required options, excellent `-h`):

- Rewrite the `scheduler` parser `description` for the single behavior: start, stop,
  restart, and status route through the `scheduler` service proc on the SASE service
  host; `run` execs the foreground orchestrator (what the host itself runs).
- Delete `restart --verify-timeout` / `-t` and its help text, and `stop --force` / `-f`
  and its help text. Both existed only for the legacy AXE lifecycle.
- If `_add_json_flag` or `_add_scheduler_overrides` loses its last caller, delete it;
  otherwise leave them. `stop` keeps no options after `--force` goes.
- Regenerate the checked-in completion snapshot: `just sync-completion-spec`, then
  confirm `tests/completion/snapshots/cli_spec.json` no longer carries the
  `scheduler`-scoped `verify_timeout` / `force` options (the `axe restart`
  `--verify-timeout` entry stays — that parser is untouched).

## Step 3 — Service control plane

`src/sase/service/control.py`:

- Delete `SERVICE_HOST_DISABLED_MESSAGE` (`:40`), `ServiceHostDisabledError` (`:50`),
  `_service_host_enabled` (`:134`), and `require_service_host_enabled` (`:139`),
  together with their four `__all__` entries.
- Delete `_feature_env_with_service_host` (`:426`) and the
  `env["SASE_FEATURE_FLAGS"] = _feature_env_with_service_host(env)` line in
  `start_service_host` (`:224`). The spawned host must inherit the ambient transport
  unchanged; forcing a now-unregistered key into it would write an `unknown_key` warning
  into every detached host's environment.
- Drop the `from sase.feature_flags import FeatureFlag, current_flags` import, and drop
  `import json` if nothing else in the module needs it.

`src/sase/main/service_handler.py`:

- Drop the `require_service_host_enabled` import and its call in
  `handle_service_command` (`:52`), and the `except ServiceHostDisabledError` arm
  (`:54-56`) with its import (`:18`).
- Keep the `load_service_environment(override_existing=True)` call for `service run` and
  its position ahead of dispatch; only the gate goes.

`src/sase/service/platform.py`:

- Delete the
  `if not current_flags().enabled(FeatureFlag.service_host): blockers.append(...)` gate
  (`:171`) and the now-unused `FeatureFlag` / `current_flags` import.
- Leave `LEGACY_SYSTEMD_UNITS` and the `SASE_FEATURE_FLAGS` capture/drift warning
  (`:594-598`) alone; a captured `~/.sase/service/env` that still carries the old key is
  handled in Step 7.

## Step 4 — Init and doctor

`src/sase/main/init_service_handler.py`: delete the flag check and its inactive
`InitPlan` (`:23-29`) plus the `FeatureFlag` / `current_flags` import. The
`_declined_for_bare_onboarding` branch stays.

`src/sase/doctor/checks_service_platform.py`: delete the `SKIP` branch (`:12-22`) and
the `FeatureFlag` / `current_flags` import. `check_service_platform` now always
evaluates `service_init_plan(force=True)`.

## Step 5 — Mobile gateway

`src/sase/integrations/mobile_gateway.py`:

- In `_service_host_owns_gateway` (`:394`), delete the
  `if not current_flags().enabled(FeatureFlag.service_host): return False` check and the
  `from sase.feature_flags import ...` line (`:399,401`). The configured-and-available
  `gateway` service-proc entry plus its enablement override is now the whole answer —
  the same shape the `telegram-rearm` phase landed in the plugin. Keep the surrounding
  `except Exception: return False`; an older `sase` without `sase.service.config` still
  has to import.
- Keep the direct `popen` start path in `_run_mobile_gateway_start`. It is the fallback
  for a machine that has not configured the gateway service proc, not a flag Off branch.
- Reword the `_delegate_gateway_start_to_service_host` diagnostic at `:422` so it no
  longer says "disable service_host or upgrade SASE"; "service host APIs are
  unavailable; upgrade SASE" is enough.

## Step 6 — The TUI

### 6a. Collector (`src/sase/ace/tui/actions/axe_display/_data.py`)

- Delete `_service_host_enabled()` (`:594-598`) and the `service_host_enabled` field on
  `AxeCollectedData` (`:175`) and its constructor argument (`:586`).
- Collapse the three `service_host_enabled` branches at `:355-382`: always import and
  call `persisted_or_current_status()` inside the existing `try/except` and derive
  `axe_running` from `service_status.host.state in {"running", "starting"}`. Delete the
  `proc = get_axe_process_module()` / `proc.is_axe_running()` fallback, the
  `proc.get_axe_status()` block that built `axe_status`, and the `read_metrics()` call.
- `get_axe_process_module` (`:194`) then has no non-test consumer, so delete it and its
  re-export from `src/sase/ace/tui/actions/axe_display/__init__.py` (`:11,32`) per the
  `symvision` memory's delete-first hierarchy. Drop the now-unused `read_metrics` import
  (`:32`) and `import types` if nothing else uses it.
- Keep the `axe_status` / `axe_metrics` fields, `AxeStatus` / `AxeMetrics` imports,
  `_invalid_config_status`, and the `degraded_status` assignment at `:428-432`: the
  degradation path is still fed by `axe_config.load_axe_config()`. After the collapse
  `axe_status` and `axe_metrics` are permanently `None`. That is intentional scope
  containment, not an oversight — record it as a `PROPOSED FOLLOW-UP:` note on
  `sase-11y.10.1.2` (retire the always-`None` `axe_status` / `axe_metrics` collector
  fields and the `update_empty_axe_display` `status` / `full_cycles` parameters they
  feed) rather than expanding this phase.

### 6b. The now-constant `_service_host_enabled` attribute

Every read is `getattr(self, "_service_host_enabled", False)`. Delete the attribute
itself and collapse each reader to its enabled arm:

- `src/sase/ace/tui/actions/_state_init_late.py:75` — delete the initializer.
- `src/sase/ace/tui/actions/axe_display/_loader_state.py:88` — delete the
  `_service_host_enabled: bool` annotation and fix the stale comment above the
  service-snapshot cache block ("populated only while the beta flag is enabled").
- `src/sase/ace/tui/actions/axe_display/_loader_refresh.py:139` — delete the
  `self._service_host_enabled = data.service_host_enabled` assignment.
- `src/sase/ace/tui/actions/axe_display/_loader_refresh.py:710-726` — keep only the host
  restart / host start arm and `return`; delete the trailing
  `_restart_axe_daemon(source="ace startup restart")` /
  `_start_axe(source="ace startup")` else-branch.
- `src/sase/ace/tui/actions/axe_display/_loader_items.py:132` — the condition becomes
  `service_status is not None`; the `else: self._append_lumberjack_items(items)` arm
  stays (it is the no-snapshot case, not a flag case).
- `src/sase/ace/tui/actions/axe_display/_render.py:102,385` — pass the host chrome
  unconditionally (see 6d).
- `src/sase/ace/tui/actions/axe_display/_render.py:312` — drop the
  `service_host_enabled=` argument to `footer.update_axe_bindings`.
- `src/sase/ace/tui/actions/axe_display/_render.py:471` — delete the early
  `footer.set_service_health(None)` return in `_push_service_health` and the "Flag off
  pushes `None`" sentence from its docstring.

### 6c. `src/sase/ace/tui/actions/axe.py`

- `_toggle_host_or_axe_daemon` (`:111-120`) becomes host start/stop only. Rename its
  docstring accordingly; keep the method name (renaming it is `tab-id`/`axe-cli`-scale
  churn and out of scope).
- `_toggle_axe_global` (`:161-170`): the other-tab "nothing running" arm calls
  `_start_service_host()`; the "only axe running" arm calls `_stop_service_host()`.
  Refresh the docstring, which still says "Start axe" / "Stop axe".
- `action_stop_axe_and_quit` (`:261`): drop the `service_host=` keyword from the
  `QuitOptionsModal(...)` construction.
- `_stop_axe_and_quit` (`:273-292`): always
  `stop_service_proc("scheduler", actor="tui", reason="ace quit")`; delete the
  `_stop_axe_daemon_result` else-arm.
- Delete `_start_axe` (`:397`), `_stop_axe` (`:412`), and `_restart_axe_daemon`
  (`:427`), and the three aliased imports at `:10-14` (`_restart_axe_daemon_result`,
  `_start_axe_daemon_result`, `_stop_axe_daemon_result`). These are the TUI direct-start
  path.
- `_set_axe_starting` / `_set_axe_stopping` / `_set_axe_restarting` and
  `_axe_worker_operation` all keep live callers through `_run_service_proc_action`,
  `_start_service_host`, `_stop_service_host`, and `_on_axe_worker_done` — keep them.
  Re-check each after the deletions instead of trusting this sentence.
- Retarget the two surviving non-flag callers of the deleted methods:
  - `src/sase/ace/tui/actions/axe_bgcmd.py:485,487` (`_show_process_selector`) —
    `"start_axe"` calls `self._start_service_host()` and `"axe"` calls
    `self._stop_service_host()`.
  - `src/sase/ace/tui/actions/axe_config_actions/_mixin.py:335` — the AXE-config-edit
    restart becomes `self._run_service_proc_action("scheduler", "restart")`. That helper
    occupies the same `_axe_worker` slot and sets `_axe_worker_operation`, so the
    existing `_axe_config_restart_saved_path` bookkeeping in `_on_axe_worker_done` keeps
    working unchanged. Verify the `outcome.axe_running` / `_axe_worker` guards above it
    still read correctly.

### 6d. Widgets and modals

- `src/sase/ace/tui/widgets/axe_info_panel.py` — `update_host_chrome`'s `enabled`
  keyword is now always `True`. Delete the parameter, the `_host_chrome_enabled`
  attribute (`:51,63`), and the `if self._host_chrome_enabled:` guard (`:211`); the host
  clause always renders. Update both call sites in `_render.py`.
- `src/sase/ace/tui/widgets/_keybinding_bindings.py:62,91` and
  `src/sase/ace/tui/widgets/_keybinding_modes.py:89,212,228` — delete the
  `service_host_enabled` parameter from `_compute_axe_bindings`, its protocol/overload
  signature, and `update_axe_bindings`. The noun is always `"service host"`, so the
  label is `f"stop service host"` / `f"start service host"`.
- `src/sase/ace/tui/modals/quit_options_modal.py:41,44,117` — drop the `service_host`
  keyword argument and the `_service_host` attribute; `_stop_target` is the constant
  `"Scheduler"`. The rendered hint row becomes `quit & stop scheduler`.
- `src/sase/ace/tui/modals/process_select_modal.py` — the `start_axe` / `axe` entries
  now start and stop the **service host**, so update their user-visible copy:
  `display_name` `"sase service"` and descriptions "Start the SASE service host" / "Stop
  the SASE service host". Keep the `process_type` literals and `ProcessSelection`'s
  `Literal[...]` unchanged — they are internal discriminants that `tab-id` and `axe-cli`
  are not chartered to rename either, and renaming them here would churn `axe_bgcmd.py`
  for no behavior change. Leave the `"AXE Control"` modal title alone (that is the
  `tab-id` phase's naming sweep).

## Step 7 — An unknown key in `SASE_FEATURE_FLAGS` must stay a warning

A captured `~/.sase/service/env` on athena or apollo may still carry
`SASE_FEATURE_FLAGS={"service_host": true}`, and `capture_service_environment`
(`src/sase/service/env.py:51`) copies that variable verbatim. Verify — do not assume —
that an unregistered key is tolerated:

- `parse_feature_flags_env` (`src/sase/feature_flags/env.py`) does no registry
  validation.
- `src/sase/feature_flags/resolver.py:80-81` emits a
  `FeatureFlagDiagnostic(severity="warning", code="unknown_key", ...)` — "unknown
  feature flag ... ignored".

Confirm end to end with a real invocation, e.g.
`SASE_FEATURE_FLAGS='{"service_host": true}' sase service status` and
`... sase flag list`, and check that neither exits non-zero on the unknown key. If a
test does not already pin this, add one (a resolver-level case asserting the warning
diagnostic and a non-fatal snapshot is enough). **State the verified behavior explicitly
in the close note** — the epic plan asks for it by name.

## Step 8 — Tests

Delete the flag-disabled cases outright — the behavior they assert no longer exists —
and keep the enabled cases with the `override_flags(service_host=True)` wrapper removed.

- `tests/doctor/test_checks_service_platform.py` — delete
  `test_service_platform_check_skips_when_beta_flag_is_off`; unwrap the other three.
- `tests/main/test_parser_service_scheduler.py` — delete
  `test_service_flag_off_fails_with_opt_in_diagnostic`; rewrite
  `test_service_run_loads_captured_env_before_beta_flag_gate` so it still pins that
  `service run` loads the captured environment before dispatch (the gate is gone, so
  assert ordering against `_handle_service_command` instead of dropping the test); drop
  the `require_service_host_enabled` monkeypatch from
  `test_service_status_does_not_override_interactive_environment`; unwrap the rest.
  `test_scheduler_help_lists_sorted_subcommands` must still pass with the new
  description and the deleted options.
- `tests/service/test_service_platform.py` — unwrap the flag overrides and delete the
  `SASE_FEATURE_FLAGS='{"service_host": true}'` key from the captured-environ fixture at
  `:135`; keep the drift-warning assertions working.
- `tests/service/test_service_executable.py`,
  `tests/service/test_service_effective_env.py`,
  `tests/service/test_service_host_runtime.py`, `tests/service/test_actions.py`,
  `tests/service/test_service_state.py`, `tests/main/test_proc_handler_kill.py`,
  `tests/test_mobile_gateway.py` — unwrap overrides, delete flag-off cases. In
  `test_mobile_gateway.py` keep the delegation test but drive it through a configured
  `gateway` entry rather than the flag.
- `tests/ace/tui/actions/test_axe_stop_quit.py` — delete
  `test_quit_modal_copy_is_flag_aware` or rewrite it as a constant assertion
  (`QuitOptionsModal()._stop_target == "Scheduler"`); drop the
  `app._service_host_enabled = True` line from
  `test_stop_and_quit_with_service_host_stops_scheduler_only` and make it the only
  stop-and-quit behavior.
- `tests/ace/tui/actions/test_service_host_keys.py` — drop
  `self._service_host_enabled = True` from `_Host` and the `_start_axe` / `_stop_axe`
  stubs at `:29,32`, which no longer exist on the mixin.
- `tests/ace/tui/actions/test_startup_observability.py` — drop the
  `_data._service_host_enabled` patch (`:216`) and the `get_axe_process_module` patch
  (`:197`); patch `sase.service.control.persisted_or_current_status` instead.
- `tests/ace/tui/test_axe_collector.py` (8 sites),
  `tests/ace/tui/test_axe_collector_overrun.py` (4 sites),
  `tests/ace/tui/test_axe_status_read_cache.py` (1 site) — every
  `patch(".../_data.get_axe_process_module")` and `patch(".../_data.read_metrics")`
  targets an attribute that no longer exists and will raise at patch time. Replace them
  with a patch of `sase.service.control.persisted_or_current_status` returning a fake
  snapshot whose `host.state` drives `axe_running`.
  `test_collector_degrades_invalid_axe_config_to_status` no longer has a
  `proc.get_axe_status` to raise from: rewrite it so `sase.axe.config.load_axe_config`
  raises the `AxeConfigError`, which is now the only producer of `degraded_status`.
- `tests/ace/tui/test_axe_lifecycle_sources.py` and
  `tests/ace/tui/test_axe_config_actions.py:382` — these pin the deleted `_start_axe` /
  `_stop_axe` / `_restart_axe_daemon` `source=` arguments. Delete the cases that only
  covered the deleted methods, and repoint the config-edit case at
  `_run_service_proc_action("scheduler", "restart")`.
- `tests/ace/tui/test_axe_worker_status_completion.py` and
  `tests/ace/tui/test_axe_live_chop_run.py` stub `_set_axe_*`, which survive — expect no
  change, but re-run them.

Grep for `service_host` across `tests/` at the end; the only surviving hits should be
service-host _feature_ names (`start_service_host`, `_service_host_owns_gateway`,
`stop_service_host`, …), never the flag.

## Step 9 — Close the flag bead, then verify

Per the `sase_flags` memory a removed beta flag's bead closes **with the removal**, not
on its `remove_by` date:

```bash
sase bead close sase-12m --note "<what you verified>"
```

Do this in the same change. Then run, in order:

1. `just install` (ephemeral `sase_<N>` workspaces may have drifted deps).
2. `just fix` (or at minimum `just fmt`) inline — it prevents avoidable formatting /
   keep-sorted failures.
3. `just _lint-flags` on its own first: it enforces that the generated schema block
   matches the registry, that no registered key lacks a non-test reference, and that no
   closed flag bead has a surviving definition. A failure here means Step 1 or Step 9 is
   incomplete.
4. `just _lint-symvision`. Expect fallout only from `get_axe_process_module`; if any
   other public symbol is reported, follow the `symvision` memory's hierarchy — delete
   dead symbols and their tests, do not whitelist. Do **not** add an `--epic-symbol`
   entry unless a later phase of _this_ epic provably consumes the symbol;
   `sase bead epic-symbols sase-11y.10.1.2` currently reports none and should still
   report none at close.
5. `just fix-tui-screenshots`, then **inspect the report before applying**. Expect churn
   in the `!x` footer help row ("stop axe" → "stop service host"), the quit-options hint
   row ("quit & stop axe" → "quit & stop scheduler"), and possibly the process selector.
   Generation is not approval: expand every update group and confirm each diff is one of
   those three text changes. Anything else is a regression to investigate, not a golden
   to accept. (18 goldens went stale earlier in this epic and were refreshed during the
   sase-12z.5 landing, so a surprise here is new, not inherited.)
6. `sase tool run check`. Do **not** run `just check-full`.

Then `sase bead epic-symbols sase-11y.10.1.2` and
`sase bead close sase-11y.10.1.2 --note "..."`.

## Close note must state

- That the `telegram-rearm` fix `f99521c` is on `sase-telegram` `origin/master`, and
  that the local editable install is still at `f05f53b`, so `sase update` is required on
  athena and apollo before deployment.
- The verified `SASE_FEATURE_FLAGS` unknown-key behavior from Step 7 (warning, not
  error), with the command that demonstrated it.
- That `sase-12m` was closed with the removal.
- The `just fix-tui-screenshots` outcome and which golden groups were inspected and why
  they changed.

## Follow-ups to record as bead notes (not new beads)

Use `sase bead note sase-11y.10.1.2 'PROPOSED FOLLOW-UP: ...'`:

- `sase update` must run on athena and apollo to pick up `sase-telegram` `f99521c`
  before this change is deployed, or `ensure_receiver_running` re-arms a second
  `getUpdates` consumer.
- `AxeCollectedData.axe_status` / `axe_metrics` are permanently `None` after this phase;
  retire them and the `update_empty_axe_display` `status` / `full_cycles` parameters
  they feed.
- `docs/` (`ace.md`, `architecture.md`, `axe.md`, `cli.md`, `getting_started.md`,
  `init.md`, `mobile_gateway.md`) still documents a flag that no longer exists between
  this phase and the `docs` phase.

## Out of scope

The ensure watchdog, the axe-start systemd scope wrapper, the `sase update` direct
restart, the `sase axe` → `sase scheduler` alias, the `axe` → `services` tab id, all
documentation, and the glossary strands. Each belongs to a named later phase of
`service_host_sunset`.
