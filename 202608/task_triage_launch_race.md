---
tier: tale
title: Defer task triage gates while a task bead's launch is in flight
goal:
  The bead_task_triage chop never raises a new TaskTriage gate or notification for a
  task bead whose detached launch is still running, while still replacing gates for
  beads that a failed launch returned to ready.
size: small
proposed_by: bbugyi200.athena.wy
create_time: 2026-08-10 09:34:33
status: wip
---

# Stop `bead_task_triage` From Re-Gating Task Beads Whose Launch Is In Flight

## Problem

Answering a `TaskTriage` gate with **Launch** sometimes produces a brand-new
`TaskTriage` notification for the same task bead seconds later, while the agent for it
is still starting.

The suspicion is confirmed, and the cause is a real race, not a display artifact.

### Why it happens

1. `execute_gate_selection` persists `response.json` first and only then calls
   `adapter.apply_side_effects(...)` (`src/sase/notification_gates/executor.py` around
   lines 221-232). From the instant the response lands, the chop's `_gate_state()`
   reports that gate as `terminal`.
2. The launch side effect does not touch the bead. `launch_task_triage`
   (`src/sase/bead/_task_gate_actions.py`) calls `submit_task_launch_task`, which only
   submits a detached background task running `sase bead work <task-id> --yes-to-all`.
3. That worker leaves the bead in `ready` for its whole prologue. `preclaim` — the first
   write that moves the bead to `in_progress` — sits at
   `src/sase/bead/cli_work_task.py:217-232`, after xprompt lookup, VCS context
   resolution, prompt rendering, the force-reuse preview, and force-reuse cleanup.
4. Meanwhile `_reconcile` in `src/sase/scripts/sase_chop_bead_task_triage.py:484-577`
   sees a recorded gate that is `terminal` and a bead that is still `ready`, drops the
   mapping (line 547), pushes the issue back into `live_tasks` (line 552), and the
   creation loop raises `g<N+1>` — which appends a fresh notification row through
   `create_gate` → `append_notification_strict`.

So the whole gap between "human answered Launch" and "worker reaches preclaim" is a
window in which any checks-lumberjack tick re-raises the gate the human just answered.

### Evidence gathered on the host

- Task-bead launch wall clock, from `~/.sase/tasks/tasks.jsonl`: 16.3s, 18.2s, and 22.3s
  for the three most recent `sase bead work <task-id>` rows. The bead is `ready` for a
  large part of each.
- Scanning every `~/.sase/interaction_requests/task_triage` bundle for "gate answered
  with `launch`, then a later generation for the same bead" yields 22 hits, 8 of them
  well inside a single launch:

  | bead      | generations | delay after the Launch answer |
  | --------- | ----------- | ----------------------------- |
  | `sase-e2` | g11 → g12   | 3.3s                          |
  | `sase-bt` | g1 → g2     | 4.6s                          |
  | `sase-d0` | g1 → g2     | 14.1s                         |
  | `sase-bo` | g1 → g2     | 29.9s                         |
  | `sase-hg` | g4 → g5     | 30.8s                         |
  | `sase-hh` | g4 → g5     | 53.3s                         |
  | `sase-hl` | g7 → g8     | 62.5s                         |
  | `sase-bp` | g1 → g2     | 89.1s                         |

  The remaining hits are hours or days later and are legitimate re-gating of a bead that
  genuinely came back to `ready`.

- The checks lumberjack runs its first tick immediately on start (`Lumberjack.run` in
  `src/sase/axe/lumberjack.py`), so an axe restart shortly after a launch reproduces
  this deterministically rather than at the 5-minute cadence.
- The behavior is currently codified as intended:
  `tests/test_axe_chop_bead_task_triage.py::test_terminal_gate_for_still_ready_task_is_regenerated`,
  plus `docs/axe.md` and `docs/notifications.md` both promise that a terminal gate on a
  still-gateable bead is replaced on the next tick.

### What must NOT change

Replacement-on-terminal is correct in general and has to stay. When a launch fails,
`rollback_task_work_launch` restores the bead to its prior status, and the human then
genuinely needs a new gate. The defect is the missing exception for launches that are
still in flight, not the rule itself.

## Fix

Introduce one invariant: **a task bead with a launch in flight is not gateable on this
tick.** The tasks store already records exactly that fact — the detached launch row
exists from the moment the side effect runs and reaches a terminal status when the
worker exits.

### 1. `src/sase/bead/task_launch.py`

- Extract the row predicate currently inlined in `_active_task_launch` (tags superset of
  `_TASK_LAUNCH_TAGS`, `command[:3] == ["sase", "bead", "work"]`, `len(command) >= 4`)
  into one shared private helper.
- Add a public `active_task_launch_bead_ids() -> frozenset[str]` that makes a single
  `read_tasks(status=ACTIVE_TASK_STATUSES, kind=DETACHED_TASK_KIND)` pass and returns
  `command[3]` for every matching row. Add it to `__all__`.
- Match on the bead ID only. Do not filter on the row's `project` field: task rows carry
  the cwd-inferred project name from `infer_project_name_from_cwd` while the chop
  iterates ProjectSpec keys such as `gh_sase-org__sase`, so a project comparison would
  silently never match. Bead IDs already carry their project prefix.
- `_active_task_launch` keeps its exact current behavior (newest-first, returns the row)
  and reuses the extracted predicate.

### 2. `src/sase/scripts/sase_chop_bead_task_triage.py`

