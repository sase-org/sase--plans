---
tier: epic
title: Service host and Services tab
goal: 'Every SASE background process on a machine — the AXE scheduler, the mobile
  gateway, plugin daemons like the Telegram receiver, user daemons, and `!` background
  commands — is owned by one `sase service` host that a platform unit (systemd user
  unit on Linux, launchd LaunchAgent on macOS) starts at boot/login, controlled from
  one `sase service` CLI and one Services tab, with every legacy supervision path
  retired as its replacement lands.

  '
phases:
- id: detach-scope
  title: Cgroup escape helper for detached work
  depends_on: []
  size: medium
  description: 'detach-scope: add a detach_scope helper that lets agents, detached
    procs, monitors/gates, and gateway bridges escape the service unit''s cgroup (systemd-run
    scope on Linux, setsid on macOS), apply it at every detach point, and rename src/sase/procs/service.py.'
- id: core-service
  title: sase-core service foundations
  depends_on: []
  size: large
  description: 'core-service: add the Rust-side service.procs config composer, the
    proc-wire service block and query field, the retention exemption, the locked service
    state store with boot-scoped stops, the status wire, and the pure restart-decision
    function, plus PyO3 bindings and Python facades.'
- id: supervision-lib
  title: Extract the shared child-supervision library
  depends_on: []
  size: medium
  description: 'supervision-lib: extract the AXE orchestrator''s child-supervision
    logic (backoff, crash-loop detection, TERM->KILL, bounded log pump) into a shared
    module and re-use it from the orchestrator with no behavior change.'
- id: service-host
  title: Service host runtime and CLI
  depends_on:
  - detach-scope
  - core-service
  - supervision-lib
  size: large
  description: 'service-host: implement the sase service host runtime and the sase
    service / sase service proc / sase scheduler CLI behind a new service_host beta
    flag, with the scheduler builtin, orchestrator handover, and the detached no-unit
    fallback.'
- id: platform-units
  title: Platform units and init integration
  depends_on:
  - service-host
  size: large
  description: 'platform-units: add sase service init/uninstall with systemd and launchd
    unit writers, the captured host environment file, legacy-unit detection, linger
    and provider-CLI checks, and the machine-scoped sase init step prompted once under
    --all.'
- id: gateway-telegram
  title: Gateway builtin and Telegram plugin migration
  depends_on:
  - service-host
  size: large
  description: 'gateway-telegram: add the gateway builtin launcher and sase mobile
    gateway pair, default the gateway off in core config, and migrate the sase-telegram
    receiver to a plugin-declared service proc while retiring the rearm tick and core''s
    origin-string hack.'
- id: services-tab
  title: Services tab in the TUI
  depends_on:
  - core-service
  - service-host
  size: large
  description: 'services-tab: rename the AXE tab display label to Services, render
    service-proc nodes with routine/job nodes nested under the Scheduler node, add
    start/stop/enable/disable keys and the host status line, replace the footer pill,
    and wire gear exclusion plus the configurable -service Procs default query.'
- id: oneshots
  title: Migrate background commands to oneshot service procs
  depends_on:
  - service-host
  - services-tab
  size: medium
  description: 'oneshots: move the ! background-command flow onto the durable proc
    store as transient oneshot service procs with recorded exit codes and durable
    rerun, render them as a distinct oneshot section on the Services tab, and keep
    legacy slot dirs readable behind a sunset flag.'
- id: rollout
  title: Live migration on athena and apollo
  depends_on:
  - platform-units
  - gateway-telegram
  size: medium
  description: 'rollout: install the platform unit on both machines, migrate the hand-written
    gateway units and the Telegram receiver per the runbook, and run the restart-with-live-work
    release gate before the new host is declared authoritative.'
- id: sunset
  title: Sunset legacy paths, docs, and glossary
  depends_on:
  - services-tab
  - oneshots
  - rollout
  size: large
  description: 'sunset: retire the ensure timer, TUI direct-start, and scope-wrapper
    paths, formalize the sase axe alias, canonicalize the services tab id, remove
    the service_host beta flag, update docs, and land the new and edited glossary
    strands via the memory-write skill.'
proposed_by: bbugyi200.athena.0m3
create_time: 2026-09-16 14:41:54
status: wip
bead_id: sase-11y
---

- **PROMPT:** [prompts/202609/service_host_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/service_host_1.md)
- **BEAD:** [sase-11y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/README.md)

