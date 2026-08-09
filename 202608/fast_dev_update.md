---
tier: epic
title: Make dev-install SASE updates fast
goal: 'Pressing `,U` on a dev (editable) SASE install completes in seconds instead
  of minutes, with every existing safety check, blocker, fallback, journal record,
  toast, and restart behavior preserved.

  '
phases:
- id: timings
  title: Instrument dev-update step durations
  depends_on: []
  size: small
  description: 'timings: record per-command and per-step wall-clock durations in the
    dev-update journal, surface the slowest steps in the result log, and add a read-only
    analysis script so every later phase has a hard before/after baseline.'
- id: unified-build
  title: Build the Rust core and LSP in one feature-unified cargo invocation
  depends_on:
  - timings
  size: medium
  description: 'unified-build: collapse the two separate cargo/maturin reconcile steps
    into a single feature-unified build so sase_core and its shared dependencies compile
    once per update instead of twice, and so the two builds stop invalidating each
    other''s cached units.'
- id: fast-profile
  title: Add a fast dev-update cargo profile
  depends_on:
  - unified-build
  size: medium
  description: 'fast-profile: add a dev-update-only cargo profile in sase-core that
    drops LTO and codegen-units=1 in favor of incremental parallel codegen, wire the
    dev-update recipes to it with an escape hatch, and prove the published wheel/CI
    profile is untouched and runtime performance does not regress.'
- id: prebuild
  title: Prebuild Rust artifacts off the interactive path
  depends_on:
  - unified-build
  - fast-profile
  size: large
  description: 'prebuild: build the Rust artifacts in the background from a dedicated
    mirror clone as soon as the update poller sees incoming sase-core commits, then
    install the stamped prebuilt artifacts during the interactive update when every
    provenance field matches, falling back to a normal build on any mismatch.'
- id: verify
  title: End-to-end verification and documentation
  depends_on:
  - timings
  - unified-build
  - fast-profile
  - prebuild
  size: small
  description: 'verify: measure the real `,U` flow against the phase-one baseline,
    re-exercise every preserved blocker/fallback/restart path on the live dev install,
    confirm `just install` and CI are unaffected, and refresh the Rust backend docs.'
proposed_by: bbugyi200.athena.wj
create_time: 2026-08-09 10:09:32
status: wip
bead_id: sase-i9
---

- **PROMPT:** [prompts/202608/fast_dev_update.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fast_dev_update.md)
- **BEAD:** [sase-i9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i9/README.md)

# Plan: Make dev-install SASE updates fast

## Problem

On a dev (editable) install, `,U` runs the comprehensive update: agent CLIs, then the
SASE leg (`src/sase/dev_update/`), then cached agent hoods. The SASE leg fast-forwards
each editable checkout and then runs reconcile steps produced by
`src/sase/dev_update/plan.py::_reconcile_steps`:

1. `uv tool install ... --editable <sase> --with-editable <plugins>`
2. `just rust-install-uv-tool` → `maturin develop --release` in
   `../sase-core/crates/sase_core_py`
3. a Python health check that imports `sase_core_rs`
4. `just rust-lsp-install-uv-tool` → `cargo build --release -p sase_xprompt_lsp`

Steps 2 and 4 are the entire cost. Everything else is noise.

## Measured evidence

Numbers below come from the existing dev-update journal
(`~/.sase/logs/dev_update.jsonl`, 118 records) with the cargo `Finished ... in X` lines
parsed out of the captured stdout of steps 2 and 4.

| Quantity                                        | Measurement                                   |
| ----------------------------------------------- | --------------------------------------------- |
| Records with Rust reconcile steps               | 110                                           |
| Runs where `sase-core` advanced                 | 37 (34%)                                      |
| Runs where `sase-core` did **not** advance      | 73 (66%)                                      |
| Cargo seconds, `sase-core` advanced             | median **294 s**, mean 287 s, 36/37 over 60 s |
| Cargo seconds, `sase-core` unchanged            | median **0.2 s**, mean 21.8 s                 |
| Core-unchanged runs still paying >30 s of cargo | 12 (up to 330 s)                              |
| Cargo seconds per update, all runs              | mean **111 s**                                |
| `uv tool install` step                          | ~0.6 s (`Resolved 55ms`, `Prepared 472ms`)    |
| Runtime version inventory                       | 0.28 s cold, 0.12 s warm                      |
| `classify_git_upstream` per root                | ~0.03 s                                       |

