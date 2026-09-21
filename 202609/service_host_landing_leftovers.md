---
tier: epic
title: Finish the service-host epic leftovers found at landing
goal: 'Restarting the sase service host no longer kills detached scheduler, hook,
  and chat-install runners. The host runtime scenarios the epic plan required are
  covered by tests. Every TUI surface, help text, and doc describes the shipped Services
  tab, service host, and scheduler. No `sase-11y` epic-symbol whitelist entry remains.

  '
phases:
- id: detach-runners
  title: Escape the service cgroup at the remaining detached runner spawns
  depends_on: []
  size: medium
  description: 'detach-runners: route the CRS, fix-hook, and summarize workflow runners,
    the mentor runner, the checks runner, hook execution, and the chat-install update
    worker through `detach_scope`, so that a `sase.service` stop or restart (`KillMode=mixed`)
    no longer SIGKILLs them. Prove that recorded PIDs and the chat-install lock fd
    survive the wrap.'
- id: services-wording
  title: Make the TUI surfaces describe the Services tab and service host
  depends_on: []
  size: medium
  description: 'services-wording: relabel the quit modal''s host-restart option, fix
    the Services-tab footer `x` hint, and rename the remaining Axe/AXE wording in
    the help title, onboarding guides, process-select modal, command-palette category,
    doctor next steps, and the screenshot example. Then regenerate and inspect the
    affected PNG goldens.'
- id: stale-docs
  title: Rewrite the docs that still describe the pre-host model
  depends_on:
  - services-wording
  size: small
  description: 'stale-docs: rewrite the Telegram receiver paragraph in `notifications.md`
    for the `telegram_receiver` service proc, and drop the removed `service_host`
    flag from three blog posts. Fix the `chat_install.restart_attempts` schema description.
    Point the mobile runbook at `sase service proc restart gateway`. Align the ACE
    and plugin docs with the relabeled restart option.'
- id: host-runtime-tests
  title: Cover the service host runtime scenarios the epic plan required
  depends_on: []
  size: medium
  description: 'host-runtime-tests: add tests for these host scenarios: concurrent
    start, stale lock, signal handling (SIGTERM/SIGUSR1), config reload, stop-vs-restart
    race, host-level crash loop, and no-oneshot-replay. They must drive the real `_ServiceHost`/`ServiceHostLock`
    code under a temporary SASE home. Fix any host defect the tests expose.'
- id: epic-symbols
  title: Resolve the sase-11y epic-symbol whitelist and the stale start label
  depends_on: []
  size: small
  description: 'epic-symbols: make private or delete the seven service facades still
    whitelisted as `--epic-symbol "sase-11y(...)"` and drop those Justfile entries.
    Move sase-telegram''s service-config test onto the public `load_service_config`
    seam. Stop journaling every host-launched scheduler start as "axe start".'
proposed_by: bbugyi200.athena.sase-11y.land
parent_bead: sase-11y
create_time: 2026-09-21 07:19:24
status: wip
bead_id: sase-11y.11
---

- **PROMPT:** [prompts/202609/service_host_landing_leftovers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/service_host_landing_leftovers.md)
- **PARENT:** [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:** [sase-11y.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.11.md)

# Plan: Finish the service-host epic leftovers found at landing

Epic `sase-11y` (Service host and Services tab) has shipped all ten phases, including
the nested sunset epic. The land agent verified each phase against source and reviewed
the ~240 non-epic commits that landed while it ran. The review found the gaps below.
Each one is caused by the epic, so the epic cannot close until they are fixed. This plan
covers only that remaining work. The land agent of the parent epic resumes the landing
(close, `just symvision`, plan-file status) after this child epic lands.

Background for every phase: the per-machine host `sase service run` runs under a
platform unit. On Linux that is the systemd user unit `sase.service`, written by
`src/sase/service/platform_units.py`, with `KillMode=mixed`. That unit owns the
`scheduler`, `gateway`, and plugin service procs as direct children. `detach_scope`
(`src/sase/detach_scope.py`) is the helper that moves long-lived detached work out of
that unit's cgroup. It uses `systemd-run --user --scope --collect` on Linux and `setsid`
on macOS, and it is a no-op outside a SASE-owned unit. Read the glossary strands before
writing user-facing text:
`sase memory read glossary:sase-service glossary:service-proc glossary:sase-scheduler glossary:service-node glossary:oneshot-service-proc -r "<why>"`.

Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
phase bead. They do not create beads.

## detach-runners — Escape the service cgroup at the remaining detached runner spawns

**Problem.** The detach-scope phase wrapped the proc supervisor, agent launches,
monitors, gate shells, and the service host's own detached fallback. It missed the
runner spawns that live under the scheduler and the Telegram receiver. They still use a
plain `subprocess.Popen(..., start_new_session=True)`. A new session does not leave the
cgroup, and `KillMode=mixed` SIGKILLs everything left in the `sase.service` cgroup on
any unit stop or restart. Triggers include:

- `sase service restart`
- the TUI's restart-with-host path (`-R/--restart-service`, used by the quit modal's
  option 3 and by Admin Center updates)
