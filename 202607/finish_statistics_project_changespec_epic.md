---
tier: epic
title: Finish and land project and ChangeSpec statistics
goal: 'The project and ChangeSpec statistics epic is installable against a released
  Rust core, project-filtered plan and question summaries are truthful, and sase-70
  is closed only after current-main integration and full validation.

  '
phases:
- id: core-release-contract
  title: Publish the statistics wire contract
  depends_on: []
  description: '''Publish the statistics wire contract'' section: complete and verify
    the sase-core-rs 0.8 release that carries agent-statistics schema v2.'
- id: filtered-counter-contract
  title: Correct filtered plan and question counters
  depends_on: []
  description: '''Correct filtered plan and question counters'' section: separate
    project-filtered run counters from intentionally global document aggregates in
    the Statistics presentation layer.'
- id: integrate-and-land
  title: Revalidate and land sase-70
  depends_on:
  - core-release-contract
  - filtered-counter-contract
  description: '''Revalidate and land sase-70'' section: integrate current heads,
    run full validation, close the epic, clean post-close Symvision findings, and
    mark the original epic plan done.'
create_time: 2026-07-19 00:08:17
status: wip
bead_id: sase-72
---

# Plan: Finish and land project and ChangeSpec statistics

## Context

Epic bead `sase-70` has four closed children. The implemented feature is present on the current mainline commits:

- `8f6d3a2d4` preserves commit-time ChangeSpec attribution in projected commit metadata and primary-workspace
  `agent_meta.json`.
- `4238206d` in the linked `sase-core` repository implements statistics wire schema v2, project and ChangeSpec
  attribution, work rollups, project filtering, runtime dimensions, status joins, PyO3 bindings, and Rust coverage.
- `fcdf2638e` adds the Python query facade and immutable work view models.
- `74b3fc732` adds the Projects view, grouping strategies, project filter, status glyph reuse, keymaps, overview table,
  behavior coverage, and visual snapshots.

The pre-integration child hashes are patch-equivalent to those mainline commits. Phase 4's only omitted patch hunks were
dead chop APIs and Symvision exemptions already removed by concurrent `sase-6v` commit `f6dc6d7c3`; no later commit
touches the epic's statistics or attribution surfaces. The Rust workspace tests, formatting, and Clippy currently pass.

The landing audit nevertheless found two in-scope gaps:

1. `pyproject.toml` requires `sase-core-rs>=0.8.0,<0.9.0`, but the linked core head containing schema v2 still declares
   `0.7.0`, and no `0.8.x` artifact is available from the package index. Local editable installs bypass this mismatch,
   so passing local binding tests do not prove released installs work. The original epic explicitly required the core
   phase to merge and publish before the facade relied on the new fields.
2. `src/sase/stats/views.py::_build_plans_questions_view` still sources plan lifecycle and question-session summaries
   from the intentionally global activity response. With a project filter active, those values remain global even though
   the epic contract says run-derived plan/question counters filter and only document-derived extras are marked
   `(all projects)`. Existing tests assert the broad panel label but not the mixed-scope values.

There is also a known current-main validation failure outside this epic: eight chop output-contract tests expect result
files that concurrent chop code no longer writes. A responsiveness soak failure from the parallel run passed on serial
rerun. Do not absorb unrelated chop repair into this epic; recheck the latest base during landing and only treat it as
an integration issue if a direct dependency on sase-70 is demonstrated.

## Publish the statistics wire contract

Open the linked `sase-core` repository through the repository skill and use its normal release workflow. The feature
commit is after tag `v0.7.0`, so do not weaken the Python requirement to an older version and do not reuse an already
published version. Complete the next semantically correct release carrying `AGENT_STATS_WIRE_SCHEMA_VERSION == 2` and
the project/ChangeSpec work fields; this is expected to be `0.8.0` because the wire gained new public capability.

Use the repository's release-plz process and existing release metadata rather than an ad hoc local-only version edit. If
an automated release proposal exists, update or advance it through the normal reviewed lifecycle. Verify all of the
following against the resulting release commit and artifact:

- the workspace and `sase_core_rs` distribution report a version satisfying `>=0.8.0,<0.9.0`;
- `agent_stats_query_runs` returns schema v2 and the `work` section through the built wheel, not only a source override;
- the project-filter request fields and `project`/`changespec` runtime dimensions survive a wheel round trip;
- `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`
  pass.

Do not declare this phase complete merely because `just install` can build a `0.7.0` local override. If publication or
release-PR authority is external to the agent, leave a precise handoff at that normal lifecycle boundary and resume only
after the real artifact exists.

## Correct filtered plan and question counters

Keep all Rust aggregation and all filesystem access off the Textual event loop. The existing worker-backed loading,
debouncing, visibility gate, stale-result comparison, and two binding calls per refresh must remain intact; this phase
needs presentation-scope correction, not a new query path.

Make the view model distinguish values derived from filtered run records from intentionally global activity/document
values:

- source the Plans summary's proposed/approved/rejected/pending values from the run response so an active project filter
  changes them;
- source the run-backed Question summary values (sessions and asking agents) from the run response;
- retain plan tier and phases-per-epic values from the global plan-document scan, and retain question-size/count values
  that only exist in global question-session files;
- label only those global tables or values as `(all projects)` while a project filter is active, rather than labeling a
  mixed-scope panel in a way that makes filtered counters appear global or global counters appear filtered.

Pin this contract with model-builder and pane behavior tests for both all-project and filtered payloads. Tests must use
different filtered and global values so an accidental source swap is observable. Preserve current project-filter, range,
grouping, stale-result, hidden-tab, and responsiveness behavior. Regenerate a PNG snapshot only if the truthful scope
labels intentionally change the rendered visual, and validate any changed golden with `just test-visual`.

Before changing TUI code, read the required `tui_perf.md` long-term memory. Before addressing any Symvision finding,
read `symvision.md`. Run `just install` first, then focused statistics/view tests and `just check` as required by the
repository.

## Revalidate and land sase-70

Treat landing as a fresh integration audit, because main and the linked core may advance while the first two phases run.
Update through the sanctioned repository workflows, inspect commits since `8f6d3a2d4` in the main repo and since
`4238206d` in `sase-core`, and fold in any new consumer, duplicate, or conflict involving commit attribution or
statistics. Do not broaden the epic to unrelated base-branch repairs.

Verify the release artifact without a local source override, then verify the checked-out sources:

1. In `sase-core`, run formatting, Clippy with warnings denied, and the full workspace test suite.
2. In the main repo, run `just install`, the focused commit-attribution and Statistics suites, `just test-visual` when
   visuals changed, and `just check`. The full check must be green on the current integrated base; if unrelated chop
   failures still exist, rebase onto their fix or report that external blocker rather than hiding it.
3. Re-show `sase-70` and all four child beads, and confirm their notes against the final source and integrated commits.

Only after those checks pass, perform the landing steps in this order:

1. Close the epic with `sase bead close sase-70`.
2. After closure, run `just symvision`. Remove stale `sase-70` epic-symbol allowances and any genuinely unused code it
   reports according to the Symvision decision hierarchy, then rerun the exact lint and `just check` after code edits.
3. Open the plans sidecar through the repository skill and set `status: done` in the frontmatter of
   `202607/statistics_project_changespec_views.md`, the original plan linked by `sase bead show sase-70`.
4. Confirm the epic is closed, the original plan says `done`, both repositories are clean except for intentional landing
   artifacts, and the released dependency contract remains satisfiable.

## Risks and boundaries

- A local linked-core build can mask an impossible released dependency. The wheel/version probe is a completion
  requirement, not an optional release follow-up.
- Global question files and plan documents cannot be retroactively scoped by project. The UI must expose that limit
  honestly while still filtering counters that come from indexed run records.
- Do not add synchronous I/O or a third binding round trip to refresh the pane.
- Do not edit SASE memory files or generated instruction shims.
- Do not close `sase-70` before the release and filtered-counter phases are integrated and validated.
