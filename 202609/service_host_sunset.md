---
tier: epic
title: Sunset legacy supervision paths, docs, and glossary
goal: 'The sase service host is SASE''s only supervisor of the scheduler: the service_host
  beta flag and every Off branch are gone, the ensure watchdog, the TUI direct-start
  path, the axe-start scope wrapper, the sase-update direct restart, and the Telegram
  rearm branch are deleted, `sase axe` is a documented alias of `sase scheduler`,
  the TUI tab id is `services`, and the docs and glossary describe the shipped system.

  '
phases:
- id: telegram-rearm
  title: Retire the Telegram receiver rearm branch
  depends_on: []
  size: small
  description: 'telegram-rearm: in the sase-telegram repo, stop `ensure_receiver_running`
    from consulting the `service_host` flag so removing that flag from sase cannot
    silently re-arm a second getUpdates consumer, and delete the rearm branch''s flag-off
    tests.'
- id: flag-removal
  title: Remove the service_host beta flag and its Off branches
  depends_on:
  - telegram-rearm
  size: large
  description: 'flag-removal: delete the `service_host` registry entry and every gate
    that reads it, delete the Off branches in the scheduler handler, the TUI, the
    mobile gateway, doctor, init, and the service control plane, make the On branch
    unconditional, and close flag bead sase-12m.'
- id: axe-cli
  title: Retire the AXE watchdogs and alias sase axe to sase scheduler
  depends_on:
  - flag-removal
  size: large
  description: 'axe-cli: delete the ensure watchdog, the opportunistic ensure on agent
    waits, the axe-start systemd scope wrapper and its doctor check, route the `sase
    update` restart through the scheduler service proc, and make `sase axe` lifecycle
    verbs a documented alias of `sase scheduler`.'
- id: tab-id
  title: Canonicalize the Services tab id
  depends_on:
  - flag-removal
  size: large
  description: 'tab-id: rename the internal ACE tab id from `axe` to `services` across
    tab_order, the app, actions, widgets, modals, and test helpers, keep `axe` as
    a normalized legacy alias for persisted and CLI input, and refresh the PNG goldens.'
- id: docs
  title: Update the documentation for the service host
  depends_on:
  - axe-cli
  - tab-id
  size: medium
  description: 'docs: rewrite docs/axe.md around the scheduler/host split, update
    ace, cli, configuration, plugins, remote_dispatch, and mobile_gateway for `sase
    service`, and retitle the mkdocs nav entry.'
- id: glossary
  title: Land the service-host glossary strands
  depends_on:
  - axe-cli
  - tab-id
  size: medium
  description: 'glossary: add the Sase Service, Service Proc, Oneshot Service Proc,
    Service Node, and Sase Scheduler strands, edit the Sase Node, Routine, Job, and
    Proc strands, and republish generated agent instructions.'
proposed_by: bbugyi200.athena.sase-11y.10
parent_bead: sase-11y.10
create_time: 2026-09-20 13:56:08
status: done
bead_id: sase-11y.10.1
---

- **PROMPT:** [prompts/202609/service_host_sunset.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/service_host_sunset.md)
- **PARENT:** [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:** [sase-11y.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.10.1.md)

# Plan: Sunset legacy supervision paths, docs, and glossary

This is the final phase of the service-host epic, broken into phases of its own. Nine
phases have already shipped the replacement: `sase service run` is a platform unit on
athena and apollo, it owns the scheduler, the gateway, and the Telegram receiver, the
Services tab renders them, and `!` background commands are oneshot service procs. What
remains is deletion.

The epic's one disqualifying outcome is **two supervisors owning the same child**. Today
five separate code paths can still start or restart the AXE orchestrator behind the
service host's back:

1. the `service_host` flag's Off branches (`sase scheduler start`, the TUI's direct
   start/stop/restart, `sase service` refusing to run at all),
2. the `sase axe ensure` watchdog, its optional systemd timer, and the opportunistic
   `ensure_axe()` call on every agent wait,
3. `sase axe start` / `restart` / `stop`, which still drive `sase/axe/process.py`
   directly and bypass the host entirely,
4. `sase update`'s `restart_after_update`, which restarts the AXE daemon directly,
5. the Telegram plugin's `ensure_receiver_running` rearm tick, whose flag check fails
   open.