# Plan: Service host and Services tab

Generalize AXE into a machine-level service runtime. Today SASE has four bespoke
supervision arrangements: AXE self-supervises through a TUI start path and an optional
ensure timer; the mobile gateway runs under hand-written systemd units on athena and
apollo; the Telegram receiver is re-armed by a 5-second AXE job tick plus a hardcoded
origin-string check in core; and `!` background commands use nine fixed slots with no
recorded exit codes. This epic replaces all four with one architecture and deletes each
legacy path as its replacement becomes authoritative. **The one disqualifying outcome is
two supervisors owning the same child**: every phase that adds a new owner must retire
the old one in the same phase or explicitly hand ownership forward.

Every phase worker MUST read the consolidated research first:

```bash
sase artifact read research:202609/service_host_and_services_tab/service_host_and_services_tab.md "Implementing a phase of the service-host epic"
```

That report carries the full rationale, the verified live findings on athena/apollo, and
the resolved disagreements. This plan states the decisions; the report defends them.

## Vocabulary (use these names everywhere; bare "service" is ambiguous)

- **Platform unit** — the systemd user unit / launchd LaunchAgent that runs the host.
- **Sase service (host)** — the per-machine foreground process `sase service run` that
  the platform unit supervises. At most one per SASE home.
- **Service proc** — a named proc the host owns: builtin (`scheduler`, `gateway`),
  plugin-declared (`telegram_receiver`), or user-defined; mode `daemon` in v1. A
  **oneshot service proc** is transient, created only on request (`!` or
  `sase service proc run`), runs detached _outside_ the host's process tree, and is
  never restarted or replayed.
- **Scheduler** — the builtin service proc that is today's AXE orchestrator (routines
  run jobs). Per the naming research, `scheduler` — not "supervisor", which collides
  with the per-proc detached supervisor — is the name; AXE stays an accepted alias.
- The TUI tab display label becomes **Services** (plural, matching Agents/Artifacts).

## Architecture (decided; see research §5–§6 for alternatives and why they lost)

- **Two-level supervision, one platform integration point.** The OS per-user service
  manager owns exactly one foreground process, `sase service run`. The host owns all
  daemon service procs as direct children and must not daemonize. One-unit-per-proc is
  rejected.
- **Control plane is file-based**: a locked state store (runtime overrides, boot-scoped
  stops keyed to the boot id, heartbeat) plus an atomic, versioned status snapshot with
  a change token; the host reconciles ~1 s and accepts a SIGUSR1 nudge. No socket.
- **Every run is a durable Proc.** Each service-proc launch/restart is a distinct
  proc-store row carrying an additive `service` block `{name, mode, source}`. Named
  service-proc rows are exempt from generic retention and keep per-service last-N;
  transient oneshots stay under generic retention. No new proc `kind` values.
- **stop vs. disable:** `stop` is a runtime override that survives host restarts and
  clears only on `start` or the next machine boot (boot-id keyed); `disable` is
  persistent machine policy (config default + a fast machine-local override with visible
  provenance: "disabled here" vs. "disabled by sase_apollo.yml").
- **Config**: a flat, layer-merged `service.procs` map. Simple procs are one line
  (`tunnel: {command: "autossh -M 0 -N buildbox"}`; string → `sh -c`, list → argv);
  builtins use `builtin:` launchers that compute argv from existing config sections
  (`axe.*`, `mobile_gateway.*`) so nothing is duplicated; complex procs are arbitrary
  executables plus a tiny env contract (`SASE_SERVICE_PROC`, a state dir, an optional
  `status.json` `{summary, state, updated_at}` the tab surfaces). **No Python plugin
  entry-point API in v1** — plugins declare service procs through the `sase_config`
  layer they already ship (corpus-before-mechanism). Per-entry fields: `description`,
  `enabled` (default true; **plugin-shipped and gateway entries default false**),
  `mode: daemon` (only value accepted in `service.procs` in v1), `command` XOR
  `builtin`, `cwd`, `env` (supports `${VAR}` references; never embeds secrets in units),
  `restart` (default `on-failure`), `success_exit_codes`, `stop_signal`, `stop_timeout`,
  `after` (ordering only), `log_max_bytes`.
- **The host resolves config from machine-level layers only** — defaults, plugin layers,
  user config, machine overlay — never project-local `sase.yml`.
