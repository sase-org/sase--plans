---
tier: tale
title: Repair the just symvision setup syntax regression
goal:
  The setup validator and its contract test parse correctly, and just symvision passes
  without changing finalizer behavior.
size: small
proposed_by: bbugyi200.athena.0cg
create_time: 2026-08-24 11:11:22
status: wip
---

# Repair the `just symvision` setup syntax regression

## Objective

Restore the Python syntax and formatting accidentally damaged in the latest
finalizer-wire change so that `_setup` can validate `sase_core_rs` and `just symvision`
can reach and run the Symvision linter. Preserve the newly adopted finalizer wire schema
and deferral behavior; this is a repair to malformed source layout, not a rollback or
behavior change.

## Context

`just symvision` delegates to `_lint-symvision`, which depends on `_setup`. During
setup, `tools/validate_test_environment` dispatches `tools/validate_sase_core_rs`; that
script currently fails to parse at line 327 before Symvision starts. Commit `570b6be4b`
joined seven formerly separate line pairs in `_validate_finalizer_contract`, including
the conditional body that causes the reported `IndentationError`. The same commit joined
five line pairs in the dedicated finalizer-contract test, leaving that test module
unparsable as well. The worktree is otherwise clean, and the diff against the parent
shows no intended semantic changes in either damaged region.

## Implementation

1. Repair `tools/validate_sase_core_rs` within `_validate_finalizer_contract` by
   restoring the accidentally removed newlines and normal indentation around the
   stale-schema error, finalizer instance and plan dictionaries, provider specification,
   resolved-plan guard, context, and submission payload. Keep all current constants,
   values, calls, and validation semantics intact.
2. Repair `tests/test_validate_sase_core_rs_contracts_tool.py` within
   `test_validate_sase_core_rs_requires_expected_finalizer_schema` by restoring the test
   function body, dictionary entries, and stale-schema mutation/assertion to separate,
   correctly indented statements. Retain the existing coverage for accepted, stale, and
   ahead finalizer schema versions.
3. Review the resulting diff against `HEAD` and the parent-side formatting to confirm
   the patch contains only the intended source-layout restoration. Do not add a feature
   flag: this repair restores an existing development command and does not introduce
   unfinished user-reaching behavior.

## Verification

1. Run `just install` first, as required for an ephemeral workspace, and confirm
   `_setup` can import and execute `tools/validate_sase_core_rs` successfully.
2. Run the focused contract test:
   `.venv/bin/pytest -q tests/test_validate_sase_core_rs_contracts_tool.py::test_validate_sase_core_rs_requires_expected_finalizer_schema`.
3. Run `just _lint-symvision` to exercise the exact private lint stage, then run
   `just symvision` to verify the reported public command completes successfully.
4. Run the required repository gate, `just check`. If it escalates or reports unusual
   test selection, run `just check-full` through `/sase_monitor` with the required
   `TESTING` and `TESTED` statuses and act on the result.

## Acceptance Criteria

- Both repaired Python files parse and satisfy the repository formatter and linter.
- The finalizer contract probe still accepts the expected schema and rejects stale and
  ahead schemas.
- `just symvision` reaches Symvision and exits successfully.
- `just check` passes, with any required escalation completed successfully.
