---
tier: tale
title: Stabilize the deep-archive typing-burst fetch test
goal: 'The deep-archive plan-filter typing-burst test remains exact under the full
  xdist suite and performs only one deep-archive fetch, without weakening its
  single-fetch regression guarantee.

  '
create_time: 2026-07-28 20:15:00
status: wip
---

# Stabilize the deep-archive typing-burst fetch test

## Problem

The full pytest suite under xdist intermittently fails:

`tests/ace/tui/test_artifacts_plans_filtering.py::test_deep_archive_typing_burst_fetches_once_and_becomes_exact`

During the simulated `needle` typing burst, the test sometimes observes one extra deep-archive fetch: the final
`calls == [(("alpha", str(tmp_path)),)]` assertion sees two calls instead of the one coalesced fetch the test requires.
The same node passes serially and in isolation, so the failure appears to depend on full-suite scheduling or shared
contention rather than a deterministic mismatch in the fixture.

This test-level race is deliberately outside `sase-am`, whose CI redesign preserves completed-run signal and reduces
duplicated setup without changing individual flaky tests.

## Work

Reproduce the extra fetch under representative full-xdist contention and identify whether it comes from filter-change
debouncing, worker/result ordering, or test synchronization. Fix the production scheduling behavior if a real typing
burst can issue the duplicate fetch; otherwise make the test wait on a deterministic settled-state contract without
loosening the requirement that one burst performs exactly one deep-archive fetch and produces an exact result.

Keep the exact node ID and its final exactness/status assertions intact. Do not mask the race with retries or change the
expected call count to tolerate duplicate work.

## Validation

Run the exact node repeatedly in isolation, its containing test module, and a representative full-xdist suite or
contention reproducer. Finish with the repository-required `just install` and `just check`.
