---
tier: tale
title: Resolve plugin console scripts at the service-host spawn seam
goal:
  The service host can launch a `command:` proc whose executable is a console script
  installed by a `uv tool install --with` plugin, so `telegram_receiver` runs instead of
  crash-looping on `[Errno 2] No such file or directory`.
size: medium
proposed_by: bbugyi200.athena.0o1
create_time: 2026-09-20 10:45:37
status: wip
---

# Resolve plugin console scripts at the service-host spawn seam

## Objective

`telegram_receiver` is stuck in `crash_loop` on athena with
`Spawn error: [Errno 2] No such file or directory: 'sase_job_tg_inbound'`. The service
host is the only SASE process-spawn seam that hands a bare console-script name straight
to `subprocess.Popen`, and a plugin's console scripts are never on the host's `PATH`.
Give that seam the same interpreter-sibling fallback the three other spawn seams already
have, and warn about an unresolvable launcher at `sase service init` time instead of
only after three crashes.

## Diagnosis (already established — do not re-derive)

The evidence below was gathered on athena on 2026-09-20; treat it as given.

**1. The executable genuinely is not on any `PATH` the host sees.**

`sase` is installed as a uv tool with four injected plugins
(`/home/bryan/.local/share/uv/tools/sase/uv-receipt.toml`). uv symlinks only the
_primary_ package's entry points into `~/.local/bin` — every `entrypoints` row in that
receipt carries `from = "sase"`. Plugin-contributed scripts exist **only** in the tool
venv's bin directory:

| script                                                              | `~/.local/share/uv/tools/sase/bin/` | `~/.local/bin/` (on `PATH`) |
| ------------------------------------------------------------------- | ----------------------------------- | --------------------------- |
| `sase_job_hook_checks` (core)                                       | yes                                 | yes                         |
| `sase_job_tg_inbound` (plugin)                                      | yes                                 | **no**                      |
| `sase_job_tg_outbound`, `sase_chop_tg_*` (plugin)                   | yes                                 | **no**                      |
| `bugyi_chop_ci_watch`, `bugyi_chop_toobig_split` (plugin)           | yes                                 | **no**                      |
| `sase_gateway`, `sase_sudo_runner`, `sase_federation_worker` (Rust) | yes                                 | **no**                      |

Neither `PATH` the host can hold contains the tool venv bin: the systemd unit inherits
`systemctl --user show-environment`'s `PATH`, and the captured `~/.sase/service/env`
`PATH` is the authoring shell's. So `Popen(["sase_job_tg_inbound", "--receiver"])` can
only ever raise `OSError(ENOENT)`.

**2. Three other spawn seams already compensate; the service host does not.**

- `src/sase/axe/chop_script_runner.py:40` `discover_chop_script` — search dirs →
  `Path(sys.executable).parent` → `PATH`. Its inline comment names this exact failure
  mode. This is why the scheduler's plugin jobs (`ci_watch`, `tg_outbound`,
  `toobig_split`) are healthy.
- `src/sase/integrations/mobile_gateway.py:361` `_resolve_gateway_command` — `PATH` →
  `_python_environment_command` (interpreter sibling). This is why the `gateway` proc is
  healthy despite `sase_gateway` also being off `PATH`.
- `sase-telegram`'s own `src/sase_telegram/executables.py:resolve_console_script`, used
  by `canonical_receiver_argv()`. This is why the _running_ receiver survived roughly
  ten `sase update` runtime-generation re-execs before it finally died.
- `src/sase/service/host_support.py:22` `entry_argv` — **no resolution at all**. It
  returns `tuple(launcher.argv)` verbatim to `_ServiceHost._launch`'s `Popen`
  (`src/sase/service/host.py:286`).

**3. Why it surfaced now.** `sase-telegram` `2f76876` (2026-09-18, released in 0.4.17)
registered `telegram_receiver` as a service-host proc with
`command: ["sase_job_tg_inbound", "--receiver"]` and made `ensure_receiver_running()`
bail out when the host owns the proc. That moved the receiver from a seam that resolves
(`resolve_console_script`) onto the one that does not. The 10:25:21 `uv tool install`
bumped the runtime generation; the live receiver re-exec'd itself one last time at
10:25:36, and once that process ended the host could not respawn it.

