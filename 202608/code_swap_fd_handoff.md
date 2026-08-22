---
tier: tale
title: Bound the guarded code-swap lock to one exec handoff
goal:
  Host-owned epic launches transfer their shared code-swap lock into sase bead work
  without leaking an inheritable descriptor into agent runners, so sase update is
  deferred only by a genuine blocking reader and reports that reader by identity.
size: small
proposed_by: bbugyi200.athena.0ba
create_time: 2026-08-22 18:41:17
status: wip
---

# Bound the guarded code-swap lock to one exec handoff

## Context and root cause

The update failure is a real kernel-lock conflict, but the processes retaining the lock
are not blocking `sase bead work` operations. The live lock inventory showed two shared
`flock` records whose original guarded-launch PIDs had already exited. Eleven surviving
`run_agent_runner.py` processes still had descriptor 3 open on the same
`~/.sase/locks/code-swap.lock` inode, and `/proc/<pid>/fdinfo/3` showed the descriptor
without close-on-exec. The holder directory contained only advisory agent-runner JSON;
there was no blocking reader identity, which is why the update surfaced the fallback
"one or more active readers did not record their identity" message.

This began with the guarded epic-launch path introduced by commit `f8a0dd858`. The
bootstrap correctly acquires the shared lock before importing editable SASE code and
marks its descriptor inheritable so `os.execvp()` can carry the lock into a fresh
`sase bead work`. The receiving reader, however, opens a second non-inheritable lock
descriptor and never adopts or closes the inherited bootstrap descriptor. Any later
child creation that honors inheritable descriptors can therefore copy the bootstrap
descriptor into detached agent runners. Those runners keep the shared lock alive long
after the launch command removes its blocking-holder sidecar and exits.

The ownership bug is in the Python code-swap handoff, not in Rust agent-launch policy:
the guard deliberately relaxes close-on-exec for exactly one transition and must bound
that exception at the receiving reader. Globally changing agent spawning would only mask
this leak for one child path and would leave other descendants vulnerable.

## Implementation

1. Define a private environment contract in `src/sase/dev_update/code_swap_lock.py` for
   the guarded lock descriptor. Have `code_swap_guarded_exec.py` publish the acquired
   descriptor immediately before `exec`, while retaining the existing disabled-lock and
   exec-failure cleanup behavior. Document that the marker authorizes exactly one
   bootstrap-to-reader handoff and must not survive into agent execution.

2. Teach `code_swap_reader_lock()` to consume that marker before using the ordinary
   open-and-lock path. Pop the marker from the environment unconditionally; accept only
   a positive open descriptor whose `fstat` device/inode matches the configured
   code-swap lock; make it non-inheritable immediately; and take/confirm the shared
   nonblocking `flock` on that descriptor. A valid descriptor becomes the reader
   context's sole owned lock and is unlocked and closed by the existing context cleanup.
   An absent, malformed, closed, or mismatched descriptor must not close an unrelated FD
   and must fall back to the current direct-reader acquisition path.

3. Preserve the current safety and diagnostic contracts: direct `sase bead work` remains
   fail-fast against a writer; guarded launches still wait before importing editable
   modules; the adopted reader writes its normal PID/operation/command holder sidecar;
   advisory agent runners never block writers; and unknown real lock holders remain
   fail-closed rather than being ignored.

4. Update module comments and helper names as needed so the descriptor lifecycle is
   explicit: bootstrap owns the lock while waiting, `sase bead work` atomically adopts
   it and restores close-on-exec, and the reader context releases it after launch
   orchestration.

## Tests

1. Extend `tests/dev_update/test_code_swap_guarded_exec.py` with a subprocess regression
   that executes a payload through `guarded_exec_argv()`, enters
   `code_swap_reader_lock()`, launches a deliberately long-lived descendant through an
   FD-inheriting spawn path, and then lets the reader exit. While that descendant is
   still alive, assert that `code_swap_writer_lock()` succeeds. This must fail on the
   current implementation because the descendant retains the bootstrap lock.

2. Assert within the guarded payload that the handoff marker is consumed and that the
   adopted lock descriptor is non-inheritable before any descendant launch. Retain the
   existing proof that the bootstrap waits behind a writer and executes its command
   exactly once.

3. Add focused cases in `tests/dev_update/test_code_swap_lock.py` for malformed, closed,
   and same-process unrelated descriptor markers. Verify safe fallback, preservation of
   unrelated descriptors, normal holder identity, and cleanup after the reader context
   exits.

4. Run `just install`, the two targeted dev-update test modules, and `just check`. If
   scoped selection escalates or reports unusual coverage, follow the repository
   guidance and hand `just check-full` to `/sase_monitor`.

## Operational verification and legacy holders

After the fix is deployed, launch a guarded epic and confirm that live agent runners do
not retain the code-swap lock inode, while a deliberately active blocking reader still
prevents an update and is named correctly. Re-run the update after this check.

Descriptors already leaked into runners by the old code cannot be retroactively made
close-on-exec by the patch. Re-inventory those legacy holders before the update. Let
them finish normally by default; if immediate recovery is still required, identify the
exact affected agents and obtain confirmation before stopping active work. Do not
unlink/replace the lock file or bypass the lock, because either action could let an
update race a genuine old blocking reader.

## Acceptance criteria

- A guarded launch maintains uninterrupted shared-lock coverage from the bootstrap
  through `sase bead work`, with a normal holder identity once Python code is loaded.
- The inherited bootstrap descriptor is consumed and non-inheritable before any agent
  runner can be spawned.
- Long-lived agent runners from a newly guarded launch do not block or defer
  `sase update`.
- Genuine blocking readers and active writers retain their current fail-fast/waiting
  behavior and useful diagnostics.
- Invalid handoff metadata cannot close an unrelated process descriptor or weaken the
  writer lock.
- Targeted regression tests and `just check` pass.
