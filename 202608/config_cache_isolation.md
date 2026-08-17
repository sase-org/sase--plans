---
tier: tale
title: Isolate merged-config cache publication from cross-test background work
goal:
  The merged-config cache and its owner-snapshot dependency cannot be replaced by a
  background caller while a test-owned config source and clock are active, and the
  config-cache flake class no longer exceeds the reproducible-flake baseline.
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.6.6.1
bead: sase-ns.6.6.6.1
create_time: 2026-08-17 06:00:35
status: wip
---

- **PARENT:**
  [202608/backlog_top_five_gates_and_flakes.md](backlog_top_five_gates_and_flakes.md)
- **BEAD:**
  [sase-ns.6.6.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.6.1.md)

# Isolate merged-config cache publication from cross-test background work

## Goal

Fix the process-global cache race behind
`tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`
and its sibling config-cache flakes. Preserve production stale-while-revalidate
semantics: an expired read returns the current merged object, one background token
refresh computes the new filesystem state, and a later read publishes a rebuilt
configuration. Do not mask the defect with retries, sleeps, or a permanent flake
baseline entry.

This is a `medium` tale because the remaining work is substantial but bounded to the
configuration cache and its test isolation. One agent can diagnose the transient
cross-thread mutation, implement the synchronization or lifecycle fix, add the focused
regressions, and run the required repository gates coherently.

## Current behavior and evidence

- The failing test patches `sase.config.core.CONFIG_DIR`, `Path.cwd`, and
  `time.monotonic`, loads one merged object, changes the machine selector, advances the
  patched clock beyond the refresh interval, and requires the triggering read to return
  that same object before a later read observes the refresh.
- The newest clean full-lane record at `b6246f1cf` fails the triggering identity
  assertion even though the node and its file pass in isolation and under three
  file-scoped contention repeats. This requires another caller to replace the
  process-global merged cache during the victim's refresh window; the token-refresh
  worker alone does not rebuild merged configuration.
- Commit `3a22ff04f` already bound token state to the `CONFIG_DIR` object, added
  setup/teardown cache clearing, and drained the named token-refresh thread. Do not
  duplicate or revert that work.
- The existing whole-suite global-state report observes no persistent cache poisoning
  after test teardown. The remaining defect may therefore be a transient background
  caller that crosses a test boundary or a cache publication race that is cold again by
  the detector's post-protocol snapshot.
- `current_config_token()` and explicit cache clearing use
  `_current_config_token_cache_lock`, but merged-config and owner-snapshot cache
  comparisons, rebuilds, and publication currently occur outside that lock. Several TUI
  paths legitimately call `load_merged_config()` from Textual worker threads, so
  production cache behavior must be thread-safe independently of pytest scheduling.

## Design and implementation

### Capture the actual competing caller

Add focused diagnostic/regression instrumentation in the test harness rather than
guessing from the node ID. Reproduce the victim with a controlled second caller that
attempts `load_merged_config()` while the selector refresh is in flight, and use the
global-state leak detector or a narrow temporary trace to identify any real suite test
whose worker survives into a successor test. Keep only durable assertions and reusable
test seams; remove temporary logging before verification.

If a specific test-owned worker is proven to survive teardown, fix that worker's
lifecycle at its owning fixture/app boundary and add a poisoner-then-victim regression.
If no worker survives and the controlled race alone reproduces the replacement, fix the
cache synchronization mechanism described below. If both contribute, address both
without broadening into unrelated TUI worker cleanup.

### Make cache generations publish atomically

Treat one config generation as a consistent unit across the token, merged document, and
selected-owner snapshot. Use the existing reentrant config-cache lock (or a small
generation/snapshot abstraction protected by it) so cache-hit checks, rebuild decisions,
and publication cannot interleave into a token/value pair assembled from different
generations. Preserve these invariants:

- The read that starts an expired-token refresh returns the prior merged object even if
  the refresh worker computes quickly.
- Only a caller that observes the published successor token can rebuild and publish the
  successor merged object and owner snapshot.
- A stale worker or explicit clear cannot overwrite a newer generation.
- Reentrant calls made while merging configuration, including owner snapshot and
  selected-overlay resolution, do not deadlock.
- Normal concurrent production callers coalesce around one coherent cached object;
  test-only `CONFIG_DIR` and clock patches cannot be observed by an unrelated caller
  after their owning test has completed.

If diagnosis instead proves the cache core already meets these invariants and the only
remaining cause is a leaked caller, leave production behavior unchanged and scope the
fix to the proven worker teardown. Do not add a broad global lock or test-specific
branch without a reproducing race.

### Regression coverage

Extend the focused config-cache tests to deterministically cover the discovered
interleaving. Assert object identity on the triggering stale read, eventual selector and
plugin/default-layer invalidation, owner-snapshot reuse before token change, and correct
successor values after publication. Retain coverage for explicit invalidation,
single-flight refresh, `CONFIG_DIR` rebinding, and drain epoch behavior.

If the fix changes the test harness or a TUI worker lifecycle, add the smallest
poisoner-then-victim test that fails on the old behavior and proves the worker is joined
or canceled before monkeypatch restoration. Avoid timing-only assertions: coordinate
threads with events and always release/join them in `finally` blocks.

## Verification

Run, in order:

1. The new deterministic regression and these three named nodes in isolation:
   `test_selector_change_eventually_invalidates_merged_config`,
   `test_owner_snapshot_reuses_parsed_overlay_until_token_changes`, and
   `test_load_merged_config_caches_plugin_layer`.
2. `SASE_CONTENTION_REPEAT=3 just test-contention tests/test_config_cache.py`.
3. A full parallel lane with the global-state leak detector enabled; this is the
   required proof because file-scoped contention already passes on the broken tree.
4. `just selection-health --fail-on-new-flake`. If only historical pre-fix records keep
   the config node red after a green full lane, add one narrowly scoped `# fixed-at:`
   directive following the baseline header convention and naming this phase and the
   verified fix point. Do not baseline post-fix failures.
5. `just install && just check`. Run any required `just check-full` only through the
   SASE monitor workflow with a follow-up action.

Inspect `git diff --check` and the final diff before closing only phase bead
`sase-ns.6.6.6.1` with a note that lists the verified nodes and gates. Record unrelated
or broader discoveries only as `PROPOSED FOLLOW-UP` notes on that phase bead.

## Acceptance criteria

- The reproducing cross-thread or cross-test interleaving is understood and covered by a
  deterministic regression.
- The triggering stale read cannot return a prematurely replaced merged object, and a
  subsequent read observes the selector/config change without retries or sleeps.
- The owner snapshot, default layer, and plugin layer retain their intended reuse and
  invalidation behavior.
- The named config-cache nodes pass in isolation, in the three-repeat contention lane,
  and in a full parallel lane.
- The reproducible-flake gate names at most failures owned by another active epic, and
  `just install && just check` passes.
