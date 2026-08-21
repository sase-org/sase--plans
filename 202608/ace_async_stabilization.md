---
tier: tale
title: Stabilize the remaining ACE lifecycle and interaction flakes
goal:
  Every ACE failure assigned to phase sase-rm.10 is fixed or proven obsolete, verified
  under focused and contention lanes, and handed to the epic land agent with accurate
  flake-retirement evidence.
size: medium
proposed_by: bbugyi200.athena.sase-rm.10
bead: sase-rm.10
create_time: 2026-08-21 05:45:48
status: done
---

- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.10.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-rm.10](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-rm.10.md)
- **COMMITS:**
  - [b8559f3](https://github.com/sase-org/sase/commit/b8559f36f00a3f46c0ee0ce7343dc50735275900)
    — fix(ace): stabilize async teardown and interaction waits

# Stabilize the remaining ACE lifecycle and interaction flakes

## Goal

Complete phase `sase-rm.10` by replacing scheduler-sensitive ACE assertions with
semantic lifecycle waits, fixing the Vim insert-session undo boundary, retiring only the
flake-baseline rows whose mechanisms are actually fixed or whose test was removed, and
recording close-ready evidence for each task assigned by the parent design.

## Current evidence and constraints

- The phase owns `sase-n5`, `sase-ni`, `sase-oe`, `sase-oz`, `sase-pd`, `sase-pe`,
  `sase-q8`, `sase-qm`, `sase-qr`, and `sase-r3`. Their live records are still `ready`
  and unassigned; this phase must not close those task beads or the parent epic.
- `sase-oe`'s comprehensive-update confirmation test and its auto-update path were
  deleted by `f1914962c`; its stale baseline row must become a `fixed-at` retirement
  rather than be recreated.
- The Models test can race the panel's initial provider-snapshot worker; the VCS
  completion test uses one-second `threading.Event.wait` assertions; the Jump-All test
  reads scrolling after a generic pause; and the Commits/update tests poll background
  work without awaiting the worker that owns completion.
- `AcePage` drains pump-free tasks but does not explicitly cancel and await Textual
  workers before returning. `AcePageGroup` also snapshots notification count even though
  tests request notifications-disabled pages, allowing startup timing to become fixture
  state.
- Textual batches ordinary inserted characters using a two-second wall-clock timer. Vim
  insert mode should make an insert session one undo batch, so CPU starvation can
  currently split `new` across undo checkpoints.
- Any phase-owned `--epic-symbol` entry must be resolved or re-keyed before phase
  closure. Discovered out-of-scope work must be recorded only as a `PROPOSED FOLLOW-UP:`
  note on `sase-rm.10`.

## Implementation

1. Make shared ACE test lifecycles deterministic.
   - Extend `AcePage` teardown to cancel and await every live Textual worker as well as
     registered pump-free tasks on both normal and exceptional exits.
   - Add focused teardown coverage proving a deliberately blocked worker is drained
     before the context manager returns.
   - Normalize `AcePageGroup`'s notifications-disabled baseline/reset contract so late
     startup notifications cannot leak between the module-scoped Vim containment
     checkouts, while retaining explicit notification-state isolation when requested.

2. Replace node-local timing races with their semantic completion signals.
   - Wait for the Models panel's initial provider snapshot before testing bucket
     drill-in and restore.
   - Remove one-second event deadlines and event-loop-blocking waits from the VCS repo
     cache-miss worker test; release the controlled worker in guaranteed cleanup and
     wait on visible loading/result state.
   - Wait for the mounted raw-frontmatter widget before reading the loaded xprompt
     binding, and wait for Jump-All's exact scroll offset instead of a generic pause.
   - In the Commits interaction test, await the collection and diff workers and then
     their UI delivery before asserting result/detail state.
   - In the managed-update test, capture the submitted session worker, await its real
     completion and callback delivery, then exercise refresh/restart assertions.

3. Make Vim undo grouping independent of scheduler delay.
   - Configure `VimTextArea` insert batching so a Vim insert session is not split by
     Textual's wall-clock checkpoint timer; preserve the explicit checkpoints already
     created by structural edits, cursor movement, and mode changes.
   - Add a deterministic regression that advances the history clock beyond the old
     threshold between typed characters and proves both bullet and ordered Ctrl-J
     continuations still undo as one insert batch after the structural prefix.

4. Reconcile durable flake accounting and phase evidence.
   - Replace live baseline rows for `sase-oe`, `sase-oz`, and `sase-qm` with scoped
     `fixed-at` directives only after their removal/fixes are verified; do not add
     baseline debt for the other nodes.
   - Run each assigned exact node or its current replacement, repeat the contention lane
     for timing-sensitive families, and run selection health. Record one evidence-rich
     close-ready note per assigned task on `sase-rm.10`, explicitly citing
     already-landed deletion when applicable.

## Verification and closure

1. Run `just install` before checks, then execute focused tests for the changed ACE
   harness, Models, Vim containment, VCS completion, xprompt loading, Vim undo, Commits,
   managed update, and Jump-All paths.
2. Run repeated `just test-contention` coverage for the assigned exact nodes that remain
   collectable, plus `just selection-health --fail-on-new-flake`.
3. Run the required repository-wide `just check`; if it escalates unusually or a full
   lane is needed, use the SASE monitor workflow rather than running `just check-full`
   inline.
4. Inspect `git diff`, re-query the assigned task records, and append close-ready
   evidence to `sase-rm.10` without changing those task statuses.
5. Run `sase bead epic-symbols sase-rm.10`, resolve or re-key every remaining symbol,
   then close only `sase-rm.10` with a note naming the exact verification performed.
