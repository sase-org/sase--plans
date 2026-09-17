---
tier: tale
title: Fix sudo working-directory execution without -D
goal:
  Run reviewed sudo commands from their requested directory without requiring sudo -D
  permission, with reliable failure cleanup and regression coverage.
size: medium
proposed_by: bbugyi200.apollo.0d
create_time: 2026-09-17 15:12:24
status: wip
---

# Fix sudo request execution without requiring sudo's -D permission

## Outcome and scope

Make reviewed `sase sudo` commands run from the requested working directory using
ordinary sudo policy, without injecting `-D`. Fix the Rust execution owner, add
behavioral regressions, verify the Python integration, and document directory and
recovery behavior. One coding agent can complete this bounded cross-repository repair;
it does not need an epic.

This plan covers the SASE execution bug. Installing `texlive-xetex` and verifying the
research-highlights PDF hook remain separate operational work for agent family `0c`. Do
not replay its settled gates, run a package install as a test, or change sudoers.

## Confirmed diagnosis

On 2026-09-17, all three approved `0c` sudo requests for
`/usr/bin/apt-get install -y texlive-xetex` recorded exit code 1 in about 0.027 seconds
with the same diagnostic:

> sudo: you are not permitted to use the -D option with /usr/bin/apt-get

Evidence can be recovered with `sase chat show --agent <member>` and
`sase sudo show <gate-id> --json`:

| Member       | Gate ID                                     | Reviewed working directory                |
| ------------ | ------------------------------------------- | ----------------------------------------- |
| `0c--gate`   | `sudo-599619e9-a524-4cef-85bc-0a7958f454bf` | `/`                                       |
| `0c--gate-0` | `sudo-afb3232a-14d4-4a1f-ad05-5fb8a76dee40` | The requesting agent's existing directory |
| `0c--gate-1` | `sudo-21237041-812a-4d4f-acd7-a0bc43177b11` | The requesting agent's existing directory |

The latest saved decision is
`~/.sase/chats/202609/gh_sase_org__sase-ace_run-0c__gate_1-20260917150439.md`. Its sudo
notification is `43ea4f19-6317-4360-ac51-652d47ad212f`, sender `sudo`, timestamp
`2026-09-17T15:04:39.645575-04:00`.

The rejection comes from sudo before apt-get starts. Changing or omitting request `cwd`
cannot fix it: Python normalizes an omitted directory, and Rust unconditionally adds
`-D <manifest.cwd>` in `run_approved_command`.