- Read the in-flight set once per tick, before the project loop. On failure, log a
  warning and continue with an empty set: a tasks-store hiccup must not block triage for
  every project, and an empty set is exactly today's behavior.
- **Creation loop** (`for bead_id, issue in sorted(live_tasks.items())`): skip any bead
  in the in-flight set before `_create_gate`. Do not create the gate, do not bump
  `generations`, do not record `gates`/`fingerprints`/`kinds`. Count it in a new
  `deferred` counter.
- **Reconcile loop**, recorded gate is `pending` and the bead is in flight: leave the
  gate exactly as it is — `continue` before the wrong-kind and fingerprint comparison
  and before the mapping delete — and count it as `skipped`. This also stops a note or
  `+1` that lands mid-launch from cancelling and re-raising a gate the human is already
  acting on.
- **Reconcile loop**, recorded gate is `terminal` or `missing` and the bead is in
  flight: keep the existing mapping delete, and let the creation-loop skip suppress the
  replacement. Because `generations` is untouched, the first tick after a failed launch
  still issues `g<N+1>` with a fresh deterministic request ID.
- Add `deferred` to `_summary(...)`. Deferrals are not changes, so the existing
  `no_triage_changes` reason logic stays keyed on `gated or canceled or resnoozed`.

### 3. Self-healing

No new state, no new marker files, no TTL. When the launch row reaches `success`,
`error`, or `killed`, the bead simply becomes gateable again on the next tick: still
`ready` (rollback) means a replacement gate, `in_progress` means the stale mapping is
cleaned up as it is today.

## Explicitly out of scope

- Do **not** move `preclaim` earlier in `launch_task_bead_work`, and do not introduce a
  `claimed` status for task launches. The preclaim deliberately sits after the
  destructive-cleanup and launch confirmations, and even an earlier claim would not
  cover the gap between the gate answer and the worker process's first store write.
- Do **not** add a time-based grace period. Active/terminal task rows are the truthful
  signal, and the tasks supervisor already reconciles rows whose submitter died
  (`_UNCLAIMED_GRACE_SECONDS` in `src/sase/tasks/runner.py`).
- Do **not** touch the `BeadSnooze` branch, the fingerprint payload, or
  `_PRESENTATION_FORMAT_VERSION` / `_GATE_CONTRACT_VERSION`. Bumping either constant
  cancels and re-raises every pending gate on the next tick.
- No `sase-core` change is needed. Both inputs already cross the binding
  (`bead_read_facade.list_issues` and `tasks.store.read_tasks` → `read_tasks_snapshot`);
  the new predicate is a thin Python filter living beside its existing sibling.
- Pending gates orphaned under a project key that has left the enabled inventory are a
  separate defect, filed as task bead `sase-ir`. Do not fix them here.

## Tests

`tests/test_axe_chop_bead_task_triage.py` (and
`tests/_axe_chop_bead_task_triage_helpers.py` for a small `patch_active_launches`
helper):

- Terminal recorded gate + bead in flight → no new gate; counters `gated=0`,
  `deferred=1`; no new bundle created.
- Same state on a later tick with the launch no longer in flight → `gated=1` and the new
  request ID ends with `-g2`, proving the generation counter survived the deferral.
- No recorded gate at all + bead in flight → nothing created and no state entry written
  for that bead.
- Pending recorded gate + bead in flight + changed presentation fingerprint → gate
  untouched (`canceled=0`, `skipped=1`), and the recorded mapping survives.
- `active_task_launch_bead_ids` raising → the chop logs a warning and behaves exactly as
  it does today.
- Keep `test_terminal_gate_for_still_ready_task_is_regenerated` green with an empty
  in-flight set, and update the counter dicts asserted in this file plus
  `tests/test_axe_chop_bead_task_triage_presentation.py` and
  `tests/test_axe_chop_bead_task_triage_snooze.py` for the new `deferred` key.

`tests/test_bead/test_task_launch.py`:

- `active_task_launch_bead_ids` returns bead IDs for active, correctly-tagged
  `sase bead work` rows and ignores terminal rows, untagged rows, non-detached rows, and
  epic plan-file launches (whose `command[3]` is a path).
- `_active_task_launch` still returns the newest active row for a bead (regression guard
  for the extraction).

## Docs and configuration

- `docs/axe.md`, `bead_task_triage` section: the sentence "If a gate becomes terminal or
  its bundle disappears while the task is still gateable, the next tick replaces it"
  gains the in-flight-launch exception.
- `docs/notifications.md`, Task Triage Notification: same exception on its "If a gate
  becomes terminal, disappears, or uses an obsolete presentation..." sentence.
- `docs/beads.md` around line 514: check whether it repeats the replacement rule and
  update it if so.
- `src/sase/default_config.yml`, the `bead_task_triage` chop `description`: state that a
  bead whose launch is in flight is deferred instead of re-gated.

## Verification

1. `just install`, then `just check`.
2. `just check-full` before landing.
3. Manual confirmation of the actual bug: answer a `TaskTriage` gate with **Launch**,
   then immediately run `sase axe chop run bead_task_triage -V` while the launch is
   still in flight. Before the fix this prints `gated=1` and creates a `-g<N+1>` bundle
   plus a notification; after the fix it prints `deferred=1` with no new bundle. Re-run
   once the worker is `in_progress` and confirm the stale mapping is cleaned up without
   a new gate.
