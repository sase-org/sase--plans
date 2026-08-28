---
tier: tale
title: Let
goal:
  A shell follow-up that forks its own agent family resolves its implied wait
  immediately instead of deadlocking on itself, and the injected history stops listing
  the forking agent as a not-shown member.
size: medium
proposed_by: bbugyi200.athena.0fn
create_time: 2026-08-28 14:03:34
status: wip
---

# Plan: Let `#fork` resume a family the forking agent belongs to

## Problem

A monitor follow-up agent deadlocked waiting on the family it was launched into.

Observed shape (project `gh_sase-org__sase`, family `0fl`):

| member      | role    | artifact timestamp | state                                      |
| ----------- | ------- | ------------------ | ------------------------------------------ |
| `0fl--plan` | root    | `20260828123052`   | completed                                  |
| `0fl--gate` | gate    | `20260828124404`   | answered, `gate_followup_agent: 0fl--code` |
| `0fl--code` | code    | `20260828124522`   | completed                                  |
| `0fl--mon`  | monitor | `20260828131124`   | `monitored`, `monitor_next_agent: 0fl--1`  |
| `0fl--1`    | code    | `20260828134433`   | **WAITING on `0fl`**                       |

`0fl--1` is the monitor's `--next` follow-up. `sase/monitor/followup_prompt.py` builds
its prompt through `fork_target_for_settled_starter()`
(`src/sase/shells/followup.py:265`), which forks the _family base_ — so the prompt
starts with `#fork:0fl` — and then spawns the follow-up as a member of that same family
via `spawn_shell_family_successor()`. The follow-up therefore forks a family it is
itself a member of.

`src/sase/axe/run_agent_directives.py:196-256` turns a top-level `#fork:<name>` into an
implied wait, and `sase/agent/fork_waits.py` binds it as a typed fork source. The
recorded dependency in `0fl--1`'s `agent_meta.json` was:

```json
{
  "kind": "family",
  "name": "0fl",
  "artifact_dir": ".../20260828123052",
  "timestamp": "20260828123052",
  "project_name": "gh_sase-org__sase"
}
```

Resolution never succeeds. `WaitDependencyForkQueries.fork_source_status()` routes a
`kind: family` source to `family_candidate_for_root()`
(`src/sase/core/wait_dependency_resolution/_index_queries.py:175-213`), which computes:

```python
effective_generation = self._family_members_after_shell_handoffs(tuple(generation))
handoffs_present = self._family_shell_handoffs_have_successors(tuple(generation))
...
is_resolved=(handoffs_present and all(c.is_resolved for c in effective_generation))
```

Both helpers live in
`src/sase/core/wait_dependency_resolution/_index_entities.py:184-204` and key off
`names_in_generation = {candidate.name for candidate in candidates}`. The generation is
built by `_aggregate_candidates(..., exclude_artifact_dir=...)`, which correctly drops
the waiter's own artifact — so `0fl--1` is **not** in `names_in_generation`. But
`0fl--mon` records `shell_followup_agent == "0fl--1"`, so
`_family_shell_handoffs_have_successors()` sees a handoff whose successor is missing and
returns `False`. `handoffs_present` is `False` forever, so the family is never
`is_resolved`, so `0fl--1` waits on itself.

Reproduced directly against the real artifacts:

```
fork_source_status(excl self): WaitDependencyStatus(state='waiting', blocked_on=())
family entity: WaitEntity(timestamp='20260828131124', is_resolved=False, is_done=True, ...)
family_candidate_for_root: FamilyCandidate(..., is_resolved=False, is_done=True, ...)
```

Every family member was `is_resolved=True` and `is_done=True`. The _only_ reason the
family was unresolved was the self-excluded handoff successor.

Nothing breaks this deadlock automatically. `wait_checks`
(`src/sase/scripts/sase_chop_wait_checks.py`) and the runner's own 60-second fallback
(`src/sase/axe/run_agent_wait_deps.py`) both call the same
`dependency_resolution_status()`, so they agree it is still waiting. The observed run
only started because the user hand-edited its directives in ACE
(`src/sase/ace/tui/actions/agents/_directive_persistence.py:310` writes `ready.json`);
`lumberjack-waits.log` has no "Dependencies satisfied" line for it.

This is not monitor-specific. Any shell follow-up that forks its own family hits it:

- monitor `--next` follow-ups (`fork_target_for_settled_starter` always returns the
  family base),
- the plan shell's feedback branch, which declares `"fork": "family"`
  (`src/sase/plan_shell/create.py:217`),
- a gate whose `next.fork` policy is `family` (`_fork_target()` in
  `src/sase/gate_shell/followup.py:320`),
- any hand-authored `#fork:<family>` in a `%family`-attached launch.

The plan shell's _coder_ branch declares `"fork": "none"` (`_coder_branch()`,
`src/sase/plan_shell/create.py:204`), which is why `0fl--code` carried no `wait_for` and
why this only surfaced on the monitor hop.

A second, cosmetic symptom: the injected history reported the forking agent against
itself. From `0fl--1`'s own main prompt:

