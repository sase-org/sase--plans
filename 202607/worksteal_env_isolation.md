---
tier: tale
title: Isolate review-runner environment under work stealing
goal:
  Review-runner finalization tests leave no commit-workflow environment behind, so the
  default work-stealing pytest schedule passes without order-dependent commit CLI
  failures.
create_time: 2026-09-09 19:53:27
status: wip
---

# Plan: Isolate review-runner environment under work stealing

## Context

The newly landed default `--dist=worksteal` pytest scheduling exposed an existing
test-isolation bug in `tests/test_axe_review_runner_finalization.py`. Its
embedded-workflow stubs assign `SASE_COMMIT_METHOD=create_proposal` and
`SASE_ACTIVE_PROJECT_DIR` directly through `os.environ` after an absent-key
`monkeypatch.delenv`. Because pytest's monkeypatch fixture did not create those
assignments, teardown does not remove them. A later `test_commit_cli.py` test on the
same worker therefore observes `create_proposal` instead of the sanitized default and
fails. Running the review finalization module followed by the commit CLI module in one
process reproduces the same six failures; the commit CLI module passes in isolation.

This is test isolation introduced into the landing path by a concurrent base-branch
change. Production update behavior does not need to change.

## Environment-isolation fix

Make both review-runner embedded-workflow stubs publish their temporary commit method
and active project directory through the test's `monkeypatch` fixture rather than
assigning the process environment directly. Keep the tests' existing assertions that the
runner sees the published values during execution, while ensuring fixture teardown owns
and removes every newly introduced key.

Review the surrounding test module for any equivalent direct environment mutation in the
same flow and normalize it consistently. Do not weaken the commit CLI's environment
semantics or mask the failure by forcing `loadfile` scheduling.

## Regression coverage and verification

Add or refine a focused regression assertion if needed so the review-runner tests
demonstrate that temporary workflow environment cannot leak to a later same-process
consumer. Verify the previously failing order directly by running the review-runner
finalization module before `test_commit_cli.py`, then exercise those modules with work
stealing. Run `just install` before repository checks and finish with `just check` under
the default scheduler.

## Landing revalidation

Preserve the completed epic state: all `sase-83` children and the epic remain closed,
the linked epic plan retains `status: done`, and the post-close `just symvision` pass
remains clean. No new Symvision exemption should be introduced.