Every one of them is deleted here. Nothing in this plan adds a feature.

## Ordering constraint that is easy to get wrong

`sase-telegram`'s `_service_host_owns_receiver()` (`src/sase_telegram/receiver.py:115`)
reads `FeatureFlag.service_host` inside a bare `except Exception: return False`. The
moment `flag-removal` deletes the enum member, that lookup raises, the helper returns
`False`, and `ensure_receiver_running()` starts re-arming a receiver that the host
already owns — two `getUpdates` consumers on athena, silently. That is why
`telegram-rearm` lands **first**, in the plugin repo, and why `flag-removal` must not
start until it has.

## No new feature flags

`sase_flags` memory: removing a flag deletes the Off branch and makes the On branch
unconditional. Every branch retired here is already dead on both live machines (rollout,
sase-11y.9, verified `sase.service` active and `sase-gateway.service` retired on athena
and apollo), so none of them needs a migration window and **no phase should create a
sunset flag**. The one on-disk legacy artifact — an installed `sase-axe-ensure.service`
/ `.timer` — is already in `LEGACY_SYSTEMD_UNITS` in
`src/sase/service/platform.py:48-52`, so `sase service init` already reports and
disables it; a stale timer that invokes a deleted subcommand fails loudly and
harmlessly. `bgcmd_legacy_slots` is a separate, still-live sunset flag from the oneshots
phase; do not touch it.

## Cross-cutting rules for every phase

- Never leave two owners for one child. If a phase deletes the last caller of a
  supervision helper, delete the helper and its tests too rather than leaving it
  reachable.
- Follow the `cli_rules` memory for every CLI change: alphabetical subcommands and
  options, short aliases, no required options, excellent `-h`.
- Any keymap or footer-label change also updates `src/sase/default_config.yml`.
- Symvision deletions follow the `symvision` memory's hierarchy: delete dead symbols and
  their tests; do not whitelist. When a phase deletes a public symbol that another phase
  of _this_ plan still consumes, use `--epic-symbol` keyed to this plan's epic bead and
  remove the entry in the phase that lands the consumer.
- After any change to rendered TUI output, run `just fix-tui-screenshots` and inspect
  the report before applying, per the `lint_and_test` memory.
- Run `sase tool run check` before finishing. Do **not** run `just check-full` unless
  explicitly instructed.
- Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
  bead; they do not create beads.

## Phase detail

### telegram-rearm — Retire the Telegram receiver rearm branch

Open the plugin with `sase repo open sase-telegram -r "<why>"` and work only in the path
it prints.

- In `src/sase_telegram/receiver.py`, `_service_host_owns_receiver()` currently answers
  "the host owns this" only when the `service_host` flag is on **and** a
  `telegram_receiver` entry is configured and available. Drop the flag check; keep the
  service-config check. The service host is now unconditional, so a configured,
  available `telegram_receiver` entry is the whole answer.
- The `except Exception: return False` around the config lookup must stay (an older
  `sase` without `sase.service.config` still has to import), but it must no longer be
  reachable via the flag import.
- Update `tests/test_receiver.py` and `tests/test_integration.py`: delete the cases that
  assert re-arming happens while the flag is off, and keep the cases that assert
  `ensure_receiver_running()` returns `None` when the service proc is configured and
  still launches when it is not.
- Do not delete `ensure_receiver_running` itself. It remains the fallback for a machine
  that has not adopted the service proc, and `docs/inbound.md` documents it.
- Verify with the plugin's own `just check`.

Leave a `PROPOSED FOLLOW-UP:` note if the plugin needs a release bump before `sase` can
depend on the new behavior; do not bump versions here.

### flag-removal — Remove the service_host beta flag and its Off branches

Delete the flag and make every On branch unconditional. Nine files read the flag today:

- `src/sase/feature_flags/registry.py:35` (enum member) and `:151-159` (definition).
- `src/sase/main/scheduler_handler.py:23` — delete `_handle_legacy_scheduler` and
  `_legacy_start` / `_legacy_status` / `_legacy_stop` / `_legacy_restart`;
  `handle_scheduler_command` dispatches straight to `_handle_service_scheduler`.
  `_run_foreground_scheduler` stays — it is what the host execs.
