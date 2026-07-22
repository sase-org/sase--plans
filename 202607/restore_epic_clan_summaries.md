---
tier: tale
title: Restore epic clan summaries on every bead-work launch
goal: Ensure fresh and retried sase bead work launches persist an epic clan summary
  without weakening single-declaration clan semantics
create_time: 2026-07-22 12:21:44
status: wip
---

- **PROMPT:** [202607/prompts/restore_epic_clan_summaries.md](prompts/restore_epic_clan_summaries.md)

# Plan

## Confirmed behavior and root cause

The suspicion is correct for retried direct launches, although not for every fresh direct launch. A fresh
`sase bead work` launch currently renders one declaring member with `%clan(..., summary_script=sase_clan_summary_epic)`,
and the runner persists that script's output on the declaring member. The gap appears after a failed or partial launch:

- force-reuse cleanup can remove the original declaring member and its artifact metadata, including the only persisted
  `clan_summary`;
- the durable rootless clan container intentionally survives cleanup;
- the next direct `sase bead work` invocation sees the reserved clan and renders every worker as a joiner, so no prompt
  contains the declaration-only summary script;
- runner summary resolution is currently gated on `directives.clan_declared`, leaving the replacement members without a
  summary.

The current machine state demonstrates the failure. The first attempted prompts for `sase-8l` and `sase-8m` contained
`summary_script=sase_clan_summary_epic`; their later successful launches used only `%id(..., clan=...)`, and the
replacement declaring-generation metadata contains no `clan_summary`. A fresh `sase-8k` launch retained its declaring
member and does contain the rich epic summary.

## Implementation

1. Extend the internal epic-work launch environment contract in `src/sase/bead/work.py` with a narrowly scoped,
   host-only epic clan-summary-script field. Put it on exactly the first phase segment returned by
   `epic_work_segment_env`, alongside the existing epic bead, plan, snapshot, and tribe metadata. Do this for both fresh
   declarations and existing-clan joins so every launch batch nominates exactly one summary-producing member while later
   phase and land segments avoid redundant script execution.

2. Consume that transient field during runner directive setup in `src/sase/axe/run_agent_directive_metadata.py` /
   `src/sase/axe/run_agent_directives.py` before nested launches can inherit it. Preserve the existing public directive
   contract and single-declaration lifecycle: do not redeclare a surviving clan and do not add summary mutation syntax
   to `%id(..., clan=...)`. Instead, use an explicit `%clan` summary script as the first-choice source and the internal
   epic-work script as the fallback for the nominated joining member.

3. Generalize the existing script-backed summary path just enough to support that nominated joiner. Resolve and persist
   the early summary with the member's already-resolved clan name and generation, use the launch-provided epic tribe
   when the join directive has no declaration-local tribe, and return the same `ClanSummaryResolutionRequest` so the
   established post-workspace-preparation refresh replaces an early fallback with the fully materialized plan summary.
   Keep all current safety properties: script failures, empty output, timeouts, and truncation remain non-fatal and
   diagnostic; literal declaration summaries remain declaration-only; ordinary clan joiners remain unaffected.

4. Add regression coverage at both halves of the contract:
   - update epic-work environment/rendering and launch assertions to prove only the first segment receives the transient
     summary script, including direct bead-ID launches and existing-clan relaunches;
   - add runner persistence coverage for a join-only epic member carrying the transient script, asserting the script
     sees the correct clan identity/tribe, `clan_summary` is written, a post-preparation refresh request is retained,
     and the host-only field is consumed rather than persisted or inherited;
   - retain fresh-declaration assertions to prove the rendered `%clan(..., summary_script=...)` behavior and precedence
     are unchanged;
   - cover a script failure or absence through the existing non-blocking path if the generalized branch is not already
     exercised by the shared parameterized tests.

## Validation

Run focused tests for epic work rendering/launch/relaunch and runner clan-summary persistence first, including at least:

- `tests/test_bead/test_work_rendering.py`
- `tests/test_bead/test_cli_work_epic_launch.py`
- `tests/test_bead/test_cli_work_epic_relaunch.py`
- `tests/test_clan_summary_persistence.py`
- `tests/test_axe_run_agent_phases_tribes.py`
- `tests/test_run_agent_directive_metadata.py`

Then run `just install` as required for an ephemeral SASE workspace, followed by the repository-mandated `just check`.
Inspect the final diff and `git status` to confirm the change is limited to epic launch/runner glue and regression
tests, with no memory files, generated instruction shims, or unrelated user changes modified.

## Acceptance criteria

- A fresh epic launch still declares its clan exactly once and produces the existing rich plan-backed summary.
- A launch that joins a surviving epic clan nominates its first member to produce and persist the same summary even when
  the old declaring member was removed.
- Only one member per launch batch runs the epic summary script.
- The summary is refreshed after workspace preparation using the existing plan snapshot/ref metadata.
- Summary generation remains decorative and can never block agent launch.
- All focused tests and `just check` pass.
