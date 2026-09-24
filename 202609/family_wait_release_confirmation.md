---
tier: tale
title: Confirm agent-family wait releases against a fresh membership view
goal:
  A %wait on an agent family (agent session) never releases while the family still has
  an active or newly handed-off shell, even when the wait resolver's membership view is
  older than the completion markers it reads.
size: medium
proposed_by: bbugyi200.athena.0r1
create_time: 2026-09-24 13:12:12
status: wip
---

# Plan: Confirm family-shaped wait releases against a fresh membership view

## Problem

A `%w:<name>` waiter can start while the agent family (agent session) it waits on is
still active. Observed on 2026-09-24: `sase-17d.10.1.2` waited on `%w:sase-17d.10.1.1`
plus `%w(bead=sase-17d.10.1.1)`. It was released at 12:39:20 and started at 12:45:38,
while the `sase-17d.10.1.1` family still had a follow-up coder (`sase-17d.10.1.1--2`)
queued and later running.

## Diagnosis (confirmed)

The suspicion is right: only waits whose target is an agent family are exposed. The
premature release happens at the moment a family member hands off to a new shell.

Timeline of the `sase-17d.10.1.1` family (all times local):

| Member          | Role / event                                                         |
| --------------- | -------------------------------------------------------------------- |
| `--plan` (root) | done `completed`                                                     |
| `--gate`        | plan gate, done `gated`                                              |
| `--code`        | done `completed` at 11:43:55; phase bead closed at 11:41:58          |
| `--mon`         | CI monitor failed; follow-up `--1` launched                          |
| `--1`           | coder. Started monitor `--mon-0` at 12:39:06-07                      |
|                 | `workflow_state.json` became `completed` at 12:39:12.7               |
|                 | `done.json` written at 12:39:45                                      |
| `--mon-0`       | artifact created 12:39:07; failed; follow-up `--2` launched 12:40:53 |
| `--2`           | queued for a runner slot 12:41-12:49, then running                   |

The `waits` lumberjack log shows the release:
`[2026-09-24 12:39:20] [wait_checks] Dependencies satisfied for sase, waited on: sase-17d.10.1.1; beads: sase-17d.10.1.1`.
The bead wait gave no protection because the phase bead was already closed at 11:41:58.
The waiter then sat in the runner-slot queue until 12:45:38.

Root cause: `wait_checks` (`src/sase/scripts/sase_chop_wait_checks.py`) reads family
**membership** from the persistent agent-artifact index with `freshness="cached"`
(`query_ace_run_index_records`). A snapshot takes seconds, and on the live host the
query alone took about 24s. It then reads **completion markers** (`done.json`,
`workflow_state.json`) live from disk in `WaitDependencyIndex.add_many`. So the markers
can be newer than the membership list.

Every family handoff (agent → monitor, agent → gate, planner → gate) creates the
successor shell _before_ the predecessor's terminal markers land. The predecessor
records no pointer to that successor. If the membership snapshot was taken before
`--mon-0` existed (or before its best-effort index upsert landed), and the markers were
read after `--1`'s `workflow_state.json` became `completed`, then the family entity
looked like this:

- `--plan`, `--gate`, `--code`: done
- `--mon`: excluded, because its follow-up `--1` is present
- `--1`: resolved, via the plan-chain rule in `artifact_is_resolved`
  (`_completed_handoff_workflow_state`)
- `--mon-0`: absent

With every visible member resolved, `_family_entity` reported resolved and `ready.json`
was written.

Reproduced against copies of the real artifacts:

- a consistent view of 12:39:13-12:39:44 (with `--mon-0` present, no `done.json` on
  `--1`) → `waiting`
- the same view with `--mon-0` missing from membership → `resolved`

The runner's own fallback (`run_agent_wait_deps.initial_dependencies_resolved`, run at
startup and every 60s) has the same problem in a narrower window.
`WaitDependencyIndex.build` lists directories and reads metas before `add_many` reads
the markers.

Sound fix: the happens-before chain is successor created → predecessor terminal →
resolver reads predecessor's markers. So any membership listing taken _after_ the
resolving marker reads is guaranteed to contain the successor. A release is safe iff:

1. a view resolves, and
2. a membership view read strictly afterwards shows no member of the waited entities
   that the resolving view lacked.

