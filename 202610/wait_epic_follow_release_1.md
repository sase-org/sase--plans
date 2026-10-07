---
tier: tale
title: Route every wait release through epic follow
goal: "Every release path decides epic follow through one shared function, so an armed
  %wait(for_epic=) target is never released past an epic it launched. The keyword stays
  opt-in (default false).

  "
size: medium
proposed_by: bbugyi200.athena.sase-1h7.5
bead: sase-1h7.5
create_time: 2026-10-07 12:24:01
status: wip
---

- **BEAD:** sase-1h7.5

# Plan: Route every wait release through epic follow

Implement phase bead **sase-1h7.5** only. The parent epic is
`plan:202610/wait_for_epic.md` (`%wait(..., for_epic=)`). Phases `contract` (sase-1h7.3)
and `reducer` (sase-1h7.4) are already on master. This tale is the `release` phase. The
working tree does not contain it: an earlier close was reopened because that commit
never landed. Implement from this plan. Do not hunt for the lost diff.

This is one tale because the approved epic already bounded the work as a single phase,
and one coder can land it from the call sites below without another planning pass. Do
not open a nested epic.

## Outcome

An armed target (`wait_for_epics_of ∩ waiting_for`) cannot release while its follow
state is `agent`, `launching`, or `blocked`, or while a promotion to `following` is
still unpersisted. A marker with no `wait_for_epics_of` field releases exactly as it
does today. `for_epic` stays default **false**. Run-now still releases immediately.

## Do not redo

Already landed, and out of scope to redesign:

- Grammar, diagnostics, and `wait_for_epics_of` persistence
  (`src/sase/macro/_directive_extract.py`, `src/sase/axe/run_agent_wait.py` launch
  payload, Python and Rust scan wires).
- Pure reducer `wait_epic_follow_reduce` in sase-core
  `crates/sase_core/src/wait_epic_follow.rs`, plus the Python collector
  `collect_epic_follow_facts` in
  `src/sase/core/wait_dependency_resolution/_epic_follow.py`. The collector already
  skips pinned `following` targets, honors `dismissed_artifact_dir`, and passes
  `cycle_epic_ids=()` until the later safety phase. Do not add cycle detection here.
- `update_agent_meta_locked` in `src/sase/core/agent_meta_update.py`.

Leave `WAIT_FOR_EPIC_DEFAULT` false. Do not edit TUI rendering, the wait modal, Jinja
`created_epic`, blocker toasts, or memory files. Those belong to later phases. Do not
edit `CHANGELOG.md` or any crate version.

## Shared decision

Add `src/sase/core/wait_dependency_resolution/_epic_follow_release.py`. Do not grow
`_epic_follow.py` (it is already over 500 lines).

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

`marker` is the `waiting.json` object (or the launch dict before that file exists).
`waiter_dir` is the waiter artifact directory. `now` is an explicit epoch float so tests
do not use the wall clock.

`WaitReleaseDecision` carries:

- `status`: the existing `WaitDependencyStatus` from `dependency_resolution_status` (the
  fresh status when confirmation ran).
- `follows`: the collector decisions for this pass.
- `patch`: one optional `WaitEpicFollowPatch`, or `None` when nothing must be written.
- `releasable`: bool.
- `confirmation_failed`: bool.

Armed targets are the ordered intersection of `wait_for_epics_of` and `waiting_for`. A
missing or non-list `wait_for_epics_of` means there are no armed targets. Implicit
targets that the contract phase kept out of `wait_for_epics_of` stay unarmed.

### When there are no armed targets

Call `dependency_resolution_status` exactly as the callers do today (`waiting_for`,
`wait_for_artifacts`, `wait_for_fork_sources`, `wait_for_beads`, `wait_for_hoods`,
`resolved_deps`, `closed_bead_ids`, `self_artifact_dir=waiter_dir`). Confirm with the
existing `confirm_dependency_resolution` only when the caller passes `fresh_index` and
the status is resolved and the wait is agent-shaped. `releasable` is the confirmed
result (or the unconfirmed `status.resolved` when there is nothing agent-shaped to
confirm, matching `_confirmation.py`). `patch` is `None`. This path must not call the
collector.

