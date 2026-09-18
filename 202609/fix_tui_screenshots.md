---
tier: epic
title: Make TUI screenshot maintenance automatic locally and check-only in CI
goal: Replace the manual visual snapshot workflow with a reviewable, staged fix-tui-screenshots
  command that updates goldens during local check-full and explicit agent calls, while
  CI checks the complete corpus without accepting it.
phases:
- id: capture-protocol
  title: Collect complete screenshot candidates without changing goldens
  depends_on: []
  description: 'capture-protocol: add an isolated, versioned candidate-capture protocol
    to the shared ACE and pager fixtures, with xdist-safe records, execution completeness
    evidence, and focused regression tests. Preserve existing rendering, convergence
    checks, and direct comparison behavior.'
  size: medium
- id: maintenance-runner
  title: Compare candidates and safely apply screenshot changes
  depends_on:
  - capture-protocol
  description: 'maintenance-runner: implement tools/fix_tui_screenshots with explicit
    check mode, governed pytest execution, exact pixel comparison, bounded stability
    verification, conservative orphan handling, recoverable application, and tests
    for failures and unchanged golden trees.'
  size: medium
- id: change-reports
  title: Make every generated screenshot change reviewable
  depends_on:
  - maintenance-runner
  description: 'change-reports: extend the existing visual report pipeline to consume
    the run manifest and report created, updated, and stale screenshots, preserving
    before images, grouped diffs, contact sheets, JSON, CI annotations, and durable
    per-run output on successful updates as well as failures.'
  size: medium
- id: workflow-integration
  title: Switch commands, exhaustive verification, CI, and agent guidance
  depends_on:
  - change-reports
  description: 'workflow-integration: expose the canonical Just recipe, call its update
    form from local check-full and its explicit check form from the existing CI visual
    job, migrate old entry points and documentation, update the two relevant reference
    memories, and validate the combined workflow before landing.'
  size: medium
proposed_by: bbugyi200.athena.0mx
create_time: 2026-09-18 10:39:48
status: wip
bead_id: sase-12z
---

- **PROMPT:** [prompts/202609/fix_tui_screenshots.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fix_tui_screenshots.md)
- **BEAD:** [sase-12z](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12z/README.md)

# Outcome and decisions

Implement the user's requested behavior: `just fix-tui-screenshots` materializes missing
or changed screenshot goldens, local `just check-full` runs it, and CI invokes
`just fix-tui-screenshots --check` to fail on drift without modifying goldens. Agents
also run the command explicitly when their work changes rendered output or snapshot
coverage. Keep it out of `just fix`, `just lint`, ordinary `just check`, and the per-SHA
Master Gate.

This is an epic because capture, filesystem application, review artifacts, and workflow
migration have different failure modes and meaningful independent acceptance tests. Each
phase is bounded direct implementation work. The dependency chain is deliberate:
integration must not expose automatic updates before safe application and useful reports
exist. No implementation changes precede approval.

Research context:
`research:202609/fix_tui_screenshots_command/fix_tui_screenshots_command.md`, read
through `sase artifact read`. Its recommendations are design input, not a replacement
for the current request. Adopt staged captures, comprehensive drift reporting,
pixel-identical no-ops, conservative orphan cleanup, renderer guards, and CI check-only
execution. Depart from its recommendation to make `check-full` non-mutating: the user
explicitly requested automatic updates there. Use one canonical recipe with an explicit
check flag; a second lint-named entry point adds no necessary behavior and risks
accidental inclusion in routine linting.

Do not add the report's suggested scheduled bot commits, master pushes, finalizer
conflict resolution, reverse-import impact selector, corpus cropping, LFS, or a renderer
change. The complete visual corpus runs in `check-full` without a heuristic skip. This
deliberately increases exhaustive-lane cost; normal agent checks remain inexpensive, and
targeted explicit invocations remain available.

Automatic generation does not establish visual correctness. Successful updates must
retain a report, announce its location, and receive agent inspection before completion.
Use the existing monitor follow-up mechanism for that inspection; do not change host
finalizers or authorize commits in the screenshot tool.

# Verified starting points

The planning checkout is at `c8c842fb3`; recheck interfaces when implementation starts
because concurrent work can change them. The research examined an earlier revision, so
its corpus counts and CI history are context rather than assertions to hard-code or
fresh measurements of this checkout.

