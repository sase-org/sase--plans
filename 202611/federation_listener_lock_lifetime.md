---
tier: tale
title: Fix federation listener lock lifetime under parallel subprocess load
goal:
  Eliminate the sase-1gz AlreadyRunning flake with deterministic lock-lifetime
  regression coverage while preserving IPC safety checks.
size: medium
proposed_by: bbugyi200.athena.sase-1qn.3
proposed_with: codex/gpt-6-astra
bead: sase-1qn.3
create_time: 2026-11-01 14:47:47
status: wip
---

- **PARENT:**
  [202611/task_backlog_epic_4.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_4.md)
- **BEAD:** sase-1qn.3

# Fix federation listener lock lifetime under parallel subprocess load

## Assignment and scope

Complete phase `sase-1qn.3` of `sase-1qn`, implementing the
`core-federation-socket-flake` phase in `plan:202611/task_backlog_epic_4.md`. The
canonical flake is `sase-1gz`. Read both beads with `sase bead read` and read the parent
design with `sase artifact read` before starting. Open `sase-core` through `/sase_repo`,
then use only its returned path and read its `AGENTS.md`.

The epic's final decision is `optional_scope = keep_five_seeds`: fix this selected flake
only. No memory changes, unrelated cleanup, binding changes, feature work, skips,
retries that hide failures, relaxed assertions, timeout increases, or flake suppressions
are authorized. Do not create task beads or close the seed task or parent epic. Record
discoveries as `PROPOSED FOLLOW-UP:` notes on `sase-1qn.3`.

This is one bounded Rust listener lifecycle change and its regression proof, sized
medium for one coding agent. The phase has no previous implementation or remaining-work
notes. At planning time both checkouts were clean; sase-core HEAD was `16a3ed0f`. The
relevant test has not changed since its module split in `b5cea78e`, and the listener
still relies on closing its file to release the lock. Recheck for intervening fixes
before editing. No production or test source was changed during planning.

## Findings and limits

Relevant sase-core files:

- `crates/sase_gateway/src/federation_worker/imp/listener.rs`
- `crates/sase_gateway/src/federation_worker/imp/tests/ipc.rs`
- `crates/sase_gateway/src/sudo_runner/platform.rs` and `execution.rs` (context for
  concurrent subprocess launches; changes are not expected)

`listener_creates_private_socket_and_rejects_symlink` creates its own
`tempfile::tempdir`, prepares a listener, asserts directory mode `0700` and socket mode
`0600`, drops the listener, then installs a symlink at its socket path. The reports
identify the second `prepare_listener` call's match arm at `ipc.rs:42`, where
`AlreadyRunning` replaces the required `UnsafeSocket`. The first listener has removed
its socket by then. The second call tries the runtime lock before reaching symlink
validation, so a retained lock is the leading explanation; a shared temporary path is
not established.

`PreparedListener` owns `_lock_file: File` and never explicitly unlocks it. Parallel
sudo-runner tests launch subprocesses with `pre_exec` callbacks. Such a fork can retain
a duplicate of the open file description until exec, even when the descriptor is
close-on-exec. Closing the parent's descriptor can therefore leave its exclusive lock
held briefly by an unrelated child.

A small pipe-synchronized Python/fcntl experiment during planning confirmed this
mechanism on the host: the parent acquired an exclusive lock on a file opened with
`O_CLOEXEC`, forked a child blocked on a pipe, closed the parent descriptor, then tried
to lock a freshly opened descriptor. It returned `WouldBlock`. Repeating with `LOCK_UN`
before close immediately reacquired the lock while the child was still alive. Both
children were reaped. This is OS-level evidence, not a fresh reproduction of the Rust
test: reproduce that test and connect the failure to this mechanism before treating it
as proven. `sase_core::store_lock::HeldStoreLock` already explicitly calls
`fs2::FileExt::unlock` in its destructor; it is useful precedent, not an API to expand
for this small gateway fix.

## Implementation

1. **Capture the failing baseline.** Record commit, host load, test parallelism,
   commands, runtime, iteration count, and exact panic. Build and run through
   `just test -p sase_gateway --lib`, never bare cargo. Run a bounded loop of the
   gateway library tests with normal parallelism so IPC and subprocess tests share the
   same process. Separate stress processes cannot inherit this test's descriptors and
   are not sufficient evidence by themselves. Stop at the first target failure and
   retain its output. If the natural race is elusive, use a synchronized regression that
   retains a duplicate descriptor and reproduces the same second-call error on the
   unchanged listener; do not substitute an unrelated noisy failure for reproduction.

