---
tier: tale
title: Eliminate the suite-level Text file busy test flake
goal:
  Executable test stubs are materialized without a writable descriptor this process can
  leak to a forked child, so no CLI test can fail with ETXTBSY under the parallel suite,
  and a regression test catches any revert.
size: small
proposed_by: bbugyi200.athena.bob-cli-o
bead: bob-cli-o
create_time: 2026-09-09 20:00:02
status: wip
---

- **BEAD:** bob-cli-o

# Eliminate the suite-level `Text file busy` (ETXTBSY) test flake

Bead: `bob-cli-o` (with `+1` evidence from `bob-cli-n.1`).

## Symptom

Under the full parallel `cargo test` run,
`tests/cli.rs::nightly_runs_shared_sync_once_then_wrapped_steps_in_order` intermittently
fails because `bob nightly` cannot execute the test's `ob` shim:

```
bob: failed to run ob sync: Text file busy (os error 26)
```

`bob-cli-n.1` independently hit the same error class in
`tests/cli.rs::capture_clip_failures_leave_vault_untouched`, where `bob` could not
execute the `BOB_CLIPBOARD_CMD` stub. Both tests pass in isolation.

## Root cause (reproduced, not inferred)

Neither shim compilation/replacement nor a shared fixture is involved. The shims are
written into per-test `TempDir`s, so no two tests share the path.

`execve` fails with `ETXTBSY` when _any_ process on the machine holds a writable
descriptor on the target inode. Both flaking tests materialize their stub through the
shared helper:

```rust
fn write_executable(path: &Path, contents: &str) {
    fs::write(path, contents).unwrap_or_else(...);   // holds a writable fd
    set_mode(path, 0o755);
}
```

`fs::write` keeps a writable descriptor open for the duration of the write. `cargo test`
runs every test in the same process, one thread per test, and those threads spawn child
processes constantly (`bob`, `git`, `sh`). A `fork`/`posix_spawn` issued by any other
test thread inside that write window copies the file-descriptor table, so the new child
inherits the writable descriptor: `O_CLOEXEC` only drops it once the child reaches
`execve`. Until the child gets there — which can be milliseconds under a loaded,
oversubscribed test run — every `execve` of that stub anywhere on the system fails with
`ETXTBSY`. The stub is executed by a _grandchild_ (`bob` spawning `ob`), well after
`write_executable` returned, which is why the failure surfaces as a `bob` error and why
it never reproduces in isolation (no other thread is forking).

This was reproduced outside the repo with a standalone harness that mimics the suite
(one thread writing + executing stubs, other threads spawning children):

| stub write strategy   | lurker threads | stub payload | iterations | ETXTBSY |
| --------------------- | -------------- | ------------ | ---------- | ------- |
| `fs::write` (current) | 4              | 256 KiB      | 100        | 49      |
| `fs::write` (current) | 8              | 256 KiB      | 100        | 84      |
| `fs::write` (current) | 16             | 1 MiB        | 100        | 88      |
| `cp` child (proposed) | 8              | 256 KiB      | 100        | 0       |
| `cp` child (proposed) | 8              | 256 KiB      | 300        | 0       |

A larger payload only widens the window that already exists for the real (tiny) stubs;
with a tiny payload and unmodified child scheduling the same harness still reproduced
the failure, just rarely (3 in ~11k iterations).

Production code is **not** affected and must not be changed:
`src/runner.rs::write_asset` already materializes cached script assets through a temp
file plus `fs::rename`, and `bob` is single-threaded (no `thread::spawn` anywhere in
`src/`), so it can never fork while holding a writable descriptor.

## Fix

Keep the writable descriptor for the executed inode **out of the test process**, so no
fork can capture it. `write_executable` writes the payload to a scratch file that is
never executed, then lets a short-lived `cp` child create the real stub. The `cp`
process owns the only writable descriptor on the stub inode, it is reaped before
`write_executable` returns, and a fork of the test process can never inherit a
descriptor that lives in another process.

This removes the hazard instead of serializing around it: no global lock, no change at
the ~800 process-spawn call sites, and no behavior change for the 43 `write_executable`
callers.

### 1. `tests/cli.rs` — replace the helper (currently around line 19572)

```rust
/// Materialize an executable stub without ever holding a writable descriptor
/// on the file the tests execute.
///
/// `fs::write` keeps a writable descriptor open while it fills the file, and
/// `cargo test` runs every test in this one process. A child that another test
/// thread forks inside that window inherits the descriptor — `O_CLOEXEC` only
/// drops it once the child reaches `execve` — and until then every attempt to
/// execute the stub fails with `ETXTBSY` ("Text file busy", os error 26), even
/// from an unrelated process such as `bob` spawning its `ob` shim. Writing the
/// payload to a scratch file that is never executed and letting a short-lived
/// `cp` child create the stub keeps the writable descriptor out of this
/// process, so no fork can capture it.
fn write_executable(path: &Path, contents: &str) {
    let payload = scratch_payload_path(path);
    fs::write(&payload, contents).unwrap_or_else(|error| {
        panic!("write executable stub payload {}: {error}", payload.display())
    });
    // Copy onto a fresh inode so rewriting a stub cannot disturb a copy that
    // is still executing.
    let _ = fs::remove_file(path);
    let output = Command::new("cp")
        .arg("--")
        .arg(&payload)
        .arg(path)
        .output()
        .unwrap_or_else(|error| {
            panic!("copy executable stub {}: {error}", path.display())
        });
    assert!(
        output.status.success(),
        "copy executable stub {}:\n{}",
        path.display(),
        format_output(&output)
    );
    fs::remove_file(&payload).unwrap_or_else(|error| {
        panic!("remove stub payload {}: {error}", payload.display())
    });
    set_mode(path, 0o755);
}

/// Scratch path for a stub payload: written in this process, never executed,
/// and removed once `cp` has copied it onto the stub path.
fn scratch_payload_path(path: &Path) -> PathBuf {
    let file_name = path
        .file_name()
        .unwrap_or_else(|| panic!("stub path has no file name: {}", path.display()));
    let unique = TEMP_COUNTER.fetch_add(1, Ordering::Relaxed);
    let mut name = OsString::from(".");
    name.push(file_name);
    name.push(format!(".{unique}.payload"));
    path.with_file_name(name)
}
```

