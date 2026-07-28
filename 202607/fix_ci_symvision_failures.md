---
tier: tale
title: Fix the SASE CI Symvision failures
goal:
  Restore deterministic SASE CI by correcting two invalid public APIs while preserving publication behavior and test
  coverage.
create_time: 2026-07-28 15:03:25
status: wip
---

- **PROMPT:** [202607/prompts/fix_ci_symvision_failures.md](prompts/fix_ci_symvision_failures.md)

# Fix the SASE CI Symvision failures

## Context and root cause

GitHub Actions CI runs `just lint`, whose Symvision stage rejects public Python symbols that have no production consumer
outside their defining module. Runs `30388332536` and `30389023609` fail for this reason; their matrix test and
visual-test jobs were cancelled after lint failed rather than reporting independent test failures.

The current deterministic failures reproduce locally with `just _lint-symvision`:

- `resolve_publication_project_key` is public and exported from `src/sase/agents_sync/commit_publication.py`, but its
  only production caller is in that same module. Its direct consumers outside the module are tests, which do not justify
  a public API under the project's Symvision rules.
- `drop_terminal_agent_publications` is public and exported from `src/sase/agents_sync/publication_outbox.py`, but it
  has no production caller at all; only a test imports it. Terminal requests are intentionally retained while active and
  quarantined queues exclude them, so deleting this unused removal API preserves the implemented retirement behavior.

A release-branch `bead-backend` job also failed once when the Rust core test
`command_helper_bridge_invokes_editor_vcs_repo_catalog` could not spawn its temporary helper. The immediately preceding
job passed against the same `sase-core` SHA, and 50 repeated local runs of both command-helper tests passed. Treat that
as a non-reproducible runner/process-spawn flake unless a newer run repeats it; do not broaden this fix or weaken Rust
coverage without new evidence.

## Implementation

1. Begin from the latest `master` and confirm that intervening changes have not already altered the two failing symbols.
2. In `src/sase/agents_sync/commit_publication.py`, make `resolve_publication_project_key` private:
   - rename it to `_resolve_publication_project_key`;
   - update its in-module production caller;
   - remove it from `__all__`; and
   - update the focused target-resolution tests to reference the private helper without changing their behavioral
     assertions.
3. In `src/sase/agents_sync/publication_outbox.py`, delete the genuinely unused `drop_terminal_agent_publications`
   helper and remove it from `__all__`. Update `tests/agents_sync/test_publication_outbox.py` to remove the test-only
   import and drop assertion. Keep the surrounding test focused on the production behavior: retrying quarantined
   publications must leave terminal entries retired while resetting the active quarantined entry. Rename the test if
   needed so its name matches the remaining assertions.
4. Keep the change limited to visibility/dead-code cleanup. Do not add a Symvision pragma, epic whitelist, or lint
   suppression, and do not change the persisted outbox schema or terminal-retirement semantics.

## Validation

1. Run `just install` before project checks, as required for an ephemeral SASE workspace.
2. Run the exact previously failing stage, `just _lint-symvision`, and require a clean exit.
3. Run the focused publication tests:
   - `tests/agents_sync/test_commit_publication_target_resolution.py`
   - `tests/agents_sync/test_publication_outbox.py`
4. Run the repository-mandated `just check` and fix/re-run until it passes.
5. Re-run `just _lint-symvision` after the full check to ensure later formatting or cleanup did not reintroduce an
   invalid public symbol.
6. Recheck GitHub Actions with `actstat`. Confirm the latest lint job no longer reports either symbol. If `bead-backend`
   repeats the same core helper-spawn failure, collect that new job's raw logs and treat it as a separately evidenced
   core issue rather than weakening or retrying unrelated SASE tests.

## Acceptance criteria

- Neither failing helper remains as an unused public API.
- Target-resolution behavior and terminal-publication retirement behavior retain focused test coverage.
- `just _lint-symvision` passes both before and after the full suite.
- `just check` passes.
- The latest actionable GitHub Actions status has no deterministic lint failure; any residual failure is reported with
  fresh job-specific evidence.