A representative slow run (2026-08-09 09:35): maturin
`Finished release profile in 2m 54s`, LSP `Finished release profile in 2m 16s`. Both
lines are preceded by `Compiling sase_core` — **the same crate is compiled twice, from
scratch, in the same update**.

Two supporting facts:

- `../sase-core/Cargo.toml` sets `[profile.release] lto = "thin"` and
  `codegen-units = 1`. That is the right choice for published wheels and the wrong one
  for a rebuild a human is waiting on.
- Both builds share `../sase-core/target/`, yet `target/release/.fingerprint/` holds two
  distinct `sase_core-*` unit directories touched minutes apart by the two builds
  (09:53:38 and 09:56:38), with identical `rustflags`, `profile`, `config`, and `path`
  hashes. The units differ only through cargo's dependency-metadata hashing, which is
  consistent with the two invocations resolving different feature-unified dependency
  graphs (`maturin` selects `sase_core_py` with `pyo3/extension-module`; the LSP build
  selects `sase_xprompt_lsp`). The same divergence explains the 12 core-unchanged runs
  that still paid 30–330 s: each build periodically invalidates units the other had
  cached.

## What must NOT change

This work is purely about latency. Every one of these stays exactly as it is:

- The confirm-preview modal, its per-leg sections, incoming-commit ranges, and the
  snapshot-gated provider/agents legs.
- Every blocker in `plugins_browser_dev_update.py::dev_update_blocking_reason`: dirty
  checkout, no upstream, diverged, ahead-of-upstream, already-current, unavailable
  reconcile step, and the `code_swap_readers_active()` guard.
- The `code_swap_writer_lock()` guard around the merge and reconcile steps.
- The `rust_health_check` step, its `repair_command` fallback that restores a published
  `sase-core-rs` wheel, and the pending-failure joining in
  `execute.py::_run_reconcile_steps` / `_run_rust_health_check_step`.
- The stale-core restore path (`plan.py::_stale_core_plan`).
- The journal, the post-update receipt/toast (diffstat + commit list), and the ACE
  restart handshake.
- The managed (non-editable) `uv tool upgrade` path, `just install`, `just check-full`,
  CI, and published wheel builds.

## Non-goals

- Auto-merging the local `sase-core` checkout in the background. The checkout only ever
  advances through a user-confirmed update.
- Introducing `sccache` or any new external build dependency.
- Changing the `release` profile used by published wheels or CI.
- Skipping the reconcile work based on which paths a commit touched. Analysis of the
  last 40 `sase-core` commits shows 37 touch `crates/sase_core`, so path-based gating
  would buy almost nothing while adding a way to ship a stale extension. Cargo's own
  freshness check already handles the no-op case in ~0.2 s.

## Cross-repo note

