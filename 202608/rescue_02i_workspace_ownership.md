---
tier: tale
title: Rescue agent 02i and complete its phase safely
goal:
  Restore single-writer ownership for agent family 02i, preserve its completed flat-pane
  query migration work, finish the approved verification and phase closure, and retire
  every redundant shell and monitor so the family releases its workspace cleanly.
size: medium
proposed_by: bbugyi200.athena.02p
create_time: 2026-08-15 14:28:14
status: wip
---

# Plan: Rescue agent 02i and complete its phase safely

## Diagnosis and scope

This is not an unrelated agent using `02i`'s checkout. It is an overlapping set of
shells in the same `02i` agent family:

- `02i--code` is the canonical code shell created after approval of
  `complete_flat_pane_query_migration.md`. Its prompt is to implement that approved plan
  and finish phase bead `sase-m6.6.1.5`.
- `02i--1` is an earlier monitor follow-up that resumed after waiting for agent 026. It
  took over the same migration, committed and pushed much of it, and continued into
  verification while the approved code shell was also active.
- `02i--2` and `02i--4` are later monitor-driven continuations whose prompts say to wait
  for `02i--code`, then audit the same migration and run the same verification. They are
  not separate feature workers, but their live Codex turns and polling loops keep adding
  contenders in the same checkout.
- `02i--mon-4` is the current sleep monitor for the same family. Its displayed
  `WAITING FOR DUPLICATE 02i` state describes this intra-family overlap.

The overlap began because a pre-existing monitor handoff resumed as `02i--1` while the
plan-approval path later launched `02i--code`. Subsequent wait handoffs recursively
created more root-role turns. The rescue must not discard valid commits or interrupt a
commit in flight, and it must not launch another `02i` continuation while consolidating
ownership.

## Completion boundary

Restore exactly one writer for the migration, finish the already-approved
`complete_flat_pane_query_migration.md` boundary, close only `sase-m6.6.1.5` when all
evidence passes, and leave its parent epic open. The rescue is complete only when no
`02i` agent shell, provider process, monitor, or polling command remains live and the
family's workspace claim has been released.

## Implementation

1. Take a fresh read-only ownership snapshot before acting. Resolve every live `02i`
   shell through `sase agent show`, its `agent_meta.json`, its stable workflow
   checkpoints, and OS process ancestry; correlate recent `live_reply.md` and
   `tool_calls.jsonl` timestamps as draft activity evidence. Do not trust historical
   PIDs blindly. Confirm whether `02i--code` is still the approved active writer and
   whether any sibling is in a commit, rebase, or repository-mutating command.

2. Establish one authoritative completion path. Prefer `02i--code`, because it is the
   plan-approved `medium_worker`, if it is still healthy and progressing. If it has
   already exited, use one rescue worker outside the contested family rather than
   launching another `02i` continuation. Before stopping anything, record the latest
   commits and any uncommitted task diff so work from `02i--1` or `02i--code` cannot be
   lost.

3. Gracefully drain the redundant monitor-derived shells. Stop `02i--1`, `02i--2`,
   `02i--4`, and `02i--mon-4` individually only after the fresh snapshot proves they are
   not in a critical write operation; do not kill the whole `02i` family or the selected
   writer by a broad name. Verify that each shell's provider descendants and ad hoc
   PID-watch loops terminate. If a shell finishes naturally during arbitration, consume
   its terminal artifact instead of sending a redundant signal.

4. Consolidate the migration state after single-writer ownership is real. Reconcile the
   pushed migration and performance commits reported by the live artifacts—including
   `d580a55c8`, `91c71b6af`, and `90058437e`—with any later commit or remaining diff.
   Preserve only task-related work, require both the main and linked-core checkouts to
   be clean and synchronized, and use the mandated SASE commit workflow for any final
   owned changes.

5. Finish the approved completion audit rather than repeating already-proven work.
   Confirm the flat Stitches, Beads, Plans/providers, and Files panes use the shared
   generation-checked off-thread Rust query session; real closed host-predicate facts,
   provider isolation, stale-result rejection, Files negation, conformance coverage, and
   selection/rollback behavior remain present. Re-run the focused suites affected by any
   final diff and require the slow Artifacts navigation benchmark to keep every measured
   action below the unchanged 16 ms p95 budget.

6. Complete the repository gates from the consolidated tree. Open the linked `sase-core`
   checkout through the required repo workflow and run its focused query and binding
   checks plus its required check gate. In SASE, run `just install`, `just check`, and
   monitored `just check-full`; inspect terminal results and fix/repeat any task-caused
   failure. Do not let another monitor continuation re-enter the `02i` checkout during
   this verification.

7. Re-read the authoritative bead state and record concise completion evidence. Close
   only `sase-m6.6.1.5` with resolution `done` if every approved-plan boundary and
   verification gate passes; otherwise leave it open with the exact remaining failure.
   Never close the parent epic from this rescue.

8. Retire the family cleanly. Let the selected worker return normally, stop any monitor
   left only to await a duplicate, and verify with both SASE status and OS process
   inspection that no `02i` runner, provider child, supervisor, test command, or polling
   loop remains. Confirm the workspace claim is released and report the final shell
   outcomes, retained commits, verification evidence, and bead status.

## Verification

- Before termination: one fresh map of shell names, roles, prompts, runner/provider
  ancestry, artifact activity, and mutation state identifies the chosen writer and the
  exact redundant shells.
- After termination: only the chosen completion path may have a live process in the
  checkout; no killed shell may retain provider or watcher descendants.
- Before bead closure: linked-core checks, SASE `just check`, monitored
  `just check-full`, focused query/session/conformance tests, and the strict slow
  navigation benchmark must all pass on the clean consolidated tree.
- At handoff: `sase agent list -a -j`, per-shell detail, workspace-claim state, and OS
  process inspection must agree that family `02i` is terminal and fully at rest.