- `src/sase/main/parser_scheduler.py` — the command description still explains both flag
  states; rewrite it for the single behavior. `restart --verify-timeout` and
  `stop --force` exist only for the legacy AXE lifecycle; delete both options and their
  help text.
- `src/sase/service/control.py:41,134-136` — delete `_service_host_enabled` and the
  "`service_host` beta flag is disabled" diagnostic; every entry point runs.
- `src/sase/service/platform.py:171` — drop the gate.
- `src/sase/main/init_service_handler.py:23` — drop the "inactive while service_host is
  disabled" `InitPlan`; the declined-marker branch stays.
- `src/sase/doctor/checks_service_platform.py:12` — drop the `SKIP` branch. The check
  now always evaluates `service_init_plan(force=True)`.
- `src/sase/integrations/mobile_gateway.py:401` —
  `_delegate_gateway_start_to_service_host` becomes the only path; delete the flag check
  and the "disable service_host or upgrade SASE" message at `:422`.
- `src/sase/ace/tui/actions/axe_display/_data.py:594-598` — delete
  `_service_host_enabled()` and the `service_host_enabled` field on the collected data
  (`:175`, `:586`), and collapse the `if service_host_enabled:` branches at `:355-382`
  so the collector always reads `persisted_or_current_status()` and never falls back to
  `proc.is_axe_running()` / `proc.get_axe_status()` / `read_metrics()`.

Then collapse every consumer of the now-constant TUI attribute `_service_host_enabled`,
which is a `getattr(self, "_service_host_enabled", False)` idiom in these places:

- `src/sase/ace/tui/actions/axe.py:112,116,161,167,261,273` —
  `_toggle_host_or_axe_daemon` becomes host start/stop only; the "nothing running on
  another tab" arm of `_toggle_axe_global` starts the host; `_stop_axe_and_quit` always
  stops the `scheduler` service proc.
- `src/sase/ace/tui/actions/axe.py:397-438` — delete `_start_axe`, `_stop_axe`, and
  `_restart_axe_daemon` (the TUI direct-start path) together with the
  `_start_axe_daemon_result` / `_stop_axe_daemon_result` / `_restart_axe_daemon_result`
  imports at `:11-12` and the `_axe_worker_operation` bookkeeping that only they used.
  Check `_set_axe_starting` / `_set_axe_stopping` / `_set_axe_restarting` for surviving
  callers before deleting them.
- `src/sase/ace/tui/actions/_state_init_late.py:75`, `axe_display/_loader_state.py:88`,
  `axe_display/_loader_items.py:132`, `axe_display/_loader_refresh.py:139,710`,
  `axe_display/_render.py:102,312,385,471`.
- `src/sase/ace/tui/modals/quit_options_modal.py:41,44,117` — drop the `service_host`
  keyword argument; `_stop_target` is always `"Scheduler"`.
- `src/sase/ace/tui/widgets/_keybinding_bindings.py:62,91` and
  `widgets/_keybinding_modes.py:89,212,228` — drop the `service_host_enabled` parameter;
  the noun is always "service host". This changes the `!x` help row text, so refresh
  `help_keymaps_changespecs` and any other affected golden.
- `src/sase/ace/tui/modals/process_select_modal.py:25,62,71,112` — the `"start_axe"` and
  `"axe"` process types start and stop the legacy daemon; retarget them at the service
  host or delete the entries that no longer have a live action.

Tests to update or delete (all of these call `override_flags(service_host=...)` or set
`_service_host_enabled`): `tests/doctor/test_checks_service_platform.py`,
`tests/service/test_service_platform.py`, `tests/service/test_service_executable.py`,
`tests/service/test_service_effective_env.py`,
`tests/service/test_service_host_runtime.py`, `tests/service/test_actions.py`,
`tests/main/test_parser_service_scheduler.py`, `tests/main/test_proc_handler_kill.py`,
`tests/test_mobile_gateway.py`, `tests/ace/tui/actions/test_axe_stop_quit.py`,
`tests/ace/tui/actions/test_service_host_keys.py`,
`tests/ace/tui/actions/test_startup_observability.py`. Delete the flag-disabled cases
outright — the behavior they assert no longer exists — and keep the enabled cases with
the override removed. `tests/service/test_service_platform.py:135` also passes
`SASE_FEATURE_FLAGS='{"service_host": true}'` in a captured environ fixture; that key
must go, and an unknown flag key in `SASE_FEATURE_FLAGS` must not become an error for
users whose captured `~/.sase/service/env` still carries it — verify the existing
behavior and state it in the close note.