- **Environment contract**: platform units get only the service manager's environment,
  which today breaks provider CLIs (verified: nvm-installed CLIs and even `sase` don't
  resolve there). `sase service init` captures the invoking shell's `PATH` and
  explicitly-listed secrets into a 0600 `~/.sase/service/env` file the host loads
  itself; units and plists never carry secrets.
- **Rust core boundary** (per the rust_core_backend_boundary memory): config
  compose/validation, wire `service` block + query field + retention exemption, the
  locked state store, the status wire, and the pure restart-decision function live in
  sase-core; process spawning, platform-unit writers, and Textual stay in Python.

## Phase detail

### detach-scope — Cgroup escape helper for detached work

The critical constraint (research §2, verified live): on athena four stale
`sase-axe-*.scope` transient units hold job-launched agents and the Telegram receiver,
because everything a child spawns inherits its cgroup and systemd stops kill the whole
control group. Once the host is a platform unit, every `systemctl --user restart`
(triggered by `sase update`, flag changes, config saves) would kill all in-flight
agents. This phase ships the fix first because it also repairs today's leaked scopes.

- Add a `detach_scope` helper (seed: `src/sase/axe/systemd_scope.py`): when the current
  process runs inside a SASE-owned unit cgroup (or AXE scope), wrap the detaching child
  in `systemd-run --user --scope --collect` on Linux; on macOS use `setsid` (the
  platform-units phase adds `AbandonProcessGroup=true` to the plist). No-op cleanly when
  systemd-run is unavailable.
- Apply it at every detach point: the proc supervisor spawn (`src/sase/procs/spawn.py`),
  agent runner launches (`src/sase/axe/run_agent_*.py`), monitor spawns
  (`src/sase/monitor/spawn.py`), gate shells, and the gateway agent-bridge command path.
- Add cgroup regression tests (assert a detached child's cgroup differs from the
  launching unit's) plus unit tests for the unavailable/no-op branches.
- Rename `src/sase/procs/service.py` (proc submission plumbing, unrelated to the new
  meaning of "service") to a non-colliding name such as `src/sase/procs/submission.py`
  and fix imports.

### core-service — sase-core service foundations

All in the sibling sase-core repo (Rust + PyO3 bindings + tests), then bump the pinned
revision and add thin Python facades, per the rust_core_backend_boundary memory.

- **Config**: `service.procs` schema and a dedicated composer (modeled on
  `compose_axe_config`): merge each entry field-by-field across layers, **replace
  list-valued fields whole in every layer** (the generic merge concatenates lists, which
  would corrupt an overridden argv `command`), treat an explicit `enabled: false` as a
  value rather than a missing key, record each entry's source layer for provenance,
  reserve builtin names, and reject `mode: oneshot` in config (v1: oneshots are
  transient-only). Python loads it fail-closed like `load_axe_config()`. Add the
  `service:` key to `src/sase/config/sase.schema.json` (it sets
  `additionalProperties: false`) and defaults (including
  `tui.procs.default_query: "-service"`) to `src/sase/default_config.yml`.
- **Proc wire**: an optional additive `service` block `{name, mode, source}` on proc
  rows (`crates/sase_core/src/procs/wire.rs`, schema version bump), a derived `service`
  boolean plus `svc:<name>` field in the procs query dialect, and the retention change:
  named service-proc rows are exempt from the generic 100-row cap and keep per-service
  last-N; rows with `source: transient` stay under generic retention.
- **State store**: a locked `~/.sase/service/state.json` (runtime overrides including
  boot-id-keyed stops, decline/handover markers, host heartbeat metadata) with the same
  flock discipline as the proc store.
- **Status wire**: a schema-versioned host/proc status snapshot (atomic write + change
  token) consumed by CLI `--json`, the TUI, and later the gateway's mobile API.
- **Restart decisions**: a pure function (inputs: restart policy, exit status,
  `success_exit_codes`, recent history; outputs: restart/give-up + delay + reason) so
  the host, CLI, and TUI all explain "retrying in 12s" identically. Include crash-loop
  thresholds compatible with the supervision-lib phase.

### supervision-lib — Extract the shared child-supervision library

Extract the orchestrator's proven child-supervision mechanics from
`src/sase/axe/orchestrator.py` (`_LumberjackRestartState`, capped exponential backoff,
crash-loop detection → mark failed + notify once, SIGTERM→SIGKILL escalation, bounded
log pump) into a reusable module (e.g. `src/sase/supervision/`). Re-wire the
orchestrator onto it with **no behavior change** (existing orchestrator tests must pass
unmodified or with mechanical import updates). The service-host phase consumes the same
module, so this replaces duplication instead of adding a fourth supervision copy. Keep
restart _decisions_ delegating to the core-service pure function once both land; until
then the extracted logic keeps its current constants.

### service-host — Service host runtime and CLI

Create the `service_host` beta feature flag with `sase flag new` (read the sase_flags
memory first): this is epic scaffolding, removed by the sunset phase. All new
user-reaching behavior in this phase is flag-gated.

- **Host runtime** (`src/sase/service/`): single-instance lifetime flock + heartbeat
  under `~/.sase/service/`; a ~1 s reconcile loop (SIGUSR1 nudge) comparing desired
  state (effective enablement + runtime overrides) to actual children; config reload via
  the existing stat-token check — added procs start, removed procs stop, changed procs
  restart, and the host itself never restarts on reload; stable bounded per-proc logs at
  `~/.sase/service/procs/<name>/output.log`; child supervision through the
  supervision-lib module; restart policy from the core-service decision function with
  `restart: on-failure` + `success_exit_codes` semantics (a disabled receiver exiting 0
  must not flap). Every child launch records a durable proc row with the `service`
  block. The host must not daemonize; `sase service start` without a platform unit runs
  the lock-guarded detached fallback and `status` says plainly "running detached (no
  platform unit installed)".
