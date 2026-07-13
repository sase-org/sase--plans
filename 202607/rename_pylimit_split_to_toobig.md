---
create_time: 2026-07-13 07:17:52
status: done
prompt: 202607/prompts/rename_pylimit_split_to_toobig.md
tier: tale
---

# Plan: Rename the pylimit split workflow and chop to toobig

## Objective

Complete the recent line-count-tool migration by renaming the SASE workflow and Athena lumberjack chop from
`pylimit_split` to `toobig_split`, while preserving the existing behavior that scans oversized Python files and launches
wait-chained split agents for them.

## Current State and Scope

- SASE owns the bundled workflow in `xprompts/pylimit_split.yml`. Despite its old public name, the embedded Python
  already resolves the `toobig` executable next to the active Python interpreter and invokes it with `--files-only` for
  both `src` and `tests`.
- The chezmoi repository owns the Athena chop configuration in `home/dot_config/sase/sase_athena.yml`; its chop name,
  description, agent-name prefix, and workflow reference all still use `pylimit_split`.
- SASE code comments, tests, user documentation, and blog examples still present `pylimit_split` as the live workflow.
- The standalone `pylimit` and `pylimit_files` scripts in chezmoi are not called by this chop or the current workflow.
  They are separate user utilities and will not be removed or renamed as part of this focused change.

## Design Decisions

- Use `toobig_split` as the workflow name and `sase_toobig_split` as the configured chop and scheduled-agent prefix.
- Make a clean rename rather than retaining a duplicate `pylimit_split` compatibility workflow. The controlled chezmoi
  consumer will be updated in the same effort, and keeping an alias would continue advertising obsolete terminology.
- Preserve the current thresholds, scanned trees, `toobig --files-only` invocation, path de-duplication, collision-safe
  child-agent naming, `%wait` chaining, chop grouping, auto-approval, hidden workflow behavior, and hourly schedule.
- Treat the chop-name change as a new scheduler identity. Its first post-deployment run may be immediately eligible
  because the persisted `sase_pylimit_split` timestamp is keyed by the old name; old run history can remain as
  historical state rather than being rewritten or deleted.

## Implementation

1. **Rename the bundled SASE workflow and its behavioral test surface.**
   - Rename `xprompts/pylimit_split.yml` to `xprompts/toobig_split.yml` without changing its established launch
     behavior.
   - Rename the focused workflow test module and its fixtures/helpers to `toobig` terminology.
   - Keep explicit regression coverage that the workflow calls the interpreter-adjacent `toobig` executable with
     `--files-only`, scans `src` and `tests` with the configured thresholds, launches nothing for an empty result,
     de-dupes files, and generates stable unique names plus a wait-chained multi-prompt.

2. **Move all live SASE references to the new public name.**
   - Update workflow-loader expectations, chop-dedup and notification fixtures, reference/parser examples, and source
     comments that currently use the real bundled workflow as their example.
   - Update the workflow reference table, CLI examples, workflow specification, and current blog explanations to use
     `#!sase/toobig_split` (and `sase_toobig_split` where the chop itself is named).
   - Audit the repository for stale `pylimit_split` references. Preserve historical CHANGELOG text that accurately names
     the workflow at the time of the recorded change; remove the old name from active code, tests, and current docs.

3. **Rename the chezmoi-managed Athena chop.**
   - In `home/dot_config/sase/sase_athena.yml`, change the chop identity to `sase_toobig_split`, describe the
     `#!sase/toobig_split` workflow, update the `%name` prefix, and invoke the renamed workflow.
   - Preserve `%group:chop`, `%auto`, the `#gh:sase` target, and `run_every: 60m` so only terminology and workflow
     routing change.
   - Roll out the new SASE workflow before applying the chezmoi consumer change, then deploy the managed config through
     the repository's normal chezmoi update/apply flow. Avoid manually mutating AXE timestamp or run-history files.

4. **Validate both repositories and the cross-repository contract.**
   - In SASE, run `just install`, the focused renamed workflow tests and related loader/parser/chop tests, then the
     required full `just check` suite.
   - Confirm the xprompt catalog/loader discovers `sase/toobig_split`, validates the YAML workflow, and no longer
     exposes `sase/pylimit_split`.
   - In chezmoi, run `just check`, inspect the managed diff, and verify the effective Athena config contains exactly one
     hourly `sase_toobig_split` chop pointing at `#!sase/toobig_split`.
   - Perform a final reference audit across both repositories: live `pylimit_split` references should be gone except for
     intentionally historical material, while the unrelated legacy shell utilities remain untouched.

## Acceptance Criteria

- The bundled workflow is invoked as `#!sase/toobig_split` and continues to discover offenders exclusively through the
  installed `toobig` package's executable.
- Athena schedules `sase_toobig_split` hourly and launches agents with the matching new prefix.
- Existing split-agent orchestration semantics and regression coverage remain intact.
- Current code, tests, and documentation no longer describe `pylimit_split` as an available workflow or chop.
- Both repositories pass their prescribed validation suites, and the chezmoi-managed configuration is deployed only
  after the renamed workflow is available.
