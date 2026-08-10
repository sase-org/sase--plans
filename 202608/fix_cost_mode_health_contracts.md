---
tier: tale
title: Repair pytest-runner recorder contracts
goal:
  Cost-mode and isolation-mode tests accurately enforce the runner's intended recorder
  behavior in every parent lane.
size: small
proposed_by: bbugyi200.athena.sase-iq
create_time: 2026-08-10 10:14:51
status: wip
---

# Repair pytest-runner recorder contracts

## Context

`tools/run_pytest` intentionally includes `cost` in `HEALTH_RECORDING_MODES` because
`just check-full` uses the cost lane and its failures must contribute to selection
health. The runner still excludes cost mode from timing recording because the cost probe
would distort timing data. Two contracts in `tests/test_run_pytest_main.py` predate that
behavior:

- `test_main_cost_mode_arms_only_the_cost_recorder` incorrectly rejects the health
  plugin even though the dedicated health test and the runner's documented mode set
  require it.
- `test_main_ace_page_group_isolation_uses_manifest_without_recorders` can inherit the
  enclosing cost lane's health request through the process environment. The test is
  intended to prove what isolation mode itself arms, so it must remove inherited
  recorder requests before invoking `main()`.

## Implementation

1. Update the cost-mode `main()` contract to require both the cost recorder and the
   health recorder, continue rejecting the timings recorder, capture the health request
   at the intercepted `execv`, and assert that its mode is `cost`.
2. In the ACE page-group isolation contract, explicitly delete all three recorder
   request environment variables before invoking `main()`. Retain its assertions that
   isolation mode loads no recorder plugins and creates no recorder requests.
3. Keep production runner behavior unchanged because it already implements the intended
   cost-lane selection-health policy.

## Verification

1. Run `just install` to refresh this workspace's editable development environment.
2. Run the two affected `tests/test_run_pytest_main.py` nodes in a normal focused pytest
   invocation.
3. Run the ACE page-group isolation node through `tools/run_pytest cost` to reproduce
   the inherited cost-lane environment and verify the test remains isolated there.
4. Run `just check` as the required repository gate; if it escalates or reports an
   unusual selection, run `just check-full` as instructed by the repository policy.
