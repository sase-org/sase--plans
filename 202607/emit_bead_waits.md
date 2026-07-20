---
tier: tale
title: Emit bead-gated waits from epic work plans
goal: Ensure every generated epic phase and land prompt waits for both agent completion
  and the corresponding phase bead closures, including retries and previews, while
  requiring the Rust core payload version that supplies the bead IDs.
create_time: 2026-07-20 12:07:15
status: done
prompt: 202607/prompts/emit_bead_waits.md
---

# Plan: Emit bead-gated waits from epic work plans

## Context and scope

`sase bead work` currently renders phase dependencies only as agent waits and renders the land segment against only the
agents launched in the current run. That is insufficient when a medium or large phase delegates to a child epic: the
phase agent can finish while its phase bead remains open until delegated work lands. The already-completed Rust phase
now supplies two additive fields needed to preserve the full work graph: each scheduled assignment's complete in-epic
`blocker_bead_ids`, including closed and delegation-excluded blockers, and the epic-wide `phase_bead_ids`, including
phases omitted from a retry wave.

This tale consumes that payload in the Python `sase bead work` path. It does not change Rust behavior, wait
parsing/resolution, waiting UI, documentation, or the parent epic lifecycle.

## Work-plan model and payload hydration

Extend the immutable Python work-plan types in `src/sase/bead/work.py` so `_PhaseAssignment` retains its blocker bead
IDs and `EpicWorkPlan` retains all phase bead IDs. Hydrate both fields in `_plan_from_payload` without translating bead
IDs into clan membership names; existing `waits_on` and `land_waits_on` remain agent-name fields and retain their
current normalization.

Update direct constructors and shared test helpers deliberately so tests distinguish agent dependencies from
bead-closure dependencies. Add payload-facing assertions that prove closed in-epic blockers and delegated phases remain
represented even when they are absent from the scheduled agent waves.

## Prompt emission and preview parity

In `render_multi_prompt`, preserve the current `%w:<agent names>` lines, then append one `%w(bead=<phase id>)` line per
assignment blocker in payload order. On the land segment, preserve the current launched-agent wait and append one bead
wait per epic-wide phase bead. Separate directives make the generated barrier explicit and preserve repeatable `bead=`
semantics.

Because dry runs, confirmation/approval, and actual launch all consume the same rendered query in
`src/sase/bead/cli_work_handler.py`, keep that single rendering path authoritative rather than creating a second preview
formatter. Assert the rendered multi-prompt contains identical bead waits for ordinary VCS and ChangeSpec launch
contexts, and that dry-run output matches the query handed to launch after confirmation apart from the existing
force-reuse rewrite.

## Rust binding compatibility floor

Advance the `sase-core-rs` dependency window in `pyproject.toml` from the current anticipated 0.10 line to the 0.11
release line carrying core commit `66360e2` and its new work-plan fields. Update the exact-minimum expectation in
`tests/test_sase_core_rs_telemetry_smoke_tool.py` using the established floor-bump pattern. Do not edit Rust crate
versions or core source, which remain release-plz owned, and do not hand-edit the lockfile while the anticipated release
is unavailable; local verification uses the linked core checkout through the existing development override.

## Verification

Add focused unit and CLI coverage under `tests/test_bead/` for:

- a later wave receiving both its blocker agent wait and one bead wait per blocker;
- a scheduled phase retaining a closed in-epic blocker as a bead-only condition;
- a retry with an open delegated phase excluded from waves while its dependents still wait on its bead;
- the land segment waiting on every phase bead, including closed or delegation-excluded phases, while retaining waits
  for currently launched agents;
- regular VCS and ChangeSpec wrappers, dry-run output, and confirmed launch query parity;
- the declared published-core minimum moving to 0.11.0.

Run `just install` first so the workspace builds and installs the linked Rust core with the new payload. Then run the
focused bead rendering/lifecycle and dependency-floor tests, followed by the required full `just check`. Inspect the
final diff and rerun any focused failures before closing only bead `sase-87.4`.
