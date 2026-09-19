---
tier: tale
title: Fix Python-hosted detached sudo relaunch
goal:
  Detached sudo execution relaunches the reviewed runner correctly after authentication
  when sase_sudo_runner is hosted by Python.
size: medium
proposed_by: bbugyi200.athena.0nf.f0
create_time: 2026-09-18 20:19:49
status: wip
---

# Fix detached sudo relaunch from the Python-hosted runner

## Goal

Make detached `sase sudo` execution work when `sase_sudo_runner` is installed as the
`sase-core-rs` Python console script. Authentication must still happen once on the
controlling terminal, after which the root executor and detached root worker must start,
publish the validated started handshake, and execute only the reviewed manifest.

## Confirmed failure

The detached runner currently asks `std::env::current_exe()` for the executable that
should be relaunched with `--internal-root-exec` and later `--internal-root-worker`.
That is correct for the native Rust binary, but the published `sase_sudo_runner` console
script calls the runner through PyO3, so `current_exe()` is CPython. After successful
PAM authentication, sudo therefore executes `python --internal-root-exec ...`; Python
rejects the private runner argument, the Rust layer reports runner error 14, and the
gate remains pending. The existing detached tests inject a fake runner path and
consequently do not exercise this production-hosted path.

## Implementation

1. In the linked `sase-core` repository, replace the assumption that a runner relaunch
   is only an executable path with an internal launcher description: an executable plus
   fixed prefix arguments. Centralize construction of a `Command` from that description
   so both privilege transitions use the same ordering:
   - sudo root executor: `sudo ... -- <program> <prefix...> --internal-root-exec ...`
   - detached root worker: `<program> <prefix...> --internal-root-worker ...`

   Preserve the native runner behavior as `<current_exe>` with an empty prefix. Preserve
   the existing test injection seam, adapting it to the launcher description rather than
   silently falling back to `current_exe()`.

2. Add a host-aware Rust entry point in `sase_gateway` for the PyO3 caller. The PyO3
   binding should obtain the active Python environment's absolute `sys.executable` and
   configure the launcher as:

   ```text
   <sys.executable> -I -m sase_core_rs.sudo_runner
   ```

   `-I` is required: the privileged relaunch must not import a lookalike `sase_core_rs`
   package from the reviewed working directory, `PYTHONPATH`, or the user site. The
   installed uv-tool/venv interpreter must continue to find its own `sase-core-rs`
   package in isolated mode. Return a runner error with a useful diagnostic if Python
   does not provide a usable absolute executable. Keep the public Python console-script
   invocation and the `sudo_runner_main(args)` Python-facing signature compatible.

3. Do not expose the launcher program or prefix as public sudo-runner command line
   options. They are host configuration, not reviewed-manifest data; allowing a caller
   to select an arbitrary program that sudo will execute would create an avoidable
   privilege-boundary hazard. Do not weaken manifest digest checks, handoff-path
   validation, environment clearing, process identity validation, or started-handshake
   publication.

4. Update nearby API documentation and comments so the two supported hosting modes are
   explicit: the native Rust binary relaunches itself, while the PyO3 console runner
   relaunches through its isolated Python environment.

## Regression coverage

Extend `sase_gateway` tests so they fail under the old path-only/current-exe
implementation and cover both relaunch hops:

- A native launcher still places `--internal-root-exec` and `--internal-root-worker`
  directly after the injected executable.
- A Python-shaped launcher preserves the exact `-I -m sase_core_rs.sudo_runner` prefix
  before each private internal mode argument.
- The detached success path still returns a schema-valid `sudo_exec_started` handshake,
  while root-spawn failures remain ledger-shaped runner errors and leave no false
  started handshake.

Add focused `sase_core_py` binding coverage for launcher derivation from the embedded
interpreter. Assert that the program is the active absolute Python executable and the
fixed prefix is isolated module execution; retain the existing help/binding smoke test
to prove the public binding remains callable. If the launcher derivation is factored
into a helper for testability, keep that helper private to the binding crate.

## Validation

From the linked `sase-core` repository:

1. Run the focused `sase_gateway` detached-runner tests and the relevant `sase_core_py`
   binding test while iterating.
2. Run `just check`, the repository's required full formatter, clippy, and
   workspace-test gate. Do not substitute a single-crate Cargo test because it would
   omit the PyO3 binding coverage.
3. Run the installed-environment smoke equivalent of
   `<venv-python> -I -m sase_core_rs.sudo_runner --capabilities` and confirm it emits
   the detached-execution capability. This smoke must not require sudo or mutate a gate.

## Acceptance criteria

- A detached request launched through the Python console runner no longer constructs
  `python --internal-root-exec` or `python --internal-root-worker`.
- Both internal relaunches use the same absolute interpreter and isolated module prefix,
  with private mode arguments appended afterward.
- The native Rust runner remains self-relaunching and synchronous `--no-detach` behavior
  is unchanged.
- No user-controlled CLI option can replace the privileged launcher.
- All focused tests and `sase-core`'s `just check` pass.

No primary `sase` CLI behavior or configuration change is required. The fix is owned by
`sase-core`; its existing release and core-revision ratchet workflows remain responsible
for propagating the corrected wheel into `sase` after the core change lands.
