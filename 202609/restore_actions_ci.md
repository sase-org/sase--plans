---
tier: epic
title: Restore SASE Master Gate and Full CI
goal: Align the required Rust core with the Python source, repair stale visual tests,
  and restore the performance floors so Master Gate and Full CI pass without weakening
  their checks.
phases:
- id: core-contract
  title: Align the source pin and published core requirement
  depends_on: []
  size: small
  description: 'core-contract: advance the Rust source pin and package floor to the
    verified release containing every required binding and behavioral fix, and validate
    the affected contracts.'
- id: visual-contracts
  title: Repair visual fixtures and regenerate reviewed goldens
  depends_on:
  - core-contract
  size: medium
  description: 'visual-contracts: correct outdated tribe and job expectations and
    SVG text assertions, then review and refresh snapshots for intentional TUI changes
    using the pinned renderer.'
- id: performance-floors
  title: Reduce scan hydration and notification copy overhead
  depends_on:
  - core-contract
  size: medium
  description: 'performance-floors: reproduce the two failing performance anchors,
    optimize the measured Python adapter overhead while preserving contracts, and
    pass the existing regression ceilings.'
- id: integrated-verification
  title: Verify the combined repair against both CI lanes
  depends_on:
  - core-contract
  - visual-contracts
  - performance-floors
  size: medium
  description: 'integrated-verification: verify the combined tree with the exact selected
    core, the exhaustive checks, visual suite, performance floors, and current Actions
    results, resolving any remaining failures in scope.'
proposed_by: bbugyi200.athena.0mh
create_time: 2026-09-17 15:12:07
status: wip
bead_id: sase-126
---