Finally, in the same change: `sase bead close sase-12m --note "<what you verified>"` per
the `sase_flags` memory (a removed beta flag's bead closes with the removal, not on its
`remove_by` date). Run `tools/check_feature_flags` — or whatever `just lint` calls for
flag integrity — to confirm no orphaned definition or bead reference survives.

`sase/repos/linked/sase-telegram` is the one external consumer of the flag; it is
already handled by `telegram-rearm`, but re-check with
`SYMVISION_EXTERNAL_REPO_PATHS=<the path sase repo open prints> just _lint-symvision` if
any URI pragma in this repo points at the plugin.

### axe-cli — Retire the AXE watchdogs and alias sase axe to sase scheduler

Four deletions and one CLI reshaping.

**The ensure watchdog.** With the host reconciling roughly every second and restarting a
crashed scheduler itself, a second healer is the exact race this epic exists to remove.

- Delete `src/sase/axe/_ensure_timer.py` and the `sase axe ensure` subparser
  (`src/sase/main/parser_ace.py:189-210`) and handler
  (`src/sase/main/axe_handler.py:107-140`, the `elif axe_sub == "ensure"` arm at `:35`,
  and the usage string at `:50-52`).
- Delete `src/sase/axe/ensure.py` and `src/sase/axe/_ensure_runtime.py` unless a
  surviving consumer needs them. `acquire_axe_ensure_lock` and `release_axe_ensure_lock`
  are re-exported from `ensure.py`; grep for other callers before deleting, and if the
  lock is still used by the host or the orchestrator, move it to the module that uses it
  rather than keeping `ensure.py` alive as a shim.
- Delete `_opportunistic_ensure_axe` and its call site in
  `src/sase/axe/run_agent_wait.py:55-62,258`. An agent wait must not start a supervisor.
- Delete `tests/test_axe_ensure.py` and the `sase axe ensure install` assertion in
  `tests/main/test_parser_command_help.py:85`.
- Do **not** touch `LEGACY_SYSTEMD_UNITS` in `src/sase/service/platform.py`;
  `sase service init` must keep detecting an already-installed `sase-axe-ensure.timer`.

**The axe-start scope wrapper.** `wrap_axe_start_in_systemd_scope`
(`src/sase/axe/systemd_scope.py:26`) exists so a TUI-launched orchestrator survived its
launching session. The host now owns the scheduler and `src/sase/detach_scope.py` is the
sanctioned cgroup-escape helper.

- Delete `src/sase/axe/systemd_scope.py` and its use in
  `src/sase/axe/_process_start.py:29,186`.
- Delete the `axe.systemd_scope` doctor check
  (`src/sase/doctor/checks_axe.py:41-44,96-121`) and the `_with_systemd_scope_issue`
  path in `src/sase/axe/status_collector.py:42,128-139`. Under the host the scheduler's
  cgroup is `sase.service`, not a `.scope`, so the check can no longer fire — it is
  dead, not merely quiet.
- Update `tests/test_axe_process_start.py` and
  `tests/test_axe_cli_chop_run_contract.py:342`.
- Keep `src/sase/detach_scope.py`, `src/sase/procs/spawn.py:143`, and
  `src/sase/monitor/spawn.py:139` exactly as they are.

**The `sase update` restart.** `restart_after_update`
(`src/sase/main/update_restart.py:29`) calls `restart_axe_daemon_result` directly, so an
update restarts the orchestrator while the host also notices it died. Route it through
`restart_service_proc("scheduler", actor="cli", reason="sase update")` and adjust
`RestartInfo` construction, which currently reports a pid the service action may not
return. `src/sase/ace/tui/update_restart.py:87` and the callers in
`src/sase/main/update_handler_live.py`, `update_handler_mode_switch.py`,
`src/sase/plugins/cli_install.py`, `cli_uninstall.py`, `cli_update.py`, and
`_required_gate_actions.py` must keep compiling and keep rendering a sensible
`render_restart_info` panel. If `RestartAxeFn` / `AxeRunningFn` in
`src/sase/main/update_types.py` lose their meaning, rename them rather than leaving
axe-shaped names on a service-host code path. `_call_restart_axe`'s signature
introspection exists only to tolerate zero-argument test fakes; simplify it if the new
callee makes it unnecessary.