- `Justfile`: `test-visual` delegates to `tools/run_pytest visual` after
  `_setup-visual`; `update-visual-snapshots` supplies the raw pytest update flag.
  `check-full` currently ends with cost and flake checks and has no visual stage. Its
  `tools/run_silent` wrappers discard successful command output.
- `tools/run_pytest`: owns worker admission, xdist, argument normalization, and
  execution for both `tests/ace/tui/visual` and `tests/pager/visual`. The fast,
  coverage, and scoped lanes exclude visual trees. Reuse this runner; do not create an
  ungoverned pytest subprocess lane.
- `tests/ace/tui/visual/png_diff.py`: `assert_png_matches(update=True)` immediately
  writes each golden and returns. Comparison raises on the first mismatch, which
  prevents subsequent captures in a multi-snapshot test. The pager fixture uses the same
  helper with a different golden root.
- `tests/conftest.py`, both visual `conftest.py` files, and `renderer_env.py` own pytest
  options, fixture configuration, renderer pins, and the Linux update guard. Exact
  comparison is the default in local and CI execution; tolerance settings are explicit
  investigation overrides, not CI defaults.
- `_ace_png_snapshot_waits.py` already proves ACE frame convergence and verifies the
  captured frame synchronously. Keep these protections. Pager captures use an
  `SvgExporter` adapter and must participate in the same capture protocol.
- `_png_diff_artifacts.py`, `_png_diff_comparison.py`, and
  `tools/render_visual_snapshot_failure_report` already supply image comparison,
  per-image evidence, HTML, Markdown, JSONL, and workflow annotations.
- `.github/workflows/ci.yml` has a dedicated `visual-test` job with visual setup,
  failure report publication, and raw artifact uploads. Keep the job identity and
  separation from Master Gate. Structural tests pin the old recipe name.
- `lint_and_test.md` and `tui_screenshot.md` contain agent workflow guidance that must
  migrate with these commands. The former also incorrectly describes CI renderer
  tolerance; correct that claim while rewriting its PNG section.

This is repository verification tooling and screenshot test support, so keep it in
`tools/` and test-support modules. Do not add application/backend behavior or duplicate
Rust core domain logic. No linked-repo edits are required.

# Command contract

| Invocation                                                                    | Behavior                                                                                                                              |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `just fix-tui-screenshots`                                                    | Capture both complete visual trees; validate candidates; apply adds, updates, and proven stale removals; print and retain the report. |
| `just fix-tui-screenshots --check`                                            | Same inventory and comparison, no golden writes; fail on any required change.                                                         |
| `just fix-tui-screenshots -- tests/ace/tui/visual/test_example.py -k example` | Targeted pytest selection; apply only captured changes; never infer or remove orphans. The path is illustrative.                      |
| `just check-full`                                                             | Existing exhaustive checks, followed by one full screenshot update stage on local supported hosts.                                    |
| CI `visual-test`                                                              | Explicit `just fix-tui-screenshots --check`, with report and artifact publication on drift or execution failure.                      |

Give the standalone tool `-c/--check` and `-h/--help`, with clear sorted help and public
short aliases for any further options. Arguments after `--` are pytest
selectors/options. Preserve quoting and invocation-directory path normalization. Allow
selectors such as paths, node IDs, and `-k`; reject options that replace the visual
lane, disable the capture plugin, request legacy writes, or undermine the declared run
scope. Account for environment-supplied pytest options, plugin-driven deselection, and
shard settings when deciding whether inventory is complete.

The tool's exit contract is `0` for fresh/check or successfully applied/update, `1` for
check-mode drift, `2` for usage/environment refusal, and `3` for execution, invalid
capture, incomplete full inventory, or application failure. Record the underlying child
exit code separately. Document that `just` may normalize a child failure code;
automation needing the distinction reads the run manifest or calls the tool directly.
Never convert broken tests into successful regeneration.

An update invocation refuses before mutation when `CI` or `GITHUB_ACTIONS` is truthy,
with remediation to use `--check`. Do not silently switch modes from environment
variables. A direct `check-full` invocation in CI therefore refuses at its update stage;
the repository's actual CI uses the explicit dedicated check job. Preserve the renderer
fingerprint gate in both modes and the Linux-only write restriction. Do not
automatically install different pins or relax equality.

# Phase capture-protocol

Implement a capture session used explicitly by the ACE and pager pytest fixtures. Add an
internal candidate-directory/run-identity option and versioned records; keep it separate
from ordinary direct assertion-helper calls so existing helper unit tests with temporary
golden roots cannot pollute the real corpus inventory.

