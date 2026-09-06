---
tier: tale
title: Repair sase-telegram CI after the SASE plan-gate API removal
goal: Restore the sase-telegram GitHub Actions matrix by migrating its plan-gate integration
  tests to the current public SASE gate-construction API.
size: small
proposed_by: bbugyi200.athena.0gr
status: done
---

# Diagnosis

`actstat --repo sase-org/sase-telegram` identifies CI run 147 on commit `9cc66ab` as the
only failing workflow. The Python 3.13 `Run tests` step stops during collection because
`tests/test_custom_gates.py` imports `create_plan_approval_gate` from `sase.plan_gate`;
the Python 3.12 matrix job is then cancelled by fail-fast. The publish workflow
succeeds, and the triggering sase-telegram commit does not touch these gate tests.

The CI workflow deliberately installs current checkouts of `sase` and `sase-core`.
Current SASE removed the `create_plan_approval_gate` convenience wrapper while retaining
`build_plan_approval_gate_spec` as the public spec builder. SASE's own tests now create
plan gates by passing that spec to `sase.notification_gates.service.create_gate`. An
in-memory compatibility probe using that exact composition collected and passed all 32
tests in `tests/test_custom_gates.py` and `tests/test_gate_shell_settlement.py`,
confirming that the stale import and four call sites are the complete causal failure.

# Implementation

1. In the linked `sase-telegram` repository's `tests/test_custom_gates.py`, replace the
   removed `create_plan_approval_gate` import with `build_plan_approval_gate_spec`.
   Reuse the file's existing `create_gate` import to compose the supported API.
2. Update each of the four tale/epic test setup call sites to create its gate as
   `create_gate(build_plan_approval_gate_spec(plan_file, request_id))`. Preserve every
   existing request ID, fixture, notification, assertion, and production code path so
   this remains a compatibility repair rather than a behavior change.

# Verification

1. Run the focused regression set for both the directly changed module and its
   fixture-dependent consumer:
   `just test tests/test_custom_gates.py tests/test_gate_shell_settlement.py`.
2. Run the repository's complete required local gate with `just check`, covering Ruff,
   mypy, and the full pytest suite.
3. Confirm the worktree contains only the intended test migration and review the diff
   for accidental fixture or runtime changes. The host's resulting GitHub Actions run
   should then exercise the same suite on Python 3.12 and 3.13; both matrix jobs must
   pass rather than one failing and the other being cancelled.