**4. Scope: this is not "all services".** `scheduler` (builtin, uses `sase_command()`)
and `gateway` (builtin, resolved) are running. Only `command:` launchers naming a
plugin-provided console script are affected — today that is `telegram_receiver` alone,
but the bug class covers any future plugin proc.

**5. The fix's target is real.** For the running host
(`sys.executable == /home/bryan/.local/share/uv/tools/sase/bin/python`),
`Path(sys.executable).parent / "sase_job_tg_inbound"` exists and is executable.

## Constraints and invariants

- **Resolve `PATH` first, interpreter sibling second.** The gateway's order, not
  `discover_chop_script`'s. This makes the change purely additive: every launcher that
  resolves today keeps resolving to exactly the same executable, and only the
  currently-failing case changes behavior. Do not reorder to sibling-first, and do not
  introduce a feature flag — there is no old branch worth keeping reachable.
- **Only rewrite `argv[0]`, and only when it is a bare name.** If `argv[0]` contains a
  path separator (`os.sep` / `os.altsep`) it is already explicit; pass it through
  untouched. Never touch `argv[1:]`.
- **Leave the Rust core alone.** `command:` → `argv` parsing is owned by
  `sase-core/crates/sase_core/src/service/config.rs:parse_command`, and
  `launcher_summary` by `service/status.rs:753`. Both are pure and declarative.
  Executable resolution depends on the _running host's_ `sys.executable` and live
  `PATH`, so it is a host-local spawn concern, not shared backend behavior — it stays in
  Python at the spawn seam. The wire format, the Rust API, and the displayed launcher
  summary must all be unchanged.
- **String-form commands stay as they are.** `command: "foo --bar"` becomes
  `("/bin/sh", "-lc", "foo --bar")`; `argv[0]` is already absolute, so resolution is a
  no-op and the inner name is still subject to the shell's `PATH`. That is a real
  remaining gap — document it and recommend the list form for plugin scripts rather than
  trying to rewrite shell text.
- **Builtin launchers are unaffected.** `scheduler` and `gateway` already resolve
  through `sase_command()` and `gateway_builtin_argv()`.
- Do **not** refactor the three existing duplicate resolvers into one shared helper in
  this tale; they have different signatures (`search_dirs`, Rust-target fallbacks) and
  folding them together would make a small fix unreviewable. File it through
  `/sase_new_task` as a follow-up instead.
- Do not change uv entry-point linking, the captured-`PATH` contract in
  `src/sase/service/env.py`, or anything in the `sase-telegram` repo.

## Implementation

1. **Add the resolver** in `src/sase/service/executable.py` (it already owns service
   executable resolution and `resolve_stable_sase_executable`).
   - Add a pure-ish helper, e.g.
     `resolve_launcher_argv(argv, *, which_fn=shutil.which, interpreter=None) -> ResolvedLauncherArgv`,
     returning the resolved argv plus an optional diagnostic string. Inject `which_fn`
     and the interpreter path so the unit tests never depend on the real environment,
     matching the injected-probe style of `sase/uv_tool/detect.py`.
   - Logic: if `argv` is empty or `argv[0]` contains a path separator, return it
     unchanged. Otherwise try `which_fn(argv[0])`; on a miss try
     `Path(interpreter).parent / argv[0]` and accept it only when `is_file()` and
     `os.access(..., os.X_OK)`. On a miss from both, return `argv` unchanged with a
     diagnostic naming the script, the interpreter bin directory searched, and the fact
     that a plugin console script is not linked onto `PATH` by `uv tool install`.
   - Export the new names from `__all__` so symvision stays green.

2. **Wire it into the spawn seam** in `src/sase/service/host_support.py:entry_argv`.
   Apply the resolver to the `launcher.argv` and `launcher.command`-as-list branches.
   Leave the builtin branches and the `/bin/sh -lc` string branch returning what they
   return today.

