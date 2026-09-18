---
tier: tale
title: Platform units and init integration
goal: Install and manage the beta SASE service host safely through systemd or launchd,
  with a captured host environment, machine-scoped onboarding, migration diagnostics,
  and no competing legacy supervisors
size: medium
proposed_by: bbugyi200.athena.sase-11y.5
bead: sase-11y.5
status: done
---

- **PARENT:**
  [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:**
  [sase-11y.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.5.md)

# Plan: Platform units and init integration

Complete phase `sase-11y.5` against the service-host runtime already present in
`src/sase/service/`. Keep the work behind the existing `service_host` beta flag and
preserve the detached fallback when no platform unit is installed. The installed unit
must run a stable, non-ephemeral `sase` executable and own exactly one foreground
`sase service run`; do not add one native unit per service proc.

## 1. Add a deterministic platform-installation model

- Add focused service modules for the shared environment contract and native platform
  integration. Keep planning/inspection pure and dependency-inject command execution so
  unit tests never call the developer's real `systemctl`, `loginctl`, or `launchctl`.
- Resolve a stable SASE executable using the existing canonical AXE-start logic, moving
  the reusable resolution into a shared helper instead of duplicating the ephemeral
  `sase_<N>` detection. Treat a missing stable executable as an install blocker, and
  make installed-unit executable drift visible to `--check`, status, and doctor.
- Derive the native identity and paths from the effective SASE home. The default home
  uses `sase.service` and `sh.sase.service`; a non-default `SASE_HOME` is refused unless
  `--force` is supplied, then uses a deterministic home-derived suffix so it cannot
  overwrite the default installation. Include the non-secret alternate home/unit
  identity in the native definition or invocation so the process can find its captured
  environment before normal configuration loads.
- Render Linux `~/.config/systemd/user/<name>.service` with `Type=exec`, the stable
  `ExecStart=... service run`, `Restart=on-failure`, `RestartSec=5`, `KillMode=mixed`,
  and `WantedBy=default.target`. Do not add `network-online.target`, and do not put
  secrets in the unit.
- Render macOS `~/Library/LaunchAgents/<label>.plist` with `RunAtLoad`, `KeepAlive` for
  unsuccessful exits, `AbandonProcessGroup=true`, explicit host stdout and stderr paths,
  and `ThrottleInterval=10`. Use `launchctl bootstrap/bootout` in the user's `gui/<uid>`
  domain; detect a background item that the user disabled and report it without
  repeatedly bootstrapping it.
- Inspect both desired content and native-manager state. Detect the legacy owners
  `sase-gateway.service`, `sase-axe-ensure.service`, `sase-axe-ensure.timer`, and
  `sh.sase.gateway`; surface exactly what must be stopped/disabled during migration so
  the new host and a legacy supervisor never own the same child. Make apply ordering
  safe: write the new definition and environment, reload/bootstrap the manager, retire
  detected legacy units, then enable/start the SASE unit; on a failed apply, return an
  actionable error rather than silently falling back to a second detached host.
- On Linux, inspect `loginctl show-user <user> -p Linger --value`. A false or
  unavailable result warns with the exact `loginctl enable-linger <user>` remediation
  and never invokes privileged elevation automatically.

## 2. Capture and load the host environment safely

- Add `~/.sase/service/env` as a deterministic, atomically replaced `0600` file under a
  private service directory. Capture only the invoking shell's `PATH`,
  `SASE_FEATURE_FLAGS`, registered providers' `SASE_<PROVIDER>_PATH` overrides and
  declared `api_key_env_vars`, and the environment variable named by
  `mobile_gateway.fcm_credential_env`; include a forced alternate `SASE_HOME` when
  applicable. Never snapshot the whole shell environment, print secret values, or embed
  those values in systemd/plist content.
- Define and test one strict escaping/parsing format that round-trips spaces, quotes,
  empty values, and newlines without evaluating shell syntax. Reject malformed entries
  with a diagnostic that names only the variable, not its value.
- Load the captured file at the earliest safe point of `sase service run`, before the
  beta-feature gate and provider/plugin/config discovery, then let service-proc child
  environments inherit it. Ensure interactive commands other than foreground `run` do
  not unexpectedly replace their current environment.
- Build readiness diagnostics against the captured environment: resolve every registered
  provider CLI through the provider registry's declared CLI/path override, check the
  configured gateway executable where applicable, and report feature flags that exist
  only in the interactive shell. Missing tools/credentials are warnings with
  remediation, not secret-bearing output.

## 3. Implement service init, uninstall, and native lifecycle control

- Replace the phase placeholder handlers with real plan/apply flows for
  `sase service init` and `sase service uninstall`. Give every public long option its
  short alias and keep help/options sorted: `-c/--check`, `-d/--diff`, `-f/--force`, and
  `-y/--yes` as applicable. Planning must be read-only; `--check` returns success only
  when unit content, captured environment, manager state, legacy ownership, executable
  path, and required readiness checks are current; `--diff` shows content/manager
  actions without exposing captured secret values.
- Make apply idempotent. Installation creates/updates the environment and native
  definition, performs the platform manager reload/bootstrap/enable/start sequence, and
  clears the service-init decline marker. Uninstall stops/unloads and disables the unit,
  removes only SASE-owned native files, reloads the manager as needed, leaves durable
  service/proc history intact, and records the deliberate opt-out so bare onboarding
  does not immediately nag to reinstall.
- Route root `sase service start|stop|restart` through the installed platform manager.
  Preserve today's lock-guarded detached behavior only when no platform unit is
  installed. Never fall back to detached mode after an installed native-manager action
  fails. Include the detected unit identity in service-host status/heartbeat output.
- Keep `sase service status` useful when the host is down by appending installation,
  legacy-owner, linger/background-item, environment, provider-CLI, gateway, and
  executable-drift diagnostics. Do not perform writes from status.

## 4. Generalize machine-scoped `sase init` steps

- Extend `InitCommandSpec` with an explicit scope (`project` by default, `machine` for
  remote-machine enrollment and the new service step). Replace the
  `machine_offer_handled` one-off with generic invocation-local scope tracking so every
  machine-scoped spec is planned/prompted/applied at most once during `sase init --all`
  or `--project`, while project-scoped specs still run per selected project. Preserve
  stable execution order, JSON/check summaries, cancellation, deferred chezmoi deploy,
  and the existing machine-assessment cache behavior.
- Register a machine-scoped Service planner/runner that reuses the exact direct-command
  installation plan. It is inactive when the `service_host` beta flag is off, honors
  check/diff/yes modes, and uses a service-state marker to remember an interactive
  decline. Add a small generic decline hook to the init spec/coordinator if needed so a
  skipped Service prompt can persist that marker without adding another command-name
  special case. `--yes` remains explicit consent to apply all offered init steps.
- Ensure machine-scoped processing does not depend on a project checkout cwd and is not
  repeated for every target in a multi-project batch. Update the onboarding tests that
  currently encode the machine-only special case, including mixed machine/project specs
  and all/check/json paths.

## 5. Add doctor coverage and focused verification

- Register a stable `sase doctor` check for service-platform readiness, reusing the same
  read-only assessment as init/status. It should skip clearly while the beta flag is
  disabled; otherwise distinguish current, drift/warnings, and blocking errors and
  provide specific next commands without leaking environment values.
- Add Linux and Darwin unit-render/assessment/apply/uninstall tests with temporary homes
  and fake command results, including permissions, stable-executable drift,
  non-default-home refusal/suffixing, linger, user-disabled launchd items, legacy-unit
  detection/retirement, idempotence, manager failure, and uninstall preservation of
  state/history. Add environment round-trip/redaction and host-load timing tests,
  parser/handler tests, lifecycle routing/fallback tests, doctor tests, and
  machine-scoped onboarding tests.
- Run the focused service, parser/handler, onboarding, and doctor suites, then follow
  the repository's required lint/test memory and run its prescribed checks (at minimum
  `just check`). Re-run `sase bead epic-symbols sase-11y.5`; remove this phase's
  Justfile allowances once `clear_service_marker`, `service_dir`, `service_state_path`,
  and `set_service_marker` are genuinely used, or re-key any intentionally deferred
  symbol to a still-open later phase/parent. Do not leave an allowance keyed to the
  closing phase.
- Record any genuinely out-of-scope discovery only as `PROPOSED FOLLOW-UP: ...` on
  `sase-11y.5`; do not create a bead. After all checks and symbol cleanup pass, close
  only `sase-11y.5` with a note naming the verified suites and native-platform
  behaviors. Do not close `sase-11y` or any ancestor.