### When there are armed targets

1. Compute `status` on `index`.
2. Before any promotion, re-confirm membership with `confirm_dependency_resolution` and
   the caller’s `fresh_index`. This is the same fresh-index check the release paths
   already use, so a late session successor is not missed. If `fresh_index` is missing,
   or confirmation is not confirmed, set `confirmation_failed`, withhold the patch, and
   set `releasable` false. Do not promote off a stale view.
3. On a confirmed view, recompute status and call `collect_epic_follow_facts` on the
   **fresh** index. Pass armed targets, `resolved_deps`, the marker’s current
   `wait_epic_follows` as `previous_follows`, waiter meta read from
   `waiter_dir/agent_meta.json` when present (so `bead_id` / `epic_bead_id` /
   `phase_bead_id` feed the deadlock guard), `now`, and `dismissed_artifact_dir`.
4. Build one patch from those decisions (below). `releasable` requires all of: fresh
   `status.resolved`, confirmation succeeded, no armed decision in `agent` / `launching`
   / `blocked`, and no pending promotion. A first transition to `following` is a pending
   promotion, so that pass is not releasable even when the new epic beads are already
   closed. The next pass, after the patch is stored, releases through ordinary bead
   waits.

Persist `launching` and `blocked` (including `since`) even when some other dependency is
still unresolved. Promote a target the first time it reaches `following` even when other
dependencies are still open. `releasable` stays false until `status.resolved`.

`none` is never persisted. A previous `launching` or `blocked` entry whose new state is
`none` is dropped. Pinned `following` entries that the collector skipped are kept
unchanged and are not re-promoted.

### Patch contents

`WaitEpicFollowPatch` holds the full replacement lists plus the compare-and-set
preimage:

- `wait_for_beads`: previous beads, then each new epic id that is not already present.
  Keep ids verbatim, including another project’s full id. Do not write them back into
  authored `%wait` text.
- `resolved_deps`: previous deps, plus the target name when that target is newly
  `following`. Do not add it for `launching` or `blocked`.
- `wait_epic_follows`: the persisted stage list. Each entry has `target`, `state`
  (`launching`, `following`, or `blocked`), `epic_ids`, `added_bead_ids`, `members`,
  `since`, `reason`, `detail`, `resume_command`, `skipped_epic_ids`. `added_bead_ids` is
  only the ids this promotion appended for that target.
- `expected`: the preimage of `waiting_for`, `wait_for_epics_of`, `wait_for_beads`,
  `resolved_deps`, and `wait_epic_follows` taken from the marker this decision was
  computed against.

`since` comes from the reducer decision (it already preserves `previous_since` when the
state is unchanged).

## Lock and apply

Move the locks out of the TUI so the chop and the runner do not import TUI actions.

Add `src/sase/core/agent_directive_lock.py`:

- `agent_directive_lock(artifacts_path)` is today’s `_agent_directive_lock` from
  `src/sase/ace/tui/actions/agents/_directive_persistence.py` (exclusive flock on
  `.agent_directive_persistence.lock`).
- `runner_slot_marker_lock()` is today’s `_runner_slot_marker_lock` (it already
  delegates to `sase.core.runner_slots.runner_slot_admission_lock`).

Point the TUI module’s private helpers at these functions so
`persist_agent_directive_update` and `_write_waiting_marker` keep the same lock order.
Do not grow `_directive_persistence.py` (523 lines) with new logic.

`apply_wait_epic_follow_patch(waiter_dir, patch) -> bool`:

1. Hold `agent_directive_lock` and, inside it, `runner_slot_marker_lock` (same
   serialization as a TUI `waiting.json` write).
2. Re-read `waiting.json`. If it is missing, or the five preimage fields differ from
   `patch.expected`, or the promoted/updated target is no longer in both `waiting_for`
   and `wait_for_epics_of`, return `False` and write nothing.