- **CLI** (read the cli_rules memory: alphabetical listings, short aliases for public
  long options, options never required):
  - `sase service init|logs|restart|run|start|status|stop|uninstall` (`init`/
    `uninstall` are implemented fully in platform-units; this phase stubs them behind
    clear errors or lands them dark). Bare `sase service` prints help/status — never
    `run`; there is no root `list` child so the central bare-group delegation rule does
    not apply. `run` is the platform-unit ExecStart / foreground entrypoint, with help
    text that contrasts it against `proc run`.
  - `sase service proc disable|enable|list|logs|restart|run|show|start|stop` — bare
    `sase service proc` delegates to `list` via the central rule. `enable`/`disable`
    write the machine-local override (like `systemctl enable`, not chezmoi-synced); the
    declarative config value stays editable through the existing scope-rail entry
    editor. `show` displays enablement provenance. `stop` records the boot-scoped
    runtime override. `proc run` submits a transient oneshot through the detached
    `submit_proc_request` path (options like `--cwd/-c`, `--label/-l`, `--project/-p`,
    `--workspace/-w`).
  - `sase scheduler` command group: `run` is the foreground orchestrator entrypoint;
    `start|stop|restart|status` delegate to `sase service proc <verb> scheduler` when
    the host owns it. `sase axe` keeps working as an alias (formal sunset later).
- **Scheduler builtin**: a `builtin: scheduler` launcher whose argv is the orchestrator
  foreground entrypoint reading existing `axe.*` config. Handover: when the host starts
  owning the scheduler, it stops a running legacy orchestrator lock-holder first, then
  starts it as a host child — never two owners.
- **Tests before the flag ever defaults on**: concurrent-start, stale-lock, crash-loop,
  signal handling, config reload, stop-vs-restart race, and no-oneshot-replay (an
  unsettled oneshot found at host start is settled as unknown, never retried).

### platform-units — Platform units and init integration

- `sase service init` / `uninstall` as an idempotent plan/apply flow with `--check/-c`,
  `--diff/-d`, `--yes/-y`, matching the existing `InitPlan` model.
- **systemd writer** (`~/.config/systemd/user/sase.service`): `Type=exec`, stable
  `ExecStart` resolved to the canonical install (reuse the
  `should_reexec_axe_start_from_canonical` logic; never an ephemeral workspace path),
  `Restart=on-failure`, `RestartSec=5`, **`KillMode=mixed`**, `WantedBy=default.target`,
  no `network-online.target` (absent in the user manager; verified), nothing beyond
  systemd 255. Check lingering and print the exact `loginctl enable-linger` remediation
  without silently elevating.
