---
tier: tale
title: Stabilize family-member relaunch prompt readiness
goal:
  Family-member relaunch tests wait for the complete prompt editor tree and remain
  reliable under serial and parallel test execution.
size: small
proposed_by: bbugyi200.athena.sase-ct
create_time: 2026-08-10 09:59:54
status: wip
---

# Plan: Stabilize family-member relaunch prompt readiness

## Context

Task bead `sase-ct` was reopened after
`tests/ace/tui/test_family_member_relaunch.py::test_completed_family_member_relaunch_dismisses_only_selected_child`
failed repeatedly in both serial and broader verification runs with Textual reporting
that `#frontmatter-raw` was not mounted. The relaunch prompt is resolved through
`schedule_relaunch_prompt_resolution`, which crosses an `asyncio.to_thread` worker
boundary. The test already uses `sase.ace.testing.wait_for`, but its success predicate
only observes the outer `PromptInputBar`. That parent can be queryable before Textual
has finished composing its nested `FrontmatterPanel`, so the waiter can return before
the UI state exercised by the following assertions is ready.

## Implementation

Update the completed- and running-member relaunch success paths in
`tests/ace/tui/test_family_member_relaunch.py` to wait for an observable descendant that
proves the prompt bar has completed the relevant nested composition, rather than
stopping when the outer bar alone is queryable. Keep the bounded shared `wait_for`
helper, the existing relaunch behavior assertions, and the cancellation/stale-row paths
unchanged. This is a test-synchronization fix; production relaunch behavior and widget
implementation should not change.

## Verification

1. Install the workspace's current development dependencies with `just install`.
2. Run the previously failing completed-family-member node repeatedly without pytest's
   random-order plugin to exercise the worker-to-mount boundary in serial execution.
3. Run the complete family-member relaunch test file, including the running-member,
   cancellation, and stale-row cases.
4. Run `just check` as the required repository-wide lint and diff-scoped test gate.
5. Recheck the diff and working-tree status, then close `sase-ct` with a note recording
   the exact focused, repeated, and repository-level verification results.
