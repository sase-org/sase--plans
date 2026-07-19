---
tier: tale
title: Container-safe forced-reuse cleanup for epic bead-work retries
goal: 'Retrying a partially-completed epic with `sase bead work` succeeds instead
  of failing with "forced reuse cleanup left agent name ''<epic>'' reserved after
  rebuild", and forced-reuse name wipes never destroy clan/family member artifacts
  when the target name is owned by a container reservation.

  '
create_time: 2026-07-18 20:37:22
status: wip
prompt: 202607/prompts/bead_work_clan_container_wipe.md
---

# Plan: Container-safe forced-reuse cleanup for epic bead-work retries

## Problem

`sase bead work <epic> -y` on an epic whose earlier phases already completed fails every time with:

```
Error: forced reuse cleanup left agent name '<epic>' reserved after rebuild; resolve the conflicting owner and retry
```

Worse, **every retry permanently deletes one completed phase agent's artifact directory and dismissed bundle** before
failing. This was observed live with epic `sase-6v`: each run of the command deleted the oldest surviving clan member's
artifacts (e.g. `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/18/20260718170002`, which belonged to
completed agent `sase-6v.3`) and then raised the identical error again.

## Root cause

The failure chain:

1. `handle_bead_work` (`src/sase/bead/cli_work_handler.py`, `force_reuse_cleanup` stage) calls
   `prepare_bead_work_force_reuse(query, expected_names=..., extra_cleanup_names=legacy_epic_cleanup_names(plan))`.
2. `legacy_epic_cleanup_names` (`src/sase/bead/cli_work_plan.py`) unconditionally returns the bare epic id (e.g.
   `sase-6v`) as an extra wipe target whenever `plan.land_agent_name != epic_id` — which is now always true, since land
   agents are named `<epic>.land`. The intent is to clean up a _legacy land agent_ that owned the bare name under the
   old naming scheme.
3. However, once any epic phase has completed, the bare epic name is reserved in the agent-name registry by the epic's
   own **clan container** entry (`container_kind: "clan"`), not by a legacy land agent. Clan container entries are
   _derived state_: `rebuild_name_registry` re-creates them from every surviving member artifact/bundle payload's
   `agent_clan` field (`_add_owner_clan` in `src/sase/agent/names/_registry_scan.py`).
4. `wipe_agent_name_for_reuse` (`src/sase/agent/names/_wipe.py`) is container-unaware. For the clan entry it:
   - seeds the container's anchor `artifacts_dir` — which is actually the **oldest surviving member's** artifact
     directory — into the wipe plan,
   - deletes that member's artifact directory and dismissed bundle (permanent data loss), then
   - rebuilds the registry, which re-derives the clan container from the remaining members.
5. Back in `_wipe_force_reuse_owner` (`src/sase/bead/cli_work_cleanup.py:170`), the name is still reserved after the
   rebuild (`found=True`, name not in `registry_names_removed`), so it raises `ForcedReuseCleanupError` — the exact
   error above.

So the wipe of a clan-container-owned name is structurally impossible: it can only "succeed" once it has destroyed every
member's artifacts and bundles. Each retry eats the next-oldest member and fails.

The wipe is also unnecessary: `reserve_registered_clan_name` (`src/sase/agent/names/_registry.py`) already handles an
existing clan container gracefully — new members launched with `%clan:<epic>` simply join the existing clan and reuse
its `clan_generation`. Retried phase agents do not need the bare epic name freed.

The same container-unaware wipe is reachable from explicit `%name:!<name>` launches via `wipe_names_for_forced_reuse`
(`src/sase/agent/launch_validation.py`): force-reusing a name owned by a clan or family container would destroy the
anchor member's artifacts and then fail at claim time with a container collision.

## Design

### 1. Make `wipe_agent_name_for_reuse` container-aware (`src/sase/agent/names/_wipe.py`)

- Add a field to `AgentNameWipeResult`: `skipped_container_kind: str | None = None`.
- In `wipe_agent_name_for_reuse`, immediately after resolving the owner entry (both the string-lookup and mapping-input
  forms), check `owner.get("container_kind")`. When it is a non-empty string ("clan" or "family"), return early with
  `AgentNameWipeResult(target_name=..., found=True, skipped_container_kind=<kind>)` — **before** building the wipe plan,
  terminating processes, or deleting anything. Container reservations are not agents; their anchor `artifacts_dir`
  belongs to a member, and the reservation is re-derived from members on every rebuild, so a container name can never be
  freed by wiping.
- Update the function docstring to document the container guard.

This single guard protects both production callers (`_wipe_force_reuse_owner` for bead work and
`wipe_names_for_forced_reuse` for explicit `%name:!` launches) from destroying member artifacts.

### 2. Interpret container skips in bead-work cleanup (`src/sase/bead/cli_work_cleanup.py`)

