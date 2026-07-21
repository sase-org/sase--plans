---
tier: tale
title: Post-preparation clan summary re-resolution
goal: Declaring clan members refresh script-backed summaries after their workspace
  and sidecars are ready, preserving the newest successful result across runner re-execs
  while keeping failures non-fatal and diagnosable.
bead: sase-8i.3
parent: sase/repos/plans/202607/race_free_epic_clan_summaries.md
create_time: 2026-07-21 11:42:01
status: wip
prompt: '[202607/prompts/post_preparation_clan_summary_reresolution.md](prompts/post_preparation_clan_summary_reresolution.md)'
---

# Plan: Post-preparation clan summary re-resolution

## Context

Clan summary scripts currently run while directives are extracted, before dependency waits, runner admission, workspace
preparation, and sidecar or linked-repository materialization. That early attempt is useful for immediate display, but a
newly created epic can persist its identity fallback because its plan and even its summary executable may not be visible
in the claimed workspace yet. The preceding epic phases already added durable attempt-labeled diagnostics and an
epic-plan snapshot; this change completes the timing fix without changing `%clan` syntax, the summary renderer, the scan
wire, or the ACE panel.

The post-preparation attempt must work on refreshed runner passes, where the epic-work environment was deliberately
consumed during directive extraction. It must also avoid losing metadata written during waits or re-exec and must never
replace an earlier successful summary with a later failure.

## Carry summary re-resolution intent

Add a small immutable carrier for the declaring member's script-backed summary inputs: the script value, clan name, clan
generation, and optional clan tribe. Expose it as an optional field on `AgentInfo`, and populate it only when the member
declares a clan with `summary_script=`. Literal summaries, clan joiners, and agents outside clans do not request a
post-preparation run.

Keep the extraction-time execution unchanged for immediate summaries and label its diagnostics with the existing
directive-extraction attempt name. The carrier records the stable directive and resolved membership inputs, not mutable
summary output or a copy of the launch environment.

## Reconstruct the launch environment without mapping drift

In `run_agent_directive_metadata`, define one authoritative mapping between epic-work environment names and persisted
metadata names. Reuse it both when consuming launch variables and when reconstructing the post-preparation overlay.
Cover `SASE_EPIC_PLAN_REF`, `SASE_EPIC_PLAN_SNAPSHOT`, `SASE_EPIC_BEAD_ID`, `SASE_PHASE_BEAD_ID`, and
`SASE_EPIC_CLAN_TRIBE`.

The reconstruction helper returns only non-empty string values represented in agent metadata. The runner builds the
summary subprocess environment from its current `os.environ` plus this overlay. That preserves environment changes made
by workspace and linked-repository preparation while restoring consumed epic variables on both the initial runner pass
and a refreshed/re-exec pass.

## Refresh and persist after workspace preparation

In `run_agent_runner`, perform the optional second resolution only after primary workspace preparation and every
applicable sidecar and linked-repository preparation branch has completed, and before bead claiming or SDD base-SHA
capture. Run the script with the prepared workspace as its working directory, the reconstructed environment, the same
20-second non-fatal resolver contract, and a distinct `post-workspace-preparation` attempt label.

Use last-success-wins semantics. If the second attempt returns no non-empty summary, leave both the extraction-time
in-memory value and the on-disk metadata untouched. If it succeeds, read the current `agent_meta.json`, merge the
refreshed summary without discarding wait/re-exec fields, update the runner's in-memory `agent_meta` with that merged
record, and write it through the standard metadata writer. Co-locate this read-merge-write helper with the existing
marker mutation helpers so the persistence pattern is independently testable. Updating both copies ensures later writes
such as `sdd_base_sha` cannot resurrect the stale extraction result.

## Contract documentation

Update `docs/agent_families.md` and the concise `%clan` reference in `docs/xprompt.md` to state that a script-backed
summary may execute during directive extraction and again after workspace preparation. Document that both attempts
receive the same clan and epic environment contract, share the timeout and non-fatal limits, and use the last successful
output. Require scripts to be read-only and idempotent because launch code can execute them more than once, including
during runner re-exec.

## Verification

Add focused coverage for:

- carrier creation only for a declaring `summary_script=` member;
- the shared consume/reconstruct mapping, including the plan snapshot and a runner pass whose only epic inputs come from
  preserved metadata;
- runner ordering after workspace, sidecar, and linked-repository preparation;
- a successful second attempt replacing the stale summary on disk and in memory, with a later metadata write retaining
  the replacement;
- a failed or empty second attempt preserving the extraction-time summary;
- attempt diagnostics distinguishing the failed early run from the post-preparation run;
- a race-shaped launch where the plan or executable is unavailable during extraction but appears during workspace
  preparation, after which the final persisted summary contains the epic title, goal fragment, and every phase title
  rather than the identity fallback;
- a control launch that can resolve immediately, proving the early display path still works.

Run the focused directive, metadata, runner, clan-summary, and epic-launch suites while iterating. Then run
`just install` followed by the mandatory `just check`. The shared PLAN renderer and ACE presentation remain unchanged,
so no PNG golden updates are expected or acceptable for this phase.

## Constraints and risks

Summary execution remains decorative and non-fatal. Do not add SASE-core or TUI changes, alter `%clan` parsing, or touch
memory files and generated instruction shims. The main risks are double-running a side-effectful user script and losing
concurrent metadata fields; the documented read-only/idempotent contract and the established read-merge-write pattern
address those risks.
