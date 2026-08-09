---
tier: tale
title: Preserve bead ownership across forced agent relaunches
goal:
  A confirmed forced relaunch can retain the overwritten agent's in-progress bead
  without weakening bead contention safeguards.
size: medium
proposed_by: bbugyi200.athena.w6
create_time: 2026-08-08 19:04:49
status: wip
---

# Honor bead ownership when force-relaunching an assigned agent

## Goal

Allow the ACE kill-and-edit relaunch produced for an assigned clan member to submit a
prompt such as `%id(!3, clan=sase-hq, bead=sase-hq.3)` while `sase-hq.3` is still
`in_progress` and assigned to the old `sase-hq.3` run. The explicit, TUI-confirmed `!`
replacement must authorize that exact successor to retain the bead association; it must
not weaken the normal protection against taking an in-progress bead from a different
agent.

## Current behavior and boundaries

- `prepare_kill_and_edit_prompt` in
  `src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py` deliberately
  preserves the full clan-member hood name and the phase `bead=` association, producing
  the prompt visible in the supplied screenshot.
- `run_agent_launch_body` in
  `src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py` parses and preflights
  the original force-reuse directive, wipes the prior name owner, and then calls
  `rewrite_force_reuse_name_directives`. The child therefore receives an ordinary
  `%id(3, clan=..., bead=...)` directive: the destructive action was confirmed, but the
  downstream bead-association check can no longer tell that the old assignee is the
  owner ACE just replaced.
- The runner claims an associated bead before a dependency wait in
  `src/sase/axe/run_agent_runner_bootstrap.py`, then promotes it immediately before
  execution through `src/sase/axe/run_agent_runner_launch.py`. The host adapters live in
  `src/sase/bead/claims.py` and `src/sase/axe/run_agent_runner_bead.py`.
- The Rust core already has the correct general mutation contract:
  `claim_for_agent_wait` quietly retains an `in_progress` bead for the same assignee,
  and `claim_for_agent_launch` can idempotently retain or reassign a non-closed bead. Do
  not broaden those shared mutations or add an unconditional "allow in progress" switch.
  The fix is to preserve and validate the trusted replacement context at the host launch
  boundary.
- Forced-reuse cleanup is destructive and must remain gated by the existing TUI
  confirmation/preflight path. Raw `%id(!...)` on non-TUI launch surfaces must continue
  to require confirmation, and ordinary `%id(..., bead=...)` launches must not acquire
  another live agent's bead.

## Implementation

1. Represent a confirmed force-reuse bead association before rewriting the prompt.
   - Extend the launch-directive inspection in `src/sase/agent/launch_validation.py`
     with a structured result for an explicit force-reuse owner and its optional `bead=`
     value. Reuse the existing `%id`/`%i`, clan, family, fenced block, disabled-region,
     and multi-segment parsing rules instead of adding a second parser.
   - Resolve `%id(!3, clan=sase-hq, bead=sase-hq.3)` to the exact owner `sase-hq.3`
     paired with bead `sase-hq.3`. Keep `force_reuse_owner_names` and force-directive
     rewriting behavior compatible for callers that only need names.
   - Reject ambiguous duplicate bead authorizations in a segment during the existing
     non-mutating preflight, before any name owner is wiped.

2. Carry the confirmed replacement pair through the rewritten TUI launch without
   exposing a user-controlled bypass.
   - In `run_agent_launch_body`, derive the per-segment replacement pair from the
     original, preflighted force-reuse prompt before cleanup. Only create the trusted
     metadata after `wipe_names_for_forced_reuse` succeeds, then rewrite `!` as today so
     history and child prompts retain their established ordinary `%id` form.
   - Thread the metadata through the existing single-launch `extra_env` path and the
     multi-prompt `segment_extra_env` path. Scope each marker to one concrete
     child/segment; do not put a bundle-wide list in every child. Merge it with existing
     swarm/fanout environment rather than replacing those values.
   - Use a `SASE_AGENT_*` one-shot environment name so the existing child-environment
     scrub prevents accidental inheritance by nested launches. Consume the marker during
     runner bootstrap and never persist it as reusable user prompt syntax.