Considered alternative: have each predecessor record a successor pointer that the
resolver requires to be present. That approach needs new done-marker/meta wire fields
across the Rust index (sase-core + revision pin) and writer changes in monitor, gate,
and plan handoffs. It also still misses failed best-effort index upserts. Confirmation
needs no wire changes and covers all handoff kinds, including future shell kinds.

## Implementation

### 1. Membership query on the index

In `src/sase/core/wait_dependency_resolution/`, add a public method on
`WaitDependencyIndex`: `dependency_member_dirs(...) -> frozenset[str]`. It takes the
same dependency inputs as `dependency_resolution_status`:

- `wait_names`, `wait_identity_deps`, `resolved_deps`
- `wait_fork_sources`, `wait_hoods`, `self_artifact_dir`

It returns the artifact dirs, normalized with `artifact_dir_key`, of every candidate
that resolution could aggregate for those targets. Place it in the query mixins, for
example `_index_queries.py` or a small new `_index_membership.py` mixin, if
`_index_queries.py` would grow too large.

- **Name `N`** (skip names memoized in `resolved_deps`), all _pre-handoff-filter_
  candidates:
  - `families[N]`, `workflows[N]`, every generation of `clans[N]`
  - the `named[N]` dir and the planner-row alias dir
- **Tribe name (`@...`)**: the dirs of rows carrying that tribe, direct or via the
  effective clan tribe.
- **Identity / fork-source dependency**: the dependency's own artifact candidate plus
  every member of `families[candidate.family_name or candidate.name]` and of its clan
  generation. Identity and fork waits can resolve through `family_candidate_for_root`,
  `_fork_family_name_status`, and `_fork_clan_status`.
- **Hood**: `_hood_candidates(hood, launched_at_or_before=<waiter cutoff>)`.
- Always exclude the waiter's own artifact dir.

### 2. Confirmation helper

Add `src/sase/core/wait_dependency_resolution/_confirmation.py` and export it from
`__init__.py`:

```python
def confirm_dependency_resolution(
    resolved_index: WaitDependencyIndex,
    fresh_index: Callable[[], WaitDependencyIndex],
    wait_names, wait_identity_deps=(), resolved_deps=(), *,
    wait_fork_sources=(), wait_beads=(), wait_hoods=(),
    closed_bead_ids=None, self_artifact_dir=None, max_rounds: int = 2,
) -> WaitReleaseConfirmation
```

`WaitReleaseConfirmation` is a small frozen dataclass: `confirmed: bool`,
`status: WaitDependencyStatus` (the latest evaluated), and
`new_member_dirs: tuple[str, ...]` (for logging).

Semantics:

- If the waiter has no agent-shaped dependencies (no names, identity deps, fork sources,
  or hoods; for example beads or time only), return confirmed without building anything.
- Otherwise, set `prev = resolved_index` and repeat up to `max_rounds` times:
  - `fresh = fresh_index()`
  - `status = dependency_resolution_status(fresh, ...)`. If it is not resolved, return
    not confirmed with that status.
  - If `dependency_member_dirs(fresh) - dependency_member_dirs(prev)` is empty, return
    confirmed.
  - Otherwise set `prev = fresh`.
- After the rounds, return not confirmed. The next tick or poll retries.
- Deleted or dismissed members that disappear from the fresh view are not a change; only
  _new_ members defer.

Add a docstring stating the happens-before argument above, so the invariant is not
"simplified" away later.

### 3. `wait_checks` chop

In `src/sase/scripts/sase_chop_wait_checks.py`, when `status.resolved` for a waiter,
call `confirm_dependency_resolution` before writing `ready.json`. Only write it when
confirmed.

- The fresh factory builds from the filesystem, reusing
  `_filesystem_dependency_rows`-style meta reads plus `WaitDependencyIndex.add_many`.
- Scope it to the waiter's project plus the `project_name`s of the resolving index's
  member dirs for that waiter.
- Copy `global_stored_tribes` (and any tribe paths) from the pass-1 index for parity.
- Cache fresh indexes per (project set, round) within one tick, so several releases in
  one tick share one fresh build. All pass-1 marker reads happen in `add_many` before
  the waiter loop, so one shared fresh build still satisfies the ordering.
