---
tier: tale
title: Eliminate the full-lane config-cache poisoning flake
goal:
  The process-global config cache and its refresh worker cannot carry test-owned state
  across pytest nodes, all affected config tests retain their strict cache assertions,
  and the exhaustive parallel lane no longer reports the sase-mv failure class.
size: medium
proposed_by: bbugyi200.athena.sase-ns.2
bead: sase-ns.2
create_time: 2026-08-16 17:19:28
status: wip
---

- **PROMPT:**
  [prompts/202608/config_cache_parallel_flake.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/config_cache_parallel_flake.md)
- **PARENT:** [202608/top_task_bead_sweep.md](top_task_bead_sweep.md)
- **BEAD:**
  [sase-ns.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.2.md)

# Eliminate the full-lane config-cache poisoning flake

## Objective

Find and remove the cross-test state transition that intermittently contaminates
`tests/test_config.py` and `tests/test_config_cache.py` on one xdist worker. Preserve
the production stale-while-revalidate cache behavior and the strict identity, token,
overlay, and loader-call assertions that expose the defect. This plan implements only
phase bead `sase-ns.2`; it does not close the parent epic, ancestor plan beads, or the
standalone task bead `sase-mv`.

## Evidence and constraints

- The titled node and its now-documented sibling set pass in isolation and when the two
  config files run serially. Full parallel runs instead fail nondeterministic subsets of
  24 config nodes, with each run's failures clustered on one xdist worker and often
  exposing real host config inside a test-owned `CONFIG_DIR`.
- `sase.config.core` owns a process-global cached token plus a daemon refresh thread.
  `clear_config_cache()` advances an epoch and resets cached values, while the autouse
  `_clear_config_caches` fixture currently performs setup-only invalidation and does not
  establish or verify a per-test refresh-worker lifecycle.
- A full-run report can fail before or after
  `test_current_config_token_refresh_is_single_flight`, so the investigation must use
  worker-order evidence and active-thread/config-call instrumentation rather than
  assuming that named test is the sole poisoner.
- The global-state leak detector in `tests/_global_state_leak_detector.py` is the
  canonical diagnostic instrument. Extend or temporarily instrument its existing
  worker/order seam where necessary; do not add a competing detector.
- Do not relax `first is second`, exact loader-call counts, overlay provenance, or token
  equality. Do not change production caching behavior merely to make tests pass, and do
  not alter `tests/reproducible_flake_baseline.txt` to hide the flake.

## Implementation

1. Reproduce the failure with the existing full-lane leak detector enabled and retain
   its per-worker execution order. Correlate each first failing config node with its
   immediate predecessors, then minimize that ordering in one pytest process. Add
   focused diagnostics around `sase-config-token-refresh` ownership and calls into
   `current_config_token()` / `load_merged_config()` only as needed to distinguish an
   orphaned refresh worker from another long-lived test thread reading config between
   fixture reset and the node's own patches.
2. Turn the minimized poisoner-before-victim sequence into a deterministic regression
   that fails on the pre-fix tree. The regression must prove the observable contract:
   after one node releases or invalidates test-owned config state, the successor's first
   token, merged config, and owner snapshot are derived only from the successor's
   patched paths, and no prior daemon can publish into or unblock the successor's cache
   generation.
3. Fix the leak at its source. If a specific test or helper leaves a thread alive, make
   that owner release and join it on every success and failure path. If the shared
   autouse boundary is incomplete, make the config isolation fixture a yield-based
   lifecycle that orders setup after the per-test `CONFIG_DIR` redirect, waits for or
   safely invalidates any test-owned refresh worker, and clears derived config/identity
   caches again before monkeypatch teardown. Keep any production edit limited to a
   lifecycle primitive that preserves the existing stale-while-revalidate API and is
   independently justified by the demonstrated race.
4. Retain the original strict test and cover the broader victim class with focused
   repetitions of both config files, including randomized same-process ordering and a
   contention run. Confirm the global-state detector reports no config poisoning and
   that test cleanup leaves no live `sase-config-token-refresh` worker capable of
   observing restored host paths.
5. Run `just install`, the focused regressions, and `just check`. Because a root
   conftest/fixture change broadens selection, run `just check-full` through
   `/sase_monitor`, then run `just selection-health --fail-on-new-flake` and inspect
   `tests/reproducible_flake_baseline.txt` to verify no baseline suppression was added.
   Repeat the full parallel lane if the first green run does not provide enough evidence
   against the nondeterministic failure mode.
6. Record any related but unverified nodes only as `PROPOSED FOLLOW-UP:` notes on
   `sase-ns.2`. Close only `sase-ns.2` with a note naming the minimized reproduction,
   focused repetitions, full-lane result, selection-health result, and unchanged
   baseline; leave task-bead and ancestor lifecycle decisions to the epic land agent.
