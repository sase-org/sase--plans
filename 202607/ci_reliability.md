---
tier: tale
goal: Restore reliable SASE CI by eliminating environment-dependent test failures,
  waiting for complete TUI renders before visual comparison, and keeping the performance
  floor sensitive to sustained regressions without failing on isolated hosted-runner
  outliers.
create_time: 2026-07-15 13:47:29
status: wip
prompt: 202607/prompts/ci_reliability.md
---

# Plan: Restore reliable SASE CI

## Context and diagnosis

Recent `actstat` output and GitHub Actions logs show three distinct failure classes:

1. Git-backed tests are not fully hermetic. Temporary repositories that create commits do not always configure a local
   test identity, and `CommitWorkflow.resume()` tests that are meant to exercise tracking behavior can load this
   repository's real `commit_hooks.after: sase init -y` configuration. Depending on the runner environment and test
   ordering, commits fail with exit 128 or resume unexpectedly executes a hook whose command is not on `PATH`.
2. The tightened PNG comparator is exposing incomplete captures, not merely rejecting benign renderer drift. Downloaded
   CI artifacts contain screenshots where most of the TUI has not painted yet, while the committed expected image is
   complete. Some incomplete dark regions remain below the material-color threshold and are caught by the area limit;
   missing text and borders are correctly caught by the material-pixel limit. Broadly relaxing either visual threshold
   would hide real UI regressions.
3. The Phase 7 performance floor is intermittently sampling hosted-runner noise. Across adjacent runs of unchanged code,
   the 5k notification snapshot median ranged from 9.77 ms to 38.71 ms, with passing values around 13.79 ms and an
   isolated 18.98 ms failure against an 18.47 ms ceiling. A different run transiently moved the mark-all-read anchor
   from roughly 36 ms to 133 ms. The latest run passes, and the only recent Rust notification-store change affects agent
   identity matching during dismissal rather than snapshot loading. This is not evidence of a sustained major
   regression, but the three-sample floor is too vulnerable to whole-run contention.

## Implementation

1. Start from the latest `origin/master` and preserve any unrelated work. Make Git-oriented tests self-contained:
   initialize a local non-signing test identity in every temporary repository before committing, and opt the commit
   resume suite into the existing shared `no_commit_hooks` fixture except in tests that explicitly verify hook behavior.
   Add or adjust focused assertions so a developer's project config, global Git identity, signing settings, and shell
   `PATH` cannot determine the outcome.

2. Strengthen the reusable ACE visual-idle barrier rather than weakening the comparator. After startup state,
   debouncers, workers, timers, and transient button state are settled, require the rendered screen to converge across
   consecutive event-loop/layout cycles before exporting the SVG. Keep the wait bounded with a diagnostic timeout. Cover
   the convergence behavior with focused tests, including a deliberately delayed paint/update, and use the shared
   barrier at capture points that can currently race modal or panel rendering. Inspect regenerated failure artifacts;
   only update PNG goldens if a complete, stable render demonstrates an intentional product-visible change.

3. Harden the Phase 7 floor against isolated host contention while retaining regression sensitivity. Increase the
   notification harness's sampling from the current three runs to a modest odd count and add a bounded confirmation
   measurement for absolute-only notification anchors that fail the first sample. Report both measurements and fail when
   the anchor remains over its ceiling on confirmation; do not retry or weaken same-process `must_beat_python` checks.
   Keep the global 1.40x factor unchanged. Raise a per-anchor factor only slightly, and only if repeated clean local and
   CI-style measurements show that the normal distribution—not an isolated outlier—needs the extra headroom; document
   the evidence beside that specific override.

4. Update focused test coverage and developer-facing comments for the resulting contracts: temporary Git repos own their
   identity, visual assertions capture a converged frame, and performance confirmation distinguishes sustained
   regressions from a single noisy sample. Avoid runtime behavior changes unless a targeted regression test proves they
   are necessary.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the affected Git and commit-resume tests under a clean temporary `HOME`, empty Git global config, and a `PATH`
   that does not expose an installed `sase` command; repeat them under pytest parallelism to catch ordering leaks.
3. Run the visual comparator unit tests and the affected ACE PNG snapshots repeatedly, then run the complete
   `just test-visual` lane. Verify every captured artifact is a fully painted UI and that the strict material-pixel gate
   still rejects a small high-contrast synthetic change.
4. Run the Phase 7 performance check repeatedly with the CI configuration. Compare all notification medians with the
   recorded baseline and confirm an injected sustained slowdown still fails both measurements while a single injected
   outlier does not pass as a regression.
5. Run `just check` to exercise formatting, lint, typing, unit, visual, and integration coverage. Re-run any failing
   focused lane after its fix, then re-run `just check` until the repository is clean.
6. Use `actstat` after the fix reaches GitHub Actions to verify the CI workflow completes successfully and inspect any
   remaining failed job rather than assuming a local pass covers runner-specific behavior.

## Risks and guardrails

- A visual wait based only on elapsed time will remain flaky; use observable render convergence and a bounded timeout.
- Accepting or regenerating incomplete PNGs would institutionalize the race; artifacts must be visually inspected before
  changing goldens.
- A broad performance-factor increase could mask real regressions; preserve the global factor and keep any exception
  anchor-local, small, and justified by repeated normal measurements.
- Shared test fixtures must not suppress dedicated hook tests or alter production commit-hook behavior.
