---
tier: tale
title: Fix sase-core macOS CI by gating Linux-only ETXTBSY replay tests
goal:
  sase-core master CI is green again on macOS. The real-kernel ETXTBSY replay tests run
  only on Linux, where their kernel semantics hold, and portable tests keep the retry
  policy covered on every unix platform.
size: small
proposed_by: bbugyi200.athena.178
proposed_with: claude/opus
create_time: 2026-11-03 09:35:44
status: wip
---

# Fix sase-core macOS CI: gate Linux-only ETXTBSY kernel replay tests

## Problem

sase-core's `CI` workflow has been red on `master` for every commit since `b0a63dc3`
(`fix(sudo-runner): serialize fixture script publication against spawns with ETXTBSY retry backstop`).
That is seven consecutive commits: `b0a63dc`, `f061f5f`, `45c6a26`, `89a5bd8`,
`65925ca`, `6cd64a9`, `f1bbb4f`. The only failing job is
`cargo fmt + clippy + test (macos-latest)`, at step `cargo test --workspace`. The ubuntu
job, the maturin smoke job, and the release-scripts job all pass, and fmt and clippy
pass on macOS too.

Every failing run fails the same two tests in `crates/sase_gateway`, and nothing else
fails (`test result: FAILED. 251 passed; 2 failed`):

```
---- sudo_runner::tests::etxtbsy::exec_while_fork_child_holds_write_descriptor_fails_with_etxtbsy ----
panicked at crates/sase_gateway/src/sudo_runner/tests/etxtbsy.rs:69:48:
called `Result::unwrap_err()` on an `Ok` value: ExitStatus(unix_wait_status(0))

---- sudo_runner::tests::etxtbsy::etxtbsy_retry_recovers_once_writer_closes ----
panicked at crates/sase_gateway/src/sudo_runner/tests/etxtbsy.rs:104:5:
expected at least one ETXTBSY retry, got 1 attempt(s)
```

Example run: https://github.com/sase-org/sase-core/actions/runs/41578286141

## Root cause

Both tests were added by `b0a63dc3` in
`crates/sase_gateway/src/sudo_runner/tests/etxtbsy.rs`. Both assume a Linux kernel rule:
`execve` of a file that some process holds open for writing fails with `ETXTBSY`. For a
`#!` script, Linux opens the script itself through `open_exec`/`deny_write_access`.

macOS (XNU) has no such rule. It execs a script that is still open for writing without
complaint. The CI evidence shows this directly:

- `exec_while_fork_child_holds_write_descriptor_fails_with_etxtbsy`:
  `Command::new(&script).status()` returns `Ok(exit 0)` even though a forked child still
  holds the write descriptor, so `unwrap_err()` panics.
- `etxtbsy_retry_recovers_once_writer_closes`: the first `command.spawn()` succeeds
  while the writer is held, so the retry closure runs once and `attempts >= 2` fails.

This is not a flake. The failure is deterministic on macOS and the production code is
correct: on macOS `retry_on_etxtbsy` never sees `ETXTBSY` and correctly returns the
first spawn. Only the tests encode a platform assumption. The third test in the file,
`publish_executable_lands_atomically_with_exec_permissions`, is platform-neutral and
passes on macOS.

## Fix

Work only in the linked `sase-core` repo. Open it with
`sase repo open sase-core -r "<reason>"` and use the printed path. No sase (Python)
change is needed. No binding or wire changes, so `sase-core-revision.txt` does not move.

### 1. Gate the two kernel-replay tests to Linux

In `crates/sase_gateway/src/sudo_runner/tests/etxtbsy.rs`:

- Add `#[cfg(target_os = "linux")]` above `#[test]` on
  `exec_while_fork_child_holds_write_descriptor_fails_with_etxtbsy` and on
  `etxtbsy_retry_recovers_once_writer_closes`. This is the gating pattern already used
  in `sudo_runner/tests/cli.rs`.
- Leave `publish_executable_lands_atomically_with_exec_permissions` ungated.
- Update the module doc comment. Say that the real-kernel replay tests are Linux-only,
  because Linux refuses to exec a file that is open for writing and macOS/XNU does not
  (on macOS the exec simply succeeds). Say that the portable tests below cover the retry
  policy on every unix platform.