3. Exempt only the bead/assignee validation covered by that replacement pair.
   - After directive extraction and final agent-name resolution, require all three
     values to agree: the marker's prior owner equals the resolved successor name under
     current-owner normalization, the marker's bead equals the directive's resolved
     `bead_id`, and the bead's current assignee equals that prior owner under the same
     identity rules.
   - When those checks pass and the bead is `in_progress`, treat the association as the
     retained ownership of the explicitly replaced run and continue through wait
     handling and launch promotion without the already-in-progress validation error.
     Preserve the bead's `in_progress` status and assignee; do not emit a redundant
     mutation, commit, or push merely to prove ownership.
   - If the marker is absent, malformed, belongs to another name or bead, or the current
     bead assignee differs, retain the existing decline/error behavior. Missing and
     closed beads remain errors at launch promotion. A forced name with no `bead=`
     association receives no bead exemption.
   - Keep the existing shutdown semantics: a retained `in_progress` assignment is not
     downgraded or released as a pre-launch `claimed` bead if the replacement runner
     exits while waiting.

4. Add regression coverage around the real ACE-to-runner flow.
   - In `tests/test_agent_launch_validation.py`, cover extraction of the exact
     clan-member prompt, family-member forms, aliases, protected regions, and the
     absence of an authorization for non-forced or beadless directives.
   - In `tests/ace/tui/test_agent_launch_non_blocking.py` and the multi-prompt launch
     tests, assert that confirmed cleanup still happens before dispatch, the
     saved/spawned prompt has `!` removed, and only the matching child receives the
     one-shot owner/bead marker. Cover composition with existing `extra_env` and a mixed
     bundle containing forced and ordinary segments.
   - In the runner wait/claim tests (`tests/test_run_agent_runner_wait_queue.py` and
     focused claim-helper coverage), reproduce `%id(!3, clan=sase-hq, bead=sase-hq.3)`
     after TUI rewriting against an `in_progress` bead assigned to `sase-hq.3`. Assert
     no validation error, no redundant bead mutation/commit/push, successful wait
     admission, and later promotion/execution.
   - Add negative cases proving an ordinary launch, a marker for another bead, a marker
     for another owner, and an `in_progress` bead held by another agent are still
     rejected or declined without mutation. Retain the existing closed-bead failure
     test.

## Validation

1. Run `just install`, as required for an ephemeral SASE workspace and to build the
   linked Rust binding used by the tests.
2. Run focused tests for launch parsing, TUI force-reuse dispatch, multi-prompt
   environment scoping, runner wait admission, and bead claim behavior, including at
   least:

   ```bash
   .venv/bin/python -m pytest -q \
     tests/test_agent_launch_validation.py \
     tests/ace/tui/test_agent_launch_non_blocking.py \
     tests/test_run_agent_runner_wait_queue.py \
     tests/test_bead/test_claims.py \
     tests/test_run_agent_runner_bead.py
   ```

3. Run `just check`. If the scoped selector escalates or reports unusual selection, run
   `just check-full` as required by the repository instructions.

## Acceptance criteria

- Submitting the screenshot's `%id(!3, clan=sase-hq, bead=sase-hq.3)` relaunch succeeds
  when the old `sase-hq.3` agent is the bead's current assignee, including while the
  successor waits on upstream agents or beads.
- The successor retains `bead_id=sase-hq.3` in its agent metadata and reaches normal
  launch promotion/execution without changing the already-correct `in_progress`
  assignment or creating no-op bead commits.
- A forced replacement cannot use its marker to take a different bead or a bead assigned
  to a different agent, and non-forced launch surfaces retain all current contention and
  confirmation safeguards.
- Single, multi-prompt, fanout, repeat, clan-member, and family-member launch behavior
  remains compatible; trusted replacement metadata is scoped to one child and is not
  inherited by nested launches.
- Focused tests and the mandatory `just check` gate pass.

## Out of scope

- Changing the deterministic epic phase naming scheme, bead-work preclaiming, or
  `#bd/work_phase_bead` content.
- Relaxing closed-bead validation or globally allowing arbitrary reassignment during
  wait claims.
- Changing Rust core claim semantics, which already provide the required idempotent
  launch mutation behavior.