```
- **Members shown:** 4 of 5 (sequential chain, oldest first)
- **Not shown:** `0fl--1` (running)
```

`resolve_agent_chat_sources()`
(`src/sase/scripts/_agent_chat_from_name_sources.py:194-215`) iterates
`find_agent_family(name).members`, which includes the current agent, and
`resolve_family_member_shell()` classifies it as
`ForkExcludedFamilyMember(status="running")`.

## Goal

`#fork` supports resuming an entity the forking agent belongs to:

1. A family member's implied `#fork:<own family>` wait resolves as soon as every _other_
   member is terminal, instead of blocking on its own absence.
2. An external waiter's semantics are unchanged: it still blocks while any family member
   — including a freshly launched shell follow-up — is live.
3. The injected fork history does not list the forking agent as a not-shown member of
   its own source.

## Approach

Fix this at the wait-resolution layer, matching the precedent already established for a
terminally failed `#fork` parent: `fork_source_status()` is _terminal-aware_ rather than
having the directive layer strip the implied wait
(`tests/test_axe_run_agent_failed_fork_admission.py` documents that shape). Two
consequences matter:

- No wire or schema change. `wait_for_fork_sources` entries already on disk keep their
  meaning, so an agent currently stuck in this deadlock unblocks on the next
  `wait_checks` tick or runner fallback without relaunching.
- The safety property `#fork` actually needs is preserved. If a _sibling_ member is
  still running, its transcript is not saved yet and the fork would silently omit it —
  so the waiter must still block. Only the waiter's own presence stops counting against
  it.

Do **not** strip the implied wait in `run_agent_directives.py`. That would also drop the
legitimate "a sibling is still live" wait.

## Implementation

### 1. Teach the family aggregate about the self-excluded waiter

`src/sase/core/wait_dependency_resolution/_index_entities.py`

Add a lookup for the artifact the caller excluded as itself:

```python
def _excluded_member_name(
    self,
    exclude_artifact_dir: str | Path | None,
) -> str | None:
    """Return the agent name of the artifact excluded as the waiter itself.

    A family member that waits on (or forks) its own family is removed from
    the aggregate by ``_aggregate_candidates``. Its name is still needed so a
    shell handoff that produced this very waiter is not read as a handoff
    whose successor never appeared.
    """
    if exclude_artifact_dir is None:
        return None
    for candidate in self.artifacts_by_dir.values():
        if same_artifact_dir(candidate.artifact_dir, exclude_artifact_dir):
            return candidate.name or None
    return None
```

`artifacts_by_dir` is already declared on `WaitDependencyEntityQueries`, and
`same_artifact_dir` is already imported there.

Change both handoff helpers to take the extra present name. Keep them static and keep
their current behavior when the argument is omitted:

```python
@staticmethod
def _family_members_after_shell_handoffs(
    candidates: tuple[ArtifactCandidate, ...],
    *,
    extra_present_names: frozenset[str] = frozenset(),
) -> tuple[ArtifactCandidate, ...]:
    present = {candidate.name for candidate in candidates} | extra_present_names
    return tuple(
        candidate
        for candidate in candidates
        if candidate.shell_followup_agent not in present
    )


@staticmethod
def _family_shell_handoffs_have_successors(
    candidates: tuple[ArtifactCandidate, ...],
    *,
    extra_present_names: frozenset[str] = frozenset(),
) -> bool:
    present = {candidate.name for candidate in candidates} | extra_present_names
    return all(
        candidate.shell_followup_agent is None
        or candidate.shell_followup_agent in present
        for candidate in candidates
    )
```

The `_PENDING_SHELL_FOLLOWUP_AGENT` sentinel (`"<pending-shell-followup>"`,
`src/sase/core/wait_dependency_resolution/_artifact_state.py:42`) is not a legal agent
name, so a gate whose follow-up has not launched yet still blocks. Do not weaken that.

Update both branches of `_family_entity()` (the rooted branch and the legacy
rootless-recovery branch) to compute
`extra_present_names = frozenset(filter(None, (self._excluded_member_name(exclude_artifact_dir),)))`
once and pass it to both helpers.

### 2. Apply the same treatment to the fork-source path

`src/sase/core/wait_dependency_resolution/_index_queries.py:175-213`

`family_candidate_for_root()` duplicates the handoff logic. Pass the same
`extra_present_names` derived from its own `exclude_artifact_dir`. Prefer factoring the
shared "generation → (effective_generation, handoffs_present)" computation into one
helper on `WaitDependencyEntityQueries` and calling it from both `_family_entity()` and
`family_candidate_for_root()`, so the two cannot drift again.

Both `%wait:<family>` (via `family_candidate` → `_family_entity`) and `#fork:<family>`
(via `fork_source_status` → `family_candidate_for_root`) then behave consistently.

### 3. Drop the forking agent from its own injected fork source

`src/sase/scripts/_agent_chat_from_name_sources.py`

In the `find_agent_family(name)` branch, skip the family member whose `artifacts_dir` is
the current agent's, using the existing idiom from `_agent_chat_from_name_tribe.py:41`
and `_agent_chat_from_name_common.py:48`:

