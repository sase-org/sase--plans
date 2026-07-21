---
tier: tale
title: Race-free epic plan snapshot at launch creation
goal: 'Every epic work launch makes the approved plan available through a durable,
  project-scoped snapshot so clan summary rendering cannot lose a race with sidecar
  synchronization, while preserving the original plan reference for display and retaining
  all existing non-fatal fallbacks.

  '
create_time: 2026-07-21 11:06:10
status: wip
prompt: '[202607/prompts/race_free_epic_plan_snapshot.md](prompts/race_free_epic_plan_snapshot.md)'
---

# Plan: Race-free epic plan snapshot at launch creation

## Context and outcome

Epic clan summaries are resolved during directive extraction, before the claimed workspace and its SDD sidecars are
prepared. A newly approved epic can therefore be launched while neither the placeholder workspace nor the primary
checkout has fetched the just-archived plan, causing `sase_clan_summary_epic` to persist the bare epic identity. The
epic-work launcher already has an authoritative `BeadProject`, the epic's approved `design` reference, and the resolved
SASE project context. It should use those launch-time inputs to create a durable local copy before spawning any segment.

The implementation will add a best-effort snapshot at a stable project-state path under
`sase_projects_dir() / <project-key> / artifacts / epic-plans / <epic-id>.md`, overwrite it for every real relaunch of
the same epic, and export its absolute path to every phase and land segment. This phase remains entirely in the Python
SASE repository: there are no Rust-core, TUI, visual-golden, memory-file, or generated-instruction changes.

## Snapshot creation in the epic launch flow

Add focused helpers around `launch_epic_bead_work` in `src/sase/bead/cli_work_handler.py` to resolve and copy the
approved plan without consulting a possibly stale launch workspace:

- Resolve the source from the bead project's own store/root and the epic's `design` reference, supporting the current
  in-tree, local, and plans-sidecar layouts. Absolute design references may be used directly, while relative references
  must be mapped back to the authoritative plans root rather than probed from the process working directory.
- Resolve the durable destination from the already-determined VCS/ChangeSpec project name, with the existing project
  inference as a best-effort fallback for regular launches without a launch context. Keep the epic identifier confined
  to a single safe filename below the project's `artifacts/epic-plans` directory.
- Create parent directories and replace the snapshot atomically so a resumed or repeated launch always sees the newest
  complete approved plan. Snapshot only real launches, after confirmation and before agents are spawned; dry runs must
  remain read-only.
- Treat missing/unreadable sources, unavailable project identity, directory creation, and copy failures as non-fatal:
  emit a warning with enough source/destination context to diagnose the failure and continue with no snapshot value.
  Existing launch rollback and bead-state behavior must remain unchanged.

Pass the returned absolute snapshot path into `epic_work_segment_env`. Unit/launch tests will pin initial creation,
overwrite-on-relaunch, dry-run behavior, and warning-plus-launch behavior when snapshotting fails.

## Segment environment and runner metadata

In `src/sase/bead/work.py`, define `SASE_EPIC_PLAN_SNAPSHOT` alongside the existing epic launch constants. Extend
`epic_work_segment_env` and `_bead_env` with an optional snapshot path and add it to every phase and land segment only
when snapshot creation succeeded. Preserve the existing `SASE_EPIC_PLAN_REF` value and all child-specific bead fields.

In `src/sase/axe/run_agent_directive_metadata.py`, consume the new host-only environment variable into an
`epic_plan_snapshot` metadata field through the same table-driven path as the other epic fields, and include that field
in `preserved_agent_metadata` so runner re-execs cannot discard it. The variable must be popped with the other host-only
epic launch inputs, while `sdd_plan_path` and `plan_committed` continue to derive exclusively from the original plan
reference. Update focused metadata tests for first extraction, join-only members, and re-exec preservation.

## Guaranteed-local summary candidate

Update `_plan_reference_candidates` in `src/sase/scripts/sase_clan_summary_epic.py` to append the expanded absolute
`SASE_EPIC_PLAN_SNAPSHOT` path after all existing plan-reference candidates, deduplicating it when necessary. Current
workspace and primary-checkout copies must retain precedence, including when the original reference is absolute; a
missing, unreadable, or invalid snapshot must simply fall through to the existing bead summary and identity fallback.

Continue calling `load_plan_display` with the original `SASE_EPIC_PLAN_REF` as `display_path`. Thus a summary loaded
from the private snapshot still renders the authored repository-relative `Path:` value and never exposes the state
directory path in the clan panel.

Add script coverage for the race shape where every checkout candidate is absent and only the snapshot exists, asserting
the full PLAN-lane title, goal, and phase rendering plus the original displayed path. Also cover precedence,
deduplication/absolute references, and missing, unreadable, or invalid snapshots falling through without changing the
non-fatal behavior.

## Launch integration, documentation, and regression coverage

Extend `tests/test_bead/test_work_rendering.py`, `tests/test_bead/test_cli_work_epic_launch.py`,
`tests/test_bead/test_cli_work_epic_summary.py`, `tests/test_bead/test_clan_summary_epic_script.py`, and the focused axe
metadata/summary suites as appropriate to verify the complete contract:

- the launcher copies from the authoritative plans store, exports the same absolute snapshot to every segment, and
  overwrites stale content on a later launch;
- directive extraction passes the snapshot variable to the declaring summary subprocess, persists it in agent metadata,
  removes it from the ambient environment, and retains it across a re-exec;
- a deliberately stale or absent workspace/primary plan still persists the rich plan summary from the snapshot, while a
  failed snapshot leaves the established candidate and bead fallback paths intact;
- the original plan reference remains the rendered and persisted association, and no snapshot contents or path leak into
  warnings or unrelated metadata fields.

Document `SASE_EPIC_PLAN_SNAPSHOT` in `docs/agent_families.md` and `docs/xprompt.md` as an epic-work-provided,
guaranteed-local best-effort plan source used by the built-in epic summary script. Keep the broader multi-run summary
script contract reserved for the subsequent epic phase.

## Validation

Run `just install` before repository checks. Exercise the focused bead work, epic summary script, directive metadata,
and clan-summary smoke tests while iterating, then run the mandatory `just check`. Any ACE PNG snapshot change is a
regression because this phase changes data availability only, not summary rendering or TUI presentation.
