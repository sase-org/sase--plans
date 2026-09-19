---
tier: tale
title: Finish platform units and init integration
goal:
  Close remaining sase-11y.5 holes in the already-landed platform-unit implementation so
  systemd/launchd install is secret-safe, Darwin-safe, test-isolated, and fully covered,
  then close only this phase.
size: medium
proposed_by: bbugyi200.athena.sase-11y.5
bead: sase-11y.5
create_time: 2026-09-19 06:24:06
status: wip
---

- **PARENT:**
  [202609/service_host_1.md](https://github.com/sase-org/sase--plans/blob/main/202609/service_host_1.md)
- **BEAD:**
  [sase-11y.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.5.md)

# Plan: Finish platform units and init integration

Phase `sase-11y.5` already has a first implementation (`3fb42fa`, plan
`202609/service_platform_units.md`). The bead is still open because the landed tree
misses several phase invariants and the verification the original tale required. Do
**not** rewrite `src/sase/service/platform.py` or the init/onboarding wiring. Patch the
holes below, add the missing tests, verify, then close only `sase-11y.5`.

Parent decisions that still bind this work:

- One platform unit owns one foreground `sase service run`. Never one unit per proc.
- Units/plists never carry secrets; `~/.sase/service/env` is `0600` and the host loads
  it itself.
- Never two owners for one child. Legacy `sase-gateway.service` /
  `sase-axe-ensure.{service,timer}` / `sh.sase.gateway` must be retired **before** the
  new unit starts.
- Non-default `SASE_HOME` refuses install without `--force`, then uses a home-derived
  suffix.
- Tests never talk to the developer's real `systemctl`, `loginctl`, or `launchctl`.
- All new user-facing behavior stays behind the existing `service_host` beta flag.

Discovered follow-up is a `PROPOSED FOLLOW-UP:` note on `sase-11y.5` only. Do not create
beads. Do not close `sase-11y` or any ancestor.

## 1. Fix Darwin apply so it cannot fight the user or a legacy owner

`apply_service_init` currently writes files, bootstraps the LaunchAgent, then boots out
`sh.sase.gateway`. That is wrong in two ways:

- A macOS-13+ user-disabled background item must be **reported** by `--check` / `status`
  / doctor and **never re-bootstrapped**. Skip `_darwin_bootstrap` when
  `inspection.user_disabled` is true. Still write definition/env so content can
  converge, and still retire the legacy label.
- Bootstrap starts the new host immediately. Bootstrapping before retiring
  `sh.sase.gateway` can put two processes on port 7629. Order Darwin apply as: write
  definition and env → retire detected/known legacy labels → bootstrap unless
  user-disabled.

Keep Linux ordering as it is (daemon-reload → retire legacy → enable/start). Do not
silently elevate linger; keep the exact `loginctl enable-linger <user>` warning.

Pin Darwin detection against a fake `launchctl`:

- `launchctl print gui/<uid>/<label>` success ⇒ active, not user-disabled.
- A failing print whose combined output contains a disabled-state marker ⇒ inactive and
  user-disabled.
- Other failures ⇒ inactive, not user-disabled.

Rendered plist must keep `RunAtLoad`, `KeepAlive={SuccessfulExit: false}`,
`AbandonProcessGroup=true`, explicit stdout/stderr paths, `ThrottleInterval=10`, and
`ProgramArguments` pointing at the stable `sase` plus `service run`. Unit and plist
still get only `SASE_SERVICE_ENV`, `SASE_SERVICE_UNIT`, and optional `SASE_HOME` — no
captured secrets.

Add a short module-level Darwin smoke checklist as a docstring on the Darwin tests (the
mac host is often offline; do not add a new docs page). Checklist items: init
`--check`/`--diff`/`--yes`, `launchctl print gui/$UID/sh.sase.service`, user-disabled
Login Item stays disabled across a second `--yes`, and `sase service start` does not
fall back to detached while the plist exists.

## 2. Keep `--diff`, status, and doctor secret-safe

`service_init_plan` builds `plan.diff` from `render_service_environment(desired_env)`,
and `sase service init --diff` prints that text. Captured API keys and FCM tokens must
never appear there.

Redact env values on **both** sides of the unified diff (same `[captured]` token already
used by `CapturedServiceEnvironment.redacted_values`) so added/removed keys are visible
and values are not. Definition/unit diffs stay unredacted because they contain no
secrets. `sase init service` already feeds redacted `new_content` into `InitPlan`; keep
that.

`--check` remains read-only and succeeds only when unit content, captured env, manager
enablement/activity, legacy ownership, executable path, and readiness warnings are
current. Status and doctor reuse `service_init_plan` and must not write.

## 3. Make uninstall work for `--force` homes and prove it

`sase service uninstall` currently omits `-f/--force` and the handler never passes
`force` through. An alternate-home unit installed with `init --force` is then blocked
from uninstall. Add `-f/--force` (short alias required) to uninstall, alphabetically
with the existing `-c/--check`, `-d/--diff`, `-y/--yes` flags, and thread `force` into
`service_uninstall_plan` / `apply_service_uninstall`.

Uninstall must still: stop/disable or bootout, remove **only** the SASE-owned definition
file, reload on Linux, leave `~/.sase/service/state.json` and proc history intact, and
set the decline marker so bare `sase init` does not immediately nag to reinstall.

## 4. Guard the real platform managers under pytest

`inspect_native_service` / apply fall back to `_default_runner`, which really invokes
`systemctl`/`loginctl`/`launchctl`. Mirror the AXE lifecycle guard:

- If `pytest_context_detected()` and no injected `runner`, `_default_runner` must refuse
  (do not spawn, do not write the user's real units).
- Allow an explicit override env only for an isolated test of the guard itself,
  following `sase.axe._process_guard` (`SASE_AXE_ALLOW_LIFECYCLE_IN_TESTS`).
- Production CLI paths are unchanged.

Keep injecting fake runners in unit tests. The guard is the backstop the phase
description called "the pytest guard so tests never touch real units."

## 5. Fill the tests the first landing skipped

Extend `tests/service/test_service_platform.py` (and the existing env / doctor / parser
/ onboarding files) with temporary `HOME`/`SASE_HOME` and fake runners:

- Linux unit text: `Type=exec`, `Restart=on-failure`, `RestartSec=5`, `KillMode=mixed`,
  `WantedBy=default.target`, `ExecStart=... service run`, no `network-online.target`, no
  secret values.
- Linux linger warning uses the exact `loginctl enable-linger <user>` string and never
  runs a privileged command.
- Linux legacy detection **and** apply retirement for `sase-gateway.service`,
  `sase-axe-ensure.service`, `sase-axe-ensure.timer`, with retirement happening before
  enable/start.
- Darwin render, inspect, apply, uninstall, user-disabled skip-bootstrap, and
  `sh.sase.gateway` retirement-before-bootstrap.
- Idempotent second apply (no extra enable/start/bootstrap when already current).
- Uninstall preserves state/history; `--force` uninstall of a suffixed identity.
- Non-default home still blocks without `--force` and suffixes both unit and plist names
  deterministically.
- Env file stays `0600`; malformed parse errors name the variable, not the value.
- Host load: `sase service run` loads captured env with `override_existing=True`
  **before** the beta-flag gate (`src/sase/main/service_handler.py`); other service
  subcommands do not override the interactive environment.
- Readiness: missing provider CLI / gateway binary against captured PATH; feature flags
  present only in the interactive shell; no secret values in the warning text. If cheap,
  also warn when a configured provider CLI resolves in the live interactive PATH but not
  the captured PATH.
- Doctor: SKIP when `service_host` is off (already covered); WARN on drift; ERROR on
  blockers; OK when current; no leaked secrets.
- Parser: uninstall `-f/--force` plus the existing check/diff/yes aliases.
- Onboarding: a `scope="machine"` **service** spec is planned/prompted/applied exactly
  once under `sase init --all` across multiple projects, including decline-once via
  `decline_init_service`. Do not rely on the `name == "machine"` `__post_init__` special
  case; that leftover may stay as compatibility for the machine spec.

Do not regenerate TUI PNG goldens. This phase does not change rendered TUI output.

## 6. Verify, clean epic symbols, and close only this bead

1. Run `just install` if this workspace venv is stale, then `just fix` (or at least
   `just fmt`) before any check monitor.
2. Run the focused suites:

   ```bash
   just test tests/service/test_service_platform.py \
     tests/service/test_service_environment.py \
     tests/doctor/test_checks_service_platform.py \
     tests/main/test_parser_service_scheduler.py \
     tests/main/test_init_onboarding_all.py \
     tests/main/test_init_onboarding_reporting.py
   ```

3. Then `just check` (the repo default). If it escalates or outruns the turn, hand it to
   `/sase_monitor` with `TESTING`/`TESTED`. Do **not** run `just check-full` unless
   `just check` escalates or selection looks wrong.
4. Run `sase bead epic-symbols sase-11y.5`. This phase currently has no `--epic-symbol`
   lines (they were re-keyed to `sase-11y.7`). If any `sase-11y.5(...)` entries
   reappear, either give the symbol a real non-test consumer or re-key the Justfile line
   to a still-open bead (`sase-11y.7`, `sase-11y.9`, `sase-11y.10`, or parent
   `sase-11y`). Closing this phase with leftovers turns unrelated `just check` red. Do
   not close `sase-11y.7`'s allowances unless they are now genuinely used outside tests.
5. Close only this bead:

   ```bash
   sase bead close sase-11y.5 --note "<what you verified>"
   ```

   The note must name the native-platform behaviors (systemd/launchd writers, env 0600,
   linger warning, user-disabled skip, legacy retirement order, secret-safe diff, pytest
   guard, machine-scoped `--all` once) and the suites that passed. Do not close
   `sase-11y` or any ancestor. Parent-close language in the epic is evidence for the
   land agent, not authorization for this phase worker.
