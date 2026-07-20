---
tier: tale
title: Top-offender test optimizations
goal: 'Reduce the post-harness wall time of the slowest audit, zoom-panel, keymap,
  and ACE PNG snapshot tests while preserving the exact collected test set, behavioral
  assertions, visual goldens, and production behavior.

  '
create_time: 2026-07-20 11:56:07
status: wip
prompt: 202607/prompts/top_offender_test_optimizations.md
---

# Plan: Top-offender test optimizations

## Context

The parent performance epic identified repeated repository scans, expensive TUI test setup, and visual rasterization as
the largest individual-test costs. The prerequisite ACE pilot-harness phase has now landed, so its fast startup policy
changes the cost profile of the zoom-panel, keymap, and visual tests. This phase must therefore measure the current
checkout before editing and use that post-harness baseline to select only demonstrable hot spots.

The relevant surfaces are the agent-artifact AST audit tests and their shared scanner, the Rust-binding scanner tests,
the modal-only zoom-panel suites, the AcePage keymap end-to-end suite, and the ACE PNG snapshot fixture and test
modules. These are test and test-support concerns; no application behavior or test selection needs to change.

## Guardrails

- Preserve every collected test, assertion, marker, and committed PNG golden. Do not skip tests, move tests out of the
  default lane, loosen PNG tolerances, or reduce the coverage gate.
- Optimize repeated setup and computation, not the behavior under test. Shared results must be immutable or freshly
  isolated where mutation could make tests order-dependent.
- Keep synthetic scanner regressions able to point at arbitrary temporary roots; any caching or fixtures for the real
  repository must not hide changes planted by those tests.
- Keep visual passing runs free of debug artifacts and retain exact local pixel equality. Failure reporting and
  intentional golden-update behavior must continue to work.
- Avoid production TUI changes unless profiling proves a test-support boundary is the actual bottleneck and the change
  is behavior-neutral. Keep the Rust core boundary untouched.

## Establish a comparable post-harness baseline

After refreshing the editable development environment, collect the targeted tests and record their node count. Run each
offender family serially with full duration reporting and consistent renderer/environment settings so file-level and
per-test wall times can be compared without xdist scheduling noise. Include the artifact audit modules, the Rust-binding
scanner module, both zoom-panel modules, the keymaps end-to-end module, and the visual snapshot suite. Use the results
to rank work and retain before/after totals for the bead notes or completion handoff.

## Collapse repeated repository parsing

Parse the Python source tree once per appropriate test-module or session boundary and share the resulting immutable
audit snapshot across assertions that currently invoke the same scanner repeatedly. Where several audit algorithms walk
the same files, introduce a narrowly scoped test helper that can reuse enumerated paths and parsed ASTs without changing
the public scanner behavior. Preserve explicit-root execution for the synthetic violation tests and make cache lifetime
and invalidation obvious.

Apply the same principle to the Rust-binding tool tests: load the tool once at a safe fixture scope and share one
real-source scan between assertions, while leaving temporary source-tree cases isolated. All existing assertions must
continue to evaluate the same names and diagnostics.

## Reduce redundant TUI setup in measured zoom and keymap offenders

Profile the post-harness zoom-panel and keymap files to distinguish app boot, message settling, file rendering, key
dispatch, and teardown costs. Consolidate compatible state transitions into one mounted app or modal only where the test
still proves every existing intermediate outcome. Prefer shared builders, seeded state, direct event-driven waits, and
grouped scenarios over repeated equivalent boots or unconditional pauses. Do not replace end-to-end key dispatch
assertions with direct action calls, and do not introduce sleep-based synchronization.

Where scenarios genuinely require clean application state or independent configuration, retain separate contexts. Re-run
the affected files repeatedly to detect leaked state, order dependence, or cleanup regressions.

## Optimize verified PNG snapshot hot spots

Use the fresh durations table to focus on visual files or helper operations with measured cost. Reuse a single AcePage
boot for related snapshots when the existing scenario can transition deterministically between asserted states, and
avoid duplicate SVG export, rasterization, or image decoding within one assertion path. If helper-level reuse is
introduced, key it by complete deterministic inputs and preserve per-node failure metadata, exact comparison, update
mode, missing-golden handling, and mismatch artifacts.

Confirm that passing tests do not write visual artifacts and that representative failure paths remain covered. Do not
regenerate goldens for performance-only changes.

## Verification and performance evidence

Re-run the same serial targeted commands and node-count collection used for the baseline, recording before/after totals
and the remaining duration leaders. Require meaningful, repeatable improvement in each changed offender family; revert
speculative complexity that does not move measured time.

Run focused scanner, zoom-panel, keymap, PNG-diff, renderer-environment, and visual snapshot tests, including repeated
runs of the stateful TUI files. Then run `just check` as the repository gate. Verify the collected test set is
unchanged, no tracked visual goldens changed, no assertions or markers were weakened, and the worktree contains only
intentional test/test-support edits. Close `sase-86.3` with concise timing and validation notes after all checks pass,
leaving parent epic `sase-86` open.