Phases `unified-build`, `fast-profile`, and `prebuild` change files in the sibling
`sase-core` repo. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "..."`) and use only the path that command prints. Per the
Rust core backend boundary, the Rust wire/API and its tests are updated there; the
Justfile recipes, `src/sase/dev_update/` planning, and the Python tests are updated
here.

---

## Instrument dev-update step durations

Nothing in the current dev-update path records how long anything took. The journal
(`src/sase/dev_update/journal.py`) stores each command's argv, return code, and output
tails, but no clock. Every later phase in this epic is a performance claim, so the clock
comes first.

Work:

1. Add duration fields to the executed-command record. In
   `src/sase/dev_update/models.py`, give `DevExecutedCommand` a
   `duration_seconds: float = 0.0` field (defaulted so existing constructions and tests
   keep working). In `src/sase/dev_update/execute.py::_run`, wrap the
   `run(argv, cwd=cwd)` call in `time.monotonic()` and populate it. `_run` is the single
   choke point every fetch, status, merge, diffstat, log, and reconcile command already
   flows through, so no other call site needs touching.
2. Add a total to `DevUpdateResult` (`duration_seconds`), set in `execute_dev_update`
   around the whole body including the early-return paths.
3. Journal it: in `journal.py`, add `duration_seconds` to `_command_record`, add
   `result.duration_seconds`, and bump `schema_version` from `1` to `2`. Any reader must
   tolerate records missing the field.
4. Surface it where the user already looks. In
   `src/sase/ace/tui/modals/plugins_browser_sase_update_summary.py` (and
   `src/sase/ace/tui/actions/_update_toast_sections.py` if the toast renders step
   detail), add one compact line naming the slowest reconcile step and its duration,
   e.g. `slowest: Rebuild sase-core-rs into the uv-tool venv (4m54s)`. Reuse
   `sase.plugins.render_common.humanize_duration`. Do not add a new toast section;
   extend the existing result log.
5. Add `tools/dev_update_timings`, a read-only script over
   `~/.sase/logs/dev_update.jsonl` (and the rotated `.1`) that prints, per
   reconcile-step label: run count, median, mean, and max duration, split by whether
   `sase-core-rs` was `updated` or `skipped` in that run. Follow the conventions of the
   existing scripts in `tools/`. It must degrade gracefully on schema-1 records that
   have no durations.
6. Capture the baseline. Run `tools/dev_update_timings` and record its output in this
   phase's bead notes. That table is what `verify` compares against.

Tests: extend `tests/dev_update/test_execute.py` and `tests/dev_update/test_journal.py`
to assert durations are recorded and journaled, using an injected clock rather than real
sleeps.

Acceptance: `just check` passes; the journal gains `schema_version: 2` with per-command
durations; `tools/dev_update_timings` prints a baseline table; no change to which
commands run or in what order.

## Build the Rust core and LSP in one feature-unified cargo invocation

Today the update runs `maturin develop --release` for `sase_core_py` and then a separate
`cargo build --release -p sase_xprompt_lsp`. Each recompiles `sase_core`. Collapse them
into one cargo invocation whose feature-unified dependency graph both artifacts share.

Work in `sase-core` (open via `/sase_repo`):

1. In `crates/sase_core_py/Cargo.toml`, add a local passthrough feature:

   ```toml
   [features]
   extension-module = ["pyo3/extension-module"]
   ```

   Do not enable it by default — the existing comment in that file is explicit that
   plain `cargo test` must leave `extension-module` off so PyO3 conversion tests can
   link libpython. Keep that property.

2. In `crates/sase_core_py/pyproject.toml`, change
   `[tool.maturin] features = ["pyo3/extension-module"]` to
   `features = ["extension-module"]` so maturin selects the same local feature the
   combined build will select. Leave `strip = true` and every other maturin setting
   alone.
3. Confirm `cargo test --workspace` and `just rust-test` still behave.

Work in this repo:

4. Add `rust-dev-install VENV=venv_dir_abs` and `rust-dev-install-uv-tool` to the
   `Justfile`, modelled on the existing `rust-install` / `rust-install-uv-tool` pair.
   The new recipe must:
   - keep the existing guards verbatim: missing `sase_core_dir`, missing `cargo`,
     missing target-venv python, and the `tools/validate_sase_core_rs_version` check
     including the `SASE_ALLOW_STALE_CORE` escape and the exit-code-3 error text;
   - keep the on-demand `maturin` install and the `CARGO_NET_RETRY` /
     `CARGO_HTTP_MULTIPLEXING` hardening;
   - run exactly one cargo build covering both artifacts:
     `cargo build --release -p sase_core_py -p sase_xprompt_lsp --features sase_core_py/extension-module`
     from `{{ sase_core_dir }}`;
   - then run `maturin develop --release` (now hitting a fully fresh cache, so it only
     packages and installs), followed by the existing
     `tools/purge_sase_core_rs_extensions` marker cleanup;
   - then install the LSP binary using the same atomic `tmp` + `chmod +x` + rename dance
     `rust-lsp-install` already uses. Leave `rust-install`, `rust-install-uv-tool`,
     `rust-lsp-install`, and `rust-lsp-install-uv-tool` in place and unchanged:
     `just install`, CI, and `docs/rust_backend.md` all reference them.
5. In `src/sase/dev_update/plan.py::_reconcile_steps`, emit a single
   `kind="rust_dev_install"` step running `just rust-dev-install-uv-tool` in
   `host_record.source_root`, in place of the current `rust_install_uv_tool` and
   `rust_lsp_install` steps, keeping the `rust_health_check` step immediately after it.
   Preserve the unavailable-step branch and its
   `"host checkout source root unavailable"` reason.
6. In `src/sase/dev_update/execute.py`, make `_has_later_rust_health_check` and the
   pending-failure branch in `_run_reconcile_steps` treat `rust_dev_install` exactly as
   they treat `rust_install_uv_tool` today, so a failed build still defers to the health
   check and can be rescued by the wheel-restore repair command.
7. Update the recipe table and prose in `docs/rust_backend.md`.

Verification for this phase (this is a measurement, not a guess):

8. On the live dev install, run the combined cargo build, then run
   `maturin develop --release`, and confirm maturin reports `Finished ... in <1s` (no
   recompilation). Then run the whole `just rust-dev-install-uv-tool` recipe from a cold
   `cargo clean` and from a warm cache, and record both wall-clock times.
9. If maturin still recompiles — meaning the feature-unification hypothesis is wrong —
   fall back to plan B rather than shipping a wash: give each of the two existing builds
   its own `CARGO_TARGET_DIR` (for example `target/uv-tool-py` and `target/uv-tool-lsp`)
   so they can no longer invalidate each other's units, and note the disk cost. Plan B
   does not deduplicate the compile but does eliminate the 12 observed core-unchanged
   runs that paid 30–330 s. Report the measured numbers for whichever path ships, and
   say plainly which one it was.

Tests: update `tests/dev_update/test_plan.py` (reconcile-step shape) and
`tests/dev_update/test_execute_reconcile.py` (ordering, pending-failure deferral,
health-check repair) for the new step kind.

Acceptance: `just check` passes; one cargo invocation per update; a `sase-core` change
no longer compiles `sase_core` twice; measured cold and warm rebuild times recorded
against the `timings` baseline; every preserved behavior in the "What must NOT change"
list still holds.

## Add a fast dev-update cargo profile

`lto = "thin"` plus `codegen-units = 1` is tuned for the published wheel. A dev install
rebuilding on the user's own machine does not need it, and it is the single largest
remaining multiplier on the ~3-minute compile.

Work in `sase-core` (open via `/sase_repo`):

1. Add to the workspace `Cargo.toml`:

   ```toml
   [profile.dev-update]
   inherits = "release"
   lto = false
   codegen-units = 16
   incremental = true
   ```

   Leave `[profile.release]` exactly as it is.

Work in this repo:

2. Point the `rust-dev-install` / `rust-dev-install-uv-tool` recipes at the new profile:
   `cargo build --profile dev-update ...` and `maturin develop --profile dev-update`.
   `just install`, `rust-install`, `rust-lsp-install`, CI, and wheel builds keep using
   `release`.
3. Fix the artifact paths. Artifacts move from `target/release/` to
   `target/dev-update/`, so the LSP binary source path in the recipe must derive from
   the profile directory rather than hardcoding `release`. Verify
   `tools/purge_sase_core_rs_extensions` and `tools/validate_sase_core_rs_version` still
   behave.
4. Escape hatch: honor `SASE_RUST_DEV_PROFILE` (default `dev-update`, `release` forces
   the published profile) in the recipes, and thread it through the reconcile step's
   environment so a user can force a published-profile rebuild without editing the
   Justfile.
5. Measure, and be willing to back off:
   - cold and warm `just rust-dev-install-uv-tool` wall-clock, before vs after;
   - runtime: run the Rust-side benches/tests (`just rust-test`) and the TUI performance
     checks described in `docs/perf_runbook.md`, focusing on the query/parse hot paths
     `sase_core` owns. Read `sase/memory/tui_perf.md` through `/sase_memory_read` before
     touching anything perf-sensitive.
   - If a meaningful runtime regression shows up, step back to `lto = "thin"` with
     `codegen-units = 16` (keeping `incremental = true`) and re-measure. Report the
     numbers either way; do not ship an unmeasured profile change.
6. Update `docs/rust_backend.md` with the new profile, what it is for, and the escape
   hatch.

Acceptance: `just check` passes; dev-update rebuilds are measurably faster with no
meaningful runtime regression; `just install`, `just check-full`, CI, and wheel builds
still use `release` and are byte-for-byte unaffected in configuration.

## Prebuild Rust artifacts off the interactive path

Even after the previous two phases, a real `sase-core` change still costs a compile the
user is sitting through. But ACE already knows new commits exist long before `,U` is
pressed: the update poller runs on `ace.updates.check_interval_minutes` (default 10) and
`recompute_interval_minutes` (default 60), and `src/sase/updates/cache.py` already
classifies each editable root's upstream state. Move the compile into that window.

Design:

1. New module `src/sase/dev_update/prebuild.py`.
2. **Mirror clone.** Maintain a dedicated clone of `sase-core` under
   `~/.sase/cache/rust-prebuild/sase-core`, created from the local checkout
   (`git clone --shared` against the existing checkout, or a plain clone using the same
   remote) and kept at the sase-core upstream ref. The live checkout is never touched,
   never merged, never locked by the prebuild. Use its own `CARGO_TARGET_DIR` under the
   same cache directory.
3. **Trigger.** From the existing update-status poll path, when the poller observes the
   `sase-core` root is strictly behind upstream and `ace.updates.prebuild_rust` is
   enabled (add the key to `src/sase/default_config.yml` with a comment; default
   `true`), schedule one background prebuild. It runs at most one at a time, under
   `nice`/`ionice`, never blocks the poll, never blocks the TUI, and is a no-op if a
   real update is executing (take the existing advisory lock, or a dedicated prebuild
   lock that the real update path checks — whichever is cleaner given
   `code_swap_lock.py`'s fail-fast contract; do not make the interactive update wait on
   a prebuild).
4. **Stamp.** On success write a JSON stamp next to the artifacts recording: sase-core
   commit sha, `Cargo.lock` sha256, `rustc --version` string, cargo profile name, Python
   ABI tag / interpreter of the target venv, and the artifact paths with their sha256
   digests.
5. **Consume.** In the reconcile step for a dev update, before running
   `just rust-dev-install-uv-tool`, check whether a stamp matches the _post-merge_ state
   on **every** recorded field. On a full match, install by copying: the prebuilt
   `sase_core_rs.abi3.so` into `crates/sase_core_py/python/sase_core_rs/` (that is where
   the editable install's `.pth` resolves it) and `sase-xprompt-lsp` into the tool
   venv's `bin/`, both via the same atomic tmp + rename the Justfile already uses. Then
   run the existing `rust_health_check` step unchanged.
6. **Fallback is mandatory and silent-to-the-user-but-journaled.** Missing stamp, any
   mismatched field, a digest mismatch, a copy failure, or a failed health check falls
   straight through to the normal build step. A cold or broken cache must never make an
   update fail or hang — worst case it is exactly as slow as today.
7. **Observability.** Journal whether the prebuild cache hit or missed and why
   (`stamp-missing`, `commit-mismatch`, `lockfile-mismatch`, `rustc-mismatch`,
   `abi-mismatch`, `digest-mismatch`, `disabled`). Show a one-line note in the update
   result log.
8. **Bounded disk.** Keep at most N (default 2) stamped artifact sets and prune older
   ones plus their target directories. Document the cache location and how to clear it.

Tests: stamp match/mismatch matrix with one field wrong per case; copy-install
atomicity; fallback-to-build on every failure mode; disabled-by-config path; prebuild
refuses to run while an update holds the lock. Keep all subprocess work injected so the
suite never invokes cargo.

Acceptance: `just check` passes; with a warm prebuild, a `,U` that advances `sase-core`
finishes in seconds and the journal records a cache hit; with the cache cold, disabled,
or invalidated, behavior and timing match the previous phase exactly; the live checkout
is never mutated by background work.

## End-to-end verification and documentation

Close the epic against reality, not against unit tests.

1. Run `tools/dev_update_timings` on the post-change journal and compare with the
   baseline captured in the `timings` phase. Report: median cargo seconds for
   core-changed runs before vs after, core-unchanged runs before vs after, and the
   prebuild hit rate. State the numbers plainly, including anything that did not improve
   as much as hoped.
2. Exercise the preserved paths on the live dev install and confirm each still behaves:
   dirty checkout blocks; diverged checkout blocks; ahead-of-upstream blocks;
   already-current reports no-op; an active `sase bead work` reader blocks via the
   code-swap guard; a wheel-installed core triggers the stale-core restore; a
   deliberately broken extension triggers the health-check repair fallback; the
   post-update toast still shows diffstat and commit list; ACE still restarts.
3. Confirm the non-dev paths are untouched: `just install` from a clean venv,
   `just check-full`, and the managed `uv tool upgrade` update path.
4. Refresh `docs/rust_backend.md` and `docs/development.md` for the new recipes,
   profile, and prebuild cache, including how to disable the prebuild and where its
   cache lives.
5. File task beads through `/sase_new_task` for anything discovered and deliberately
   left undone.