- `sase service init` re-applying a changed unit
- systemd recovering a crashed host

An in-flight mentor, CRS, fix-hook, or summarize run is killed in every one of these
cases. These runs survived a legacy AXE restart. The rollout phase reported this as a
follow-up (sase-11y.9, SIGKILLed `sase_job_*` processes in the journal).

Spawn sites to wrap (line numbers as of this plan):

- `src/sase/ace/scheduler/workflows_runner/starter.py`: the three runner `Popen`s:
  - CRS runner, around line 263
  - fix-hook runner, around line 393
  - summarize runner, around line 497
- `src/sase/ace/scheduler/mentor_runner.py` (around line 115)
- `src/sase/ace/scheduler/checks_runner.py` (around line 157)
- `src/sase/ace/hooks/execution.py` (around line 153)
- `src/sase/integrations/chat_install.py`: the chat update worker (around line 110),
  reachable from the Telegram receiver service proc

**Approach.**

- Follow the existing idiom in `src/sase/procs/spawn.py` and
  `src/sase/monitor/spawn.py`:
  `launch = detach_scope(argv, description=..., unit_prefix=...)`, then
  `Popen(launch.argv, start_new_session=launch.start_new_session, ...)`.
- Choose a distinct `sase-<kind>` unit prefix per runner kind so leaked scopes are
  attributable.
- These callers write `proc.pid` into hook and mentor status suffixes and later probe
  liveness. `systemd-run --scope` registers the scope and then execs the command in
  place, so the PID is unchanged. Keep that guarantee tested, following the live-scope
  test pattern in `tests/test_detach_scope.py`.
- `chat_install` passes `pass_fds=(lock_fd,)`. Prove, with a live test gated like the
  existing live-scope test, that the inherited lock fd survives the systemd-run exec. If
  it does not, leave that one spawn unwrapped, add a comment explaining why, and record
  a `PROPOSED FOLLOW-UP:`.
- Unit tests: for each site, monkeypatch that module's `detach_scope` to return an
  escaped command, then assert that `Popen` receives the wrapped argv and the session
  policy.
- Outside a SASE unit, for example hooks launched from a TUI shell, `detach_scope` is a
  no-op. Assert that too, so TUI-launched behavior is unchanged.
- Do not wrap routine or job processes that the scheduler itself supervises. Those die
  with the scheduler by design.

## services-wording — Make the TUI surfaces describe the Services tab and service host

Rendered text the epic left stale or contradictory:

