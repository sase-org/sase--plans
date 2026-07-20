---
tier: tale
title: Restore pytest environment isolation under work stealing
goal: Prevent review-runner tests from leaking commit-dispatch environment variables
  into later workers so the work-stealing full suite is deterministic and the sase-83
  landing remains fully validated.
create_time: 2026-07-20 13:25:56
status: wip
prompt: 202607/prompts/restore_test_env_isolation.md
---

# Plan: Restore pytest environment isolation under work stealing

## Context

The post-landing `just check` for epic `sase-83` began running pytest with the work-stealing scheduler introduced by
`8e544a398`. The update-feature tests remained green, but the full suite reproducibly produced six failures in
`tests/test_commit_cli.py`; that file passes alone and alongside the initially suspected review tests. The first failure
shows that `SASE_COMMIT_METHOD=create_proposal` unexpectedly survives into a commit-CLI test that expects the default
method.

The contaminator is in `tests/test_axe_review_runner_finalization.py`. Its setup removes agent environment variables
through `monkeypatch.delenv`, but two nested `expand_embedded` fakes later write `SASE_COMMIT_METHOD` and
`SASE_ACTIVE_PROJECT_DIR` directly through `os.environ`. Pytest therefore cannot restore those writes when the test
ends. The former loadfile scheduling happened to mask the leak; work stealing exposes the cross-test dependency.

## Phase 1: Make review-runner environment mutation reversible

Replace the direct process-environment writes in both affected `expand_embedded` fakes with pytest-monkeypatch-managed
updates. Cover every environment variable those fakes introduce, including `SASE_COMMIT_METHOD` and
`SASE_ACTIVE_PROJECT_DIR`, so test teardown restores the exact pre-test state. Keep the production workflow behavior and
the existing assertions unchanged; this is test isolation, not a change to commit dispatch semantics.

Audit the rest of `tests/test_axe_review_runner_finalization.py` for the same direct-mutation pattern and fix any
equivalent leak found in that file. Do not broaden the change into unrelated test cleanup.

## Phase 2: Lock in scheduler-independent behavior

Add or strengthen focused coverage that proves the review-runner tests leave the relevant environment keys unchanged
after completion and that the commit CLI still resolves its default method when no method is supplied. Exercise the
known ordering under pytest work stealing, not only each test file in isolation, so the regression would fail if a
future fake again mutates the shared worker environment outside monkeypatch control.

Run the focused review-runner and commit-CLI tests with the repository's parallel work-stealing configuration. Then run
`just check` on the integrated branch. Diagnose any remaining failures rather than weakening test selection or scheduler
settings.

## Phase 3: Reconcile and finish the sase-83 landing

After the fix and full validation pass, close `sase-83` idempotently and verify the epic and all three child beads are
closed. Run `just symvision` after the close so expired `sase-83` symbol exemptions cannot remain hidden; remove any
stale whitelist entries or unused code it reports and revalidate affected tests.

Resolve the epic plan through the configured plans sidecar and ensure `202607/agent_cli_update_awareness.md` has
`status: done` in its frontmatter. Preserve the already-landed done state if it is current rather than creating a no-op
edit. Finish with clean status checks in both the primary and plans repositories, committing only changes owned by this
follow-up work through the required SASE commit workflow.