- **launchd writer** (`~/Library/LaunchAgents/sh.sase.service.plist`): `RunAtLoad`,
  `KeepAlive={SuccessfulExit: false}`, `AbandonProcessGroup=true`, explicit log paths,
  `ThrottleInterval=10`, installed via `launchctl bootstrap gui/$UID`. A macOS-13+
  user-disabled Login Item is reported as such by `--check`/`status` and never
  re-bootstrapped in a loop. Unit-test against a fake `launchctl` and add a manual smoke
  checklist (the mac host is often offline).
- **Environment file**: `init` captures the invoking shell's `PATH` plus explicitly
  configured secret variable names into 0600 `~/.sase/service/env`; the host loads it
  itself so Linux and macOS share one mechanism. `init --check`, `status`, and
  `sase doctor` resolve every configured agent-provider CLI and the gateway binary
  against the host's effective PATH and flag interactive-shell-only variables.
- **Legacy detection**: `init` detects and plans the migration of `sase-gateway.service`
  (which holds port 7629), `sase-axe-ensure.{service,timer}`, and the `sh.sase.gateway`
  plist. A decline is remembered so `--check` doesn't nag.
- **`sase init` integration**: replace the `machine_offer_handled` special case with an
  `InitCommandSpec.scope = "machine"` attribute; machine-scoped steps (`machine`,
  `service`) run once per invocation outside the per-project loop, so `-a|--all` prompts
  exactly once. Add the `service` step to `init_registry.py`.
- **Safety**: `init` refuses under a nonstandard `SASE_HOME` without `--force`; under
  `--force` unit/plist names carry a home-derived suffix. Keep the pytest guard so tests
  never touch real units.

### gateway-telegram — Gateway builtin and Telegram plugin migration

- **Gateway builtin** (`builtin: gateway`): computes argv from `mobile_gateway.*` via
  `_prepare_mobile_gateway_launch`, adds the agent-/helper-bridge arguments the
  hand-written unit is missing, and **execs the gateway binary directly — never
  `sase mobile gateway start`**, whose post-health pairing POST would mint a fresh
  pairing challenge on every restart and write codes into the service log. Add
  `sase mobile gateway pair` (asks the running gateway for a challenge); when a
  host-owned gateway is enabled, `start` points at the host instead of fighting for
  port 7629.
- **Defaults**: the `gateway` entry ships `enabled: false` in core defaults and is
  enabled per-machine in the athena/apollo machine overlays (rollout phase applies
  them). A default-on loopback HTTP API on machines that never pair a phone is wrong.
- **Telegram** (cross-repo: open sase-telegram with the repo skill): the plugin ships a
  `sase_config` layer declaring `telegram_receiver` (`enabled: false` by default;
  `restart: on-failure` with `success_exit_codes` covering the clean
  exit-when-disabled). `ensure_receiver_running` no-ops when the service proc is
  configured, behind a sunset flag in the sase repo. Delete core's hardcoded
  `telegram-receiver` origin check (`src/sase/ace/tui/update_restart.py`). Migrate the
  `~/.sase/telegram_is_enabled` marker to config enablement **in the athena machine
  overlay only** — Telegram allows one `getUpdates` consumer per bot and SASE has no
  cross-machine lock, so the chezmoi-synced base config must never enable it. The offset
  file and receiver code are unchanged.

### services-tab — Services tab in the TUI

Read the tui_perf memory before implementing; new keybindings must also land in the
`src/sase/default_config.yml` keymaps (gotchas memory). Display-label rename only:
change `_TAB_DISPLAY_NAMES`/color in `src/sase/ace/tui/widgets/tab_bar.py` to "Services"
and accept `--tab services` as an alias; the internal tab id stays `axe` until the
sunset phase (≈667 visual snapshots regenerate on id changes — pay once).

Target sketch:

```text
 Services · host ● running 4d · sase.service
 ● Scheduler            running · 11 routines
 │  ▌ hooks …             (routine and job nodes, unchanged folding/actions)
 ● gateway              running · 127.0.0.1:7629
 ● telegram_receiver    running · sase-telegram
 ◌ tunnel               disabled here
 ✕ beta_thing           unavailable: plugin missing
── oneshots ──
 ▷ #1 pytest -k foo     running · 1m
 ✓ #2 make lint         exit 0 · 4m ago
```

- One node per **configured** service proc — running, stopped, disabled (dimmed, with
  provenance), and unavailable (with the reason inline). You cannot enable a node that
  isn't rendered. The host status line is chrome, not a node, and shows what to press to
  start the host when it is down.