2. **Add deterministic regression coverage before fixing the listener.** Prefer a child
   test module of `listener` so it can inspect private state without exposing production
   internals. Retain a `File::try_clone` of the acquired lock, drop the prepared
   listener, install the same symlink, and require `prepare_listener` to return
   `UnsafeSocket` while the duplicate remains alive. On platforms where descriptor
   duplication has different semantics, use the supported Unix fork/pipe mechanism with
   only async-signal-safe operations in the child and guaranteed child cleanup.
   Establish that the regression fails on the original implementation and that it
   recreates the reported lock error. A separate fresh temporary directory does not fix
   descriptor inheritance.

3. **Release the owned lock explicitly at the correct lifecycle boundary.** Use a small
   private RAII guard created only after successful lock acquisition. Its destructor
   calls fully qualified `fs2::FileExt::unlock`. Keep it alive through socket cleanup
   and listener disposal, and release it on post-acquisition preparation errors as well
   as normal shutdown. Preserve the existing nonblocking `AlreadyRunning` refusal and
   lock-file inode; never unlink the lock file to work around contention. A failed
   competing acquisition must not unlock the active owner's lock. Keep private modes and
   socket/symlink checks unchanged. Do not replace this with sleeps, lock retries, or
   global test serialization.

4. **Protect exclusivity and safety.** Cover rejection while a listener is genuinely
   alive, successful immediate restart after disposal while a duplicate lock descriptor
   survives, and the deterministic symlink case. Verify that failed contenders leave the
   active owner protected and that the guard releases an acquired lock on preparation
   errors. Exercise the guard's error-path ownership directly if that avoids a
   production test hook. Keep the original private-directory, private-socket, and
   symlink-rejection assertions. If the reproduced cause differs from the hypothesis,
   make the smallest evidence-backed fix in the responsible fixture or listener and
   document why the implementation changed.

## Verification and evidence

Read `lint_and_test.md` through `/sase_memory_read`. Use `/sase_monitor` for commands
that exceed the synchronous turn limit; include continuation work and evidence paths in
its follow-up. For `sase tool run` escalation, join the printed run rather than
restarting it. Do not globally install anything.

- Run the new regression against the original listener and the fixed listener to
  demonstrate red/green behavior. Retain exact outputs.
- Run focused listener/IPC tests and the full `sase_gateway` library suite. Preserve the
  hermetic environment supplied by `scripts/check.sh`.
- Repeat the same loaded baseline loop after the fix: at least 20 complete
  gateway-library iterations, and at least twice the baseline iterations needed to
  reproduce, subject to a documented bounded wall-clock budget. Also repeat the fast
  deterministic regression at least 100 times. If the natural baseline did not fail, say
  so explicitly and rely on the proven failing deterministic schedule rather than
  claiming a measured flake rate.
- Briefly mutation-check the safety assertions: make the socket mode wrong, then
  separately bypass symlink rejection, and confirm the corresponding focused tests fail.
  Restore each mutation immediately; do not retain it.
- Run `just fmt`, then `sase tool run check` from sase-core. If any tracked sase files
  change, run its check too. Do not run `just check-full`. No Python binding or wire
  changes are expected, so do not invent a core revision SHA; let host completion handle
  any required pin movement.
- A gate failure reproduced identically on the clean base is not a reason to leave the
  phase open. Record the base comparison and any existing tracker in a
  `PROPOSED FOLLOW-UP:` note. Fix failures caused by this work.

Acceptance requires a concrete explanation of the original second-call failure, a
regression that fails without the fix, a passing loaded loop, unchanged safety
assertions with negative proof, and a passing core check or documented identical
clean-base failures under the phase-worker rule. Record command results and limitations
without claiming that a finite stress run proves all scheduling interleavings.

## Phase completion

Append
`TASK RESOLVED: sase-1gz — <cause, base, before/after counts, regression and gate evidence>`
to `sase-1qn.3` only after acceptance is met. Run `sase bead epic-symbols sase-1qn.3`
immediately before closing. The planning read found no entries, but recheck; resolve any
new entries or re-key them to an appropriate still-open bead. Close only `sase-1qn.3`
with `sase bead close sase-1qn.3 --note "<what was verified>"`. Its parent epic and the
seed task remain for the epic lander. Finish with `/sase_final` and a commit decision
for every changed repository; never create commits manually.
