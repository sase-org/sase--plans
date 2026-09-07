---
tier: tale
title: Unblock bead-work relaunches held by stale retry-agent bead claims
goal:
  sase bead work automatically relaunches phases whose beads are claimed by dead
  retry-descendant agents, preserves slots whose retry descendant is still live, and
  only blocks on genuinely foreign assignees.
size: medium
proposed_by: bbugyi200.athena.01d
create_time: 2026-09-07 07:40:57
status: wip
---

# Unblock `sase bead work` Relaunches When A Bead Is Held By A Stale Retry Agent

## Problem

`sase bead work <epic>` aborts with a blocked-cleanup error when a phase bead's assignee
is a retry-derived descendant of the planned relaunch owner:

```
BLOCKED  (blocked) sase-x7.4 bead=sase-x7.4  bead sase-x7.4 is assigned to
sase-x7.4.r0.r0, which does not match the relaunch owner sase-x7.4
Error: bead sase-x7.4 is assigned to sase-x7.4.r0.r0, which does not match the
relaunch owner sase-x7.4
```

How this state arises: the ace retry-edit action
(`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py`) relaunches a failed agent
under a derived name allocated by `allocate_retry_name`
(`src/sase/agent/names/_retry.py`, template `<base>.r@` → `sase-x7.4.r0`,
`sase-x7.4.r0.r0`, ...). The retry agent claims the phase bead, so the bead's `assignee`
becomes the retry name. When that retry also fails (or is interrupted), the stale claim
persists, and every later epic relaunch is hard-blocked until the user manually
dismisses agents in ace. In a multi-epic invocation (`sase bead work xr x7 -Y`) the
error also aborts the whole command mid-way.

## Root Cause

`_require_compatible_assignee` in `src/sase/bead/cli_work_cleanup_targets.py` only
accepts an assignee whose `current_owner_agent_name_key` exactly matches the slot's
`owner_name` or `launch_name`. It does not recognize that `sase-x7.4.r0.r0` is a
generated descendant of `sase-x7.4` (the same logical work slot), and it never consults
whether the assignee agent is even alive.

Two facts make the fix safe and purely local to this repo's CLI cleanup layer:

1. The Rust core's `preclaim_epic_work_plan` unconditionally reassigns every non-closed
   phase bead to the selected launch agent (and `claim_for_agent_launch` reassigns
   in-progress task beads). No core/wire change is needed; the Python guard is the only
   thing that blocks the relaunch, so it alone must learn to distinguish "foreign owner"
   from "same-slot stale descendant". This respects the Rust-core backend boundary:
   relaunch-cleanup policy for the `sase bead work` CLI already lives in this module.
2. The identity facade already exposes the lineage primitive:
   `agent_name_ancestors(name, identity)` (`src/sase/core/agent_identity_facade.py`)
   returns the dot-segment ancestor chain (for `sase-x7.4.r0.r0`: hood `sase-x7`, then
   `sase-x7.4`, `sase-x7.4.r0`, and the name itself), handling owner-root/globalized
   spellings. Liveness helpers (`_record_is_live`, `_record_current_state`) already
   exist in `cli_work_cleanup_targets.py`.

## Desired Behavior

When classification finds a bead whose assignee does not match the relaunch owner:

- **Foreign assignee** (not lineage-related to the slot): keep raising the existing
  `ForcedReuseCleanupError` with the current message, verbatim. This guard is the only
  protection against stomping another agent's claim and must stay intact.
- **Lineage-descendant assignee with no live agent** (no artifact record for the name,
  or only dead records — failed / interrupted / done): treat the assignee as compatible.
  Classification proceeds exactly as if the assignee matched (REMOVE of the failed owner
  artifact, relaunch selected), and the existing preclaim/claim steps reassign the bead
  to the new launch. This is the reported scenario and must become fully automatic,
  including under `--yes-to-all`.
- **Lineage-descendant assignee with a live agent record** (running or waiting): do not
  destroy or steal. Classify the slot as PRESERVE with a detail naming the descendant
  (e.g. `live retry sase-x7.4.r0.r0 is working bead sase-x7.4`), so the epic relaunch
  skips that phase and continues with the rest. This mirrors the existing
  live-family-member preserve semantics, and land gating stays correct because the land
  segment always waits on `%w(bead=<phase>)` for every phase bead (`render_multi_prompt`
  in `src/sase/bead/work.py`), not only on agent names. Preserving (rather than the KILL
  used for WAITING owners of a reused name) is deliberate: the descendant's name is not
  needed by the relaunch, so killing a queued retry would be gratuitous destruction.