3. **Make the failure self-explanatory** in `src/sase/service/host.py`. The
   `except OSError` around `Popen` currently records bare `str(exc)`. When the resolver
   produced a diagnostic for this entry, append it to the recorded `spawn_error` so
   `sase service status` and the TUI Services tab say _why_ the name did not resolve,
   not just that it did not. Keep the original `OSError` text intact — same convention
   as the SSH-denial remediation sentence added in `6087c0a8e`.

4. **Preflight the launchers** in `src/sase/service/platform.py`. In
   `build_service_platform_plan`, next to the existing
   `warnings.extend(readiness_warnings(desired_env))` /
   `effective_ssh_agent_warnings(...)` block, add a warning per available _and_ enabled
   `command:` proc whose `argv[0]` resolves nowhere. Put the check itself in
   `service/executable.py` beside the resolver so `platform.py` stays under its toobig
   limit (the precedent `6087c0a8e` set when it moved the SSH probe into
   `service/ssh_agent.py`). This surfaces on `sase service init`; `sase service status`
   already reports the runtime failure, so do not duplicate it there.

5. **Tests.**
   - `tests/service/test_service_host_runtime.py`: extend the existing `_entry_argv`
     coverage — a bare name found on `PATH` resolves to the `PATH` hit even when a
     sibling also exists (ordering guard); a bare name absent from `PATH` but present
     next to the interpreter resolves to the sibling; a name with a path separator is
     passed through; a name found nowhere is passed through unchanged; `argv[1:]` is
     never modified; builtin launchers are unchanged.
   - New unit tests for the resolver in `tests/service/` covering the non-executable
     sibling, the empty-argv guard, and the diagnostic text.
   - `tests/service/test_service_host_runtime.py`: a `Popen` `OSError` for an
     unresolvable bare name records a `spawn_error` that carries both the OS text and
     the diagnostic.
   - `tests/service/test_service_platform.py`: the plan warns for an enabled,
     unresolvable `command:` proc and stays silent for a resolvable one, a disabled one,
     and an unavailable one.

6. **Docs.**
   - `docs/configuration.md` §`service`: in the `service.procs.<name>.command` row (or a
     short paragraph under the table), state that a bare `argv[0]` is looked up on
     `PATH` and then next to the running interpreter, that this is what makes plugin
     console scripts work under `uv tool install --with`, and that the string/shell form
     does not get that fallback.
   - `docs/init.md` §"Service host": document the new readiness warning and its remedy,
     alongside the existing SSH-agent readiness notes.

## Verification

- `just install` first (this workspace's venv may be stale), then `sase tool run check`.
  Do **not** run `just check-full`.
- Run `just fix` inline before handing any long verification to `/sase_monitor`.
- No TUI-rendered output changes, so PNG visual snapshots are not involved.
- Live confirmation requires the fix to be _installed_, since the running host executes
  `/home/bryan/.local/share/uv/tools/sase/bin/sase`. After landing: `sase update`, then
  `systemctl --user restart sase.service` (or let the host reconcile), then confirm
  `sase service status` shows `telegram_receiver` as `running` rather than
  `backoff`/`crash_loop`, and that `~/.sase/service/procs/telegram_receiver/output.log`
  gains fresh `Starting Telegram long-poll receiver` lines. Propose that restart through
  `/sase_gate` rather than running it unprompted.

## Acceptance criteria

1. `entry_argv` resolves a bare `argv[0]` to an absolute path when the executable is
   next to the running interpreter but absent from `PATH`, and is otherwise
   byte-for-byte identical to today's output for every launcher shape.
2. A `PATH` hit still wins over an interpreter sibling.
3. `sase service init` warns about an enabled `command:` proc whose executable resolves
   nowhere, before the host ever crash-loops on it.
4. A spawn `OSError` for an unresolved bare name records both the OS error text and the
   reason it could not be resolved.
5. The Rust core, the wire schema, and `launcher_summary` are unchanged.
6. `sase tool run check` passes.