Notes for the implementer:

- `TEMP_COUNTER`, `Ordering`, `OsString`, `PathBuf`, `Command`, and `format_output` are
  already in scope in `tests/cli.rs`; add no new imports beyond what the code needs.
- Keep `set_mode` as the single place that applies `0o755` (`cp` creates the copy with
  the umask default).
- `cp` is already an implicit dependency class for this suite (it shells out to `git`,
  `sh`, and `bash`), and the stubs are `#!/bin/sh` scripts, so no new portability floor
  is introduced. No `cfg` branches: on a non-Unix host the suite already cannot run its
  `/bin/sh` stubs.
- Do **not** touch `nightly_runs_shared_sync_once_then_wrapped_steps_in_order` or
  `capture_clip_failures_leave_vault_untouched`; their shared-sync-once and wrapped-step
  ordering assertions must survive this change untouched.

### 2. `tests/dataview_parity.rs` — same helper, same fix (currently around line 1632)

That binary has its own copy of `write_executable` (2 call sites) with the identical
hazard. Apply the same implementation there, following the file's existing convention of
duplicating small helpers rather than introducing a shared module. A shorter comment
that points at the same failure mode is fine.

### 3. `tests/cli.rs` — regression coverage

Add one test immediately after the last existing test (after
`nightly_failed_step_still_runs_later_steps_and_exits_nonzero`, before the
`fn done_tasks_source` helper block):

```rust
#[cfg(unix)]
#[test]
fn executable_stubs_stay_executable_while_other_threads_fork() {
    // Guards the `Text file busy` flake (bead bob-cli-o): a stub must never be
    // written through a descriptor this process holds, because any child
    // forked during the write inherits it and makes `execve` of the stub fail
    // with ETXTBSY until that child execs.
    ...
}
```

Shape:

- Start N=4 background threads that spawn `true` in a tight loop until an `AtomicBool`
  stop flag is set. These are the forks that would capture a leaked descriptor.
- On the main thread, loop ~60 times: `write_executable` a stub whose body is
  `#!/bin/sh\nexit 7\n` followed by ~256 KiB of `#` comment padding (padding widens the
  window that a reverted helper would leave open, so the regression is caught reliably
  rather than once in a thousand suite runs; `exit 7` runs before `sh` ever reads the
  padding).
- Execute each stub. Treat `Err(error)` with `error.raw_os_error() == Some(26)` as the
  regression and panic with a message naming the flake; assert the exit code is `7`
  otherwise.
- On the first iteration also assert the stub round-tripped: contents equal the payload,
  `assert_unix_mode(&stub, 0o755)`, and the stub's directory contains exactly the stub
  (no leftover `.payload` scratch file).
- Stop the threads and `join` them before returning.

Expected behavior: the current helper fails this test essentially always (~50 of 60
iterations hit ETXTBSY in the standalone harness at these parameters); the fixed helper
cannot fail it, because the hazard is removed by construction rather than by timing.
Runtime is roughly 0.2 s. The lurker threads only endanger stubs written through a
leaked descriptor, and after this change no test in the binary has one, so the new test
cannot destabilize the tests running beside it.

## Verification

1. `cargo test` — full suite green; run it at least 5 times to confirm stability.
2. `cargo clippy --all-targets --all-features` — clean.
3. `cargo fmt --check` — clean.
4. Prove the regression test has power: temporarily restore the old `fs::write` +
   `set_mode` body of `write_executable`, run
   `cargo test executable_stubs_stay_executable_while_other_threads_fork` and confirm it
   fails with the ETXTBSY message, then restore the fix and confirm it passes again.
   Report both observations.
5. `just all` (fmt + lint + test) as the final gate.

## Rejected alternatives

- **Global `RwLock` (exclusive while writing a stub, shared around every spawn).** This
  is the textbook fix, but the shared side has to wrap every `fork`, which means
  touching ~800 `.output()` / `.spawn()` call sites and renaming them to a wrapper; it
  also serializes stub writes against all process spawning.
- **Temp file + `fs::rename`.** Does not help: `rename` keeps the inode that was written
  through the leaked descriptor, so `execve` still sees a writer.
- **Retry on `ETXTBSY` in `src/native/ob.rs` (and every other production spawn).** The
  hazard is created by the test harness, not by `bob`; adding retries would put
  workaround code in production for a condition production cannot hit, and would mask
  genuine spawn failures.
- **Retrying the failed `bob nightly` invocation inside the test.** Would hide real
  ordering failures and re-run a command that has already mutated the vault.