3. Otherwise update only `wait_for_beads`, `resolved_deps`, and `wait_epic_follows`.
   Leave `wait_until`, queue fields, and every other key untouched. Write atomically.
4. Mirror those three fields into `agent_meta.json` through `update_agent_meta_locked`,
   without clobbering other meta keys.
5. Call `update_agent_artifact_index_for_marker_mutation` for the waiter dir after the
   writes.

Register the new writer in `tests/test_agent_artifact_marker_mutation_audit.py`
`_REVIEWED_MARKER_MUTATION_CONTEXTS`. The context key is `path:function`. Copy the
lifecycle call `update_agent_artifact_index_for_marker_mutation`. Match `mutation_calls`
to what the audit scanner actually sees; run the audit test and fix the entry until
`test_tracked_marker_mutation_sites_are_reviewed` passes. A meta write that only goes
through `update_agent_meta_locked` is already reviewed there.

Also add `set_waiting_until(waiter_dir, wait_until) -> None` in the same release module.
Under the same two locks, re-read `waiting.json` and change only `wait_until`, then
refresh the index. If the file is missing, do nothing. This is the two-stage clobber
fix. Register it in the same audit map if the scanner treats it as its own mutation
context.

## Call sites

All four release paths call `resolve_wait_release`. They apply `patch` when it is not
`None`. They write `ready.json` only when `releasable` is true **and** this pass did not
apply a patch. A failed apply (compare-and-set abort) leaves the waiter parked.

### Runner initial check

`src/sase/axe/run_agent_wait.py` and `initial_dependencies_resolved` in
`src/sase/axe/run_agent_wait_deps.py`.

Today the fast path calls `initial_dependencies_resolved` **without**
`wait_for_epics_of` and, on true, proceeds without writing `waiting.json`. Thread
`wait_for_epics_of` through. Build the synthetic marker from the launch arguments and
call `resolve_wait_release` with the same `build_index` closure the function already
uses as `fresh_index`.

- `releasable` and no `duration` and no `wait_until`: keep today’s “Dependencies already
  satisfied, proceeding without waiting” path.
- Otherwise, when there are dependencies: build the first `waiting.json` as today, fold
  `patch` into that dict **before** `write_waiting_marker`, and park. Do not release on
  the pass that folds a promotion in.

Keep `initial_dependencies_resolved`’s bool return for existing callers: it is true only
when `releasable`. The initial-check caller in `run_agent_wait.py` needs the patch, so
return the decision from a shared helper and let the bool wrapper read `.releasable`.

### Parked-runner fallback

`waiting_marker_dependencies_resolved` (60s fallback in the same wait loop). Re-read
`waiting.json`, call `resolve_wait_release`, apply a patch, and return true only when
`releasable` and no patch was applied on this pass. Never release on a promotion pass. A
compare-and-set abort returns false.

### AXE `wait_checks`

`_process_one_waiter` in `src/sase/scripts/_chop_wait_checks_run.py` (478 lines; keep
the edit a thin call). Replace the inline `dependency_resolution_status` plus
`confirm_dependency_resolution` block with `resolve_wait_release`, passing the chop’s
existing `fresh_index`.

- Apply the patch. If a patch was applied, or `releasable` is false, do not write
  `ready.json`. Count the waiter unresolved unless confirmation failed.
- On `confirmation_failed`, keep today’s `deferred_unconfirmed` logs and return.
- On `releasable` with no patch, write `ready.json` exactly as today
  (`{"resolved_deps": waiting_for}`).
- On an unresolved status with no patch, keep today’s terminal-blocker logging off
  `decision.status`.

### Kill / dismiss

`_resolve_waiters_before_artifact_delete` in
`src/sase/ace/tui/actions/agents/_killing_utils.py`.

For a waiter whose armed set contains the deleted run: call `resolve_wait_release` with
`dismissed_artifact_dir` set to the deleted artifact dir **before**
`_memoize_completed_dependency`. Pass a fresh index rebuilt the same way
`build_wait_dependency_index` is already built in this function.