```python
current_artifacts_dir = os.environ.get("SASE_ARTIFACTS_DIR")
current = (
    Path(current_artifacts_dir).expanduser().resolve(strict=False)
    if current_artifacts_dir
    else None
)
```

Filter members whose resolved `artifacts_dir` equals `current` _before_ calling
`resolve_family_member_shell()`, so the forking agent lands in neither `members` nor
`excluded`. Apply the identical filter in the `find_agent_clan(name)` branch above it,
so a clan member forking its own generation does not try to read its own unwritten
transcript.

When `SASE_ARTIFACTS_DIR` is unset (CLI use outside an agent), behavior is unchanged.
The existing "at least one shown member is required" guard still fires with the same
`No agent with chat history found for: <name>` error when self-exclusion empties the
source.

## Tests

Add to `tests/test_monitor_wait_dependency.py` (it already owns
`_monitor_handoff_family` and the handoff-successor cases at lines 248-449):

- `test_monitor_handoff_successor_does_not_wait_on_its_own_family` — build the `0fl`
  shape: root, gate with `gate_followup_agent`, code, settled monitor with
  `monitor_next_agent` pointing at a live successor, plus that successor with no
  `done.json`. Assert `dependency_resolution_status(...)` with the successor's own
  artifact dir as `self_artifact_dir` and the `kind: family` fork source is `resolved`.
- `test_external_waiter_still_blocks_on_a_live_handoff_successor` — same fixture, a
  third-party `self_artifact_dir`. Assert `waiting`. This is the regression guard for
  over-broadening.
- `test_family_member_waiting_on_own_family_blocks_on_a_live_sibling` — same fixture
  plus a second live member that is _not_ anyone's handoff successor. Assert the
  in-family waiter is still `waiting`.
- A case asserting a gate whose `next_action` is set but whose follow-up has not
  launched still blocks for an in-family waiter (`_PENDING_SHELL_FOLLOWUP_AGENT` is not
  satisfied by self-exclusion).

Add to `tests/test_fork_history.py` (or the `#fork` source tests it pairs with):

- Forking a family from a member of that family omits the forking agent from both
  `members` and `excluded` — assert the rendered header shows `Members shown: N of N`
  and no `Not shown:` row naming the current agent.
- The same with `SASE_ARTIFACTS_DIR` unset leaves current behavior intact.

Also confirm `tests/test_run_agent_wait_deps.py`, `tests/test_gate_wait_dependency.py`,
`tests/test_clan_wait_dependency.py`, and
`tests/test_axe_run_agent_failed_fork_admission.py` still pass — they cover the
neighbouring resolution paths.

## Verification

Manual check against the recorded artifacts confirmed the fix and its scoping. With the
self-excluded waiter counted as a present handoff successor:

```
WITH FIX  fork_source_status: WaitDependencyStatus(state='resolved', blocked_on=())
WITH FIX  is_resolved("0fl"): True
EXTERNAL  fork_source_status: WaitDependencyStatus(state='waiting', blocked_on=())
EXTERNAL  is_resolved("0fl"): False
```

The external waiter still blocks because `0fl--1` is live — exactly the property that
must not regress.

After the change the reported case takes the `initial_dependencies_resolved()` fast path
in `src/sase/axe/run_agent_wait.py:118-143`, printing
`Dependencies already satisfied, proceeding without waiting` and never writing
`waiting.json` — so ACE never shows a member waiting on its own family.

## Docs

- `docs/xprompt.md` — next to the existing paragraph explaining that an already-failed
  `#fork` parent skips its implied wait, state that a `#fork` target the new agent is
  itself a member of does not count the new agent against the target's completeness; the
  wait still holds for any other live member.
- `docs/agent_families.md` (around line 339, the `#fork:<family>` contribution
  paragraph) — state that the forking agent is not listed among its own source's
  members.
- `docs/monitors.md` (around line 255, "the starter's full prior conversation, via
  `#fork`") — note that the follow-up joins the family it forks and does not wait on
  itself.

## Out of scope

- **Not** stripping the implied wait in `src/sase/axe/run_agent_directives.py`. The
  implied wait remains, and remains correct for live siblings.
- **Clan self-membership waits.** `_fork_clan_status()` already excludes the waiter and
  resolves correctly whenever the generation has other members. The only gap is a
  single-member clan forking itself, where the empty generation returns `waiting` — but
  such a fork has no transcript to inject at all, so the right outcome is an error, not
  a wait. Track separately if it ever surfaces; the history-side self-exclusion in step
  3 is included here only because it is the same three lines as the family case.
- Any change to how `sase agent list` or ACE renders `waiting_for`.

## Follow-up

File a task bead through `/sase_new_task` if `just check` surfaces an unrelated failure.

Run `just check` before finishing. If it takes long, hand it to `/sase_monitor` with the
`TESTING` / `TESTED` status pair. Because this changes shared wait-dependency
resolution, run `just check-full` through `/sase_monitor` before landing.