- **Unresolvable live state** for a lineage descendant (the existing
  `active_status_for_record` "unknown live state" path): keep the blocking error. Never
  guess about a possibly-running agent.

The lineage test must be directional: an allowed name (slot `owner_name`,
`slot.owner_name`, or `slot.launch_name`) must appear among the _assignee's_ ancestors
(compared via `current_owner_agent_name_key`). The reverse relation (assignee is an
ancestor of the owner, e.g. bead assigned to `sase-x7` while relaunching `sase-x7.4`)
stays blocked.

## Implementation

All production changes are in `src/sase/bead/cli_work_cleanup_targets.py`:

1. Extend `_AgentOwnerView` with a name index, e.g.
   `records_by_name_key: dict[str, tuple[AgentArtifactRecordWire, ...]]`, populated in
   `load_agent_owner_view()` from `_record_agent_name(record)` keyed by
   `current_owner_agent_name_key(name, identity)`. (The existing scan already covers
   retry agents' artifact records; no extra scan.)
2. Replace `_require_compatible_assignee` with a conflict resolver that receives the
   `_AgentOwnerView` (reuse `view.identity` instead of re-creating
   `AgentIdentitySnapshot.current()` per call) and returns `CleanupTarget | None`:
   - `None` when there is no assignee, the assignee is directly allowed, or the assignee
     is a lineage descendant with no live record → caller proceeds with its normal
     classification.
   - A PRESERVE `CleanupTarget` (named for the classified owner, with `current_state`
     from the live descendant's `_record_current_state`, detail naming the descendant
     and bead, plus the descendant's `artifacts_dir`/`generation`) when a lineage
     descendant is live.
   - Raises the existing `ForcedReuseCleanupError` otherwise.
3. Thread the view through the call sites: `_classify_artifact_record` and
   `_classify_stale_registry_owner` take `view` (identity comes from it) and return the
   resolver's PRESERVE target instead of their normal result when one is produced. The
   family-member path already promotes preserved member targets to a slot-level
   PRESERVE, and `_classify_clan_owner` is unaffected.
4. Determinism across passes: the same classification runs at preview,
   `revalidate_bead_work_launch_selection`, and `_verify_cleanup_target_still_selected`
   (before the wipe). The resolver must be a pure function of the scanned records +
   assignee map so repeated runs agree; a descendant dying between preview and confirm
   surfaces as the existing "would become destructive after preview; rerun" error, which
   is acceptable.

## Tests

Add a new test module (keep test files under 500 lines; do not grow the existing
700-line `tests/test_bead/test_cli_work_epic_launch_cleanup.py`), e.g.
`tests/test_bead/test_cli_work_cleanup_assignees.py`, reusing the established harness
patterns (`seed_diamond` / `seed_task`, fake-home agent artifact writers,
`_phase_slot`-style slot builders — extract shared helpers into
`tests/test_bead/cli_work_helpers.py` if needed rather than importing from a test
module). Cover at least:

1. Reported scenario: slot `sase-x7.4`-style owner with a FAILED artifact record, bead
   assigned to a dead `.r0.r0` descendant (with its own FAILED record) → selection has
   no blocked targets, the owner is a REMOVE target, and the slot is in `launch_names`.
2. Assignee is a descendant with **no** artifact record at all → same outcome as (1).
3. Assignee is a **live** descendant → slot classified PRESERVE, excluded from
   `launch_names`, no error, and the preserve detail names the descendant.
4. Foreign assignee (unrelated bead/agent name) → still BLOCKED, exact existing message
   preserved.
5. Directionality: assignee that is an ancestor of the owner stays BLOCKED.
6. Task-bead relaunch path (classification-level with a task slot carrying
   `launch_name`): dead `.r0` descendant assignee no longer blocks.
7. Revalidation stability: a preview built with a dead-descendant assignee revalidates
   cleanly when nothing changed.

## Verification

- `just install` first (ephemeral workspace clones may have stale virtualenvs), then
  `just fmt` and `just check` (whole-repo lint gates + diff-scoped tests).
- Escalate to `just check-full` through a monitor only if the scoped selection escalates
  or reports an unusual selection, per the two-speed verification rule.

## Out Of Scope

- Rust core (`sase-core`) changes: preclaim/claim reassignment semantics already support
  the fix; no wire or binding change.
- Making a multi-epic `sase bead work a b ...` invocation continue past one epic's
  unrelated launch failure.
- Auto-dismissing/wiping the dead retry descendant's own registry entry or artifacts —
  it stays visible as history in ace; only the relaunch block is removed.