1. At each assertion, preserve current rendering and convergence checks, then write
   candidate PNG/SVG and metadata to this run's scratch directory instead of comparing
   or writing the golden. Continue to the next snapshot assertion in the same test. Do
   not suppress ordinary assertions, exceptions, teardown failures, timeouts, or
   convergence failures.
2. Use repository-relative golden paths under exactly the ACE and pager PNG roots as
   canonical identities. Include test node ID and source location, candidate hash and
   paths, root identity, capture sequence, and relevant comparison settings. Use
   collision-resistant artifact identities rather than the existing lossy slug alone.
   Reject absolute paths, traversal, symlink escapes, and duplicate canonical golden
   ownership, even if bytes agree.
3. Make writes worker-local or individually atomic; merge deterministically in the
   controller. Never have xdist workers append uncoordinated records to a shared JSONL
   file. Record collected/executed nodes, skips/xfails, unexpected deselection, worker
   loss, and completion markers separately from captures.
4. Define full inventory as a successful unrestricted visual run with complete execution
   evidence for both roots. Expected exclusion of non-visual tests is not incomplete
   inventory. Other selection, skipped/xfail coverage, missing worker records, or
   collection-only execution cannot prove orphanhood. Fail closed for a requested full
   run that lacks evidence; targeted runs explicitly report their limited scope and
   disable pruning.
5. Leave ordinary compare calls usable for diagnostics and parity tests. Keep optional
   renderer imports out of fast-lane collection paths; import visual capture support
   lazily and keep image-dependent tests in the visual lane.

Validate capture of multiple differing snapshots in one test, ACE/pager root separation,
duplicate names across workers, malformed/escaping paths, incomplete worker output, and
ordinary test failures. Hash the golden tree before and after candidate collection and
require no changes. Include a real small xdist fixture project, not only mocks of the
manifest merger.

# Phase maintenance-runner

Add `tools/fix_tui_screenshots` as the single orchestration entry point. Keep
implementation modules focused rather than extending large existing files into
monoliths. At this phase it may be exercised directly in the prepared visual
environment; public Just/CI integration waits for the reports phase.

1. Perform preflight validation, acquire a checkout-local exclusive maintenance lock,
   and create a unique run directory under `.pytest_cache/sase-visual/`. Refuse an
   overlapping maintenance run clearly rather than overwriting its scratch data. Record
   renderer identity, arguments, mode, scope, start state, pre-existing dirty golden
   paths, and the baseline bytes/hashes of both roots. Never stage files, reset the
   tree, or require an otherwise clean checkout.
2. Invoke the existing governed visual runner with candidate collection enabled.
   Preserve its argument and suite-gate behavior. Retain logs and partial candidates for
   failed runs, but do not apply anything until all tests and capture validation have
   succeeded. A zero-test selection is an error.
3. Compare all captures against the run-start baseline. Treat dimensions plus decoded
   RGBA pixels as equality, with a byte-equality fast path. An encoding difference alone
   must not rewrite a golden or change its mtime. Reuse the current exact comparator; do
   not make transparent dimension changes equal just because padding produces zero
   changed pixels. Keep explicit diagnostic tolerances on the low-level compare path;
   the maintenance command itself operates with exact equality in both modes.
4. Classify `created`, `updated`, `unchanged`, and `stale`. Stale means an existing PNG
   in one of the two allowed roots with no capture in a proven full inventory. Selected
   runs neither report definitive stale entries nor delete unvisited files. Reject
   incomplete full inventories before applying anything.
5. Before applying created/updated candidates, rerun their owning test nodes once
   through the same governed runner into a separate verification directory. Compare
   every capture of those nodes and their ownership/key sets with the first pass. Any
   fresh-run instability or execution failure aborts the whole update and preserves both
   attempts for diagnosis. This is a bounded second pass, not retry-until-green.
   Check-only mode needs no second pass. Maintain the existing intra-test convergence
   guarantees as well.
6. Produce the complete machine-readable change manifest before writing goldens. It is
   the input to the reports phase and contains run scope/completeness, status, counts,
   per-change paths, baseline and candidate hashes, dimensions, pixel statistics, and
   evidence locations. Preserve pre-update expected bytes, including stale files. Report
   baseline dirty paths separately so a no-op command does not claim to have created
   someone else's working-tree diff.
