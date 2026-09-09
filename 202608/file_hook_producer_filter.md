---
tier: epic
title: Prevent duplicate digest-suffixed research Highlights PDFs
goal:
  Each newly added consolidated research report invokes Bob exactly once from its
  canonical checkout path, producing only the intended basename while preserving
  artifact durability and existing file-hook behavior by default.
phases:
  - id: core-producer-filter
    title: Add producer-aware file-hook filtering to SASE
    depends_on: []
    description:
      "core-producer-filter: extend the file-hook configuration and dispatch contract
      with a validated producer filter, document artifact-store path semantics, and
      cover backward compatibility and dispatch behavior."
    size: medium
  - id: research-provider-policy
    title: Restrict research Highlights generation to committed report events
    depends_on:
      - core-producer-filter
    description:
      "research-provider-policy: update the sase-research-artifacts provider to exclude
      artifact-copy events while retaining commit, SDD, and finalizer reconciliation
      paths, with provider-level contract tests and documentation."
    size: small
  - id: e2e-verification
    title: Verify one canonical Highlights output across the coordinated repositories
    depends_on:
      - core-producer-filter
      - research-provider-policy
    description:
      "e2e-verification: exercise artifact creation, commit dispatch, finalizer
      reconciliation, effective chezmoi configuration, and Bob dry-run naming to prove
      exactly one canonical output without deleting historical vault files."
    size: small
proposed_by: bbugyi200.athena.0b7
bead_id: sase-s5
create_time: 2026-09-09 19:50:18
status: wip
---

- **PROMPT:**
  [prompts/202608/file_hook_producer_filter.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/file_hook_producer_filter.md)
- **BEAD:**
  [sase-s5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s5/README.md)

# Plan

## Diagnosis and root cause

The duplicate is not created by a hidden second write inside `bob highlights create`. It
is produced by two independent SASE file-hook events for the same logical research
report:

1. `sase artifact create` copies the report into durable artifact storage. The storage
   layer deliberately names the copy `<source-stem>-<sha256-prefix>.md`; for example,
   the stored copy of `conditional_launch_admission.md` is
   `conditional_launch_admission-ad048d84997e.md`, and the suffix exactly matches the
   first twelve characters of the file's SHA-256 digest.
2. Artifact-time dispatch preserves the original repository-relative path for filter
   matching, but replaces the executable absolute path with that durable stored path.
   The detached runner appends this absolute path as the command's final argument.
3. Bob intentionally derives both its PDF basename and `--include-id` marker from the
   input Markdown filename. A dry run against the stored copy therefore plans
   `conditional_launch_admission-ad048d84997e.pdf`, while a dry run against the
   canonical report plans `conditional_launch_admission.pdf`.
4. The report is subsequently committed. Commit-time dispatch sees the canonical
   checkout path and runs the same hook again, producing the intended basename. Commit
   and finalizer events deduplicate through their deterministic commit-SHA batch, but
   artifact batches use a different identity and therefore do not deduplicate with the
   later commit batch.

The retained file-hook evidence demonstrates both successful executions: artifact batch
`45d482f2ab8045728c4332f08f6fac54` used the digest-suffixed stored path, and commit
batch `1b24e216b46bbc9f325499a7` used the canonical
`202608/conditional_launch_admission/conditional_launch_admission.md` path. The two
Markdown inputs have identical SHA-256 content.

The durable artifact copy is necessary because hook execution is detached and the
original source can move. The correct fix is therefore to make event source an explicit
hook-selection dimension and configure this basename-sensitive hook to run only for
committed repository events. Do not weaken artifact storage naming or teach Bob to
silently strip strings that merely resemble hashes.

## Scope and compatibility

- Change the `sase` repository's file-hook configuration and dispatcher.
- Change the `sase-research-artifacts` repository's `research-highlights` provider
  policy. Open that repository through `/sase_repo` before editing it.
- Keep the existing chezmoi declaration
  (`use: sase-research-artifacts@research-highlights` plus the local Bob command); the
  provider's filters should flow through the existing deep-merge contract without a
  machine-specific workaround.
- Make no `bob-cli` source change. Its basename behavior is intentional and serves as
  the end-to-end oracle for this fix.
- Do not delete existing digest-suffixed PDFs, reference notes, annotations, or artifact
  copies. Some historical reference notes contain user-authored highlights. Inventory
  leftovers for a separately approved cleanup instead.

## Phase 1: Add producer-aware file-hook filtering to SASE