- Apply a promotion or stage patch first.
- If that target’s decision is `launching`, or `blocked` with reason
  `target_dismissed_during_launch`, do not memoize the target as resolved and do not
  write `ready.json` because of this dismiss.
- If the pass applied a promotion, do not write `ready.json` on this pass either. The
  next chop or fallback releases once the pinned beads are closed.
- If the target is not armed, keep today’s memoize-or-ready behavior unchanged.

The reducer already maps a dismissed reserved member with an empty epic set to `blocked`
/ `target_dismissed_during_launch`, and a recorded epic to `following`. Do not
reimplement that.

### Run-now

`_apply_wait` in `src/sase/ace/tui/actions/agents/_wait_actions.py` already writes
`ready.json` with `unwait: true` and does not consult dependency resolution. Leave that
path alone.

### Two-stage rewrite

In `run_agent_wait.py`, after dependencies resolve and a duration floor is set, the code
does `waiting_data["wait_until"] = deadline` and
`write_waiting_marker(artifacts_dir, waiting_data)`. That dict was captured at park
time, so it clobbers follows, derived beads, and pins written while the runner was
parked. Replace that write with `set_waiting_until`. Keep the in-memory deadline only
for the subsequent sleep.

## Wires

Add `wait_epic_follows` to `AgentMetaWire` and `WaitingMarkerWire` in both repos. Open
sase-core with `sase repo open sase-core -r "Add wait_epic_follows to the scan wires"`
and read that checkout’s `AGENTS.md` before editing. Work only in the printed path.

Follow the trailing additive pattern of `wait_for_epics_of` (serde default, omit when
empty, no schema bump). Commit `988af8f3bb` (`finalizer_status`) is the older template;
`created_epics` / `wait_for_epics_of` are the copies already in the tree. Do **not**
bump `AGENT_SCAN_WIRE_SCHEMA_VERSION`. An empty list must not change byte-stable scan
payloads.

Rust (`crates/sase_core/src/agent_scan/wire.rs`):

- New `WaitEpicFollowEntryWire` with the persisted fields above, every field
  `#[serde(default)]`. Use `skip_serializing_if` for empty vecs and `None` options,
  matching `CreatedEpicWire`’s style so absent markers stay stable.
- Append `wait_epic_follows: Vec<WaitEpicFollowEntryWire>` as the last field of
  `AgentMetaWire` (after `wait_for_epics_of`, around line 738) and of
  `WaitingMarkerWire` (after `wait_for_epics_of`, around line 1052).
  `#[serde(default, skip_serializing_if = "Vec::is_empty")]`.

Rust readers in `crates/sase_core/src/agent_scan/scanner.rs`: extend
`agent_meta_from_object` (the `wait_for_epics_of` assignment near line 1515) and
`waiting_marker_from_object` (near line 1823). Coerce leniently: drop malformed entries,
never fail the scan. Mirror `coerce_created_epics`.

Python (`src/sase/core/agent_scan_wire_markers.py`): a frozen `WaitEpicFollowEntryWire`
dataclass and a trailing `wait_epic_follows` field on `AgentMetaWire` and
`WaitingMarkerWire`, defaulting to an empty list. Lenient
`wait_epic_follows_from_value`, exported the same way as `wait_for_epics_of_from_value`.

Python converters in `src/sase/core/agent_scan_wire_conversion.py`: set the field in
both `_agent_meta_from_dict` and `_waiting_marker_from_dict`.

Update
`tests/test_core_agent_scan_wire_agent_meta.py::test_finalizer_status_is_trailing_wire_field`
so `wait_epic_follows` is the last `AgentMetaWire` field, and the default is `[]`.
Update the matching Rust field-order or scan fixture tests that fail. Round-trip one
populated entry through both the Python converter and the Rust scanner.

No new Python binding is required. The existing scan path must surface the field. A
declaration that commits both repositories gets `sase-core-revision.txt` from the host.
Do not hand-edit the pin unless `sase tool run check` in sase reports a missing binding
that only a pin bump fixes. Rebuild the local extension before Python tests that go
through the Rust scanner (`just rust-install` in sase, or `sase tool run` if that recipe
is guarded).

