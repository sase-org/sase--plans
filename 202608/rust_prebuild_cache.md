---
tier: tale
title: Prebuild Rust artifacts off the interactive update path
goal:
  ACE prebuilds exact, stamped sase-core artifacts in an isolated background cache,
  confirmed dev updates install matching artifacts in seconds, and every miss safely
  falls back to the existing Rust build with full observability.
size: large
proposed_by: bbugyi200.athena.sase-i9.4
bead: sase-i9.4
create_time: 2026-08-09 13:01:07
status: wip
---

- **PARENT:**
  [202608/fast_dev_update.md](https://github.com/sase-org/sase--plans/blob/main/202608/fast_dev_update.md)
- **BEAD:**
  [sase-i9.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i9/sase-i9.4.md)

# Plan: Prebuild Rust artifacts off the interactive update path

## Goal

When ACE's existing update-status worker observes that the editable `sase-core` checkout
is strictly behind its upstream, it should start one low-priority background build
against an isolated mirror and cache a fully stamped extension/LSP artifact set. After
the user confirms the real update and the live checkout has fast-forwarded, the Rust
reconcile path should install that set in seconds only when every provenance and digest
check matches. Every disabled, absent, stale, corrupt, busy, or unhealthy cache case
must be journaled and must immediately fall back to the existing
`just rust-dev-install-uv-tool` path without weakening its health-check/repair behavior.

## Current state and constraints

- `src/sase/ace/tui/actions/update_toast.py` already performs startup and periodic
  update checks in a Textual worker thread with an overlap guard. The timer callback is
  intentionally thin, and the worker must remain non-blocking from the user's point of
  view.
- `UpdateStatus.components` identifies an editable core candidate with `role="core"`,
  its live `source_root`, and its `upstream_ref`; cached snapshots are locally
  revalidated with `classify_git_upstream`, so a surviving editable core row is strictly
  behind and not dirty/diverged.
- `src/sase/dev_update/execute.py` takes the fail-fast code-swap writer lock before
  fast-forwarding roots and running reconcile steps. Background work must never touch or
  lock the live checkout, and an interactive update must never wait for the prebuilder.
- The earlier epic phases shipped target-dir isolation rather than a true unified Cargo
  build: `just rust-dev-install-uv-tool` builds the PyO3 extension and LSP in separate
  `target/uv-tool-py` and `target/uv-tool-lsp` trees, using the `dev-update` profile by
  default and honoring `SASE_RUST_DEV_PROFILE`.
- The existing Rust health check, published-wheel repair command, pending-failure
  joining, stale-core restore, preview/blocker behavior, result receipt/toast, restart
  handshake, managed update path, `just install`, release profile, and CI must remain
  intact.
- The implementation should not require a `sase-core` source change: its current
  workspace profile, PyO3 `extension-module` feature, and maturin configuration already
  provide the build inputs. Open that linked repository through `sase repo open` for
  verification and keep it unmodified unless implementation exposes a concrete need.

## Implementation

### 1. Add a self-contained prebuild cache module

Create `src/sase/dev_update/prebuild.py` with pure/injectable helpers plus a private
module entry point for detached production and interactive consumption.

- Use `~/.sase/cache/rust-prebuild/` (resolved through the existing SASE path helpers)
  with these durable areas:
  - `sase-core/`: a dedicated shared/no-checkout clone sourced from the live checkout;
  - `sets/<key>/`: one immutable, per-provenance build set containing isolated PyO3 and
    LSP target directories, copied artifacts, and `stamp.json` written last;
  - a non-blocking `prebuild.lock` that is held for the full producer run so at most one
    process builds at a time.
- Resolve the observed upstream ref to an exact commit in the live repository, create or
  refresh the mirror from local objects only, and force the mirror's detached HEAD to
  that commit. Never fetch into, checkout, merge, reset, or write generated output under
  the live checkout.
- Before doing work, reclassify the live checkout and exit unless it is still strictly
  behind at the requested upstream commit. Also probe the code-swap writer state
  non-blockingly and exit when a real update is already executing. Do not take a shared
  code-swap lock for the build, because that would defer the interactive writer.
- Run producer subprocesses through the available `nice` and `ionice` wrappers. Build
  the abi3 PyO3 wheel with maturin and the LSP with cargo using the selected
  `SASE_RUST_DEV_PROFILE`, separate `CARGO_TARGET_DIR` values inside the set, and the
  existing cargo retry/multiplexing and PyO3 forward-compatibility environment. Extract
  `sase_core_rs.abi3.so` from the wheel and retain `sase-xprompt-lsp` as the two
  installable artifacts.
- Treat missing tools, git state changes, clone/build/extraction errors, and process
  contention as best-effort prebuild failures. Record a small atomic last-result
  diagnostic for debugging, but never surface them as an ACE failure.
- Stamp a completed set with a schema version, exact core commit, `Cargo.lock` SHA-256,
  `rustc --version`, cargo profile, resolved target interpreter, target Python ABI tag,
  artifact-relative paths, and artifact SHA-256 digests. Write the stamp atomically only
  after both artifacts are complete.
- Keep the newest two completed sets. Prune older set directories (including their
  per-set Cargo targets) after a successful production run; ignore incomplete temp
  directories when consuming and clean stale ones best-effort.

### 2. Schedule production from the existing ACE update worker

Extend `src/sase/ace/tui/actions/_update_toast_config.py` with a parsed
`prebuild_rust: bool = True` setting. Add `ace.updates.prebuild_rust: true`, with a
comment, to `src/sase/default_config.yml`; add the boolean to
`src/sase/config/sase.schema.json`, configuration docs, and schema/config parser tests.

In `src/sase/ace/tui/actions/update_toast.py`, after the worker obtains a successful
`UpdateStatus`, pass the editable core component and config to a scheduler helper in
`prebuild.py`. The scheduler should only launch a detached
`python -m sase.dev_update.prebuild produce ...` process and return; clone inspection
and every build command belong in the child. Preserve the current update-check overlap
guard and UI-thread marshaling. Make prebuilding independent of whether the indicator or
startup toast is enabled: checks may return early only when all enabled consumers,
including Rust prebuild, are disabled.

Tests must inject process launch and prove that startup/periodic checks remain guarded,
that no eligible core or `prebuild_rust: false` launches nothing, that a strictly-behind
editable core launches once per worker result, and that scheduling never waits for the
child.

### 3. Represent and consume the cache in the reconcile plan

Extend the dev-update models with:

- a `rust_prebuild_install` reconcile-step kind placed immediately before the existing
  `rust_dev_install` fallback; and
- a defaulted `DevRustPrebuildResult` (hit/miss plus reason) on `DevUpdateResult`, so
  existing constructors remain source-compatible.

Update `src/sase/dev_update/plan.py` to retain the editable core record/path (including
host-sibling/SASE_CORE_DIR resolution for stale-core restores and Python-only rebuilds)
and emit an available prebuild-install command using the same target interpreter and
profile as the normal Rust step. Keep the existing unavailable normal-build branch and
the existing health step immediately after the two alternatives.

The consume entry point should be deliberately fail-open:

1. Load the merged `prebuild_rust` setting; report `disabled` when false.
2. Recompute post-merge commit, lockfile digest, rustc version, profile, interpreter,
   and ABI. Search only completed stamps and require an exact match for every field.
3. Validate both artifact digests before touching either destination.
4. Copy the extension into
   `crates/sase_core_py/python/sase_core_rs/sase_core_rs.abi3.so` and the LSP into the
   target venv's `bin/` with per-file temp + chmod + atomic replace semantics, then run
   the existing extension-purge tool so stale extension filenames cannot win import
   resolution.
5. Probe the copied extension with the target interpreter. If the copy or probe fails,
   report a miss and let execution run the normal build.

Use a stable machine-readable marker/exit convention between the command and executor.
Map all cache outcomes to one of the designed journal reasons: `stamp-missing`,
`commit-mismatch`, `lockfile-mismatch`, `rustc-mismatch`, `profile-mismatch`,
`abi-mismatch`, `interpreter-mismatch`, `digest-mismatch`, `copy-failure`,
`health-check-failed`, or `disabled`. A zero exit denotes a verified hit; every other
outcome is a cache miss, not an update failure.

Update `src/sase/dev_update/execute.py` so a verified prebuild hit skips only the paired
`rust_dev_install` command, while any miss runs it exactly as today. The existing
`rust_health_check` then runs once regardless of which install path won, preserving its
repair and pending-failure semantics. Thread the `DevRustPrebuildResult` through every
success, failure, and early-return result.

### 4. Add observability without changing receipts or restart behavior

- Bump the dev-update journal schema and serialize the prebuild hit/miss plus exact
  reason in `src/sase/dev_update/journal.py`; leave older records readable and adjust
  `tools/dev_update_timings` wording to accept schema 2+ records.
- Include the same object in `sase update --json` output.
- Add one compact line such as `rust prebuild: hit` or
  `rust prebuild: miss (commit-mismatch)` to the CLI result panel and the ACE tracked
  update task's existing result-log summary. Do not add a new toast section or alter the
  post-update receipt contents, diffstat, or commit groups.
- Document the cache location, default two-set retention, exact-match contract,
  `ace.updates.prebuild_rust: false` escape hatch, and safe cache-clearing procedure in
  `docs/rust_backend.md` and `docs/configuration.md`.

## Tests

Add focused tests, with all subprocesses and process launches injected so pytest never
invokes git, cargo, rustc, maturin, nice, or ionice:

1. `tests/dev_update/test_prebuild.py`
   - mirror creation/update targets the observed upstream commit and never issues a
     mutating command against the live checkout;
   - the producer lock refuses overlapping builds and the code-swap writer probe makes a
     producer a no-op during a real update;
   - successful production writes the full stamp last, hashes both artifacts, and prunes
     to two completed sets;
   - a match succeeds, while changing exactly one of commit, `Cargo.lock`, rustc,
     profile, interpreter, ABI, or either digest yields the corresponding miss;
   - missing/corrupt stamps, extraction/build failures, atomic copy failures, disabled
     config, and failed copied-extension probes degrade to a miss;
   - each destination uses a temporary file and atomic replace, and the executable bit
     is preserved for the LSP.
2. `tests/dev_update/test_plan.py` pins the prebuild-install -> normal-build -> health
   ordering, interpreter/core-root/profile arguments, stale-core restore, and existing
   unavailable-host behavior.
3. `tests/dev_update/test_execute_reconcile.py` pins hit-skips-build, every miss-runs-
   build, health-check execution after both paths, and unchanged build-failure repair
   behavior.
4. `tests/dev_update/test_journal.py`, main JSON/render tests, and ACE update-summary
   tests assert the new structured outcome and one-line result log.
5. `tests/ace/tui/test_update_toast_automatic_checks.py`,
   `tests/ace/tui/test_update_toast_config.py`, and config-schema tests cover scheduling
   and the new setting without weakening the existing poller cadence/overlap behavior.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run focused Python tests for dev update, automatic update checks/config, update
   render/JSON/summary, and config schema.
3. Run `just check`; escalate to `just check-full` if scoped selection broadens or
   reports unusual coverage.
4. In the opened `sase-core` checkout, verify `cargo metadata --no-deps` and the
   existing `dev-update` profile/feature inputs without changing the checkout.
5. Exercise a real producer against an incoming core commit, inspect the stamp and
   artifact digests, then perform the confirmed dev update and verify:
   - the interactive reconcile logs `rust prebuild: hit`;
   - the normal Rust build command is absent;
   - the existing health check passes and both installed artifacts execute/import;
   - the update completes in seconds and the journal contains the hit.
6. Re-run with the cache removed, disabled, and deliberately invalidated one provenance
   field at a time. Verify every case records the exact miss reason and executes the
   unchanged normal build path without waiting, hanging, or mutating the live checkout
   in the background.

## Follow-up handling

Do not create task beads from this phase. Record any objective work deliberately left
out as `PROPOSED FOLLOW-UP:` on `sase-i9.4` for the epic land agent to triage. Do not
duplicate the already-recorded proposal about revisiting true single-Cargo-build
packaging.
