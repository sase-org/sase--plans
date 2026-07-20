---
tier: tale
title: Eliminate the pytest distribution tail
goal: 'The fast suite uses a measured, repeatedly validated xdist scheduling mode
  that reduces end-of-run stragglers without changing test selection or assertions,
  while retaining a documented one-variable fallback to loadfile.

  '
create_time: 2026-07-20 12:10:54
status: wip
prompt: 202607/prompts/distribution_scheduling.md
---

# Plan: Eliminate the pytest distribution tail

## Context

The host worker-token phase now grants a solo `tools/run_pytest` invocation up to 28 workers, but the runner still
hard-codes `--dist=loadfile`. That scheduler keeps every test file on one worker, so a few heavy files can strand work
after most workers become idle. Pytest-xdist 3.8.0 also provides `worksteal`, which redistributes pending tests from
workers with long queues, but it can execute tests from one file on different workers and therefore must be treated as
an ordering/isolation change.

The implementation must preserve every selected test and assertion, keep inline snapshot update modes serial, leave
host-global worker-token accounting unchanged, and provide an immediate `loadfile` fallback through `SASE_PYTEST_DIST`.

## Implementation

1. Establish comparable loadfile and worksteal baselines at the token-governed worker count. Record pytest wall time,
   the end-of-run tail, worker utilization, and slowest files/tests using the same fast-suite selection and otherwise
   equivalent environment. Use the results to confirm that worksteal materially improves distribution rather than merely
   shifting fixed overhead.
2. Audit the suite for assumptions that loadfile currently masks, including module-level mutable state, session/module
   fixtures with worker-local side effects, explicit worker identity handling, shared non-temporary resources, and tests
   whose correctness depends on another test in the same file running first. If repeated worksteal execution exposes
   such a dependency, repair the isolation without weakening coverage; if that cannot be done safely in this phase,
   retain loadfile and split only the measured straggler test files along existing helper boundaries, moving tests and
   assertions verbatim.
3. When worksteal passes the safety and performance bar, make it the runner default and resolve `SASE_PYTEST_DIST` as
   the distribution-mode override. Validate the override early with an actionable pytest usage error, keep `loadfile` as
   the supported fallback, preserve serial inline-snapshot behavior, and allow normal caller selectors/options to flow
   through unchanged.
4. Extend the runner unit tests to cover the worksteal default, the loadfile environment fallback, invalid override
   diagnostics, command construction with governed worker grants, and the absence of xdist arguments in serial snapshot
   modes. Update development and configuration documentation with the new default, accepted override values, fallback
   example, and measured comparison.

## Validation

- Run the focused runner tests while iterating.
- Run the complete fast suite under worksteal at least three times and require all runs to pass, watching for
  order-dependent failures and recording comparable timings. Exercise `SASE_PYTEST_DIST=loadfile` once to prove the
  fallback command and suite path remain usable.
- Re-run the loadfile/worksteal timing comparison after implementation to confirm the tail and wall-time improvement at
  the same worker count.
- Run `just install` before repository checks, then run `just check` as the required final repository gate. Do not close
  `sase-86.4` unless these checks pass.

## Boundaries and handoff

This phase changes only Python-side test infrastructure, its tests, and its docs; it does not alter worker-token
capacity, test markers, coverage thresholds, visual goldens, or Rust core behavior. Close only child bead `sase-86.4`
after successful validation, leaving parent epic `sase-86` open for its remaining phases.
