---
tier: epic
title: Finish the service-host sunset leftovers found at landing
goal: "No code path outside the sase service host starts the scheduler, nothing reads
  the retired AXE desired-state marker, `sase scheduler` offers no option it ignores,
  the Services-tab collector carries no permanently-empty legacy fields, and every
  non-blog doc describes the shipped service host, scheduler alias, and Services tab.

  "
phases:
  - id: chat-restart
    title: Route the chat-install post-update recovery through the service host
    depends_on: []
    size: medium
    description:
      'chat-restart: make the chat-install update worker bring the scheduler back
      through the `scheduler` service proc instead of `start_axe_daemon`, retitle its
      messages to "scheduler", and delete `start_axe_daemon` plus every
      `_process_start.py` helper that loses its last caller (sase-152).'
  - id: desired-state
    title: Retire the AXE desired-state marker
    depends_on:
      - chat-restart
    size: medium
    description:
      "desired-state: derive the scheduler's desired state from the service host instead
      of `~/.sase/axe/desired_state.json`, rebase the `axe.health` doctor check and the
      status collector on it, and delete the marker module, its `record_desired_state`
      plumbing, and its tests."
  - id: scheduler-cli
    title: Delete the scheduler options the service path ignores
    depends_on: []
    size: small
    description:
      "scheduler-cli: remove `-A/-H/-q/-z` from `sase scheduler start|restart` and their
      `sase axe` aliases and remove `restart -j`, keeping those overrides on `sase
      scheduler run`, then regenerate the CLI spec and completion snapshots."
  - id: tui-dead-state
    title: Retire the Services-tab fields the flag removal emptied
    depends_on: []
    size: medium
    description:
      "tui-dead-state: delete `AxeCollectedData.axe_status` / `axe_metrics`, the app's
      `_axe_status` / `_axe_metrics`, the `status` / `full_cycles` parameters they feed,
      and the `None` arm of `KeybindingFooter.set_service_health`, keeping the render
      pixel-identical."
  - id: docs-core
    title: Fix the service-host and scheduler reference docs
    depends_on:
      - chat-restart
      - desired-state
      - scheduler-cli
    size: medium
    description:
      "docs-core: remove the `service_host` flag, whole-system-status, heartbeat-verify,
      and ignored-option claims from `docs/axe.md`, `configuration.md`, `cli.md`,
      `architecture.md`, `init.md`, `getting_started.md`, `README.md`, and `index.md`,
      and fix the anchors those heading changes break."
  - id: docs-surfaces
    title: Rename the AXE tab and AXE restarts across the remaining docs and help text
    depends_on:
      - chat-restart
      - tui-dead-state
    size: medium
    description:
      "docs-surfaces: say Services tab and scheduler restart in `ace.md`, `plugins.md`,
      `integrations.md`, `perf_runbook.md`, `vcs.md`, `query_language.md`,
      `rust_backend.md`, `notifications.md`, `mentors.md`, runner-slots troubleshooting,
      `llms.md`, and the tabs infographic prompt, plus the stale `!x` and plugin-install
      help strings in `src/`."
proposed_by: bbugyi200.athena.sase-11y.10.1.land
parent_bead: sase-11y.10.1
create_time: 2026-09-21 03:59:09
status: wip
---

- **PROMPT:**
  [prompts/202609/service_host_sunset_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/service_host_sunset_finish.md)
- **PARENT:**
  [202609/service_host_sunset.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_sunset.md)

# Plan: Finish the service-host sunset leftovers found at landing

The service-host sunset epic has shipped all six of its phases. The `service_host` flag
and its Off branches are gone. The ensure watchdog, the scope wrapper, and the direct
update restart are deleted. `sase axe start|stop|restart|status` alias `sase scheduler`,
the tab id is `services`, and the glossary strands have landed. The landing review found
four gaps between that shipped state and the epic's goal. Each gap below is either left
behind by one of the epic's phases or blocks the epic's headline claim that the service
host is the scheduler's only supervisor:

1. **A sixth direct-start path.** The chat-driven update worker still starts the
   orchestrator itself (task bead sase-152). It is also the last caller of
   `start_axe_daemon`, and so the last thing that writes `desired_state.json` as
   `running`.