7. Apply only the computed changes after rechecking the golden baseline against
   concurrent external edits. Use per-file atomic replacement plus a durable write-ahead
   journal and backups. Roll back handled write errors and normal interruption to the
   invocation's baseline, including pre-existing dirty bytes. Preserve the Git index and
   every unrelated path.
8. Be precise about crash guarantees: staging protects the tree throughout test
   execution; replacing many files is not a single filesystem transaction. After an
   uncatchable kill during apply, detect the unfinished journal on the next invocation.
   An update invocation may safely restore the recorded baseline before starting again,
   provided paths still match known before/after hashes; otherwise refuse and identify
   conflicts. A check invocation must refuse without even performing recovery writes.
   Never discard backups or silently declare a partial apply complete.

Test the outcome matrix on synthetic small corpora: clean, missing, mismatched,
encoding-only, dimension-only, and stale images in both modes; targeted pruning
suppression; test/teardown/worker failures; failed determinism verification;
renderer/non-Linux/CI refusal; pre-existing dirty files; concurrent edits and
invocations; write failure rollback; process termination and recovery. Prove check mode
leaves file contents, inventory, mtimes, and Git index unchanged while allowing scratch
reports. Keep subprocess/argument tests runnable without installing the visual extra;
put actual image tests in a visual-marked module.

# Phase change-reports

Extend `tools/render_visual_snapshot_failure_report` and its helpers instead of forking
another report stack. Retain support for existing `failure.json` records from direct
diagnostic comparisons. Add the new versioned run/change records as an explicit input,
not an unscoped scan that could mix old runs.

1. Emit a bounded console summary with scope, status, counts, dirty-before paths, and
   the exact report/manifest locations. Distinguish an applied update from drift,
   environment refusal, broken tests, and interrupted application. For check-mode drift,
   print `run: just fix-tui-screenshots`, followed by the instruction to inspect the
   report.
2. Retain per-run self-contained HTML, Markdown summary, JSONL change records,
   actual/expected/diff PNGs, and available source SVGs. For created images show that
   there was no baseline; for stale removals show the previous image and lack of a
   producing assertion. Never label either as an ordinary pixel diff.
3. Group cascaded updates by dimensions and diff-mask signature, with stable
   representatives and access to every member. Provide a representative contact sheet,
   bounded console groups, and the complete list in HTML/JSON. Include pixel bounding
   boxes. Add terminal-cell regions only when recorded export geometry proves the
   mapping; otherwise omit them rather than inferring cells from a filename. Grouping
   identifies similar regions, not proven causes.
4. Preserve test-source links, useful CI annotations, annotation escaping, and summary
   links to uploaded reports. A successful update must produce the same useful evidence
   as check-mode drift. Build review artifacts before apply; report-generation failure
   must not leave accepted changes without a report. Finalize applied/failed status
   atomically after application.
5. Keep outputs in unique run directories and publish an atomic latest-run pointer only
   to that run's results. A later clean check must not erase the report of an earlier
   update. Failed preflight should replace stale status with a current refusal record,
   not imply that an old report describes it.

Extend `tests/test_render_visual_snapshot_failure_report.py` for legacy input, new
kinds, grouping, empty runs, escaping, and report links. Add image-dependent
contact-sheet tests to the visual lane. Verify before-image fidelity after successful
application and that an aborted run never appears as applied. Test an
unreadable/malformed manifest and failure before reporting/application.

# Phase workflow-integration

1. Add a positional-arguments `fix-tui-screenshots *args` recipe using `_setup-visual`
   and the workspace virtualenv, preserving `SASE_JUST_INVOCATION_DIR`. Add the full
   update stage at the end of local `check-full`, after existing checks succeed. Print
   its compact report visibly: either run it outside `tools/run_silent` or print the
   exact just-completed manifest summary in a separate step. Do not rerun the suite to
   obtain output and do not print a previous run's summary after failure.
2. Switch the existing `.github/workflows/ci.yml` `visual-test` job to the explicit
   check command. Preserve setup, timeout discipline, HTML publication, Markdown
   summary, annotations, and raw artifact uploads, adapting paths to the current run. On
   execution failure upload available logs/partial captures and report that failure; do
   not imply the corpus merely needs accepting. CI has no golden-writing fallback,
   tolerance relaxation, bot commit, or PR.