- The Scheduler node is special only in presentation: today's routine/job nodes nest
  under it with existing folding, actions, output views, and scope-rail config editing
  retained (`bgcmd_list.py` + `actions/axe_display/` evolve into the service tree).
- Keys: `x` start/stop the selected proc, `r` restart daemon / rerun oneshot / run job,
  `!!` new oneshot, `!e` enable/disable (help text says it edits machine policy), `!x`
  host start/stop. The `Q` quit modal offers "stop Scheduler", not "stop host".
- Footer pill: `AXE` becomes a service-health pill (`SVC 3/3` / `SVC !`). A failed
  service proc stays loud here and in notifications — hidden from the gear must never
  mean invisible failure.
- **Gear + Procs pane**: the proc gear excludes rows carrying the wire `service` marker
  (and monitor rows, as today); delete the `session_id is None` miscount that shows the
  live receiver as a gear chip in every session. Procs pane: add the `service` boolean
  and `svc:<name>` field to the query dialect wiring, seed the query bar from
  `tui.procs.default_query` (default `-service`) **only when no committed query is
  persisted** — a user-cleared query stays cleared; include service metadata in the
  filter cache key.
- TUI flags: `--no-axe`/`--restart-axe` gain `--no-service`/`--restart-service` names
  (old names become sunset aliases).
- Consume the atomic status snapshot off-thread; generation-aware refresh; measure
  navigation p95 < 16 ms under a restart storm before calling the phase done.

### oneshots — Migrate background commands to oneshot service procs

- Rewire the `!` flow (`actions/axe_bgcmd.py` → `bgcmd.py`) onto the durable proc
  store's detached path (`submit_proc_request`) with a `service` block
  `{mode: oneshot, source: transient}`: recorded exit codes, kill, streaming, and a
  durable rerun replace the nine slot dirs. Keep the existing UX: project → workspace →
  command-history modals, `#1–#9` display indices via the existing `bgcmd-slot:<n>`
  concurrency keys, output view, kill/rerun/dismiss.
- **Limit active oneshots, not history**: a completed command never blocks a new one.
- Oneshots run **only on request** — never on host start, config reload, or crash
  recovery (unsettled rows settle as unknown). They run outside the host's cgroup via
  detach-scope, so host restarts don't kill them.
- Render the oneshot section of the Services tab (divider, `▷/✓/✗` glyphs, muted
  palette, exit-code chips), visually distinct from daemon nodes.
- Legacy slot dirs (`~/.sase/axe/bgcmd/`) stay readable behind a sunset flag until they
  age out; `sase service proc run` and `!` share one code path.

### rollout — Live migration on athena and apollo

Operational phase on the real machines (apollo via Tailscale SSH — read the tailnet
reference memory). Follow the research §9 runbook:

1. `sase service init --diff` on each machine (plans the unit + disabling the legacy
   `sase-gateway.service`), then apply; enable the gateway in each machine overlay
   (chezmoi-managed config).
2. Verify `curl 127.0.0.1:7629/api/v1/health` and controller `sase machine status`;
   rollback restores the legacy unit on failure.
3. On athena: kill the legacy receiver proc row, drop the `tg_inbound` job from the
   `telegram` routine in chezmoi config (`tg_outbound` stays), enable
   `telegram_receiver` in the athena overlay only; verify exactly one Telegram
   `getUpdates` consumer before and after.
4. Remove the chezmoi-managed gateway unit only after both machines pass. Stale
   `sase-axe-*.scope` units clear as their agents exit.
5. **Release gate** (mock tests cannot establish this): under the real unit, restart the
   host while three things are live — a job-launched agent, a running `!` oneshot, and a
   gateway-launched agent. All three must survive. Additionally, an agent launched for
   each configured provider must resolve its CLI and credentials under the host's
   captured environment.

### sunset — Sunset legacy paths, docs, and glossary

- Retire the parallel watchdogs (each behind its sunset flag where callers need a
  migration window, per the sase_flags memory): the ensure timer
  (`sase axe ensure install` + `_ensure_timer.py`), the TUI direct-start path, the
  axe-start `systemd-run --scope` wrapper, and the Telegram rearm tick's old branch.
  Remove the `service_host` beta flag (delete Off branches, close the flag bead).
- Formalize `sase axe` as a documented alias of `sase scheduler`; canonicalize the tab
  id `axe` → `services` (`tab_order.py`, session state migration, snapshot
  regeneration).
