---
tier: tale
title: Prevent feature-flag tests from overwriting host SASE config
goal:
  Feature-flag tests remain inside pytest sandboxes and the live SASE config is safely
  restored.
size: medium
proposed_by: bbugyi200.athena.0a3
create_time: 2026-08-21 19:30:21
status: wip
---

# Prevent feature-flag tests from overwriting the host SASE configuration

## Outcome

Make the feature-flag CLI journey tests write only inside pytest's per-test sandbox, add
a regression guard that fails before any host-path write can occur, and restore the live
SASE configuration to its pre-test state without changing the clean chezmoi source.

## Root cause and current state

- `tests/feature_flags/test_cli_journeys.py` imports `CONFIG_DIR` directly from
  `sase.config.core` at module collection time. At that point the value is the real
  `~/.config/sase` directory.
- The autouse `_isolate_sase_home` fixture later monkeypatches
  `sase.config.core.CONFIG_DIR` and `sase.config.targets.CONFIG_DIR`, but Python's
  copied module-level `CONFIG_DIR` binding in the test module is unchanged.
- `_seed_portable_config()` therefore writes the host files before exercising the CLI:
  it replaces `~/.config/sase/sase.yml` with exactly
  `feature_flags:\n  ref_sync_gesture: true\n` and writes `timezone: UTC\n` to
  `~/.config/sase/sase_extra.yml`.
- The reset times align with focused and full-suite executions of that new journey test.
  The birth time of `sase_extra.yml` aligns with the first focused test run, and its
  current bytes match the test fixture exactly. The later repeated mtime aligns with
  another run.
- The chezmoi source file itself was never changed. A scoped chezmoi update restored
  `sase.yml`, whose bytes now match `home/dot_config/sase/sase.yml`. The 14-byte
  `sase_extra.yml` has no chezmoi source and is the remaining test artifact.
- The persistent feature-flag implementation is not the writer: CLI and Config Flags
  mutations target `~/.sase/feature_flags.json`, and the Rust binding joins that fixed
  filename to SASE home.

## Implementation

1. Reconcile with the in-flight feature-flag polish work that owns
   `tests/feature_flags/test_cli_journeys.py`, so the fix is applied to the version that
   will land rather than creating a competing copy. Do not disturb unrelated changes
   from that work.
2. Replace the collection-time value import with a runtime lookup of
   `sase.config.core.CONFIG_DIR` after pytest fixtures have run. Prefer passing the
   resolved directory explicitly into the seeding helper so the write target is visible
   at the call site.
3. Before the helper creates or writes anything, assert that the resolved config
   directory is contained by the worker's `SASE_PYTEST_SANDBOX_DIR`. This must fail
   closed if the sandbox variable is absent, malformed, or does not contain the target.
4. Add narrow regression coverage for the collection-time trap: reject module-scope
   by-value imports of `CONFIG_DIR` in tests that seed config files, while continuing to
   allow runtime module lookup or imports performed inside a test after fixtures start.
   Keep the guard focused on host-write safety rather than broadly banning harmless
   config imports.
5. Recover the live machine state after the code guard is in place:
   - Reapply only `~/.config/sase/sase.yml` from chezmoi and verify its bytes match the
     opened canonical source with no chezmoi diff.
   - Recheck that `~/.config/sase/sase_extra.yml` is still the 14-byte test payload and
     retains the test-correlated birth/mtime evidence. If so, move it to a timestamped
     recovery location outside `~/.config/sase` so the intended prior absence is
     restored and the artifact remains recoverable. If its bytes or metadata have
     changed, preserve it in place and report the conflict instead of guessing.
   - Audit the rest of `~/.config/sase` for fixture payloads introduced during the same
     interval; recover only files with equally conclusive provenance.
6. Do not modify the chezmoi repository: its canonical `sase.yml` is already clean and
   is the recovery source, not the cause.

## Verification

1. Record byte hashes, sizes, and mtimes for the live `sase.yml` and the recovery state
   of `sase_extra.yml`.
2. Run the CLI journey test serially, then under the repository's parallel pytest lane.
   Confirm the sandbox assertion passes and all feature-flag journey assertions still
   exercise real config/state behavior inside temporary roots.
3. Compare the live-file fingerprints after each test run. `sase.yml` must remain byte
   identical with an unchanged mtime, and no `sase_extra.yml` may reappear.
4. Run the related feature-flag state, CLI-set, CLI-journey, and Config Flags pane
   journey tests to ensure persistent state still uses `~/.sase/feature_flags.json` and
   neither CLI nor TUI mutations touch YAML config layers.
5. Run `just install` before repository verification if the workspace environment is
   stale, then run `just check` as required for SASE file changes. Report any unrelated
   pre-existing gate failure separately; do not weaken the new host-path safety checks.
6. Finish with a scoped chezmoi diff/status check and a final live-file fingerprint
   comparison, demonstrating that repeated execution can no longer reset the managed
   file.

## Safety and rollback

- Never run an unpatched copy of the leaking journey test while a real host config path
  is reachable.
- Preserve unrelated dirty work in both SASE and chezmoi checkouts.
- The only live-file removal is the conclusively identified `sase_extra.yml` fixture
  artifact, and it is moved to a recovery location rather than destroyed. Restoring it
  is a single move if later evidence shows it was intentional.
- If the canonical chezmoi source and the live `sase.yml` diverge for any reason other
  than the known test payload, stop recovery and surface the diff rather than
  overwriting user changes.