1. **Quit modal option 3** (`src/sase/ace/tui/modals/quit_options_modal.py`).
   - **Problem:** it is labelled "Restart TUI & scheduler" / "Reopen the TUI and restart
     the scheduler." The action actually relaunches with `-R/--restart-service`, which
     calls `restart_service_host()` (`actions/axe_display/_loader_refresh.py`), a full
     host restart.
   - **Keep the behavior.** A host restart is what reloads updated host, gateway, and
     plugin code. It is already documented in `docs/plugins.md` ("restarts sase's TUI
     and the service host through the same restart path as the `Q` restart action") and
     in the `-R` help ("Restart the service host on startup").
   - **Fix the text:** relabel both bindings, the row title, the row description, and
     the dim key hint to name the service host.
   - **Option 2:** it says "The scheduler keeps running." Reword it so it does not imply
     only the scheduler matters, for example "The service host keeps running."
   - **Option 1** ("Quit & Stop Scheduler") is correct. Leave it.
2. **Services-tab footer `x` hint** (`src/sase/ace/tui/widgets/_keybinding_bindings.py`,
   the `axe_current_view == "axe"` branch).
   - **Problem:** with no service proc selected, the footer advertises `x` as
     "start/stop service host". Bare `x` is a no-op there, and the host toggle is `!x`.
     Since the Services-tab key-routing fix, `x` on host chrome, empty selection, and
     nested scheduler rows does nothing.
   - **Fix:** show the host toggle under its real bang binding, or drop the entry.
   - **Tests:** update
     `tests/test_keybinding_footer_status.py::test_keybinding_footer_axe_bindings`.
3. **Help modal title.** `src/sase/ace/tui/modals/help_modal/binding_common.py`
   `TAB_DISPLAY_NAMES["services"]` is `"Axe"`, which renders "Axe Tab". Make it
   "Services" and update `tests/ace/tui/test_popup_panel_tab_switch_keymaps.py`.
4. **Agents onboarding tab rows** (`src/sase/ace/tui/widgets/agent_onboarding.py`
   `_TAB_ROWS["services"]`).
   - **Problem:** the row is still
     `("AXE", "#FF5F5F", "Monitor the Axe daemon and automation.")`.
   - **Fix:** match the tab bar's label and accent from `widgets/tab_bar.py`
     (`_TAB_DISPLAY_NAMES`, `_TAB_COLORS`). Prefer reading them over duplicating the
     literals. Describe the Services tab (service host, service procs, scheduler
     routines and jobs, oneshots).
5. **Services-tab guide** (`src/sase/ace/tui/widgets/axe_onboarding.py`).
   - **Stale strings:** "What is Axe?", "Axe is the daemon...", "Axe starts
     automatically with sase tui", "x: start or stop Axe (with the Axe row selected)",
     "edit the selected AXE config", "the full Axe guide", "mentors Axe keeps running".
   - **Rewrite** for the shipped model:
     - the host
     - service procs, including the Scheduler node with its nested routines and jobs
     - oneshots
   - **Keys:** `x` start/stop the selected service proc, `r` restart or rerun, `!x` host
     start/stop, `!e` enable/disable, `!!` new oneshot.
   - **Verify every claim against the code** before writing it, for example whether
     `sase tui` auto-starts the host and under which flag.
   - Class and module identifiers may keep their `axe` names.
6. **Process-select modal** (`src/sase/ace/tui/modals/process_select_modal.py`): the
   title "AXE Control" and the module docstring. Use Services vocabulary.
7. **Command palette category** `"Axe"`: `src/sase/ace/tui/commands/types.py`
   `CommandCategory` and `CATEGORY_ORDER`, plus the rows in
   `commands/_app_metadata_actions.py`. Rename it to `"Services"`, and update any help
   or palette tests that assert the category.
8. **Doctor next step.** `src/sase/doctor/checks_external_mirror.py` `_NEXT_STEPS_AUTH`
   says "in the AXE daemon's environment". Point it at the service host's captured
   environment (written by `sase service init`).
9. **Screenshot example.** `src/sase/main/parser_screenshot.py` shows
   `sase screenshot --keep -- -t axe`. Use `-t services` and regenerate any CLI help,
   spec, or completion snapshots that embed it.
   - The `tui_screenshot` reference memory has the same `-t axe` example. Change it only
     through the `/sase_memory_write` skill.

Then regenerate the PNG goldens these changes touch with `just fix-tui-screenshots`,
targeted after `--` (quit modal, help panel, Services-tab footer, onboarding, command
palette). Inspect the report's update groups before applying. Every pixel change must be
one of the intended wording or label changes. Read the `tui.md` reference memory before
changing TUI code.

## stale-docs — Rewrite the docs that still describe the pre-host model

This phase depends on services-wording so its prose matches the final UI labels.

- **`docs/notifications.md`, the Telegram receiver adoption paragraph** (around line
  1511).
  - **Stale claims:**
    - a job tick re-arms the receiver
    - the receiver "survives `sase axe stop`"
    - a package upgrade needs a manual receiver retirement "until sase-11w lands"
  - **Rewrite it** for the plugin-declared `telegram_receiver` service proc:
    - enabled only in the owning machine's overlay
    - controlled with `sase service proc stop|restart|disable telegram_receiver`
    - a TUI update restarts the host and therefore the receiver
    - a CLI `sase update` restarts only the scheduler, so a receiver upgrade then needs
      `sase service proc restart telegram_receiver`
  - Check the current sase-telegram behavior through the `/sase_repo` skill.
  - Check open task sase-11w's scope, and keep or remove the reference according to what
    is still true.
- **Blog posts.** Drop the `service_host` beta-flag / "legacy Axe daemon" clauses. The
  flag no longer exists.
  - `docs/blog/posts/structured-agentic-software-engineering.md` (around line 261)
  - `docs/blog/posts/hello-sase-your-first-15-minutes.md` (around line 130)
  - `docs/blog/posts/why-coding-agents-need-orchestration.md` (around line 403)
  - Historical posts such as `axe-background-daemon.md` and `changespecs-in-practice.md`
    stay as written.
- **`src/sase/config/sase.schema.json`, `chat_install.restart_attempts`.** The
  description still says "axe start attempts when axe is not running". Match the
  `docs/configuration.md` wording (scheduler start attempts through the service host).
- **`docs/mobile_mvp_runbook.md`** (around line 104). It tells the operator to restart
  `sase mobile gateway start` after an upgrade. Add the host-owned case:
  `sase service proc restart gateway`.
- **`docs/ace.md` and `docs/plugins.md`.** Make the `Q` restart and Admin Center
  update-restart descriptions use the relabeled "restart TUI and service host" wording.
- Sweep the other `docs/` pages for user-facing "AXE tab", "Axe daemon", or
  `service_host` text that contradicts the shipped system, and fix it. The `axe.*`
  config keys, the `sase axe` alias, and `axe_*` file names are legitimate.

## host-runtime-tests — Cover the service host runtime scenarios the epic plan required

The epic plan required these tests before the host became authoritative:
"concurrent-start, stale-lock, crash-loop, signal handling, config reload,
stop-vs-restart race, and no-oneshot-replay". None of them exist in Python. No test
touches `run_service_host`, `ServiceHostLock`, `_install_signal_handlers`,
`_reconcile_once`, `_settle_exit`, or `handover_scheduler`.
`tests/service/test_service_host_runtime.py` covers only spawn/argv/marker behavior, and
crash-loop is covered only at the decision-function level (`tests/test_supervision.py`,
`tests/service/test_service_restart.py`).

Code under test:

- `src/sase/service/host.py`: `_ServiceHost`, `run_service_host`, reconcile, spawn, and
  settle
- `src/sase/service/host_lifecycle.py`: `run_host`, signal handlers
- `src/sase/service/control.py`: `ServiceHostLock`, `start_service_host`,
  `stop_service_host`, the start lock
- `src/sase/service/host_support.py`: builtin argv, scheduler handover

Add scenario tests under `tests/service/` for:

- **Concurrent start:** two racing `start_service_host()` calls converge on exactly one
  host holding the lifetime lock.
- **Stale lock:** a leftover lock file or host record from a dead pid does not block a
  new host.
- **Signal handling:** SIGTERM stops the children with their configured stop signal and
  timeout, releases the lock, and exits cleanly. SIGUSR1 triggers an immediate
  reconcile.
- **Config reload:**
  - an added entry starts
  - a removed entry stops
  - a changed entry restarts
  - the host pid never changes
- **Stop-vs-restart race:** a stop marker racing a restart ends in a single child and a
  consistent state file, never two children.
- **Host-level crash loop:** a child that keeps exiting non-zero is restarted with
  growing backoff and the host stays up.
- **No-oneshot-replay:** an unsettled transient oneshot row present at host start is
  settled as unknown and never relaunched.

Test constraints:

- Use a temporary `SASE_HOME`, config layers that declare cheap real children (for
  example `python -c` sleepers), and bounded waits through the repo's sanctioned wait
  helpers. The `just check` test-wait lint rejects ad-hoc helpers.
- Never touch the platform lifecycle; the pytest guard in `platform_runner.py` must stay
  in force.
- Keep each test fast and deterministic. Prefer driving the reconcile loop directly over
  sleeping for its 1 s cadence.

If a test exposes a real host defect, fix it in this phase. Record anything out of scope
as a `PROPOSED FOLLOW-UP:`.

## epic-symbols — Resolve the sase-11y epic-symbol whitelist and the stale start label

The Justfile `_lint-symvision` recipe still whitelists seven
`--epic-symbol "sase-11y(...)"` entries (with a `# sase-11y entries: ...` comment). They
go stale when epic sase-11y closes. Read the `symvision` reference memory first. The
decisions are already made; apply them:

| Symbol                       | File                  | Consumers today                                                                             | Decision                                                                                                                                                                                                    |
| ---------------------------- | --------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CapturedServiceEnvironment` | `service/env.py`      | in-file return type only (plus tests)                                                       | privatize to `_CapturedServiceEnvironment`, drop it from `__all__`, update `tests/service/test_service_platform.py`                                                                                         |
| `ServiceFieldProvenance`     | `service/config.py`   | in-file only                                                                                | privatize, drop it from `__all__`                                                                                                                                                                           |
| `compose_service_config`     | `service/config.py`   | in-file (`load_service_config`), sase tests, sase-telegram's `tests/test_service_config.py` | privatize to `_compose_service_config`, drop it from `__all__`, update `tests/service/test_service_config.py`                                                                                               |
| `readiness_warnings`         | `service/platform.py` | in-file (`service_init_plan`) plus tests                                                    | privatize to `_readiness_warnings`, drop it from `__all__`, update the monkeypatch target strings and the import in `tests/service/test_service_platform.py` and `tests/service/test_service_executable.py` |
| `service_platform_supported` | `service/platform.py` | none                                                                                        | delete it, drop it from `__all__`, remove the now-unused `platform_kind` import                                                                                                                             |
| `resolve_service_enablement` | `service/status.py`   | none, not even tests                                                                        | delete it and drop it from `__all__`. Keep `_enablement_override_to_wire` and `_service_proc_config_to_wire`, which the status build still uses                                                             |
| `clear_service_enablement`   | `service/state.py`    | one test only; no CLI or TUI path clears an override                                        | delete it, drop it from `__all__`, trim the clearing half of `tests/service/test_service_state.py::test_setting_and_clearing_enablement`                                                                    |

- **sase-telegram test.** sase-telegram's CI installs sase from a git checkout, so its
  test must not import the privatized name.
  - Open sase-telegram with the `/sase_repo` skill.
  - Rewrite
    `tests/test_service_config.py::test_default_config_declares_disabled_service_receiver`
    to use the public `sase.service.config.load_service_config()`. Monkeypatch
    `sase.service.config.load_config_layers` to return the plugin layer it builds today.
  - That version passes against sase both before and after the privatization, and it
    becomes part of this phase's commit obligations.
- **Justfile.** Delete the seven Justfile `--epic-symbol "sase-11y(...)"` lines and
  their comment.
  - `sase bead epic-symbols sase-11y` must then print no entries.
  - `just _lint-symvision` must report no service symbol. The `sase-14j`
    `_agent_bead_touches.py` findings belong to another epic.
- **Keep `src/sase/procs/service.py`.** The compatibility facade has no in-repo
  importer, but sase-telegram's `src/sase_telegram/inbound.py` imports
  `sase.procs.service.submit_proc_request`.
- **Lifecycle start label.** `src/sase/axe/orchestrator.py` journals every orchestrator
  start with `source=os.environ.get("SASE_AXE_START_SOURCE", "axe start")`. Nothing sets
  that variable any more, so every host-launched scheduler start is recorded as "axe
  start".
  - Default the label to the real entrypoint (for example `scheduler run`).
  - Keep the env override that `tests/test_axe_lifecycle_journal.py` exercises.
  - Update any surface or test that asserts the old default.

## Verification (every phase)

- Run `just fix`, then `just check`. Do not run `just check-full`.
- Phases that change rendered TUI output must also run the targeted
  `just fix-tui-screenshots` and inspect the report.
- Phases that change a linked repo must also run that repo's `just check`.