The host has Ubuntu sudo `1.9.15p5-3ubuntu5.24.04.2`. Its
[sudoers manual, Chdir_Spec and runcwd](https://manpages.ubuntu.com/manpages/noble/man5/sudoers.5.html#Chdir_Spec)
documents separate permission for selecting a directory with `-D`, such as `CWD=*` or
`runcwd=*`. Normally sudo inherits the invoking process's directory. The failure is the
runner's assumption that command authorization also permits `-D`; it is not an apt-get
option conflict or evidence of a package-manager failure. The exact local sudoers rule
was not read because those files are not readable unprivileged. The receipts establish
the effective rejection. No new privileged reproduction was run.

The reviewed sources contain the defect both at the current core checkout and at the
SASE-pinned core revision. Current Rust tests assert `-D` in expected argv, while their
fake sudo skips every option before `--`; they cannot expose this policy error.

## Ownership and files

Before reading or editing the other repository, use `/sase_repo` and run
`sase repo open sase-core -r "Implement reviewed sudo working-directory repair"`. Use
the returned checkout and its `AGENTS.md`; do not assume a sibling checkout path.

The implementation owner is `sase-core`:

- `crates/sase_gateway/src/sudo_runner.rs`: command construction, execution lifecycle,
  and the existing fake-sudo regression fixtures.
- `crates/sase_core/src/sudo.rs`: existing manifest and ledger contracts; inspect to
  preserve their behavior, with no schema changes expected.
- `crates/sase_core_py/src/lib.rs` and
  `crates/sase_core_py/python/sase_core_rs/sudo_runner.py`: existing binding and
  installed runner entry point; these delegate to the same Rust implementation.

In `sase`, use `docs/sudo.md` and the existing tests in `tests/test_sudo_runner.py`,
`tests/test_sudo_gate.py`, `tests/test_sudo_ssh.py`, and
`tests/test_sudo_acceptance.py`. The Python adapters under `src/sase/sudo/` should
remain thin. Do not duplicate command execution policy in Python.

## Implementation

1. Add a regression that reproduces the compatibility failure without real sudo. Extend
   the existing fake-sudo fixture so it rejects `-D`/`--chdir` in sudo's option region,
   before `--`. It must still allow an application's literal `-D` argument after `--`.
   Record actual child cwd and unambiguous argv boundaries, instead of relying
   exclusively on the fixture's current flattened `$*` log. This test should fail on the
   old runner. The fixture must not execute apt-get or acquire privilege.

2. In `run_approved_command`, remove the injected `-D` and directory argv elements.
   Apply `Command::current_dir(&manifest.cwd)` to the approved command's sudo child.
   Retain the absolute production `/usr/bin/sudo` path, optional cached-auth `-n`,
   `-u <run_as>`, `--`, and the reviewed command argv verbatim. Avoid process-global
   `set_current_dir`, shell wrappers, `/usr/bin/env` command wrappers, automatic
   retries, and apt-get-specific branches. Apply this on the execution host, so local
   requests and requests relayed through `sase sudo exec` share the same fix.

3. Keep authentication, cache probing, and timestamp invalidation independent of the
   requested directory. A missing, removed, or non-traversable cwd must fail before the
   approved executable runs; never fall back to the runner's directory. Report the
   existing runner-error outcome with a useful directory/startup diagnostic. Address the
   directly related cleanup path: after authentication, the current `?` propagation from
   command launch can bypass final `sudo -k`. Ensure cleanup is attempted on this error
   path before returning, while preserving the original failure. A failure during
   cleanup must not turn the run into a success. Keep the change focused on the sudo
   execution lifecycle, without inventing new statuses.

4. Preserve manifest hashing (including cwd), command selection, run-as, reviewed
   environment policy, PAM/TTY handling, process-group timeout/cancellation, output
   bounds, and stop-on-failure behavior. A post-authentication command exit failure
   still produces an approved gate with failed ledger entries. A runner startup failure
   keeps the gate pending. `SUDOED` means approval settled, not that apt-get succeeded;
   no gate-status redesign is part of this fix.

5. Update `docs/sudo.md` to explain that cwd is set before invoking sudo, must be
   accessible to the invoking user on the execution host, and normally becomes the
   command's directory. Administrator-configured `CWD`/`runcwd` remains authoritative;
   do not promise to override host policy. Root-only directories are not a supported
   fallback case. Explain the reported `-D` rejection and that changing request cwd
   cannot repair an old runner. A settled failed request needs a new reviewed request
   after deploying the fixed runner, rather than repeated approval of the old gate.

## Regression coverage

Use the existing Rust test-only executable injection; do not add a production
environment variable or CLI option that replaces `/usr/bin/sudo`.

- Exercise the runner end to end against the rejecting fake sudo with an apt-get shaped
  command. Assert successful ledger entries, no injected directory option, correct
  run-as and exact command argv, and the actual requested cwd. The fake must simulate
  the command, never execute the package manager.
- Cover cwd `/` and a distinct temporary directory containing spaces; verify a harmless
  relative-path read in that directory or an equivalent observable child behavior.
  Assert that the parent process cwd is unchanged.
- Cover both a successful cache probe (`-n` execution) and an expired/failed probe
  (interactive-auth path); neither may inject directory options.
- Cover a nonexistent cwd and a directory removed after authentication. Assert no
  approved command dispatch, runner failure rather than success, and final
  timestamp-invalidation attempt. Test a permission-denied directory when the test
  user's privileges permit a reliable assertion, avoiding root-dependent flakes.
- Retain the existing authentication failure, digest mismatch, timeout, ledger
  validation, environment clearing, stop-on-failure, and output-policy coverage.
- At the Python seam, use existing coverage and add only missing behavioral cases: exact
  cwd survives manifest handoff locally and over SSH, and a runner startup error leaves
  approval pending. Tests should exercise the contract rather than duplicate Rust's
  command-construction implementation.

## Build, delivery, and verification

Read `lint_and_test.md` through `/sase_memory_read` before changing SASE files. Follow
both repositories' instructions and use the standard build and finalizer paths.

1. Run `just check` or `./scripts/check.sh` in the opened `sase-core` checkout. Its
   canonical check includes formatting, clippy, and workspace tests with the PyO3
   binding; a standalone `cargo test -p sase_core` is insufficient and misses the
   gateway runner. Use Python 3.12 or newer as required by that checkout.
2. Build/install the changed core into the SASE workspace environment with its supported
   `just install`/`just rust-install` workflow. Verify which interpreter, `sase_core_rs`
   package, and `sase_sudo_runner` entry point are actually selected; an unchanged
   globally installed runner cannot demonstrate the fix.
3. Run the four focused Python sudo test modules, then `just check` in `sase` for
   tracked changes. Use `/sase_monitor` for long-running verification and follow the
   repository's `just check-full` rule if the eventual diff triggers it. Do not run
   visual tests for this nonvisual repair unless the implementation unexpectedly touches
   presentation.
4. Deliver the core fix through host-owned finalization and the established core
   revision/release adoption workflow. Account for `sase-core-revision.txt` and the
   installed core package: use an available committed revision containing the repair
   when advancing the pin, never an unrelated revision or an uncommitted placeholder. Do
   not manually bump Rust crate versions; release-plz owns them. No new wire API or
   binding is expected. Report any remaining core publication or installation step
   explicitly instead of claiming the running host is repaired from source edits alone.
5. After the host actually has the repaired runner, an optional real-host acceptance
   check uses a new `/sase_sudo` request with harmless exact argv such as
   `["/usr/bin/pwd"]` from an accessible distinct cwd and
   `["/usr/bin/apt-get", "--version"]`. Bryan approves through the normal terminal flow.
   Confirm cwd and successful ledger entries without modifying packages or sudoers. This
   human/PAM check must not block deterministic automated verification; state plainly
   whether it was performed.

## Acceptance criteria

- The pre-fix rejecting fake demonstrates the observed failure and passes with the
  repaired runner for both authentication paths.
- Approved argv stays exact, cwd is applied to the child, and the parent cwd stays
  unchanged. No privileged shell or sudoers policy expansion is introduced.
- Invalid cwd never dispatches the approved executable or reports success, and
  authenticated failure paths still attempt timestamp cleanup.
- Local and remote paths keep the existing manifest and ledger contracts and pass the
  required Rust and Python checks.
- Documentation and the completion report distinguish the confirmed sudo invocation
  defect, source/test repair, actual runner deployment, and any unperformed host smoke
  check. The package installation is not represented as completed by this repair.