**`sase axe` becomes an alias.** `sase axe start|stop|restart|status`
(`src/sase/main/axe_handler.py:41-48,205-218,254-294,296-352,354+`) still drives
`sase/axe/process.py`. Retarget those four verbs at `handle_scheduler_command` so
`sase axe start` and `sase scheduler start` are the same command.

- `sase axe {job,routine,maintenance}` (with the `chop` / `lumberjack` aliases at
  `axe_handler.py:33,37`) manage the scheduler's routine/job tree, not its process
  lifetime; they stay.
- `sase axe bgcmd-launch` is an internal durable entrypoint; leave it.
- Say the alias out loud in `-h`: the `axe` parser's help becomes something like "Alias
  of `sase scheduler` plus the routine/job tree", and `sase scheduler`'s description
  names `sase axe` as the accepted former name. Keep both alphabetical.
- Once the four verbs delegate, `start_axe_daemon_result`, `stop_axe_daemon_result`,
  `restart_axe_daemon_result`, `canonical_axe_start_command`, and
  `should_reexec_axe_start_from_canonical` may have no live callers. `sase service` uses
  the canonical-executable logic for `ExecStart`, so check each one individually and
  delete only what is genuinely dead, per the `symvision` memory.
- `--no-service` / `--no-axe` and `--restart-service` / `--restart-axe`
  (`src/sase/main/parser_ace.py:83-85,127-129`) already carry both spellings. Leave the
  aliases; only make sure the help text names the service host, and note that `dest` is
  still `no_axe` / `restart_axe` — renaming the dest is `tab-id`-scale churn and is out
  of scope here.

### tab-id — Canonicalize the Services tab id

`src/sase/ace/tui/tab_order.py` currently reads `SERVICES_TAB: TabName = "axe"` with
`"services"` as an alias pointing the other way. Invert it.

- `TabName` becomes `Literal["artifacts", "agents", "services"]`, `TAB_ORDER` becomes
  `("agents", "artifacts", "services")`, `SERVICES_TAB` becomes `"services"`, and
  `"axe"` joins a `LEGACY_SERVICES_TABS` frozenset that `normalize_tab_name` maps
  forward — exactly the shape `LEGACY_ARTIFACTS_TABS` already has for
  `changespecs`/`patches`.
- Replace the hardcoded `"axe"` tab comparisons with `SERVICES_TAB`:
  `_app_layout.py:74`, `_app_watchers.py:33,62,75,175`,
  `_app_action_availability.py:214,216,408,426`, `actions/axe.py:273,391`,
  `testing/ace_page.py:102,194`, `testing/ace_page_group.py:87`. Import the constant
  instead of reintroducing a literal; `_app_action_availability.py` already imports
  `ARTIFACTS_TAB` for precisely this reason.
- `src/sase/main/parser_ace.py:100-114` — `--tab` keeps accepting `axe`, but `services`
  becomes the documented value and `axe` moves into the compat list beside
  `changespecs`/`patches` at `:115`.
- Session-state migration: any persisted or replayed tab value must survive. The entry
  points are `normalize_tab_name` (already the funnel for `AceApp.__init__`,
  `src/sase/ace/tui/app.py:325`), `src/sase/ace/tui/commands/types.py:259`
  (`__post_init__` already rewrites legacy tab values and needs an `axe` → `services`
  arm), and `src/sase/ace/testing/ace_page.py:34-37` (`_LEGACY_STATE_VALUE_ALIASES`
  gains `("tab", "axe"): "services"`). A TUI restart that hands the old id forward must
  land on the Services tab, not fall back to `agents` — cover that with a test.