3. Retain `test-visual` as a simple supported check alias and `update-visual-snapshots`
   as a simple supported update alias to the same maintenance runner. These aliases
   retain no alternate implementation and need no temporary deprecated behavior branch.
   Keep `test-visual-contention` as a check-only diagnostic using the existing runner.
   Retire direct `--sase-update-visual-snapshots` writes: reject that raw pytest option
   with an actionable command replacement, migrate all repository callers, and remove
   the immediate-write branch from fixture-driven execution. Do not let aliases, raw
   pytest, or CI environment settings bypass staging or guards. This is a developer/test
   workflow migration, not a new SASE runtime feature; it needs no runtime feature flag.
4. Update `docs/development.md`, the command summary in `README.md`, and
   `src/sase/ace/tui/fonts/README.md`. Search other tracked non-memory docs for active
   old instructions and change those affected, including stale claims that `test-visual`
   is the sole execution path. Document ACE and pager roots, exact check behavior,
   selectors, full-only pruning, failure recovery, and renderer upgrade steps. Leave
   historical measurements labeled as history.
5. As the agent-guidance portion of the requested migration, update exactly the
   canonical reference notes `sase/memory/lint_and_test.md` and
   `sase/memory/tui_screenshot.md` using `/sase_memory_write` and audited reads. Replace
   old manual-update instructions and stale CI tolerance prose; state when agents must
   run the new command and that local `check-full` can modify goldens. Require
   inspecting each creation/removal and all update groups, expanding groups with
   unexpected differences; generation is not approval. Explain that full and targeted
   visual commands may require `/sase_monitor`. For a mutating verification run use a
   follow-up that inspects the report and golden changes before finalization; do not
   attach a prepared host-completion intent that skips that inspection. Run
   `sase memory init` to republish; never hand-edit `AGENTS.md` or provider shims. Do
   not rewrite immutable decisions.
6. Update Justfile and CI structural tests, including `tests/test_justfile_lint.py`,
   `tests/test_github_actions_ci_workflow.py`, and
   `tests/test_github_actions_ci_master_gate.py`. Assert the update call and visible
   successful summary in `check-full`; assert explicit check-only CI; forbid the new
   command from ordinary check/lint/fix and Master Gate. Add spy execution tests for
   argument forwarding, alias routing, and exit propagation, not only string matching.
   Ensure fast/scoped collection still works without Pillow or visual pins.

# Combined validation and acceptance

Use each phase's focused tests while developing; follow the repository's required
`just check` policy after tracked changes. Before handing long verification to a
monitor, run `just fix` (at minimum `just fmt`) inline. All `just check-full` runs use
`/sase_monitor` with `TESTING` / `TESTED` and an explicit report-review follow-up.
Increase the timeout to account for the added visual run and, on updates, the bounded
verification pass. Do not run the contention harness or repeat full suites routinely.

On the integrated implementation:

1. Exercise controlled miniature corpora for the complete outcome matrix before using
   the real committed screenshots. In a single revision, compare the existing low-level
   check results with the new collector's results for stable, changed, and
   multi-snapshot tests. The new path must report every drift the old path sees, plus
   later assertions previously hidden by its first failure.
2. Run a targeted ACE and pager capture through the real runner, confirming invocation
   from a subdirectory, renderer preflight, xdist behavior, and report generation.
   Verify targeted runs preserve all unrelated goldens.
3. Run the combined tree's local `just check-full` through a monitor. Inspect its newly
   applied report in the follow-up, including every creation/removal and representative
   plus member coverage for each update group. If it exposes unintended UI changes or
   nondeterminism, fix those causes rather than accept them as baseline maintenance.
   Refresh legitimate stale goldens as part of completing this workflow migration; do
   not hide unrelated execution failures.
4. Run `just fix-tui-screenshots --check` after the update and require success with an
   unchanged golden tree. Use this as the no-op/idempotence verification; the focused
   tests already prove a second fix preserves bytes and mtimes. Use the CI check
   environment to prove update refusal and explicit check behavior without repeatedly
   running the entire corpus.
5. Record the commands, outcome counts, report paths, visual review conclusions, and
   full-run timing. Confirm reports and journals stay in ignored scratch locations, only
   intended files/goldens changed, and no renderer pins or unrelated production code
   were changed merely to make the suite pass.

Acceptance requires all four phases, passing focused/structural checks and the combined
verification, reviewable automatic local updates, a no-write CI check, and current agent
instructions. A recipe rename or a successful blind golden refresh alone does not
complete the work.