2. **A marker that stopped being written.** `sase scheduler start` and `stop` now go
   through the service host and no longer write `~/.sase/axe/desired_state.json`, but
   the `axe.health` doctor check and the deep status collector still read that file. A
   stale `running` marker left by an old `sase axe start` makes `sase doctor` warn "axe
   is desired running, but its orchestrator is down" after a deliberate
   `sase scheduler stop`, and tells the user to run `sase scheduler start`.
3. **Options that do nothing.** `sase scheduler start|restart` and the `sase axe`
   aliases still accept `-A/-H/-q/-z`, and `restart` still accepts `-j`. The legacy
   handlers that read them were deleted in flag-removal, and the service path ignores
   all of them.
4. **Dead TUI state and stale docs.** `AxeCollectedData.axe_status` and `.axe_metrics`
   are always `None` now. The footer's `set_service_health(None)` "legacy AXE pill" path
   has no production caller. About twenty doc files still describe the `service_host`
   flag, the deleted whole-system `sase axe status` snapshot, the heartbeat-verified
   restart, or the "AXE tab".

Nothing here adds a feature. Like its parent, this plan deletes or corrects.

## Cross-cutting rules for every phase

- No feature flags. The options, marker, and fields retired here are already inert on
  both live machines. This is the same judgment the parent plan made when flag-removal
  deleted `restart --verify-timeout` and `stop --force` outright.
- Delete per-symbol against real consumer evidence, following the `symvision` memory:
  delete dead symbols and their tests, and make a symbol private when only its own file
  uses it. Before deleting a public symbol, grep the linked plugin repos (open them with
  `sase repo open <name> -r "<why>"`).
- Follow the `cli_rules` memory for any CLI change, and regenerate generated CLI
  artifacts with `just sync-completion-spec` when parsers change.
- After any change to rendered TUI output, run `just fix-tui-screenshots` and inspect
  the report before applying anything, per the `lint_and_test` memory.
- Run `sase tool run check` before finishing. Do **not** run `just check-full`.
  symvision currently reports `bead_touch_glyph` and `ordered_bead_verb_chips` from
  sase-14j. They are not this plan's to fix; leave them. Also leave the known failures
  already tracked elsewhere: `tests/llm_provider/test_usage_config.py` (sase-14u /
  sase-14v) and `tests/test_test_shards.py` drift (sase-14r).
- Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
  bead. They do not create beads.

## Phase detail

### chat-restart — Route the chat-install post-update recovery through the service host

`src/sase/integrations/chat_install.py` `_update_worker` calls `_ensure_axe_running()`
after the update command. That function checks `is_axe_running()` and, when the check is
false, calls `_restart_axe()`, which calls `_chat_install_worker.restart_axe()` with
`start=start_axe_daemon`. The result is a detached orchestrator (`sase scheduler run`)
that the service host did not start.

- Replace that recovery with the service host. The intent of the chat flow is "after a
  remote update, the scheduler is running again", and the host already restarts a
  crashed scheduler under its restart policy. So the worker should call
  `start_service_proc("scheduler", actor="chat-install", ...)` from
  `sase.service.actions`. That call clears any boot-scoped stop marker and nudges the
  host. If the host itself is down, also call `start_service_host()` from
  `sase.service.control`. Then poll the scheduler proc's state through
  `persisted_or_current_status()` for a bounded time. Keep
  `chat_install.restart_attempts` only if it still has a meaning under the new flow,
  such as the poll budget. If it no longer means anything, delete the key from the
  config schema, `default_config.yml`, and `_chat_install_config.py`, and update
  `docs/configuration.md` (the `chat_install.restart_attempts` row) and
  `docs/integrations.md` (around line 418) in this phase.
- `sase update`, which this worker runs first, already restarts a _running_ scheduler
  through `restart_after_update()`. Do not restart it a second time. Only start it when
  it is not running afterwards.
- Retitle the user-facing strings: "axe restart failed" becomes a scheduler message, and
  the worker log lines ("starting axe", "axe restart succeeded") become scheduler lines.
  Keep exit code 5 for "update succeeded but the scheduler did not come back" unless a
  caller proves otherwise. Check `sase-telegram`, which renders chat-install results,
  for string matching before you change a message.