- Docs: `docs/axe.md` (scheduler + host split), `docs/ace.md` (Services tab, Procs query
  field), `docs/configuration.md` (`service:` section), `docs/plugins.md` (plugin
  service procs), `docs/remote_dispatch.md` / `docs/mobile_gateway.md` (replace the
  hand-written unit recipes with `sase service init`), `mkdocs.yml`.
- **Glossary** (authorized by the approving user's request; go through the
  `/sase_memory_write` skill, then `sase memory init`). Five new strands:
  - **Sase Service** (aliases: service host) — "The sase service is the per-machine host
    process (`sase service run`) that starts, restarts, and stops that machine's service
    procs, with at most one per SASE home. `sase service init` registers it as a
    platform unit — a systemd user unit on Linux, a launchd LaunchAgent on macOS — so it
    starts at boot or login; without one, `sase service start` runs it detached.
    systemd/launchd supervises the sase service itself, never each service proc. Say
    'service proc' for what it runs and 'platform unit' for its registration; bare
    'service' is ambiguous."
  - **Service Proc** — "A service proc is a named proc the sase service owns, declared
    under `service.procs` by core (builtin), a plugin's config layer, or the user, or
    created at runtime as a transient oneshot. Its mode is `daemon` — kept running under
    its restart policy — or `oneshot` — run once to completion. Control one with
    `sase service proc` or its Services-tab node; `enabled: false` in a machine overlay
    keeps it off that machine. Every launch is a distinct durable Proc carrying a
    `service` marker, hidden from the Procs tab and proc gear by default."
  - **Oneshot Service Proc** (aliases: oneshot, background command) — "A oneshot service
    proc runs once to completion and is never restarted. Transient oneshots are created
    at runtime by the TUI's `!` background commands or `sase service proc run`; they run
    outside the sase service's process tree, so restarting the sase service does not
    kill them. Distinct from a job, which a routine runs on a schedule."
  - **Service Node** — "A service node is one selectable row of the Services tab: a
    service-proc node, a oneshot node, or — nested only under the Scheduler's node — a
    routine node and its job nodes. A service-proc node is stable across process
    restarts; section dividers and the host status line are chrome, not nodes."
  - **Sase Scheduler** (aliases: scheduler, AXE) — "The sase scheduler is the builtin
    service proc behind `sase scheduler` that runs SASE's background automation: it
    starts one process per routine and restarts crashed routines, and routines run jobs.
    The sase service owns the scheduler's process lifetime; the scheduler owns only its
    routine/job tree. AXE is its former name and remains an accepted alias." And four
    edits: **Sase Node** generalizes to one selectable row of a hierarchical TUI view
    (keeping the Agents taxonomy, adding "on the Services tab, a service node");
    **Routine** and **Job** replace AXE with the scheduler as owner (keeping the
    Lumberjack/Chop aliases); **Proc** gains "Runs of service procs carry a `service`
    marker; the Procs tab hides them by default" and drops the now-misleading
    "background task(s)" aliases. Do not add "platform unit" or "service proc
    definition" strands — implementation vocabulary, not agent-facing terms.

## Cross-cutting rules for every phase

- Never leave two owners for one child; handover is explicit and tested.
- CLI work follows the cli_rules memory (alphabetical, short aliases, options never
  required, excellent `-h`).
- New/changed keymaps also update `src/sase/default_config.yml`.
- sase-core changes update the Rust wire/API, bindings, and tests in the core repo, then
  the pinned revision and Python callers here.
- Feature flags only through `sase flag new`; sunset flags for every deprecated branch
  that must stay reachable.
- Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
  bead (no new beads from epic phase workers).

## Risks

- **Silent boot failure** stalls automation invisibly: mitigated by
  `Restart=on-failure`/`KeepAlive`, a TUI banner with one-key host start, `sase doctor`,
  and a readable heartbeat error even while crash-looping.
- **Supervision depth** (platform → host → scheduler → routines → jobs → agents) is
  accepted; all layers share supervision-lib. Flattening routines into host children is
  deliberately deferred.
- **Deferred by decision** (corpus-before-mechanism): declarative healthchecks and
  `when:` conditions, configured (non-transient) oneshots, a Python plugin proc API, a
  socket control plane, gateway-exposed mobile service control, namespaced proc IDs.
