---
tier: tale
title: Stabilize bounded log pipe shutdown under load
goal:
  Monitor and proc supervisors close output promptly without dropping buffered output
  when descendants retain pipe writers.
size: medium
proposed_by: bbugyi200.athena.sase-lk
bead: sase-lk
create_time: 2026-08-15 17:52:58
status: wip
---

- **PROMPT:**
  [prompts/202608/stabilize_bounded_log_pipe_close.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stabilize_bounded_log_pipe_close.md)
- **BEAD:**
  [sase-lk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-lk/README.md)

# Stabilize bounded log pipe shutdown under load

## Goal

Complete task bead `sase-lk` by removing the scheduling-sensitive shutdown behavior that
makes monitor supervisor tests exceed their no-hang bounds or lose already buffered
output when a descendant keeps the output pipe open.

## Context

`BoundedLogPipe.close()` currently gives its drain thread a fixed five-second join even
when the caller configures a shorter close-drain budget. Under heavy parallel load, a
descheduled drain thread can therefore consume the monitor tests' entire five-second
no-hang allowance. The drain loop also checks its deadline before polling the file
descriptor, so it can discard bytes that were already readable when the deadline was
reached. Both the monitor and proc supervisors use this shared pipe with a 0.5-second
close-drain budget.

## Implementation

1. Refine `BoundedLogPipe` shutdown so `close()` is bounded by its configured drain
   budget (plus only a small scheduling/poll allowance) rather than an unrelated
   five-second join, while keeping descriptor cleanup and callback-error propagation
   safe if the daemon thread finishes asynchronously.
2. Reorder or otherwise reshape draining so bytes that are immediately readable are
   consumed before the close deadline stops further waiting, preserving output tails
   without waiting for EOF from lingering descendants.
3. Add focused pipe-level regression tests that deterministically cover an open writer,
   readable data at/after the close deadline, and prompt return when the drain worker is
   delayed. Retain monitor-level assertions for the partial-line, grandchild-held
   stdout, and chatty TERM-ignoring command scenarios, adjusting only test structure or
   timing where needed to prove behavior rather than mask the race.
4. Review both monitor and proc supervisor call sites for compatibility with the new
   close contract and update documentation or call-site parameters only if the shared
   API requires it.

## Verification

1. Run the direct log-pipe and monitor supervisor tests, including repeated and parallel
   stress runs of the three nodes named by `sase-lk`.
2. Run `just install`, then the repository-required `just check`; escalate to the
   monitored `just check-full` lane if scoped selection broadens or reports unusual
   selection behavior.
3. Run `just selection-health --explain` and confirm the affected nodes no longer have a
   newly reproduced failure attributable to the patched behavior.
4. Reinspect the final diff and close `sase-lk` with a note naming the tests and gates
   that passed.