`sase-core` agents run `sase tool run check` from the sase-core checkout, never bare
`cargo` or a raw `just check`. `just fast` and targeted `just test -p …` are the inner
loop. `just check` under `sase tool run` is the gate and needs a long timeout (10
minutes or more).

## Docs

Default stays false. Document the opt-in keyword, not the future flip.

- `docs/macros.md` Supported Directives row for `%wait`: mention the opt-in
  `for_epic=true|false` follow. Completion-matrix `%wait` cell: parenthesized form also
  completes `for_epic=` (`true` / `false`). Syntax block near the existing `%wait`
  examples: add `%wait(planner, for_epic=true)` and `%wait(planner, for_epic=false)`,
  with one sentence each. Add a short subsection under the wait directive: grammar
  (`true`/`false`, per occurrence, colon form uses the default), the four error cases
  already implemented (`wait-for-epic-without-agent`, `wait-for-epic-invalid-value`,
  `wait-for-epic-conflict`, `wait-for-epic-plan-row`) by meaning, and the release rule
  (follow the epics that armed agent launched; release when those epic beads close;
  `launching` and `blocked` stay parked; absent field never follows). State that the
  default is false. Say `for_epic` is not tribe `@epic`.
- `docs/agent_sessions.md` “Epic bead-work example” (the paragraph that assigns tribe
  `@epic`): one sentence that `%wait(for_epic=)` follows epics an agent launches and is
  unrelated to tribe `@epic`.
- `docs/axe.md` in the `wait_checks` section, replace the sentence “A wait on an
  epic-approved planner waits for that planner, not for the host-owned epic it launched;
  use bead waits or a wait on the launched epic clan for that.” An ordinary wait still
  releases when that planner finishes. `%wait(planner, for_epic=true)` also waits until
  the epic that planner launched is closed. The runner initial check, the parked-runner
  fallback, `wait_checks`, and kill/dismiss share that decision. The default remains
  false.

## Tests

Add `tests/test_wait_epic_follow_release.py`. Build fixture trees the way
`tests/test_wait_epic_follow_collector.py` does (`tests/_agent_names_fixtures.py`
`make_agent`, `WaitDependencyIndex.empty().add_many`). One helper runs the same snapshot
through the initial-check helper, `waiting_marker_dependencies_resolved`, the chop’s
per-waiter decision, and (when a run is being dismissed) the kill path, and asserts they
agree.

Required cases, from the epic contract:

| Snapshot                                                                  | Agreement                                                                                                                                                 |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Target ends with no plan, a tale, or `plan_committed`                     | state `none`, releasable as today                                                                                                                         |
| Tale coder later has a recorded epic                                      | that epic is promoted                                                                                                                                     |
| Plan still in review                                                      | state `agent`, not releasable                                                                                                                             |
| `plan_rejected`                                                           | stays parked (today’s predicate)                                                                                                                          |
| `epic_approved` on the monitor path before a record exists                | `launching` or `blocked`, not releasable; after `created_epics` is present, `following` and a promotion                                                   |
| Proc fallback (`epic_launch_argv.json`, no epic yet)                      | `launching`, then `following` or `blocked`; a `launching` target does not release                                                                         |
| Skip mode (reserved, no argv)                                             | `blocked` `launch_skipped` with a resume command, not releasable                                                                                          |
| Record missing, bead-store attribution returns the epic                   | `following` (mock `attributed_epic_ids` the way the collector tests do)                                                                                   |
| Several recorded epics                                                    | every id is appended, in reducer order                                                                                                                    |
| Phase worker                                                              | inherited `epic_bead_id` is not followed; a child id in `created_epics` is                                                                                |
| Waiter’s own epic id                                                      | guard skips it (`skipped_epic_ids`), and a target whose only epic was skipped does not promote                                                            |
| Epic id from another project                                              | appended verbatim                                                                                                                                         |
| Marker without `wait_for_epics_of`                                        | no collector call effect; releasable matches today’s status                                                                                               |
| Already-`following` pin, plus a newer same-name run with a different epic | the new epic is not appended                                                                                                                              |
| Closed epic bead, including resolution `canceled` or `superseded`         | first pass promotes and is not releasable; second pass, with the bead in `closed_bead_ids`, is releasable                                                 |
| `ready.json` already present                                              | chop still skips the waiter (no un-release after a reopen)                                                                                                |
| Dismiss a `launching` armed target                                        | kill path persists `blocked` `target_dismissed_during_launch` and does not memoize the name; the other three paths are not given `dismissed_artifact_dir` |
| Run-now                                                                   | existing unwait ready write still happens and does not call `resolve_wait_release`                                                                        |

