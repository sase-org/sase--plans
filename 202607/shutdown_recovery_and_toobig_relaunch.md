---
tier: tale
title: Recover interrupted agent families and relaunch toobig_split
goal: Deterministic epic retries self-clean interrupted family handoffs, derived shutdown
  state is reconciled safely, and toobig_split relaunches with working repository
  authentication.
create_time: 2026-07-22 14:37:32
status: done
---

- **PROMPT:** [202607/prompts/shutdown_recovery_and_toobig_relaunch.md](prompts/shutdown_recovery_and_toobig_relaunch.md)

# Plan: Recover interrupted agent families and relaunch `toobig_split`

## Summary

Harden deterministic epic relaunches against the exact state left by an interrupted plan-to-code handoff, reconcile the
safe derived indexes that drifted around the shutdown, and relaunch the valid `toobig_split[sase]` proposals after
confirming Axe can authenticate to the repositories needed during workspace preparation.

The incident has two distinct causes which should not be conflated:

- `sase-8k.3--code` died without `done.json`, but its artifact-backed family reservation survived. The existing
  `stale_running_cleanup` correctly released dead workspace claims, while `sase bead work` deliberately refused to wipe
  the family container. Recovery therefore required manually wiping the family members before retrying the epic.
- `toobig_split` generated eight valid, deduplicated, sequential proposals. Its first agent failed while fetching the
  plans sidecar because SSH public-key authentication was temporarily unavailable after restart; the seven downstream
  agents correctly remained queued on their failed predecessor until the clan was dismissed. The plugin source and
  machine configuration are valid, and its once-per keys were released after the failed action.

The deep audit also found repairable derived artifact-index drift. The four malformed JSON markers and three pinned
workspace claims with missing registry rows predate this shutdown and must be reported but preserved; they are not safe
to delete under this incident cleanup.

## Implementation

1. Make forced reuse of deterministic bead-work phase names family-aware.
   - Extend `src/sase/bead/cli_work_cleanup.py` so an expected phase name that resolves to a family container is handled
     by discovering and wiping every concrete member owned by that family, then rebuilding/rechecking the registry and
     requiring the expected container name to be gone before bead state is mutated or any replacement agent launches.
   - Preserve the current safety boundaries: an expected clan container remains an error; the bare legacy epic clan may
     still be skipped only through `extra_cleanup_names`; wipe errors, an empty/unresolvable family, or any residual
     member/container reservation fail closed before launch.
   - Keep the existing confirmed-relaunch semantics for `sase bead work -y`: concrete prior owners may be terminated and
     removed, but unrelated names, clans, families, artifacts, notifications, and workspace claims must not be touched.

2. Make name-reuse relationship discovery understand the current sharded artifact layout.
   - Replace the legacy one-directory-deep traversal in `src/sase/agent/names/_wipe.py` with the shared artifact-path
     iterator from `sase.core.agent_artifact_paths`, covering both legacy and `YYYYMM/DD/timestamp` layouts across all
     projects and workflow directories.
   - Continue deriving the transitive wipe set from names, timestamps, parent/retry references, and outgoing handoff
     suffixes so plan/code family members and their dismissed child bundles are removed together.
   - Preserve best-effort handling of unreadable unrelated artifacts and retain the structured `AgentNameWipeResult`
     accounting used by bead-work cleanup.

3. Add regression coverage for shutdown-shaped family state.
   - In `tests/test_agent_name_wipe.py`, build related plan/code members in the day-sharded layout and prove that wiping
     a concrete family member finds the complete related artifact/bundle set, releases only its claims, and removes all
     derived reservations after registry rebuild.
   - In `tests/test_bead/test_cli_work_epic_launch_cleanup.py` and `tests/test_bead/test_cli_work_epic_relaunch.py`,
     model an expected phase reserved by a family whose completed plan member and dead, incomplete code member survived
     an interruption. Verify cleanup resolves the members, removes the family container, rewrites forced `%id`
     directives, and launches the retry only after cleanup succeeds.
   - Retain explicit tests that clan-container conflicts, member-wipe failures, and residual reservations abort without
     bead mutations or partial launches.

4. Reconcile only safe derived shutdown state and relaunch the failed chop.
   - After the code and tests pass, enter Axe maintenance mode for the short reconciliation window. Run the artifact
     index GC recommended by `sase doctor`, then verify the artifact and dismissed-bundle indexes again. Record any
     remaining warnings from the four pre-existing malformed source markers without deleting or rewriting their
     historical artifacts.
   - Recheck that all non-pinned RUNNING claims have live PIDs or have been released, that the current `sase-8k` agents
     remain healthy, and that the three older pinned failed-agent holds are unchanged.
   - Confirm the active Axe process has a live `SSH_AUTH_SOCK` with an available identity and that a repository access
     probe succeeds before launching. If authentication is unavailable, stop without creating another failed proposal
     clan and report the credential prerequisite.
   - With authentication healthy, run `sase axe chop run 'toobig_split[sase]' -L run_every --chop-verbose`, exit
     maintenance mode, and verify a new `toobig-*` clan is linked to a new `launched` chop run with its head running and
     its remaining sequential members waiting on the expected predecessors. Do not edit `bugyi-chops` or the chezmoi Axe
     configuration unless this verification contradicts the already-confirmed valid proposal output.

## Validation

- Run `just install` before repository checks, as required for an ephemeral workspace.
- Run focused tests for agent-name wiping and epic relaunch cleanup.
- Run `just check` for the complete SASE validation suite.
- Re-run `sase doctor` checks for the agent index, workspace registry/missing checkouts, and Axe health; distinguish the
  repaired derived-index drift from the explicitly preserved pre-incident warnings.
- Confirm `sase agent list -j` and the new chop run history show no stale family reservation, no newly orphaned
  workspace claim, and the expected live/waiting `toobig-*` chain.

## Acceptance Criteria

- Re-running a ready epic can safely replace an interrupted phase family without manual member-by-member cleanup.
- Forced reuse traverses both legacy and day-sharded artifact layouts and remains fail-closed for ambiguous containers
  or partial cleanup.
- The repair does not remove unrelated permanent names, historical artifacts, pinned failed-agent holds, or active
  agents.
- Safe artifact-index drift is reconciled and revalidated; unrelated corrupt historical markers are preserved and
  clearly reported.
- `toobig_split[sase]` launches a fresh sequential clan after repository authentication is verified, with no workspace
  preparation/authentication failure in the new run.
