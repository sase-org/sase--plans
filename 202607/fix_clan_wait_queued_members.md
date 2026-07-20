---
tier: tale
title: Count queued clan members in whole-clan wait resolution
goal: 'A %wait on a clan name (and on a tribe that resolves to a clan generation)
  only releases the waiter once every clan member has actually completed, including
  members still queued behind intra-clan dependencies, so fork follow-ups like sase-7z.f2
  no longer launch against an incomplete clan and fail.

  '
create_time: 2026-07-20 08:46:34
status: wip
prompt: 202607/prompts/fix_clan_wait_queued_members.md
---

# Plan: Count queued clan members in whole-clan wait resolution

## Incident and root cause

Agent `sase-7z.f2` (a `#fork:sase-7z` follow-up with an implied `%wait` on the `sase-7z` clan) was released at 08:30:53
on 2026-07-20 while the clan was only 6/9 done: members `.7`, `.8`, and `.land` were still queued behind intra-clan `%w`
dependencies. The fork workflow's parent resolution then correctly refused the incomplete clan (`AgentClan.is_complete`
requires every member to have a successful outcome), and the agent failed with
`RuntimeError: Invalid fork parents: ... No agent with chat history found for: sase-7z`.

Evidence from the run artifacts: the last _started_ member (`sase-7z.4`) finished at 08:27:59; at that moment `.7`,
`.8`, and `.land` each had a `waiting.json` marker and no `done.json`. The next `wait_checks` chop pass resolved the
wait and wrote `ready.json` (the runner log shows the plain "All dependencies satisfied" path, not the sase-80.4 runner
fallback).

Root cause: `WaitDependencyIndexQueries.clan_candidate` in `src/sase/core/wait_dependency_resolution/_index_queries.py`
aggregates the newest clan generation through `_aggregate_candidates`, whose `exclude_queued` parameter defaults to
`True`. A queued member (`waiting.json` present, no `done.json`) is therefore invisible to a whole-clan wait, so the
wait resolves as soon as every member that ever _started_ is done. The queued-member exclusion was introduced by commit
`5ca4379b7` ("fix: unblock queued family waits") to break a family/workflow queue deadlock, long before clans existed;
clans inherited the default when `clan_candidate` was added in `f3bc42caa`. Families were already flipped back to
`exclude_queued=False` in `01da41927` (sase-6n.4) because sequential promotion needs queued members to block completion
— clans were left on the stale default.

The same hole exists in `tribe_candidate` (same file): when a tribe entity is a clan generation, its members are
aggregated with the queued-excluding default, so a `%w:@<tribe>` waiter can treat a partially-queued clan as a complete
entity.

Both the `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`) and the sase-80.4 runner fallback
(`src/sase/axe/run_agent_wait.py`) share this resolution code, so fixing the index queries fixes every release path at
once. The resolution module is pure Python with no `sase_core_rs` involvement, so no Rust-core change is needed.

## Fix

In `src/sase/core/wait_dependency_resolution/_index_queries.py`:

1. **`clan_candidate`**: pass `exclude_queued=False` to `_aggregate_candidates` so queued members join the aggregate. A
   queued member has `is_resolved=False` / `is_done=False`, so the clan candidate stays unresolved until every member of
   the newest generation truly completes. This matches the fork side's strict `AgentClan.is_complete` semantics and the
   TUI's member accounting (which already displays queued members as unfinished, e.g. "6/9 done").

2. **`tribe_candidate`**: pass `exclude_queued=False` in its clan-generation member aggregation so a clan with queued
   members is not offered as a complete tribe entity.

Deliberately out of scope, with a code comment or plan-note rationale:

- **`workflow_candidate`** keeps the queued exclusion. It still guards the original `5ca4379b7` queue-deadlock scenario
  for workflow-name aggregates, and no incident implicates it. Do not change its behavior in this tale.
- Intra-clan member waits are unaffected by the fix: the epic launcher (`src/sase/bead/work.py`) spells member and land
  waits as explicit member names (`%w:<member>,...`), which resolve through `named` candidates that never apply the
  queued filter. Whole-clan waiters are non-members (fork follow-ups), so including queued members cannot create an
  intra-clan wait cycle, and self-exclusion via `exclude_artifact_dir` is preserved.

## Secondary diagnostic improvement

In `src/sase/scripts/agent_chat_from_name.py`, `_resolve_fork_source` raises
`No agent with chat history found for: <name>` when a clan exists but is not complete. That message sent this
investigation toward chat-history resolution instead of the wait barrier. Raise a distinct message for the
incomplete-clan branch, e.g. `Clan '<name>' is not complete: <k>/<n> members done` (derived from the clan members'
outcomes), while leaving the no-such-agent wording for genuinely unknown names. Update any tests asserting the old
wording.

## Testing

- `tests/test_clan_wait_dependency.py`: add a regression test where the newest generation has done members plus one
  member with `waiting.json` and no `done.json` — `is_resolved("<clan>")` must stay false; flip the queued member to
  done and assert it resolves. Mirror the incident shape (several done members, multiple queued members) in at least one
  case.
- `tests/test_tribe_wait_dependency.py`: add the analogous case — a clan generation containing a queued member must not
  produce a complete tribe candidate; once all members are done it must.
- `tests/test_axe_chop_wait_checks.py` (helpers in `tests/_axe_chop_wait_checks_helpers.py`): add a chop-level
  regression test reproducing the incident — a waiter whose `waiting_for` names the clan, six done members, and queued
  members with `waiting.json` — and assert no `ready.json` is written.
- Fork-message coverage for the incomplete-clan wording, wherever `agent_chat_from_name` fork-source errors are already
  tested.
- Run `just check` before finishing.

## Risks

- **Stuck waiters**: any workflow that (incorrectly) relies on the current lax behavior — a whole-clan waiter expected
  to run while members are still queued — will now wait longer. That is the documented "all clan members" contract, and
  the fork guard already made the lax release fail anyway.
- **Pathological self-waits**: a clan member waiting on its own bare clan name would now deadlock against other queued
  members instead of releasing early. No launcher generates this shape; the epic launcher uses explicit member names.
  Not worth special-casing in this tale.
- **Runner-slot parking**: `waiting.json` also marks root agents parked for a runner slot, so a slot-parked clan member
  now blocks whole-clan waiters. This is correct (the member has not run), and whole-clan waiters that are fork
  follow-ups are exempt from slot claims, so no slot deadlock is introduced.