- Build lazily: ticks that release nothing must not pay for a filesystem build.
- On deferral:
  - log
    `[wait_checks] Deferred release for <cl_name>: dependency membership changed since the resolving view (new: <dirs>)`,
    or the blocking labels when the fresh view is unresolved
  - count it in a new summary field `deferred_unconfirmed`
  - do not treat it as an error
- Keep the `SASE_CHOP_SCAN_FULL_WALK` path correct. Confirmation still applies, because
  the full walk has the same list-then-read ordering.

### 4. Runner fallback

In `src/sase/axe/run_agent_wait_deps.py`, `initial_dependencies_resolved` should, after
its pass-1 status resolves, confirm with a factory that rebuilds
`build_wait_dependency_index(project_name)` with `global_stored_tribes` set the same
way. It returns `True` only when confirmed. This covers the startup fast path and the
60s `waiting_marker_dependencies_resolved` fallback.

Do **not** add re-validation of `ready.json` in `run_agent_wait.py`. The runner's index
is single-project, so re-validating would strand cross-project waits that only the chop
can resolve. With confirmation in the chop, `ready.json` stays trusted.

### 5. Docs

In `docs/axe.md`, in the `wait_checks` section, add a short paragraph:

- a release is confirmed against a fresh on-disk membership view taken after the
  resolving read
- a family that gained a member since then (for example a coder that just started a
  monitor or gate) stays parked until a later tick
- the runner's fallback applies the same confirmation
- mention the `deferred_unconfirmed` summary counter

## Tests

Put the core tests in a new `tests/test_wait_dependency_release_confirmation.py`,
reusing `tests/_agent_names_fixtures.make_agent` and the
`tests/_monitor_wait_dependency_helpers` helpers (`_monitor_member`,
`_write_completed_workflow_state`, `_update_meta`). Build the incident shape on a tmp
projects root with family `lane`:

- root `lane--plan`: done, completed
- `lane--code`: done, completed
- `lane--mon`: monitor done, failed, followup `lane--1` launched
- `lane--1`: plan-chain code member, completed `workflow_state.json`, no `done.json`
- `lane--mon-0`: running monitor member, parent `lane--1`, no `done.json`

Cases:

- A stale pass-1 index built from rows that omit `lane--mon-0` resolves `lane`. This
  documents the hazard. `confirm_dependency_resolution` with a filesystem factory
  returns not confirmed, and its status is blocked on `lane`.
- A fully completed family is confirmed in one round, with exactly one factory call.
- A factory whose first fresh view adds a new, resolved member and whose second view is
  stable is confirmed in round 2. A factory that keeps adding members is not confirmed
  after `max_rounds`.
- Bead-only and time-only waiters are confirmed with zero factory calls.
- Identity-dependency and hood variants of the stale-membership case are not confirmed.

Chop tests go next to `tests/test_axe_chop_wait_checks_plan_families_handoffs.py` and
use `tests/_axe_chop_wait_checks_helpers`.

- Monkeypatch the index path (`query_ace_run_index_records` /
  `wait_rows_from_index_records`) to drop the `lane--mon-0` row. Assert that no
  `ready.json` is written and that the summary reports `deferred_unconfirmed=1`.
- With complete rows, `ready.json` is still written. The existing chop suites must keep
  passing unchanged.

Runner tests go in `tests/test_run_agent_wait_deps.py`. Monkeypatch
`build_wait_dependency_index` to return a stale index first and the real index second.
`initial_dependencies_resolved` returns `False`. A consistent completed family still
returns `True`.

## Verification

Run `just check` through `sase tool run` (per the lint/test memory). Before the
confirmation fix, the new stale-membership tests must fail against the current code;
confirm this with a quick local red run. They must pass after the fix.

## Out of scope (note in the final report, do not implement)

- **Phase-bead close timing.** Phase beads close when the coder submits (11:41:58 here),
  before the family's CI monitor and follow-up coders finish. So a bead wait does not
  guard a family's tail.
- **Launch admission.** `launch_admission_runtime._resolve_agent_wait` resolves
  `find_named_agent` for multi-unit launch plans and has its own family semantics. It
  should be audited separately.
- **Index freshness.** The chop's cached index query is slow (about 24s observed), and
  marker-mutation index upserts are best-effort.
