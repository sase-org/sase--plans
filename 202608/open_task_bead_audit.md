---
tier: tale
title: Audit and prune the open task-bead backlog
goal:
  Every non-closed task bead is revalidated, obsolete entries are closed with durable evidence, and actionable work
  remains open.
size: medium
proposed_by: bbugyi200.athena.qv
create_time: 2026-08-01 06:57:44
status: wip
---

- **PROMPT:** [prompts/202608/open_task_bead_audit.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/open_task_bead_audit.md)

# Audit and prune the open task-bead backlog

## Goal

Revalidate every non-closed task bead against current `master`, close only the tasks whose evidence now establishes
completion, duplication, staleness, or non-reproducibility, and preserve actionable work with concise supporting
evidence. This is a `tale` because the work is one coherent backlog transaction: a single follow-up agent can keep one
fresh inventory, make consistent resolution choices, and avoid closure races without implementation phases.

## Starting evidence and provisional disposition

The planning audit found 12 non-closed task beads on `master` at `d462e97bb`. Treat this table as a hypothesis to
revalidate, not permission to close a bead whose state or evidence has since changed.

| Bead      | Provisional disposition                                      | Evidence to confirm                                                                                                                                                                                              |
| --------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-cc` | Keep `ready`                                                 | Compact `list` has the type glyph, but its Rust `handle_list` port remains dormant work rather than a duplicate.                                                                                                 |
| `sase-cd` | Keep `ready`                                                 | `sase bead search provider --format compact` still emits only the status glyph, unlike `list`.                                                                                                                   |
| `sase-ce` | Close `done`                                                 | `a692be5d7` refreshed the affected clan/panel/tribe goldens, `e9ae2dbac` stabilized slow-tool rendering, and `just test-visual` passed 393 tests.                                                                |
| `sase-cf` | Close `canceled` as not reproducible                         | Its exact SIGKILL capacity test passed directly and during a 28-worker full-suite run.                                                                                                                           |
| `sase-cg` | Close `canceled` as no longer reproducible                   | The exact watchdog test, the full 393-case visual suite, and the full 28-worker suite passed the named cases; `e9ae2dbac` also fixed the slow-tool snapshot path.                                                |
| `sase-ch` | Close `superseded` as duplicate                              | It is the same five-provider `sase_beads` drift report as closed `sase-cm`; current `sase validate` reports `init skills --check` healthy.                                                                       |
| `sase-cj` | Keep `ready`                                                 | This targets the linked `sase-telegram` development install contract and has not been invalidated from the primary repo alone.                                                                                   |
| `sase-ck` | Close `canceled` as stale/branch-specific                    | The exact attribution regression passes on current master; `4fd54a967` and `3a98c68df` contain the relevant routing/refactor fixes and predate the report.                                                       |
| `sase-cl` | Keep `ready` and add evidence                                | It recurred after `sase-c7`/`c82eff9a0`; the planning full suite spent 92.05 seconds in `test_save_dismissed_bundle_is_fast_with_many_existing_bundles`, matching the recursive `dismissed_bundles` scan report. |
| `sase-cn` | Close `done`                                                 | Its attributed commit `86d820f0b` is an ancestor of master, and the current floor has since advanced to `sase-core-rs>=0.17.4`. No live worker owned the still-`in_progress` bead during planning.               |
| `sase-cq` | Close `canceled` as stale if the sidecar check remains green | Current `sase validate` reports `plan links validate` healthy; inspect the audited plans-sidecar history before recording the final reason.                                                                      |
| `sase-cr` | Close `canceled` as stale if validation remains green        | Current `sase validate` reports `init memory --check` healthy after `f0e1a25e6`; inspect the audited chezmoi/current-memory state before recording the final reason.                                             |

## Execution

1. Re-read the bead lifecycle rules through `sase memory read sase_beads.md` and take a fresh machine-readable inventory
   with `sase bead list --type task --format json --limit 0`. Compare IDs, status, assignee, timestamps, dependencies,
   and notes with the table above. Do not close a task that acquired new contradictory evidence, was claimed by a live
   worker, or is no longer in the expected state. Use `sase agent list -j` to recheck live ownership, especially for
   `sase-cn`.

2. Use `sase repo open` with specific audit reasons before reading the plans sidecar, `sase-core`, chezmoi, or
   `sase-telegram`; use only the paths it returns. Inspect relevant recent history and current files, without modifying
   those repositories. Confirm that `sase-cc`, `sase-cd`, and `sase-cj` are still distinct actionable tasks. Confirm
   that `sase-cl` is a post-`c82eff9a0` recurrence rather than a duplicate of `sase-c7`, and append a bead note with the
   current performance evidence if it remains accurate.

3. Reproduce closure evidence before changing bead state:
   - Run `sase validate`; require all four checks to pass for the `sase-ch`, `sase-cq`, and `sase-cr` decisions.
   - Run `just test-visual`; require the named `sase-ce`/`sase-cg` snapshots to pass without updating goldens.
   - Run the exact `sase-ck` attribution node, `sase-cf` SIGKILL-capacity node, and `sase-cg` watchdog node. Repeat the
     two timing-sensitive nodes enough times to distinguish a one-off green run from a persistent flake.
   - Run the standard 28-worker `just test` lane once and record the outcomes of the named flake tests even if unrelated
     tests fail. Do not misattribute unrelated failures to these beads.
   - Verify `86d820f0b` is still reachable from `HEAD`, that the configured `sase-core-rs` lower bound is at least
     `0.17.3`, and that the published-minimum smoke reports the expected floor before closing `sase-cn`.

4. For each confirmed closure, call `sase bead close` individually with a specific `--note`, `--reason`, and explicit
   `--resolution`; do not use `--force` and do not batch beads that need different evidence. Use `done` for completed
   work (`sase-ce`, `sase-cn`), `superseded` for the exact duplicate (`sase-ch`, pointing to `sase-cm`), and `canceled`
   for failures that are stale or no longer reproducible (`sase-cf`, `sase-cg`, `sase-ck`, `sase-cq`, `sase-cr`). If any
   prerequisite does not hold, keep that bead open and append a note describing the contradictory result instead.

5. The full planning run independently exposed two failures in `tests/test_sdd_file_writes.py`: the flat-sidecar and
   seeded-parent fixtures now omit required committed-plan `title` and `goal` fields. Search the refreshed bead
   inventory for an existing task covering those exact failures. If none exists and they still reproduce on unchanged
   master, create one standalone task bead with the exact node IDs and diagnostics, refine it while `open`, then mark it
   `ready`; do not fix it as part of this backlog-cleanup tale.

## Final verification

- Rerun `sase bead list --type task --format json --limit 0` and account for every task from the fresh starting
  inventory as either deliberately closed or deliberately retained; report any bead that changed concurrently.
- Use `sase bead show`/`history` on every closed bead to verify its resolution, reason, and evidence note, and confirm
  the kept tasks retain their prior actionable status.
- Run `sase bead doctor` and `sase validate` to ensure the event store, projections, references, generated state, and
  plan links remain healthy after the closure commits.
- Leave primary and linked source worktrees unmodified. Summarize the final close/keep matrix, the exact verification
  results, and any newly filed unrelated task; do not claim the full suite is green if only unrelated failures remain.