Distinguish the two wipe populations in `prepare_bead_work_force_reuse`:

- **Extra cleanup names** (the legacy bare epic id from `legacy_epic_cleanup_names`): a result with
  `skipped_container_kind` set is the _expected_ outcome for an epic retry — the bare name is the epic's own clan
  container, which the relaunch happily joins. Treat it as success and continue silently. A genuine legacy land agent (a
  plain agent owner) is still wiped exactly as today.
- **Expected `%name:!` names** (phase and land agent names): a container owner is an unresolvable conflict for this
  launch. Raise `ForcedReuseCleanupError` with an actionable message, e.g.
  `"agent name '<n>' is reserved by a <kind> container and cannot be force-reused; dismiss or clean up the container's members, then retry"`.
  This still aborts before any bead mutation (same guarantee as today) but with a truthful message and no data
  destruction. (Realistic trigger: a prior phase agent that was grown into an agent family reserves the bare phase name
  as a family container.)

Implement by threading a flag (e.g. `allow_container_skip: bool`) through `_wipe_force_reuse_owner`, or by splitting the
loop into two passes — implementer's choice. Keep the existing `found`/`registry_names_removed` residual check for
non-container owners unchanged. Update the `prepare_bead_work_force_reuse` docstring to describe the container
semantics.

### 3. Reject force-reuse of container names at validation (`src/sase/agent/launch_validation.py`)

Harden `validate_launch_name_requests`: in the `request.force_reuse` branch (currently an unconditional `continue` once
`allow_force_reuse` is set), look up the reserved clan/family name sets (reuse the existing lazy single-load pattern)
and raise `_AgentNameClanCollisionError` / `_AgentNameFamilyCollisionError` when the force-reused name is
container-owned. This gives explicit `%name:!<clan>` launches an immediate, clear error instead of a doomed launch that
dies later at claim time.

Note the bead-work path is unaffected: it rewrites `%name:!` directives to plain `%name:` before the launcher validates,
and the bare epic name only ever appears as `%clan:`, never as `%name:`.

### 4. Tests

- `tests/test_agent_name_wipe.py`:
  - Wiping a name whose registry entry is a clan container: returns `found=True` with
    `skipped_container_kind == "clan"`, and the member artifact directory, dismissed bundle, and registry entries all
    survive untouched.
  - Same for a family container (`skipped_container_kind == "family"`).
- `tests/test_bead/test_cli_work_epic_launch.py` (follow the existing fake-wipe/fake-launch patterns):
  - **Regression test for this bug**: epic retry where the fake wipe reports `skipped_container_kind="clan"` for the
    bare epic id → the launch proceeds, the rewritten prompt reaches the launcher, and no error is raised.
  - Expected phase/land name whose fake wipe reports a container skip → `ForcedReuseCleanupError` path: aborts with the
    new message before any bead mutation or launch (mirror the assertions in
    `test_work_force_reuse_cleanup_failure_aborts_before_mutation`).
  - The existing `test_work_stale_owner_round_trip_wipes_and_rewrites` must keep passing: the legacy bare name is still
    passed to the wipe (the guard lives inside the wipe/result interpretation, not in name selection).
- `validate_launch_name_requests` hardening: a force-reuse request naming a clan-owned and family-owned name raises the
  respective collision error (extend the existing launch-validation tests where they live).

## Non-goals

- No "wipe a whole container and all its members" capability. Force-reusing a container name stays an error; users
  resolve container conflicts by dismissing/cleaning members through existing flows.
- No recovery of already-deleted artifacts (there are no backups of agent artifact directories); the epic's code commits
  are unaffected and live in git.
- No Rust core changes: the agent-name registry and wipe logic are Python-only in this repo, and this is
  launcher-internal behavior, not shared frontend/backend domain logic.

## Verification

- `just install`, then targeted runs:
  `pytest tests/test_agent_name_wipe.py tests/test_bead/test_cli_work_epic_launch.py tests/test_bead/test_cli_work_collisions.py`,
  then full `just check`.
- Manual end-to-end (after the fix is deployed to the installed `sase` tool): `sase bead work sase-6v -y` should print
  the retry plan, skip the clan-container wipe silently, launch `sase-6v.8`/`sase-6v.9` (which join the existing
  `sase-6v` clan generation), and schedule `sase-6v.land`. Until then the command must not be retried with the installed
  CLI, because each attempt destroys the oldest surviving member's artifacts.

## Risks

- The container guard changes `wipe_agent_name_for_reuse` semantics for any future caller that expected containers to be
  wiped — none exist today; the new result field makes the skip observable.
- The validation hardening (change 3) turns a previously "allowed" (but doomed and destructive) explicit
  `%name:!<container>` launch into an upfront error; this is intentional and strictly safer.