- **PROMPT:** [prompts/202609/restore_actions_ci.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/restore_actions_ci.md)
- **BEAD:** [sase-126](https://github.com/sase-org/sase--beads/blob/main/pages/sase-126/README.md)

# Restore SASE Master Gate and Full CI

## Scope and execution

The user requested investigation with `actstat`, diagnosis, and repair, with a validated
proposal before implementation. This plan covers both the fast master gate and the
scheduled exhaustive lane. Separate implementation phases are justified because
dependency alignment is a prerequisite for independent visual and performance repairs,
followed by verification of their combined tree.

Planning inspected the checkout at SASE commit
`ff088437985ffa0f679ae59406280202f58b9279` without changing tracked files. Diagnostic
tests produced only disposable test output. All paths below are relative to the
implementer's checkout. Open any other repository through `/sase_repo`; do not assume a
particular sibling or numbered workspace path. The Rust fixes already exist upstream, so
this plan does not initially require Rust source changes.

## Evidence and root causes

### Master Gate: source and dependency requirements lag the Python consumers

`actstat` identified
[Master Gate run 35260881979, number 925](https://github.com/sase-org/sase/actions/runs/35260881979)
on SASE `0af5b08151ac275b81cc1d2e790b29d1d43df53d`. Lint fails at
`Check pinned core bindings`; shards 1, 3, 5, 6, and 7 contain 30 failed tests. The
newer run [35261584562](https://github.com/sase-org/sase/actions/runs/35261584562) was
still running during investigation and showed the same affected jobs failing.

`sase-core-revision.txt` pins `e210d18a80a3d16066ff81dd2866c68ab7a24786`, the `0.34.42`
release. Both `pyproject.toml` and `uv.lock` retain that package floor. CI faithfully
builds this pinned source, so its wheel is missing:

- `plan_agent_publication_batches`
- `service_config_compose`
- `service_restart_decide`
- `service_state_mutate`
- `service_state_read`

The remaining assertions also depend on newer Rust behavior. Upstream history maps every
observed Master Gate failure to an existing fix:

| Core commit                                | First release | Relevant behavior                                                            |
| ------------------------------------------ | ------------- | ---------------------------------------------------------------------------- |
| `a09386dbce873219ef21f44859d50535378e7ab3` | `0.34.43`     | Canonical Claude Fable usage-window identities and legacy-cache coalescing   |
| `51ae484`                                  | `0.34.44`     | Service configuration composer and binding                                   |
| `77492cc`                                  | `0.34.45`     | Agent publication batch planner                                              |
| `244634e143412a52231911f3f20b2dca376272a3` | `0.34.46`     | Edit `job_timeout` through its existing legacy `chop_timeout` source         |
| `a756136`                                  | `0.34.46`     | Service restart and state bindings                                           |
| `478795a4d5330f56d37431c96d38f64e9479411e` | `0.34.47`     | Registered-project aliases redirect to linked repositories by path or remote |

Verified target: core release commit `b4f7de3aee7ece9c9b4fa61c1776aa72f30f9de0`
(`0.34.47`). The read-only command `python tools/ratchet_core_window --check` confirmed
that PyPI has a complete `0.34.47` release and recommends `>=0.34.47,<0.35.0`.

The existing local environment already contains core `0.34.47`. Without modifying
source, `tools/check_sase_core_rs_bindings` passed all 651 required bindings, and 73
tests across all 11 affected test modules passed in 9.38 seconds. That is strong
evidence for the dependency diagnosis; it is not proof that a fresh installation or the
full suite passes.

### Full CI: visual fixtures and goldens lag intentional product changes

[Full CI run 35257902913](https://github.com/sase-org/sase/actions/runs/35257902913) on
SASE `14403c1594b63deac2d9dceec2adbb6c3d7eb79a` has a completed failing
[visual job](https://github.com/sase-org/sase/actions/runs/35257902913/job/105330392158):
572 failed, 387 passed, and one skipped. Of the failures, 568 are PNG mismatches; four
fail assertions or fixture validation before snapshot comparison.

The downloaded GitHub artifact `ace-visual-artifacts` (artifact ID `10514902031`) was
inspected in memory. For `changespec_initial_120x40`, exactly 250 pixels differ, all in
the title at bounds `(693, 45, 728, 62)`. The expected image says `sase ace (v0.7.1)`
and actual says `sase tui (v0.7.1)`. This is the intentional CLI rename in SASE commit
`4fc9e7ae49`; 502 failures report that same 250-pixel difference. The remaining image
differences require review individually or in clearly explained groups; they cannot all
be attributed to the title rename.

The four pre-comparison failures are:

- `test_help_guide_agents_png_snapshot`: the helper searches raw SVG for
  `Welcome to sase's TUI`, but only decodes nonbreaking spaces. The apostrophe is XML
  escaped. Existing XML-based styled-text helpers provide a suitable seam.
- `test_wait_modal_png_snapshot`: its fixture passes `tribe="#verification"`, which
  violates the current tribe identity contract.
- `test_agents_collapsed_panel_png_snapshot`: expects the public label `@chop` while
  current rendering correctly uses `@job`.
- `test_tribe_panel_display_config_png_snapshot`: likewise expects `@chop`.

The representative title snapshot and all four pre-comparison failures reproduced
locally with core `0.34.47` (five failures in 15.14 seconds). A core pin bump alone
therefore cannot repair the visual lane. The actual CI comparisons used exact pixel
equality; do not assume that a general statement about CI tolerance means these
differences are renderer noise.

### Full CI: two performance anchors have inadequate headroom

The same run's
[performance job](https://github.com/sase-org/sase/actions/runs/35257902913/job/105330392169)
fails only the Phase 7E floor step. Its report records:

| Anchor                                                                | Measured median                   | Existing ceiling |
| --------------------------------------------------------------------- | --------------------------------- | ---------------- |
| `scan_agent_artifacts.synthetic_6p_200pp.scan_facade`                 | 285.859 ms                        | 281.973 ms       |
| `notification_store.synthetic_5k.notification_store_5k_load_snapshot` | 18.702 ms; confirmation 19.198 ms | 18.466 ms        |

Previous reports for runs `35167160896`, `35195996130`, and `35227620108` passed these
anchors at respectively 192.238/10.087 ms, 262.853/17.258 ms, and 267.209/18.322 ms. No
changes to the Python scan/notification paths or benchmark files occurred between
`88175f34` and `14403c15`; the corresponding core pin change was in gate-decision code.
Hosted-runner variation is plausible, but the notification confirmation failed too, so
this is not established as a transient flake.

A focused local measurement using the current core and Python 3.14 yielded 281.154 ms
for the full scan (8 samples after 2 warmups) and 20.665 ms for cached 5k notification
reads (5 samples after 1 warmup). Profiling, separately from those timings, identified:

- `agent_scan_wire_from_dict` / `_record_from_dict` consume most of the Python portion
  of the scan; `_agent_meta_from_dict`, `_done_marker_from_dict`, and
  `family_shell_from_mapping` perform repeated mapping projections and legacy
  conversions. The Rust call itself is substantial but is not the entire cost.
- Cached notification reads still clone 4,736 surviving records with
  `dataclasses.replace`; that accounts for almost all profiled snapshot load time. Fresh
  top-level objects are an intentional API guarantee and must remain so.

These profiles identify implementation targets, not a license to change behavioral
contracts or enlarge thresholds. Reproduce with CI's Python 3.12 and the selected
release wheel before accepting performance conclusions.

## Phase core-contract

1. Recheck the current SASE revision, `actstat`, source pin, package requirement, and
   target release. Use `/sase_repo` to inspect `sase-core` if needed.
2. Update `sase-core-revision.txt` to the full verified `0.34.47` release SHA above.
   Update the package floor to `sase-core-rs>=0.34.47,<0.35.0` and refresh only the
   necessary `uv.lock` entries with the existing `tools/ratchet_core_window` workflow.
   Its exit status 2 means a ratchet is pending or applied, not an operational error.
   Inspect its proposed target if a newer release has appeared; keep the pin and package
   floor on a jointly verified release.
3. Preserve the source pin, wheel provenance, and required-binding checks. The current
   workflows are exposing a real contract mismatch; changing them to use floating HEAD
   or suppressing missing-binding failures is not the repair.
4. Validate the exact release wheel in an isolated environment, including the binding
   checker and `tools/validate_sase_core_rs`. Also validate the source-built wheel for
   the selected SHA through CI or an equivalent local build. A development wheel
   reporting the right version alone is insufficient evidence.
5. Run the affected modules together:

   ```text
   tests/llm_provider/test_claude_usage.py
   tests/llm_provider/test_claude_fable_usage_identity.py
   tests/main/test_repo_handler_open_configured.py
   tests/test_axe_config_backend.py
   tests/test_check_sase_core_rs_bindings_tool.py
   tests/service/test_service_state.py
   tests/service/test_service_restart.py
   tests/service/test_service_config.py
   tests/agents_sync/test_publication_manifest.py
   tests/agents_sync/test_publication_reconciliation.py
   tests/agents_sync/test_inventory_history.py
   ```

The release-only `release-core-floor-smoke` job in `ci.yml` installs the exact declared
minimum. Follow its smoke/contract procedure so the package floor is proven as well as
the source pin. No new domain implementation or tests that merely assert a hardcoded
version are needed for this phase.

## Phase visual-contracts

1. Use `just install-visual` and verify the committed renderer fingerprint. Start with
   the five locally reproduced failures, explicitly selecting `-m visual` when invoking
   pytest directly; the default pytest marker expression excludes them even when
   individual node IDs are supplied.
2. Fix SVG content assertion handling through parsed text/XML entity decoding,
   preferably reusing the existing helpers in
   `tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py`. Keep assertions about
   rendered content meaningful across styling spans and escaped apostrophes. Add focused
   helper coverage if changing shared parsing behavior.
3. In `test_ace_png_snapshots_agents_modals.py`, provide a valid canonical tribe name
   for the wait-modal fixture. In `test_ace_png_snapshots_agents_panels.py` and
   `test_ace_png_snapshots_agents_tribe_panel.py`, update user-visible job labels and
   related assertions to the current public names. Preserve internal legacy
   storage/group keys where the implementation still deliberately uses them.
4. Run the visual suite, review expected/actual/diff/source artifacts, and refresh
   goldens for intentional changes with `--sase-update-visual-snapshots`. The title
   rename explains the dominant cluster. Inspect the larger differences (including help,
   job/routine wording, indicators, and newer panels) against current behavior and
   history before accepting them. Correct test setup or product regressions where
   appropriate; do not bulk-accept unexplained differences.
5. Keep the pinned fonts, package versions, and current comparison tolerances. Retain
   legacy snapshot filenames where they serve compatibility. Run the full visual lane
   again without update mode and require it to pass.

Expected file scope is visual helpers, affected visual tests, and reviewed PNGs under
`tests/ace/tui/visual/snapshots/png/`. Product edits should be necessary to correct a
demonstrated regression, not to restore obsolete labels.

## Phase performance-floors

1. Reproduce both anchor workloads using Python 3.12 and the selected core release. Keep
   sample sizes, workload sizes, warmups, and report metadata identical to the existing
   harness. Record unprofiled timings separately from profiler output.
2. Reduce unnecessary Python adapter work in
   `src/sase/core/agent_scan_wire_conversion.py` and
   `src/sase/core/agent_scan_wire_family_shell.py`. Investigate repeated full mapping
   copies, projections, and absent-shell legacy-key scans first. Preserve optional
   defaults, unknown-field tolerance, nested/flat shell precedence, Patch aliases,
   queue-capacity aliases, clan/family handling, and schema checks. Use the existing
   wire and shell tests and add focused regression cases for any new fast path; ensure
   source dictionaries are not mutated.
3. Reduce clone overhead in `src/sase/notifications/store.py` while preserving its
   documented fresh-top-level-object guarantee on both cache hits and misses. Compare an
   appropriate shallow-copy implementation with `dataclasses.replace` for this concrete
   dataclass rather than guessing. Reuse one private clone path where possible. Test
   caller mutation isolation, include-dismissed behavior, field preservation, and cache
   invalidation after file or in-process updates. Do not return cached mutable
   notification objects directly or change the existing nested-value semantics as an
   incidental optimization.
4. This phase targets binding adaptation and object copying. Shared domain changes, if
   unexpectedly necessary, belong in Rust core and require its full binding verification
   and an updated released pin/floor before final validation.
5. Run the relevant agent-scan wire/shell and notification-store/cache tests, plus
   `tests/perf/phase7/test_phase7_check_regression.py`. Run the full
   `just phase7-perf-check` after the optimization and retain its report. Both
   identified anchors must pass their existing ceilings with useful headroom; the other
   anchors and same-process Rust/Python comparisons must still pass. Do not weaken
   thresholds, skip scenarios, relabel the workload, or add unbounded retries to obtain
   green results. If hosted variance remains after optimization, document the evidence
   and seek a revised plan for any policy change.

## Phase integrated-verification

1. Reconcile all phase changes and recheck the exact selected core release, source SHA,
   `pyproject.toml`, and lockfile. Re-run the binding and contract probes after the
   final source changes, since they can introduce additional requirements.
2. Read `lint_and_test.md` through `/sase_memory_read`. Run `just fix` before monitored
   verification, inspect its diff, and run `just check` for the final changes. Resolve
   all new lint/type/format failures. Read the Symvision memory before addressing any
   Symvision diagnostic.
3. Run `just check-full` through `/sase_monitor` with the required `TESTING` / `TESTED`
   status pair. Run long visual, floor, or contract commands through the same monitor
   mechanism. The full visual suite and Phase 7E floor are explicit acceptance checks;
   do not assume the ordinary fast suite covers them.
4. Keep Master Gate's eight complete fast shards and the scheduled Full CI split.
   Preserve lint-job parity, required-binding checks, timeout/concurrency behavior,
   visual comparison rules, performance contracts, and the release-floor smoke.
5. Refresh `actstat` and inspect every failed or newly completed job relevant to the
   repaired revision. The scheduled run was still in progress during planning, so its
   unfinished coverage, contention, and Python matrix jobs are unresolved evidence, not
   passing results. Resolve any additional failures in the repair's scope and compare
   failures against their exact tested SHA. Do not rerun old source and present it as
   validation of new changes.
6. Completion is host-owned: submit the repository declaration through `/sase_final`
   rather than manually creating commits, branches, or PRs. Once the host makes the
   changes available to GitHub, verify Master Gate and Full CI for the repaired tree
   through the authorized workflow. If remote execution is not yet possible, report the
   precise local evidence and outstanding remote verification rather than claiming
   Actions is green.

## Acceptance criteria

- Source-built and exact-minimum published core expose every binding required by the
  final Python tree; all previously failing Master Gate test modules pass.
- The visual suite passes without update mode, invalid tribe fixtures, stale public
  labels, or relaxed pixel tolerances; changed goldens have been reviewed.
- Both failing performance anchors and the remaining Phase 7E anchors pass with
  unchanged workload and threshold semantics; behavioral regression tests pass.
- The combined tree passes the required lint/scoped and exhaustive verification, and
  current Actions results are checked with clear attribution to their SHA.
- No test is skipped or weakened to hide the original failures, and no unrelated
  dependency, workflow, or durable-memory changes are included.