- Leave alone the many unrelated `"axe"` strings that are **not** tab ids: the `source`
  query token (`agent_query/tokenizer.py:49`,
  `query_profile/profiles/_agents_live.py:39`), agent-launch source values
  (`scheduler/mentor_runner.py:104`, `scheduler/workflows_runner/starter.py`),
  `BGCMD_STATE_DIR = sase_subdir("axe") / "bgcmd"` (`tui/bgcmd.py:37`, a legacy state
  path read behind `bgcmd_legacy_slots`), config segment names
  (`modals/axe_entry_editor_types.py:179`), and the `runners_modal` / `jump_all_modal` /
  `command_palette_modal` section keys. Changing a state path or a query token here
  would be a behavior change, not a rename. If a display label in `jump_all_modal.py:58`
  or `command_palette_modal.py:37` still reads `AXE`, update the label but keep the key.
- Do not rename modules or the `#axe-dashboard` widget id. This phase renames one
  identifier's value, not the AXE-era file layout; a file rename would collide with
  `docs` and explode the diff.
- The tab label already renders as "Services", so the id change should be pixel-neutral.
  Run `just fix-tui-screenshots` anyway and confirm the report is `updated=0`; if
  anything moved, inspect each group before applying. Note that 18 goldens went stale
  earlier in this epic (epic bead sase-11y notes #3 and #4) and were refreshed during
  the sase-12z.5 landing, so a dirty golden here is new, not inherited.

### docs — Update the documentation for the service host

Documentation is the last place the retired paths still exist. Work from the shipped
CLI, not from this plan's prose: run `sase service -h`, `sase service proc -h`,
`sase scheduler -h`, and `sase axe -h` and document what they print.

- `docs/axe.md` (1702 lines, 173 "axe" mentions) — the big one. Split the page into the
  **sase service** host (platform unit, `sase service init`, state, status, enable vs.
  stop) and the **scheduler** service proc (routines and jobs, which are unchanged).
  Delete the `sase axe ensure install` watchdog section (`:74`, `:143`, `:1577`) and the
  recovery advice built on it — recovery is now `Restart=on-failure` plus `sase doctor`.
  Retitle the page for the scheduler and mention that `sase axe` is the accepted former
  name.
- `docs/ace.md` — the Services tab, its keys (`x`, `r`, `!!`, `!e`, `!x`), the host
  status line, the `SVC` footer pill, and the Procs pane's `service` / `svc:<name>`
  query fields and `tui.procs.default_query` default.
- `docs/cli.md:509` — replace the `sase axe ensure install` row; add `sase service` and
  `sase service proc` rows; mark `sase axe` as an alias of `sase scheduler`.
- `docs/configuration.md:5320` — replace the ensure-timer paragraph; document the
  `service:` section (`service.procs`, per-entry `description`, `enabled`, `mode`,
  `command` XOR `builtin`, `cwd`, `env`, `restart`, `success_exit_codes`, `stop_signal`,
  `stop_timeout`, `after`, `log_max_bytes`) and that the host resolves machine-level
  layers only — never a project-local `sase.yml`.
- `docs/plugins.md` — how a plugin declares a service proc through its `sase_config`
  layer, and that plugin-shipped entries default to `enabled: false`. State plainly that
  there is no Python plugin proc API in v1.
- `docs/remote_dispatch.md` and `docs/mobile_gateway.md` — replace the hand-written
  systemd unit recipes with `sase service init`, and note that the gateway defaults off
  and is enabled per machine in the overlay.
- `mkdocs.yml:24` — `AXE Automation: axe.md` becomes a Services/scheduler entry.
- Run the repo's Markdown formatter (`just fmt`) and whatever link check `just lint`
  performs; a renamed heading breaks in-page anchors elsewhere in the docs tree.

Do not rename `docs/axe.md` itself. External links and the blog posts under
`docs/blog/posts/` point at it; a redirect is a separate decision, and the blog posts
are dated historical records that should not be rewritten.

### glossary — Land the service-host glossary strands

Memory files are not ordinary files. Use the `/sase_memory_write` skill before editing;
authorization comes from this plan (the approving user asked for the glossary work, and
the parent epic plan names it). Write the canonical strands under
`sase/memory/glossary/`, never `AGENTS.md` or `CLAUDE.md`, then run `sase memory init`
to republish.

Five new strands. The wording below is the approved text from the epic plan; adjust only
where shipped behavior differs, and say so in the close note if you do.

- `sase-service.md` — keyword **Sase Service**, alias `service host`: "The sase service
  is the per-machine host process (`sase service run`) that starts, restarts, and stops
  that machine's service procs, with at most one per SASE home. `sase service init`
  registers it as a platform unit — a systemd user unit on Linux, a launchd LaunchAgent
  on macOS — so it starts at boot or login; without one, `sase service start` runs it
  detached. systemd/launchd supervises the sase service itself, never each service proc.
  Say 'service proc' for what it runs and 'platform unit' for its registration; bare
  'service' is ambiguous."
- `service-proc.md` — keyword **Service Proc**: "A service proc is a named proc the sase
  service owns, declared under `service.procs` by core (builtin), a plugin's config
  layer, or the user, or created at runtime as a transient oneshot. Its mode is `daemon`
  — kept running under its restart policy — or `oneshot` — run once to completion.
  Control one with `sase service proc` or its Services-tab node; `enabled: false` in a
  machine overlay keeps it off that machine. Every launch is a distinct durable Proc
  carrying a `service` marker, hidden from the Procs tab and proc gear by default."
- `oneshot-service-proc.md` — keyword **Oneshot Service Proc**, aliases `oneshot`,
  `background command`: "A oneshot service proc runs once to completion and is never
  restarted. Transient oneshots are created at runtime by the TUI's `!` background
  commands or `sase service proc run`; they run outside the sase service's process tree,
  so restarting the sase service does not kill them. Distinct from a job, which a
  routine runs on a schedule."
- `service-node.md` — keyword **Service Node**: "A service node is one selectable row of
  the Services tab: a service-proc node, a oneshot node, or — nested only under the
  Scheduler's node — a routine node and its job nodes. A service-proc node is stable
  across process restarts; section dividers and the host status line are chrome, not
  nodes."
- `sase-scheduler.md` — keyword **Sase Scheduler**, aliases `scheduler`, `AXE`: "The
  sase scheduler is the builtin service proc behind `sase scheduler` that runs SASE's
  background automation: it starts one process per routine and restarts crashed
  routines, and routines run jobs. The sase service owns the scheduler's process
  lifetime; the scheduler owns only its routine/job tree. AXE is its former name and
  remains an accepted alias."

Four edits to existing strands:

- `sase-node.md` generalizes from "one row of the Agents tab's agent tree" to one
  selectable row of a hierarchical TUI view, keeping the whole Agents taxonomy and
  adding "on the Services tab, a service node".
- `lumberjack.md` (keyword **Routine**) — the scheduler, not "the AXE orchestrator",
  starts and restarts it. Keep the Lumberjack aliases.
- `chop.md` (keyword **Job**) — the same substitution. Keep the Chop aliases.
- `proc.md` — add "Runs of service procs carry a `service` marker; the Procs tab hides
  them by default", and drop the now-misleading `background task` / `background tasks`
  aliases, which `oneshot-service-proc.md` now claims.

The glossary web declares `link_reference: implicit` and `link_rendering: inline`, so
prose references between these strands resolve without explicit `[[...]]` markup; check
`sase memory read glossary:sase-service ... -r "<why>"` renders the closure you expect
before finishing.

Do **not** add "platform unit" or "service proc definition" strands — the epic plan
rules them out as implementation vocabulary, not agent-facing terms.

Verify with `sase memory init` followed by `sase tool run check`, and confirm the
regenerated `CLAUDE.md` / `AGENTS.md` roster lists all five new terms.

## Risks

- **The Telegram ordering trap** described above. If `flag-removal` lands before
  `telegram-rearm` reaches the installed plugin, athena runs two `getUpdates` consumers.
  The `flag-removal` worker should confirm the plugin change is in the installed
  `sase-telegram` before removing the enum member, and say so in its note.
- **Over-deletion in `axe-cli`.** `sase/axe/process.py` and `sase/axe/_process_start.py`
  hold logic the service host still needs (canonical executable resolution, desired
  state). Delete per-symbol against real consumer evidence; a deletion that breaks
  `ExecStart` resolution breaks boot on both live machines.
- **Docs anchors.** `docs/axe.md` is linked from blog posts, `docs/cli.md`, and
  `docs/configuration.md`. Restructuring it will break in-page anchors that are not
  caught by tests; grep for `axe.md#` before renaming headings.
- **Goldens.** `flag-removal` changes the `!x` help row text and `tab-id` may shift
  nothing at all. Both phases must run `just fix-tui-screenshots` and inspect the
  report; generation is not approval.