Also:

- Compare-and-set: compute a patch, change `wait_for_beads` on disk as a concurrent `w`
  edit would, `apply_wait_epic_follow_patch` returns false, and the file is unchanged.
- `set_waiting_until` preserves `wait_epic_follows`, derived `wait_for_beads`, and
  `resolved_deps` while updating `wait_until`.
- After a promotion, the published `wait_for_beads` list produces an `awaits` edge.
  Extend `tests/artifact_links/test_agent_wait_bead_projection.py` rather than inventing
  a new projection rule.
- Wire round-trip and the trailing-field order test above.

Extend the caller suites only where the shared helper is not what they execute:
`tests/test_run_agent_wait_deps_initial.py`, `tests/test_run_agent_wait_fallback.py`,
`tests/test_wait_dependency_release_confirmation.py` (confirmation still withholds a
promotion), `tests/test_axe_chop_wait_checks*.py`, and
`tests/test_kill_named_agent_dismiss_waiting.py`. A focused test in each file that an
armed `launching` planner does not write `ready.json`, plus one chop test that a second
pass after the patch does. Keep new public symbols used from those callers or tests so
symvision stays clean. Export from `wait_dependency_resolution/__init__.py` only symbols
that those external callers import.

## Verification

Read `sase/memory/lint_and_test.md` with `sase memory read` before finishing, because
this tale edits tracked sase files.

- Targeted pytest for `tests/test_wait_epic_follow_release.py` and the extended caller
  and wire tests while iterating.
- In sase-core, after the wire edit: `sase tool run check` from that checkout (long
  timeout).
- In sase: `sase tool run check`. Never `just check-full`. Never a raw `just check` or
  raw `cargo`.
- Rebuild the venv extension before sase tests that scan through Rust.

A check failure that reproduces identically on the clean base does not keep the bead
open. Known failures already noted on sase-1h7.5, and not caused by this phase:
`tests/ace/tui/test_app_import_budget.py` sitting on the module-count cap, symvision
private imports of `_runs` in `src/sase/agents_sync/v2_snapshot_io.py` and
`src/sase/ace/tui/widgets/decks/final/overview_card.py`, sase-core `cargo fmt` drift in
files this tale does not touch, and a flaky discard-guard test. If one of those is the
only red, confirm it on a clean tree, record a short `PROPOSED FOLLOW-UP:` on this bead
that cites the existing note, and close anyway. Fix failures this tale introduces (the
trailing-field assertion and the marker-mutation audit will go red until updated; those
are in scope).

## Close this bead only

Do not set status by hand. Do not create beads. Do not close parent epic `sase-1h7` or
any ancestor. A phase description that mentions closing an ancestor is preparation for
that ancestor’s land agent, not permission.

Discovered follow-up goes on this bead:

```bash
sase bead note sase-1h7.5 'PROPOSED FOLLOW-UP: <one-line summary — detail>'
```

Before closing:

```bash
sase bead epic-symbols sase-1h7.5
```

If any `--epic-symbol` entry still names this phase, resolve the symbol or re-key that
Justfile line to a bead that stays open (parent `sase-1h7`, or a later phase such as
`sase-1h7.6` / `sase-1h7.7`). `sase bead close` refuses while leftovers remain.

Then:

```bash
sase bead close sase-1h7.5 --note "<what you verified>"
```

The note names the tests that passed, the sase and sase-core checks, and any
pre-existing failure you recorded and closed over.