- Delete `start_axe_daemon`, `restart_axe` in `_chat_install_worker.py`, and every
  helper in `src/sase/axe/_process_start.py` that loses its last non-test caller,
  together with their re-exports in `sase/axe/process.py` and `sase/axe/__init__.py` and
  their tests. Keep `canonical_axe_start_command` and anything else that
  `src/sase/service/executable.py` still needs, because it builds the service host's
  `ExecStart`. Deleting it would break boot on both live machines. Keep
  `stop_axe_daemon_result`, because `service/host_support.py:handover_scheduler` uses it
  to adopt a legacy orchestrator.
- Update the chat-install tests to fake the service actions instead of
  `start_axe_daemon`.

The child epic's land agent closes sase-152 once this phase lands. The phase worker does
not.

### desired-state — Retire the AXE desired-state marker

After `chat-restart`, nothing writes `desired_state.json` as `running`. The only
remaining `write_desired_state` call sites are `_process_stop.py`'s `stop_axe_daemon*`
behind `record_desired_state` (the host's `handover_scheduler` passes `False`) and
whatever start path survives. The marker's readers are:

- `src/sase/doctor/checks_axe.py:_check_axe_health`, the `axe.health` check.
- `src/sase/axe/status_collector.py:_collect_desired_state`, which fills
  `AxeStatusRequest.desired_state`. The Rust classifier uses that field to tell `down`
  from `stopped`, and it feeds `checks_deep_axe.py`.
- `src/sase/axe/lifecycle_journal.py:_lifecycle_record`, which stamps the marker into
  each journal entry.
- `src/sase/ace/tui/actions/event_refresh/_surface_tokens.py`, which lists
  `desired_state.json` as a watched AXE root file.

Replace the marker with the service host's own view of the `scheduler` proc:

- Derive `AxeDesiredStateRecord(state, source, timestamp)` from the `scheduler` entry of
  `persisted_or_current_status()`. Base the state on the proc's `desired` value and its
  stop marker (`ServiceStatusProc.stop`), and name the service host as the source. Keep
  the Rust wire shape unchanged (`AxeStatusRequest.desired_state` is already optional).
  If the classifier needs a value the host cannot supply, pass `None` rather than
  inventing one, and say so in the close note. Per the Rust core boundary memory, touch
  `../sase-core` only if the wire contract itself must change. It should not need to.
- Rebase `axe.health` on the same source. Warn only when the host wants the scheduler
  running (enabled, desired running, no stop marker) and no orchestrator is live. Point
  `next_steps` at `sase service status` / `sase scheduler start`. Keep the check id
  `axe.health` and its `axe` group, because the id is a stable doctor selector. Retitle
  it for the scheduler service proc.
- Drop `desired_state` from new lifecycle journal records, or fill it from the same
  derived record. Readers must tolerate old entries that still carry the field.
- Delete `src/sase/axe/desired_state.py`, the `record_desired_state` /
  `desired_state_source` parameters that become meaningless, the surface token, and
  their tests. Leave any `desired_state.json` already on disk alone. Nothing reads it
  after this phase.

### scheduler-cli — Delete the scheduler options the service path ignores

`src/sase/main/parser_scheduler.py` attaches `add_scheduler_overrides` (`-A`, `-H`,
`-q`, `-z`) to `restart`, `run`, and `start`, and `add_json_flag` to `restart` and
`status`. `src/sase/main/scheduler_handler.py:_handle_service_scheduler` reads none of
them. Only `run` (`load_axe_config_with_overrides`) uses the overrides, and only
`status` (`handle_service_proc_show`) uses `-j`.

- Keep the overrides on `run` only. Keep `-j` on `status` only.
- Apply the same change to the `sase axe start|restart` alias parsers in
  `src/sase/main/parser_ace.py`, which reuse the same builders. The alias must stay
  identical to `sase scheduler`.
- Before deleting, grep this repo and the linked plugin repos for callers that pass
  these options, such as the TUI, `sase update`, `sase-telegram`, or scripts under
  `tools/`, and retarget each one.
- Make each `-h` say what the command actually does: `start`/`stop`/`restart` ask the
  service host to act on the `scheduler` service proc and return once the request is
  recorded.
- Regenerate `cli_spec.json` and the completion snapshots (`just sync-completion-spec`),
  and update the parser-help tests.

### tui-dead-state — Retire the Services-tab fields the flag removal emptied

flag-removal made the collector read only the service host. Since then, nothing produces
`AxeStatus` or `AxeMetrics` for the TUI:

- In `src/sase/ace/tui/actions/axe_display/_data.py`, `AxeCollectedData.axe_status` and
  `.axe_metrics` are always `None` (see `_collect_axe_status_data_inner`).
- `_loader_refresh.py` copies them into `self._axe_status` / `self._axe_metrics`. These
  are declared in `actions/axe.py`, `axe_display/_loader_state.py`, and
  `actions/_state_init_late.py`.
- `_render.py` (the zero-routine branch near line 198) feeds them into
  `AxeDashboard.update_empty_axe_display(status=..., full_cycles=...)`. That in turn
  feeds `_AxeStatusSection.update_display(status, is_running, full_cycles, ...)`.
  `AxeDashboard.show_empty` / `update_display` also pass `status=None`.

Delete the two collector fields, the two app attributes, and the `status` /
`full_cycles` parameters along the whole chain. Also delete any status-section rendering
that only runs for a non-`None` `AxeStatus` or a nonzero `full_cycles`, since it is now
unreachable. Check every other caller of `_AxeStatusSection` and
`AxeDashboard.update_display` before narrowing a signature.

`KeybindingFooter.set_service_health(health: ServiceHealth | None)` in
`widgets/_keybinding_status.py` documents `None` as "restores the legacy AXE pill", but
`_push_service_health` always passes a derived `ServiceHealth`. Narrow the setter to
`ServiceHealth` and fix its docstring. Delete the `set_service_health(None)` case in
`tests/ace/tui/test_services_phase_closure.py`. The footer's initial `None` state, which
renders the RUNNING/STOPPED pill before the first snapshot arrives, is still live
startup behavior. Keep it, and describe it as the pre-snapshot pill.

This should be pixel-neutral. Run `just fix-tui-screenshots` and confirm the report
shows `updated=0`. If anything moved, inspect every group before applying and explain
the change in the close note.

### docs-core — Fix the service-host and scheduler reference docs

Work from the shipped CLI (`sase service -h`, `sase service proc -h`,
`sase scheduler -h`, `sase scheduler status -h`, `sase axe -h`) as it stands after
`chat-restart`, `desired-state`, and `scheduler-cli`, not from this plan's prose. The
landing review found these stale statements. Re-grep before editing, because line
numbers drift.

- `docs/axe.md`:
  - Lines 83-84 and 141 say "whole-system health snapshot" / "schema version 2". Change
    them to the `scheduler` service proc's status, with `-j` emitting its proc record.
  - Lines 119-125: drop `sase axe status --json` from the `axe_routine_job_contract`
    list.
  - Lines 137-139 and 220-222 describe a heartbeat-verified restart and
    `restart --json`. Say instead that `restart` asks the host to stop and start the
    proc and returns immediately.
  - Lines 145-146 and 678-679 put overrides on `sase scheduler start`. Move them to
    `sase scheduler run` / `sase axe routine run` or the `axe:` config.
  - Replace the whole "Whole-System Status" section (about lines 176-216) with a short
    "Scheduler Status" section.
  - Lines 414: "a successful install restarts axe".
  - Lines 1529-1531 and 1564-1565 cover the desired-state marker, which is gone after
    `desired-state`.
  - Lines 1543-1556 cover the lifecycle journal's restart records, the three-attempt
    heartbeat restart, and the "Axe restart failed" notification. None of these exist
    any more.
  - Retitle "AXE Tab Views" (about line 1378) to "Services Tab Views", call it the
    "Services tab" throughout, and fix the in-page `#axe-tab-views` link.
- `docs/configuration.md`:
  - Replace the `### sase axe status|start|stop|restart` sections (about lines
    5295-5349) with the alias behavior. Note that `-v/--vcs-provider` is a `sase axe`
    group flag only.
  - Lines 3337: "CLI flags on `sase scheduler start`".
  - Lines 4738-4749, 6284-6290, 468, and 6676 describe AXE restarts. Make them scheduler
    service proc restarts, and change "TUI+AXE restart" to "TUI and service host
    restart".
  - Line 908: "Axe tab" becomes "Services tab".
- `docs/cli.md`: lines 506-507 (the restart `-j` and whole-system status rows, whose
  link is `axe.md#whole-system-status`), line 601 (the AXE restart wording), and
  line 319.
- `docs/architecture.md`: lines 16-17 and 120-129 describe the beta service host gated
  by `service_host`, the legacy `sase axe` control surface, and "an axe scope".
- `docs/init.md`: lines 25-27, 169, and 194-204 cover the `service_host` flag gate and
  `sase -f service_host service init` examples. The examples become plain
  `sase service init [--check|--diff|--yes]`.
- `docs/getting_started.md`: lines 129-132 (the flag-gated Services tab and "legacy Axe
  daemon") and line 328.
- `README.md` lines 43 and 123, and `docs/index.md` line 154. Use the mkdocs nav's
  "Scheduler and Service Host" naming.

Do not rename `docs/axe.md`, and do not edit `docs/blog/posts/`. Before renaming any
heading, grep the docs tree for its anchors (`axe.md#`, `#whole-system-status`,
`#axe-tab-views`) and fix every link. Run `just fmt` and the docs link checks in
`just lint`.

### docs-surfaces — Rename the AXE tab and AXE restarts across the remaining docs and help text

Same method as `docs-core`: follow the shipped behavior and re-grep for current line
numbers.

- `docs/ace.md`:
  - Lines 1096 and 2783: the `!x` row reads "Start / stop service host or axe". There is
    no separate axe target any more. Make the same fix in
    `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (about line 412) and
    `patches_bindings.py` (about line 184).
  - Lines 2820-2821: the tab id is `services`, and `--tab axe` is a legacy alias.
  - Every other place that calls the tab "Axe"/"AXE" becomes "Services": lines 959, 971,
    1269, 1350, 1578-1608, 2825, 2889, 3119, 3190-3204, 4438-4469, and 7677. Rename the
    "Axe Control" heading (about line 3058) and fix its anchors.
  - Lines 3158-3174, 2978-2979, and 3016-3017 describe TUI+AXE restarts and "restart
    AXE". Make them restarts of the service host or the `scheduler` proc.
- `docs/plugins.md`: the sample output at line 253 ("Axe restarted (pid …)") and the
  restart wording at lines 281-302, 337-338, 412-413, and 475. Fix
  `src/sase/plugins/_required_gate_preview.py` (about line 29) too.
- `docs/integrations.md`, lines 418-419: chat-install recovery. `chat-restart` may
  already have rewritten this. If so, only confirm it.
- `docs/perf_runbook.md`:
  - Lines 224 and 273 use `sase axe status` for spawn rate and "Job load". Point them at
    `sase axe routine status` and `metrics.json` instead.
  - Line 437: `current_tab "axe"` becomes `"services"`.
  - Line 867: "AXE tabs".
- `docs/vcs.md`, lines 125 and 746-754: `sase axe --vcs-provider hg start` never reaches
  the host-run scheduler. Document `SASE_VCS_PROVIDER` in `service.procs.scheduler.env`,
  and keep `-v` for `sase axe job|routine run`.
- `docs/query_language.md` line 6, `docs/rust_backend.md` line 88,
  `docs/notifications.md` lines 693 and 842, `docs/mentors.md` line 9,
  `docs/troubleshooting/runner-slots.md` lines 111 and 117, `docs/llms.md` lines
  2308-2309, and `docs/images/sase_tui_tabs_infographic.prompt.md` line 9.

The help-modal string changes alter rendered help text. Run `just fix-tui-screenshots`
and inspect every updated `help_*` golden before applying. Run `just fmt` and
`just lint`.

## Risks

- **Over-deletion in `chat-restart`.** `_process_start.py` still holds the
  canonical-executable logic that `sase service init` puts into `ExecStart`. Delete
  helpers one at a time, with evidence for each, and run the service executable and
  platform tests before finishing.
- **Doctor regressions in `desired-state`.** `axe.health` and `checks_deep_axe` are the
  operator's last remaining whole-scheduler health view (sase-153 tracks bringing back a
  CLI view). Cover each of these cases with a test: host wants the scheduler running and
  it is live; it is live but was deliberately stopped; it is down unexpectedly; the host
  is not installed at all.
- **Scripts passing removed options.** `scheduler-cli` turns silently ignored options
  into argparse errors. Grep the linked repos and `tools/` first. This is the same
  trade-off that flag-removal accepted for `--verify-timeout` / `--force`.
- **Goldens.** `tui-dead-state` should move nothing. `docs-surfaces` changes help text.
  Generating new goldens does not count as approving them.
