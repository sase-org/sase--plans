---
tier: tale
title: Keep session fork waits blocked while a session successor is queued
goal:
  A `#fork:<session>` (or identity) wait never releases while any member of that session
  is still live or queued for a runner slot, and a failed shell that handed off to a
  follow-up no longer counts as a terminal session failure, without reintroducing the
  sibling dependency-wait deadlock or changing hold settlement.
size: medium
proposed_by: bbugyi200.athena.0s3
create_time: 2026-09-25 11:01:50
status: wip
---

# Keep `#fork:<session>` waits blocked while a session successor is queued

## Problem

`0rv.w0.f0` was launched with `#fork:0rv.w0 ...`, which implies a wait on the whole
`0rv.w0` agent session. It started at 09:48 local while `0rv.w0` was still working:
`0rv.w0--1` (the monitor follow-up coder) had not started yet, and `0rv.w0--mon-0` ran
after it.

Timeline (local time, 2026-09-25), taken from the `0rv.w0*` artifact dirs and the waits
lumberjack log:

- **07:23** `0rv.w0--code` starts (session `0rv.w0`, parent = root `0rv.w0--plan`).
- **08:22** `0rv.w0.f0--plan` is launched with
  `wait_for_fork_sources=[{kind: "session", name: "0rv.w0", artifact_dir: <0rv.w0--plan dir>}]`.
- **08:27** `0rv.w0--code` hands off to monitor `0rv.w0--mon`
  (`just fix-tui-screenshots && sase tool run check`).
- **09:45** The monitor command exits 1 (`monitor_state: "failed"`).
- **09:46:00** The monitor launches follow-up `0rv.w0--1` in the same session.
- **09:46:35** `0rv.w0--1` crosses its dependency barrier (`wait_completed_at`), then
  parks for a **runner slot**. All 8 slots are busy, so the runner writes a
  `waiting.json` queue marker with `slot_requested_at`.
- **09:47:01** The monitor writes `done.json`: `outcome: "monitored"`,
  `monitor_state: "failed"`, `monitor_followup_outcome: "launched"`,
  `monitor_followup_agent: "0rv.w0--1"`.
- **09:48:28** `wait_checks` logs `Dependencies satisfied for sase, waited on: 0rv.w0`
  and writes `ready.json` for `0rv.w0.f0--plan` (**bug**).
- **10:03** `0rv.w0--1` finally gets a runner slot. At 10:31 it hands off to monitor
  `0rv.w0--mon-0`.

A plain name wait on `0rv.w0` (the waiting `0rv.w0.f0--code`) still correctly blocks.
Only the `#fork` session-source path released early.

## Root cause

A `#fork:<session>` source resolves in `WaitDependencyForkQueries.fork_source_status`
(`_index_fork_queries.py`). It calls `agent_session_candidate_for_root(root)` and
releases on `(session.is_resolved and session.is_done) or session.is_failed`.

