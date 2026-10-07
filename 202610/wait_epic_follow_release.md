---
tier: tale
title: Follow through in every wait release path
goal:
  Route every wait release path through one shared epic-follow decision so an armed
  target is never released past an epic it launched.
size: medium
proposed_by: bbugyi200.athena.sase-1h7.5
bead: sase-1h7.5
create_time: 2026-10-07 09:36:24
status: wip
---

- **PARENT:**
  [202610/wait_for_epic.md](https://github.com/sase-org/sase--plans/blob/main/202610/wait_for_epic.md)
- **BEAD:**
  [sase-1h7.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h7/sase-1h7.5.md)

# Plan: Follow through in every wait release path

Implement epic phase `sase-1h7.5` (the `release` phase of `%wait(..., for_epic=)`). The
parent epic plan is `plan:202610/wait_for_epic.md`. Read that phase, plus its Contract
sections Persistence, Promotion and pinning, and Edge Cases, before editing. Where this
plan and the research report disagree, this plan and the parent epic plan win. The
research report is `sase artifact read explicit:ccd173b1ad8ee7cd437bf3e7`.

The reducer and the fact collector already exist and are not wired into any release
path. Do not reimplement their rules. `collect_epic_follow_facts` in
`src/sase/core/wait_dependency_resolution/_epic_follow.py` calls
`wait_epic_follow_reduce`. It skips targets whose persisted state is already
`following`. Cycle facts stay empty; the safety phase fills them.

Default stays false. A marker with no `wait_for_epics_of` never follows. Do not add a
feature flag, do not edit `sase/memory/`, and do not hand-edit `CHANGELOG.md`.

## Outcome

Every release path asks one function whether a waiter may proceed. An armed target in
`agent`, `launching`, or `blocked` keeps the waiter parked. The first time an armed
target reaches `following`, that same function returns a promotion patch. The patch
appends the epic ids to the bead wait, pins the target, and records the follow entry.
The promotion pass itself never writes `ready.json`. The next evaluation releases
through the ordinary bead-wait path.

## Shared decision

Add `src/sase/core/wait_dependency_resolution/_epic_follow_release.py`. Keep it out of
`_epic_follow.py`, `_resolution.py`, and the chop module so those files do not grow.

```text
resolve_wait_release(
    index,
    marker,
    *,
    waiter_dir,
    closed_bead_ids,
    now,
    dismissed_artifact_dir=None,
    fresh_index=None,
) -> WaitReleaseDecision
```

`marker` is the `waiting.json` object, or the dict the runner is about to write.
`WaitReleaseDecision` carries:

- `status`: the existing `WaitDependencyStatus` from `dependency_resolution_status`
- `follows`: the `EpicFollowDecision` list from the collector
- `promotion`: patch or `None`
- `stage`: patch or `None`
- `releasable`: bool
- `deferred_unconfirmed`: bool

Armed targets are the order-preserving intersection of `wait_for_epics_of` and
`waiting_for`, strings only. A missing or non-list field is empty. Empty armed targets
skip the collector. `releasable` is then exactly `status.resolved`, and both patches are
`None`. That is today's behavior.

Otherwise call `collect_epic_follow_facts` with the armed targets, `resolved_deps`, the
marker's `wait_epic_follows`, the waiter's `agent_meta.json` as `waiter_meta` (so
`bead_id`, `epic_bead_id`, and `phase_bead_id` feed the deadlock guard), `now`, and
`dismissed_artifact_dir`.

`releasable` is true only when all of these hold:

- `status.resolved`
- confirmation passed, when `fresh_index` was given
- no returned decision has state `agent`, `launching`, or `blocked`
- `promotion` is `None`

Pinned targets produce no decision. Their epic ids are already in `wait_for_beads`, so
`status.resolved` covers them. State `none` does not block. Do not include the time
floor in `releasable`. The runner still sleeps until `wait_until` after `ready.json`,
and the chop still writes `ready.json` without reading the clock.

Before emitting a promotion, when `fresh_index` is not `None`, call
`confirm_dependency_resolution` on the full marker the same way `_process_one_waiter`
does. If it does not confirm, drop both patches, set `releasable` false and
`deferred_unconfirmed` true, and return. Stale membership must not pin a follow. When
`fresh_index` is `None`, skip confirmation; unit tests of the decision use that. Every
caller that can release or promote passes a fresh index.

### Patches

One patch may update several targets. Apply promotion and stage together.

Promotion, for each new `following` decision:

1. Append `epic_ids` that are not already in `wait_for_beads`, preserving existing
   order. Record only the newly appended ids as `added_bead_ids`. An id the user already
   authored stays authored.
2. Append the target to `resolved_deps` when it is absent. Leave other memos alone.
3. Upsert the `following` entry in `wait_epic_follows`.

Stage, for each `launching` or `blocked` decision: upsert that entry and do not change
beads or `resolved_deps`. State `none` or `agent` removes that target's entry when one
exists. Never persist `none` or `agent`.

Entry shape, matching the contract:

```text
{target, state, epic_ids, added_bead_ids, members, since, reason, detail,
 resume_command, skipped_epic_ids}
```

`state` is `launching`, `following`, or `blocked`. `since` comes from the decision. Omit
empty optional strings. `added_bead_ids` is empty on a pure stage upsert.

### One collector change

`collect_epic_follow_facts` builds member facts only after the entity resolves, so a
kill of a still-launching sole member never reaches reducer rule 6. When
`dismissed_artifact_dir` is set and that directory is the target's only member, treat
the target as resolved for that call so the reducer sees `member_dismissed`. Other live
members keep the current `agent_resolved` result. Do not otherwise change collector or
reducer rules.

## Lock and compare-and-set

Move `_agent_directive_lock` and `_runner_slot_marker_lock` from
`src/sase/ace/tui/actions/agents/_directive_persistence.py` into
`src/sase/core/agent_directive_lock.py` as `agent_directive_lock` and
`runner_slot_marker_lock`. The chop and the runner must not import TUI actions. Update
the two call sites in the TUI module. Lock order stays directive lock, then the
runner-slot lock.

Add `apply_wait_epic_follow_patch(waiter_dir, expected, patch) -> ApplyResult` in the
new release module. Under those locks it re-reads `waiting.json` and aborts without
writing when any of these differ from `expected`:

- `waiting_for`
- `wait_for_epics_of`
- `wait_for_beads`
- `resolved_deps`
- a touched target is no longer in both `waiting_for` and `wait_for_epics_of`

On success, merge the patch into `waiting.json` and mirror onto `agent_meta.json`:
always `wait_epic_follows`; on a promotion also the unioned `wait_for_beads` and
`resolved_deps`. Write both files atomically. Refresh the artifact index once after the
locks release. Register the writer in
`tests/test_agent_artifact_marker_mutation_audit.py`. An aborted compare-and-set leaves
both files unchanged and is not releasable.

Add `apply_wait_until_preserving_follows(waiter_dir, wait_until)` beside it. Under the
same locks, re-read `waiting.json` and change only `wait_until`. Use it from the
dependency-then-time rewrite in `src/sase/axe/run_agent_wait.py` instead of dumping the
in-memory `waiting_data` dict. That dict was captured at park start and currently
clobbers follows, derived beads, and pins. Update the mutation-audit counts for
`wait_for_dependencies` to match the moved write.

## Callers

All of these call `resolve_wait_release`. None of them write `ready.json` on a pass that
returns a promotion or a stage patch.

### Runner initial check

`wait_for_dependencies` in `src/sase/axe/run_agent_wait.py` builds the initial marker
dict, then resolves it before the fast path. Fold promotion and stage into that dict and
write it with `write_waiting_marker`. If the decision is not releasable, park, even when
the agent predicate is already resolved and even when the epic bead is already closed.
The fast path that skips the wait runs only when the decision is releasable, `duration`
is `None`, and `wait_until` is `None`, which is today's extra condition plus the follow.

Keep `initial_dependencies_resolved` returning `bool` so its existing callers stay
valid. Add optional `wait_for_epics_of` and `previous_follows` arguments that default
empty. Empty armed targets return `status.resolved` after the same confirmation the
helper already performs. Non-empty armed targets return `decision.releasable` and do not
write. `wait_for_dependencies` should call the shared decision once rather than resolve
twice.

### Parked-runner fallback

`waiting_marker_dependencies_resolved` re-reads `waiting.json`, resolves, applies a
patch when present, and returns true only when `releasable` is true and this call did
not apply a promotion or a stage change. A compare-and-set abort returns false. The
60-second interval stays as it is.

### AXE `wait_checks`

In `src/sase/scripts/_chop_wait_checks_run.py`, `_process_one_waiter` parses
`wait_for_epics_of` and `wait_epic_follows` (missing or non-list means empty) and calls
the shared decision with the chop's existing `fresh_index`. Apply patches. A
compare-and-set abort or `deferred_unconfirmed` increments `deferred_unconfirmed`, logs
the existing confirmation line, and does not write `ready.json`. When the agent
predicate is resolved but a follow blocks, log the target state and do not treat it as a
terminal-blocker or `unknown_outcome`. Write `ready.json` only when `releasable`.

### Kill and dismiss

In `_resolve_waiters_before_artifact_delete`, the index is built while the directory
still exists. For a waiter whose armed set contains the deleted name, call the shared
decision with `dismissed_artifact_dir` set to that directory before any name memo:

- `following`: apply the promotion. That pin is the only `resolved_deps` write. Do not
  write `ready.json` on this pass.
- `launching` or `blocked`: apply the stage patch. Do not memoize the target. Do not
  write `ready.json`.
- `agent`: do not memoize that armed target and do not write `ready.json`. Other live
  members still have to finish. The sole-member collector change above is what turns a
  sole launching member into `blocked` instead.
- `none`, or a target that is not armed: keep today's memoize-or-ready path.

### Run-now

`src/sase/ace/tui/actions/agents/_wait_actions.py` keeps releasing immediately. Do not
call the shared decision from run-now.

## Wires

Add `wait_epic_follows` to `WaitingMarkerWire` and `AgentMetaWire` in Rust and Python.
Open the linked repo with `sase repo open sase-core` and read its `AGENTS.md` before
editing it.

Follow the `wait_for_epics_of` field, not a schema bump:

- Rust: trailing field, `#[serde(default, skip_serializing_if = "Vec::is_empty")]`, a
  small `WaitEpicFollowWire` struct with `serde(default)` on every field. No
  `deny_unknown_fields` on the scan wires. Update `agent_meta_from_object` and
  `waiting_marker_from_object` in `crates/sase_core/src/agent_scan/scanner.rs` with a
  lenient list coercion: drop entries that lack a string `target`, ignore unknown keys,
  coerce id lists to strings.
- Python: the same dataclass and a `wait_epic_follows_from_value` coercion in
  `src/sase/core/agent_scan_wire_markers.py`, used from both converters in
  `agent_scan_wire_conversion.py`. `asdict` already emits the field. If the markers
  module would cross the `toobig` limit, put the new type and coercion in a sibling
  module and re-export them.
- Do not bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` (it stays 12) and do not change
  `SUPPORTED_AGENT_SCAN_WIRE_SCHEMA_VERSIONS`. Release reads `waiting.json` directly.
  The index refresh after the patch rewrites the row. Empty on a stale cached row
  matches the `wait_for_epics_of` contract.
- Exhaustive Rust struct literals that construct `AgentMetaWire` or `WaitingMarkerWire`
  must set the new field or use `..Default::default()`. The compiler lists them. Known
  constructors include the two scanner functions and the test helpers under
  `agent_scan/` and `fleet_contract/`.

This phase adds no binding. Do not hand-edit `sase-core-revision.txt`. The host writes
that pin when it commits sase-core and then sase. Before `sase tool run check` in sase,
run `just rust-install` so the venv extension matches the sase-core checkout.

## Docs

In `docs/macros.md`, document `for_epic` in the `%wait` row of Supported Directives, in
the Directive Completion Matrix row for `%wait` / `%w`, and next to the `%wait`
examples. Cover the grammar, the four error messages already implemented by the contract
phase, and the semantics. The default in this phase is still false. State explicitly
that `for_epic=` is not a wait on tribe `@epic`: `%wait:@epic` still waits for the next
agent or clan in that tribe.

In `docs/axe.md`, rewrite the sentence that a wait on an epic-approved planner does not
wait for the epic it launched. An ordinary wait still stops at the planner. A wait that
armed `for_epic=true` stays parked through the launch and then waits on the recorded
epic beads. `wait_checks` and the runner fallback use the same decision, persist
`launching` and `blocked`, and write `ready.json` only when the decision is releasable.

## Tests

Add `tests/test_wait_epic_follow_release.py`. For each row below, build one fixture
snapshot and assert that `resolve_wait_release`, `initial_dependencies_resolved`,
`waiting_marker_dependencies_resolved`, the `wait_checks` chop, and the kill/dismiss
path agree. Reuse the projects-dir helpers in `tests/_axe_chop_wait_checks_helpers.py`
and the existing chop and kill tests rather than copying a second runner. Drive the chop
through the same `wait_checks` entry those tests use, and drive dismiss through
`delete_agent_artifacts` or `_resolve_waiters_before_artifact_delete`.

| Snapshot                                                                   | Agreement                                                                                                               |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Target ends with no plan, a tale, or `plan_committed`                      | `none`, releasable, no follow fields written                                                                            |
| A later session member has a recorded epic                                 | promotion appends that id                                                                                               |
| Plan still in review                                                       | `agent`, not releasable                                                                                                 |
| Plan rejected                                                              | stays parked, same as today's predicate                                                                                 |
| `epic_approved` without `created_epics`, then a record appears             | `launching` does not release; the later record promotes                                                                 |
| Proc fallback or `skip` with `epic_launch_argv.json` and no epic           | `launching`, then `blocked` with `launch_skipped` or `launch_ended_without_epic` and a resume command; neither releases |
| Record write missing, bead-store attribution returns an id                 | promotion uses that id                                                                                                  |
| Failed launch                                                              | stays parked; no `ready.json`                                                                                           |
| Several epics on one run or several members                                | every id is appended once                                                                                               |
| Waiter `epic_bead_id` equals the only recorded epic                        | guard skips it, state `none`, id is in `skipped_epic_ids`, releasable                                                   |
| Phase or land worker target                                                | inherited epic is not followed; a child epic it launched is                                                             |
| Same target already `following`                                            | a newer run's epic is not appended                                                                                      |
| Marker has no `wait_for_epics_of`                                          | releasable exactly as today, even if the target has `created_epics`                                                     |
| Followed epic id is in `closed_bead_ids` on the evaluation after promotion | releasable; canceled and superseded beads need no special case                                                          |
| Epic id from another project                                               | the full id is appended unchanged and passed through `closed_bead_ids_for_waits`                                        |
| `--plan` name absent from `wait_for_epics_of`                              | releases when that row completes                                                                                        |

Also add these focused cases:

- A promotion pass does not write `ready.json`. The next chop or fallback does, once the
  bead is closed.
- Compare-and-set aborts after `waiting_for` changes the way a `w` edit would, and
  neither file gains the epic id.
- `apply_wait_until_preserving_follows` changes `wait_until` and keeps follows, derived
  beads, and pins. Extend `tests/test_run_agent_wait_fallback.py` if its sleep loop can
  observe the rewrite; otherwise the helper test is the proof and the runner must call
  that helper.
- Dismiss of an armed sole member that is `launching` persists `blocked` with reason
  `target_dismissed_during_launch` and does not memoize the name.
- Dismiss of an armed member that already has `created_epics` promotes before memoizing.
- Run-now on an armed waiter still releases immediately and does not append beads.
  Extend the existing run-now test under `tests/ace/tui/`.
- After promotion, `agent_meta.json` `wait_for_beads` contains the new ids.
  `portable_metadata` in `src/sase/agents_sync/inventory_io.py` publishes them, and
  `project_agent_wait_bead_rows` emits `awaits`. Extend
  `tests/artifact_links/test_agent_wait_bead_projection.py`. Do not add a second
  projection rule.
- Scan-wire round-trip: a waiting marker and an agent meta with one follow entry survive
  Rust scan and Python `from_dict`. An older payload without the key yields an empty
  list.

Do not add a re-park path for an epic that reopens after release. Bead waits do not
re-park, and this phase must not either.

## Verification and bead close

Read `sase/memory/lint_and_test.md` with `sase memory read` before finishing, because
this changes tracked sase files. If Symvision fails, read `sase/memory/symvision.md` the
same way. New public symbols must be used.

Run `sase tool run check` from the sase checkout. Run `sase tool run check` from the
sase-core checkout after the wire change. Do not run `just check-full`. A failure that
reproduces on the clean base does not keep the bead open: record it with
`sase bead note sase-1h7.5 'PROPOSED FOLLOW-UP: <summary — detail>'` and close anyway.

This work completes only `sase-1h7.5`. Do not close parent `sase-1h7` or any ancestor.
Do not create beads. Record any other discovered follow-up with `sase bead note` as a
`PROPOSED FOLLOW-UP:` line. Before closing, run `sase bead epic-symbols sase-1h7.5` and
clear any `--epic-symbol` leftovers by resolving them or re-keying the Justfile line to
a still-open bead. `sase bead close` refuses while leftovers remain. Then:

```bash
sase bead close sase-1h7.5 --note "<what you verified>"
```

Do not set the bead status by hand.

## Out of scope

- Loading follows into the TUI Agent model, render cache, counts, and phrasing (phase
  `model`).
- Blocker inbox notifications and cycle facts (phase `safety`).
- The teal `↪` row, lane, toast, and timeline (phase `tui`).
- The wait-modal toggle, CLI export, Jinja `created_epic`, and Telegram text (phase
  `surfaces`).
- Flipping the default to true, and `sase bead work` emitting `for_epic=false` (phase
  `flip`).
- `sase agent wait --for-epic` and any memory edit.
