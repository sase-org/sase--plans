---
tier: tale
goal: The human-readable `sase plan list` dashboard shows each plan's configured SASE
  project name while preserving canonical project identity in stored metadata and
  stable JSON output.
create_time: 2026-07-15 09:01:16
status: wip
prompt: 202607/prompts/plan_list_project_display_names.md
---

# Plan: Show SASE project names in the plan dashboard

## Context

`sase plan list` currently builds proposed rows from notification action data and approved rows from agent metadata or
the artifact directory. Those sources intentionally contain canonical project keys such as `gh_sase-org__sase` and
`gh_bobs-org__bob-cli`. The Rich `Agent/Project` column renders those keys directly, even though SASE project records
expose the configured user-facing names `sase` and `bob-cli`.

The repository already centralizes this display-only conversion in
`sase.project_display_names.project_display_name_for()`. That helper reads project lifecycle records through the core
facade, caches the canonical-key-to-display-name map, and safely falls back to the input for unknown, legacy, or
temporarily unreadable projects. The fix belongs in the Python CLI rendering boundary: no Rust core behavior, project
records, notifications, agent metadata, or artifact paths need to change.

## Implementation

1. Update the plan inventory's Rich rendering path to resolve the project component of proposed and approved
   `Agent/Project` cells with `project_display_name_for()` before composing it with the agent name. Keep the inventory
   rows' canonical `project` value intact so collection, deduplication, metadata interpretation, and the documented
   stable `--json` projection retain their existing identity contract.
2. Apply the display projection in one shared rendering seam used by both tables. Preserve the current `-` handling and
   rely on the central helper's no-op fallback when a canonical key has no known display-name mapping; do not infer
   names by parsing `gh_*` strings.
3. Extend `tests/test_plan_inventory.py` with regression coverage that supplies canonical project keys and a controlled
   display-name mapping, then verifies:
   - proposed dashboard rows render the configured project name;
   - approved rows render the configured project name whether the canonical key came from explicit agent metadata or the
     artifact-path fallback;
   - canonical keys do not leak into the corresponding human-readable `Agent/Project` cells;
   - unknown projects still render unchanged and sentinel/missing values retain current behavior; and
   - `plan_inventory_to_json()` continues to expose the canonical project value, guarding the display-only boundary and
     stable JSON contract.

## Validation

Run the focused plan inventory test module first. Then, because implementation changes touch Python source and tests,
run `just install` followed by the repository-required `just check`. Finally, smoke-test both `sase plan list` and
`sase plan list --json` against configured projects to confirm the dashboard shows names such as `sase`/`bob-cli` while
JSON continues to carry canonical project keys.

## Risks and boundaries

The main regression risk is accidentally rewriting canonical identity at collection time, which could silently change
the stable JSON API or future programmatic consumers. Keeping conversion at the Rich rendering seam avoids that. The
cached resolver already degrades to the canonical value when project discovery fails, so listing historical plans
remains robust even after a project is removed or disabled.