1. **A slot-queued successor is hidden.** `agent_session_candidate_for_root`
   (`_index_queries.py`) calls `_aggregate_candidates` with the default
   `exclude_queued=True`. `ArtifactCandidate.is_queued` is currently "`waiting.json`
   exists, no `done.json`, no archived completion" (`_index.py::_add_prepared`). The
   queued-member exclusion was added on 2026-07-06 (`5ca4379b7`, "unblock queued family
   waits") so that session members parked on their _dependency_ barrier cannot deadlock
   each other through their own session's aggregate. For example, sibling attach
   children waiting on the root's identity would block each other. Runner-slot admission
   arrived on 2026-07-12 (`28f563f3f`) and reuses the same `waiting.json` file as its
   queue marker. That silently widened "queued" to include members that have already
   crossed their dependency barrier and are only waiting for capacity. Such a member
   cannot take part in a wait cycle, but it is a live session member that has not run
   yet. The name path (`_agent_session_entity`) already includes all queued members
   (`exclude_queued=False`), which is why the plain `%wait:0rv.w0` stayed blocked.
2. **A handed-off failure still counts as a session failure.** With `0rv.w0--1` hidden,
   the failed monitor's declared follow-up is "missing". The monitor therefore stays in
   the effective generation, and `is_failed = any(member.is_failed)` becomes true. The
   `or session.is_failed` branch then releases the fork waiter. The same false release
   happens during the shorter window where the successor is still parked on its own
   dependency barrier: the repro below shows this. The documented contract in
   `docs/xprompt.md` is that a fork's implied wait "still holds for any other live
   session member".

Reproduced on current `master` with the helpers in
`tests/_monitor_wait_dependency_helpers.py`. The session is root done, failed monitor
with a launched follow-up, and an external fork waiter:

| Successor state               | fork-source wait   | plain name wait |
| ----------------------------- | ------------------ | --------------- |
| running (no marker)           | waits (correct)    | waits           |
| slot-queued (crossed barrier) | **releases (bug)** | waits           |
| dependency-parked             | **releases (bug)** | waits           |

The fix below was prototyped in a throwaway copy. The repro then waits in all three
rows, and 1439 existing tests across the wait, fork, hold, runner-slot, session, clan,
tribe, and dismiss suites pass unchanged.

## Design

The change is surgical: only the root-session aggregate that `#fork:<session>` and
identity (`wait_for_artifacts`) waits use changes behavior. The name, clan, and workflow
aggregates, the release confirmation, and hold settlement keep their current semantics.

- **Split "queued" into its two meanings.** Keep `ArtifactCandidate.is_queued` as-is,
  meaning any pre-run `waiting.json`. Add `is_dependency_parked`: queued **and** with no
  evidence that the dependency barrier was crossed. Only dependency-parked members need
  exclusion for deadlock avoidance.
- **Include slot-queued members in the root-session aggregate** used by wait resolution.
  Dependency-parked members stay excluded, which preserves the attach-chain deadlock
  protection.
- **Report a root session as failed only once it has settled.** Every declared shell
  follow-up must be present, and no effective member may still be pending. A failed
  member that handed off, or that has a live member after it, no longer releases a fork
  waiter. A fully terminal failed session still releases a fork waiter as today, because
  forking a failed session is supported.
- **Hold settlement keeps today's semantics.** `agent_session_settled` opts out of
  slot-queued inclusion explicitly. A hold armer's own later session members can be
  parked behind that very hold at runner admission. Counting them would pin the hold
  until its TTL.

## Implementation

All paths are relative to the sase repo root. No `sase-core` or Rust changes are needed:
wait-dependency resolution has no `sase_core_rs` binding.

### 1. `src/sase/core/wait_dependency_resolution/_types.py`

Add `is_dependency_parked: bool = False` to `ArtifactCandidate`, directly after
`is_queued`. Add a short comment on the two fields. `is_queued` means the artifact has a
pre-run `waiting.json` marker (a dependency wait or a runner-slot queue).
`is_dependency_parked` is the subset still behind its own dependency barrier.

### 2. `src/sase/core/wait_dependency_resolution/_artifact_state.py`

Add a public helper:

```python
def waiting_marker_crossed_dependency_barrier(
    artifact_dir: Path,
    meta: Mapping[str, Any],
) -> bool:
    """Return whether a pre-run ``waiting.json`` is a runner-slot queue marker."""
```

- Return `True` when `meta["wait_completed_at"]` is a non-empty string.
  `record_wait_completed_at` stamps it durably before the runner removes its dependency
  marker and before runner-slot admission publishes a queue marker. The key is present
  in both the on-disk meta dict and `asdict(AgentMetaWire)` rows from the artifact
  index, so both index-build paths see it.
- Otherwise read `artifact_dir / "waiting.json"` with the module's `read_json_dict`.
  Return `True` only if the marker has a non-empty string `slot_requested_at`. That
  covers the runner-slot marker of an agent that never had a dependency wait, so no
  `wait_completed_at` stamp exists.
- Otherwise return `False`. An unreadable, missing, or malformed marker fails closed to
  the historical "parked" classification.

The docstring should explain why both signals exist. Do **not** infer "not parked" from
the absence of declared `wait_for*` meta keys. Existing tests
(`tests/test_axe_chop_wait_checks_plan_agent_sessions_identity.py`) write
dependency-shaped markers without declaring waits in meta, and treating those as live
would reintroduce the sibling deadlock.

### 3. `src/sase/core/wait_dependency_resolution/_index.py` (`_add_prepared`)

Directly after the existing `is_queued` computation, add:

```python
is_dependency_parked = (
    is_queued
    and not waiting_marker_crossed_dependency_barrier(artifact_dir, meta)
)
```

Pass `is_dependency_parked=is_dependency_parked` to the `ArtifactCandidate(...)`
constructor and import the helper from `._artifact_state`. The marker is only read for
queued artifacts that lack the stamp, so the index-build cost stays proportional to
waiting markers.

### 4. `src/sase/core/wait_dependency_resolution/_index_entities.py` (`_aggregate_candidates`)

Add a keyword `exclude_slot_queued: bool = True`. Replace the queued filter with "drop
the candidate when
`exclude_queued and candidate.is_queued and (exclude_slot_queued or candidate.is_dependency_parked)`".
The defaults keep every existing call site's behavior identical: clan and session-name
entities pass `exclude_queued=False`, and workflow entities use the defaults. Update the
matching `_aggregate_candidates` declaration in the `_ForkQueryIndex` protocol in
`_index_fork_queries.py`.

### 5. `src/sase/core/wait_dependency_resolution/_index_queries.py` (`agent_session_candidate_for_root`)

- Add a keyword `exclude_slot_queued: bool = False` and forward it to
  `_aggregate_candidates`. Wait resolution is the default consumer and gets the correct
  behavior by default.
- After computing `effective_generation` and `handoffs_present`, compute
  `settled = handoffs_present and not any(_member_is_pending(c) for c in effective_generation)`.
  Report `is_failed=settled and any(c.is_failed for c in effective_generation)`. Leave
  `is_resolved`, `is_done`, and `is_identity_success` unchanged.
- Add a module-level helper:

  ```python
  def _member_is_pending(candidate: ArtifactCandidate) -> bool:
      """Return whether a session member may still run or change its outcome."""
      return not (
          candidate.has_done_marker
          or candidate.archived_completion is not None
          or candidate.is_resolved
      )
  ```

  `is_resolved` covers plan-chain handoff members that finish without a `done.json`, for
  example a submitted planner root with a completed workflow state.

- Update the docstring or comments to state both rules. Slot-queued members are live;
  dependency-parked members are excluded to avoid wait cycles. A session is failed only
  once it has settled.
- Update the `agent_session_candidate_for_root` declarations in the protocols in
  `_index_fork_queries.py` (`_ForkQueryIndex`) and `_index_identity_queries.py`
  (`_IdentityQueryIndex`) to include `exclude_slot_queued: bool = False`.

`fork_source_status` and `identity_status` need no logic change. Their existing
predicates become correct once the candidate reports live members and settled failures
faithfully.

Semantics check for `identity_status`: previously an unsettled failed session took the
name fallback (`is_resolved(name, newer_than=session.timestamp)`), which kept waiting.
Now it reaches `is_resolved and is_identity_success`, which is false while a member is
pending, so it also waits. A settled failure still takes the name fallback. The existing
test `test_identity_wait_failed_plan_agent_session_generation_keeps_waiting` must keep
passing.

### 6. `src/sase/core/agent_hold_liveness.py` (`agent_session_settled`)

Call `index.agent_session_candidate_for_root(root, exclude_slot_queued=True)`. Add a
one-line comment: hold settlement intentionally ignores runner-slot-queued members,
because a hold armer's own later members may be parked behind that very hold, and this
preserves pre-fix hold semantics. `agent_session_settled` reads only `is_resolved`, so
the settled-failure change does not affect it.

### 7. Tests

Put them in a new file, `tests/test_wait_dependency_slot_queued_session_members.py`, and
keep existing large files under the `toobig` limits. Reuse
`tests/_monitor_wait_dependency_helpers.py`: `_monitor_handoff_agent_session`,
`_agent_session_fork_source`, `_identity_dep`, and `_update_meta`. Reuse
`tests._agent_names_fixtures.make_agent` and, for the chop test,
`tests/_axe_chop_wait_checks_helpers.py` (`make_waiting_agent`, `run_wait_checks`).
Build indexes with
`build_wait_dependency_index("proj", projects_root=tmp_path / ".sase/projects")` and
evaluate with
`dependency_resolution_status(..., self_artifact_dir=<external waiter dir>)`.

Fixture shapes:

- **Slot-queued successor with a stamp:** successor meta gets
  `wait_for=["monitor-lane--code"]` and `wait_completed_at="2026-08-13T09:01:30+00:00"`.
  Its `waiting.json` is `{"slot_requested_at": "...", "waiting_for": []}` with no
  `done.json`.
- **Slot-queued successor without a stamp** (no-dependency agent): no
  `wait_completed_at`; `waiting.json` has a `slot_requested_at`.
- **Dependency-parked successor:** meta `wait_for=[...]`, no stamp; `waiting.json` is
  `{"waiting_for": ["monitor-lane--code"]}`.

Required cases:

1. **Incident regression.** Use a failed monitor (`monitor_state="failed"`, follow-up
   launched) and a slot-queued successor with a stamp. None of the three waits resolves:
   the external fork-source wait, the identity wait on the root, and the plain name wait
   on `"monitor-lane"`. Also assert that the successor candidate has
   `is_queued and not is_dependency_parked`. Then write the successor's `done.json`
   (`outcome: "completed"`), delete its `waiting.json`, rebuild the index, and assert
   that the fork-source wait resolves.
2. **Slot-queued successor without a stamp** blocks the fork-source wait.
3. **Dependency-parked successor** blocks the fork-source wait (settled-failure rule).
   Assert that the candidate has `is_dependency_parked`.
4. **A settled failed session still releases a fork waiter.** Use a failed monitor with
   no follow-up (`followup_outcome=None`, `followup_agent=None`, no successor) and every
   other member done. The fork-source wait resolves, preserving terminal-aware fork
   semantics.
5. **A failed member followed by a live member blocks.** Use a failed `--code` member
   and a later running member of the same generation with no `done.json` and no marker.
   The fork-source wait stays unresolved.
6. **Attach-chain deadlock protection is preserved.** A root is done, and two
   dependency-parked siblings each wait on the root's identity. Each sibling's identity
   wait resolves with `self_artifact_dir` set to that sibling. This is an index-level
   twin of the existing chop test
   `test_queued_agent_session_siblings_do_not_mutually_block_parent_dependency`.
7. **Hold settlement is unchanged.** A root is done, and an attached member of the same
   session is slot-queued. `agent_session_settled(root_dir, "proj", cache)` returns
   `True`, using `AgentSessionIndexCache(records=(), allow_scans=True)` with the
   projects root pointed at `tmp_path`. Monkeypatch
   `sase.core.agent_hold_liveness.sase_projects_dir` to return
   `tmp_path / ".sase/projects"`. In contrast,
   `index.agent_session_candidate_for_root(root).is_resolved` is `False`.
8. **Chop-level regression.** Build the incident fixture plus an external waiter
   (`make_waiting_agent(tmp_path, "monitor-lane", wait_for_fork_sources=[...])`). Run
   `run_wait_checks(tmp_path, monkeypatch)` and assert that `ready.json` is **not**
   written. Follow the patterns in
   `tests/test_axe_chop_wait_checks_plan_agent_sessions_handoffs.py`.

Every existing test must keep passing unchanged, in particular:

- `tests/test_monitor_wait_dependency_handoff.py`
- `tests/test_gate_wait_dependency*.py`
- `tests/test_axe_chop_wait_checks*.py`
- `tests/test_wait_dependency_release_confirmation.py`
- `tests/test_run_agent_wait_*.py`
- `tests/test_run_agent_runner_slot*.py`
- `tests/test_agent_hold_*.py`
- `tests/test_clan_wait_dependency.py`
- `tests/test_tribe_wait_dependency.py`
- `tests/test_axe_run_agent_failed_fork_admission.py`

### 8. Docs

In `docs/agent_sessions.md`, after the paragraph starting "A monitor or gate shell that
ended unsuccessfully ...", add two or three sentences:

- A session member that has crossed its own dependency waits and is only queued for a
  runner slot is a live member, so bare-session `%wait` and `#fork` targets stay blocked
  until it finishes.
- Members still parked on their own dependency waits are left out of the session
  aggregate so that sibling waits cannot deadlock.
- A failed shell that handed off to a follow-up does not release a `#fork` wait while
  that follow-up is pending.

Keep the existing prose style and wrap width, and run `just fmt`. Do not edit
`CHANGELOG.md`, which release-please manages.

## Verification

1. Run the new test file plus the existing suites listed above with targeted `pytest`.
2. Run `sase tool run check`, the agent default. Follow the repo's lint and test rules:
   do not run `check-full` unless explicitly asked, and route long runs through a
   monitor if needed.
3. Optionally sanity-check against live state with a read-only Python snippet. Build
   `build_wait_dependency_index("gh_sase-org__sase")`. Look up the root artifact for
   session `0rv.w0` (its `--plan` member) and confirm that
   `agent_session_candidate_for_root(root)` reports `is_resolved=False` while any
   `0rv.w0` member is still live or queued. This is optional and depends on live state.

## Non-goals and risks

- The `workflow_candidate` aggregate and the release-confirmation membership in
  `dependency_member_dirs` are unchanged. Workflow aggregates intentionally keep the
  queued exclusion. Changing them is out of scope because no incident depends on it.
- Holds that fence their own session's follow-ups are a separate, pre-existing design
  question. This change preserves hold behavior exactly.
- Risk: a fork waiter on a session whose failed shell declared a follow-up that never
  materializes, for example because it was deleted, now keeps waiting instead of
  releasing. This matches the existing name-path semantics
  (`test_failed_monitor_handoff_waits_for_missing_successor`) and the documented
  "implied wait holds for any other live session member" contract.
- Risk: a slot-queued member now keeps fork and identity waiters on its session parked
  until it runs. This cannot deadlock through waits, because the member has already
  crossed its dependency barrier and waiters hold no runner slot. The only possible
  cycle is an explicit hold whose pre-run armer is itself waiting on that session. That
  is the existing, notified hold-deadlock case (`_notify_hold_deadlock`), and the hold
  TTL still guarantees forward progress.