- Do not weaken any assertion in the Linux tests. They stay exactly as they are and
  still run on the ubuntu CI job and on local Linux hosts.

### 2. Add portable tests of the retry policy (macOS keeps coverage)

Gating step 1 removes all `retry_on_etxtbsy` retry coverage on macOS. To restore it, add
two deterministic, platform-neutral tests to the same file. They inject a synthetic
`ETXTBSY` from the spawn closure instead of relying on kernel behaviour:

- `retry_on_etxtbsy_retries_synthetic_etxtbsy_then_returns_the_spawn`: the closure
  returns `Err(io::Error::from_raw_os_error(libc::ETXTBSY))` for the first two attempts,
  then spawns a trivial `/bin/sh -c 'exit 0'` command (with stdio set to null). Assert:
  - the closure ran exactly 3 times
  - the returned child exits successfully
  - `is_etxtbsy(&io::Error::from_raw_os_error(libc::ETXTBSY))` is true
- `retry_on_etxtbsy_returns_non_etxtbsy_errors_immediately`: the closure always returns
  `Err(io::Error::from_raw_os_error(libc::ENOENT))`. Assert that it ran exactly once and
  that the returned error's `raw_os_error()` is `Some(libc::ENOENT)`.

Notes:

- `retry_on_etxtbsy` and `is_etxtbsy` are `pub(crate)` in `sudo_runner/platform.rs` and
  are already re-exported through `super::super::*`. `libc` is already a `sase_gateway`
  dependency. Add `use std::io;` if needed.
- Under `cfg(test)`, `retry_on_etxtbsy` takes the fixture spawn lock around each
  attempt. The synthetic closures must not take that lock themselves, because the mutex
  is not reentrant. They do not need it.
- Do not add a retry-budget-exhaustion test. `ETXTBSY_RETRY_ATTEMPTS` is private and the
  test would cost about 500 ms of sleeps.

### 3. Keep macOS free of unused-import warnings

The macOS job runs clippy with `-D warnings`, so check that every import in `etxtbsy.rs`
is still used once the Linux tests are compiled out.

- Function-local imports go away with their gated function: `AsRawFd` and `CommandExt`
  inside the first test.
- The remaining module-level imports are all still used by the ungated tests:
  - `std::fs` (`fs::read_to_string`, `fs::read_dir`)
  - `PermissionsExt` (`.permissions().mode()`)
  - `Command`
  - `super::super::*` (`output_with_etxtbsy_retry`, `retry_on_etxtbsy`, `is_etxtbsy`)
  - `super::support::*` (`publish_executable`)
- If any import ends up used only by Linux-gated code, move it into that function or
  give it the same `cfg` instead of allowing the lint.

### 4. Doc-accuracy touch-ups (comments only)

These doc comments state the ETXTBSY rule as if it applied everywhere. Qualify each one
as Linux kernel behaviour, for example "On Linux, exec-ing a file fails with `ETXTBSY`
while…":

- `retry_on_etxtbsy` in `crates/sase_gateway/src/sudo_runner/platform.rs` ("Exec-ing a
  file fails with `ETXTBSY` while any process holds it open for writing").
- `FIXTURE_SPAWN_LOCK` in `crates/sase_gateway/src/sudo_runner/tests/support.rs`, which
  has the same sentence.

Do not change any code there. The retry backstop stays `cfg(unix)` and harmless on
macOS.

## Verification

1. Run a targeted test while iterating, from the sase-core checkout:
   `just test -p sase_gateway etxtbsy`. On Linux, all five tests in the module must run
   and pass: the 2 kernel replays, the publish test, and the 2 new portable tests.
2. Run the full gate. In sase-core it is guarded, so run it as `sase tool run check`
   (never bare `just check`), with a foreground tool timeout of at least 10 minutes. It
   must pass: fmt-check, features, clippy `-D warnings`, every test, and the script
   tests.
3. macOS cannot run locally. After the change lands, the confirmation is the next
   `cargo fmt + clippy + test (macos-latest)` job on `master`, checked with
   `actstat --repo sase-org/sase-core`. The implementing agent should state this
   explicitly in its final report and not claim the macOS job is green before it is.

## Commit

Use one Conventional Commit in sase-core, for example
`test(sudo-runner): gate kernel ETXTBSY replay tests to Linux and cover retry portably`.
Do not touch any `version` field or `CHANGELOG.md`.