Introduce `filters.producers` as an optional list whose accepted values are the existing
file-hook producer identities: `artifact`, `commit`, `sdd`, `finalizer`, and `dispatch`.
Omission must retain today's behavior and match every producer. An explicit list is
AND-ed with the existing project, sidecar, path, agent, operation, and cause dimensions.

Implement the contract in the established Python file-hook boundary:

- Extend `src/sase/config/file_hooks.py` with the typed producer filter, direct-value
  and YAML validation, parsing, matching, and fail-soft diagnostics. Keep the
  authoritative producer vocabulary shared with `src/sase/file_hooks/audit.py` rather
  than duplicating drifting string sets.
- Thread the dispatch producer through matching in `src/sase/file_hooks/dispatch.py` so
  both the preflight match and batch payload construction select the same hook/event
  pairs. A filtered artifact event should record a normal `no_match` producer audit and
  should create no batch or runner process.
- Extend `src/sase/config/sase.schema.json`, the human and JSON output in
  `src/sase/main/file_hook_handler.py`, and the file-hook documentation. Bump the
  versioned `sase file-hook list --json` schema because its filter object gains a field.
  Document that artifact events execute against durable content-addressed paths whose
  basenames may carry a digest, and recommend producer filtering for hooks whose output
  identity depends on the input basename.
- Update unit and schema tests in `tests/test_file_hooks.py`,
  `tests/test_config_schema_extensions.py`, and `tests/test_file_hook_cli.py`. Cover
  valid and invalid producers, omitted-filter backward compatibility, rendered/JSON
  inventory, and AND semantics with the other dimensions.
- Add a regression in `tests/test_file_hook_engine.py` or
  `tests/test_file_hook_dispatch_regression.py` that drives the same logical report
  through artifact, commit, and finalizer producers. With a committed-only hook, assert
  that artifact dispatch is `no_match`, commit dispatch creates one batch using the
  canonical path, and finalizer reconciliation reuses that commit batch without a second
  spawn.

Run `just install` before verification, then run focused file-hook/config tests and
`just check`. If scoped selection broadens or reports an unusual selection, use the
documented monitored full-check path.

## Phase 2: Restrict the research Highlights provider

In the `sase-research-artifacts` repository, add `producers: [commit, sdd, finalizer]`
to the packaged `research-highlights` filter. This keeps every durable committed-file
route, including finalizer repair, while excluding the early `sase artifact create`
event that exposes the content-addressed basename.

Update `src/sase_research_artifacts/provider.py`, README/provider documentation, and the
provider/filter tests. Assert that the installed provider exposes the producer list and
that its existing sidecar, path, agent, operation, and companion-file exclusions remain
unchanged. The plugin already targets the unreleased SASE line containing the provider
registry; keep its dependency floor coordinated with the SASE version that actually
contains `filters.producers`.

Run `just install` and `just check` in the plugin checkout using the coordinated SASE
source. Include its wheel/entry-point contract if dependency or packaged metadata must
change.

## Phase 3: Coordinated end-to-end verification

Exercise the real provider through the coordinated SASE/plugin environment and the
existing chezmoi configuration:

1. Confirm `sase validate` succeeds and `sase file-hook list --json` reports
   `research-highlights` with producers `commit`, `sdd`, and `finalizer` plus the local
   `bob highlights create --include-id` command.
2. In an isolated temporary research sidecar and Bob directory, create one consolidated
   report, register it as an explicit artifact, then commit it and run finalizer
   reconciliation. Use a controlled Bob command or dry-run wrapper so the test does not
   write to `~/bob`.
3. Assert the artifact producer records `no_match` and launches nothing; the commit
   producer launches exactly once with the canonical Markdown basename; and finalizer
   reconciliation returns `batch_already_present` without another launch.
4. Run `bob highlights create --include-id --dry-run` on the canonical test report and
   assert the planned PDF and marker id contain no digest suffix.
5. Re-run the focused cross-repository regression suites. Before landing the epic's
   combined SASE tree, run `just check-full` only through `/sase_monitor`, as required
   by the repository verification policy.

## Acceptance criteria

- The effective research hook never executes for producer `artifact`.
- A newly added consolidated research report produces one hook execution from its
  canonical checkout path, and finalizer reconciliation remains idempotent.
- Bob plans `<report-stem>.pdf` and marker id `<report-stem>`; no content-digest suffix
  reaches either value.
- File hooks without `filters.producers` continue to match artifact and committed-file
  events exactly as before.
- Invalid producer names fail schema/runtime validation with actionable diagnostics,
  while `sase file-hook list` exposes the effective producer policy.
- SASE and plugin checks pass, the coordinated full SASE verification passes before
  landing, and no personal vault files are removed or rewritten during verification.
