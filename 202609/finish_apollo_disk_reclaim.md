---
tier: tale
title:
  Finish sase-10r — safe runner-exit scratch cleanup and low-free-space pressure age
goal:
  "The sase-10r.2 behavior that never landed actually ships: a launched agent runner
  removes its own per-launch cargo-targets and agent-tmp directories at exit when no
  live process still uses them, the Rust-owned managed-tmp pressure pass uses a 1h
  minimum age whenever the free-space floor is breached, and the sase-10r epic is closed
  with honest bead records."
size: medium
proposed_by: bbugyi200.athena.0kk
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0kk](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kk.md)
- **COMMITS:**
  - [59dde52](https://github.com/sase-org/sase/commit/59dde523c37cbc0d629cdab41e82d89c986cdccb)
    — feat(axe): clean launch scratch on runner exit

# Finish the sase-10r epic: land the missing sase-10r.2 work, then close the epic

## Why this plan exists

`sase-10r` ("Reclaim apollo root disk and stop SASE build scratch from refilling it",
plan `plan:202609/apollo_disk_reclaim_1.md`) has three phases that are all marked
closed/done, but the epic is still in progress and **phase `sase-10r.2` never landed**:

- Phase 1 (`sase-10r.1`, apollo emergency reclaim) was operations-only and is genuinely
  done (apollo went from 9.3G to 110G free).
- Phase 3 (`sase-10r.3`, sibling pytest scratch reaping) landed as commit `df5fbbaccf`.
- Phase 2 (`sase-10r.2`, "Runner-exit scratch cleanup and low-free-space pressure
  reaping") closed at 2026-09-14 12:11Z with a "done" note, but master has **no** commit
  referencing `sase-10r.2`, no runner-exit cleanup, no
  `tests/test_run_agent_runner_scratch_cleanup.py`, and no low-free-space min age
  anywhere. The phase ran on another host (kellys_mbp) and edited the old Python
  implementation in `src/sase/core/managed_tmp_reaper.py`.

### The conflict to respect (sase-zw / sase-zw.8)

While `sase-10r.2` was running, `sase-zw.8.1` landed commit `6a60ee5fb7` (07:39 EDT)
which **replaced the Python managed-tmp reaper with a thin, schema-checked adapter over
the Rust owner** (`sase-core` `crates/sase_core/src/managed_tmp.rs`, binding
`sase_core_rs.reap_managed_tmpdir`) and made every horizon/pressure threshold
configurable under `managed_tmp` in `sase.yml`. The `_pressure_trigger`,
`DEFAULT_PRESSURE_MIN_AGE_SECONDS`-in-Python, and `_has_fresh_descendant` code that
`sase-10r.2`'s plan targeted no longer exists in Python, which is almost certainly why
that phase's changes never landed.

The `sase-zw` landing audit note (the note that spawned the `sase-zw.8` remaining-work
epic, `plan:202609/disk_footprint_remaining_work.md`) is explicit: shared backend
decisions belong in Rust, "Python duplicates the already-landed Rust scratch owner" was
a defect, and the Python side must "avoid a second pressure policy". `sase-zw.8` is
still active: phase `sase-zw.8.4` is running and **phase `sase-zw.8.5` ("pressure":
unify disk inventory, pressure decisions and owner delegation) is queued behind it and
will rework the same Rust pressure contract** (`pressure_plan`, thresholds, forwarded
policy) and the same `managed_tmp.pressure` config.

Therefore:

- The low-free-space min-age change MUST be implemented in the Rust owner, not in
  Python. Python only resolves a config value and forwards it on the wire.
- Keep the change small and additive so it merges cleanly with `sase-zw.8.5`, and leave
  coordination notes on the `sase-zw.8` beads (step 5) so that phase preserves it.
- Do not touch `sase-zw.8`'s scope (inventory, doctor thresholds, proc/run/object
  owners, the `disk_pressure` chop policy).

### Good news already in Rust

The Rust `pressure_plan` already reports `size_and_free_space` when both triggers fire,
so the Python "trigger-precedence trap" described in the original plan (size trigger
masking a breached free-space floor) is gone. The remaining gap is only the minimum age:
`reap_pressure_candidates` applies `pressure_min_age_seconds` (12h default) uniformly,
so during an emergency every same-day cargo target is still too young to reap.

### Safety finding that changes the original phase-2 design

Deleting the runner's `TMPDIR` / `CARGO_TARGET_DIR` unconditionally after
`finalize_runner_shutdown` (as the original plan specified) is **unsafe**:

- `/sase_monitor` handoff: `sase monitor start` runs inside the agent's Bash tool, so it
  inherits the runner env. `procs/spawn.py::_supervisor_env` and
  `procs/supervisor.py::_child_environment` copy `os.environ` and start the monitored
  command in a new session. The runner is then killed back to `finalize` and exits while
  the monitored command (typically `just check` / a cargo build) is still writing into
  `cargo-targets/<key>` and `agent-tmp/<key>`.
- Any `sase proc` started from inside the agent, provider background jobs (Claude Code
  `run_in_background` output lives under `$TMPDIR/claude-<uid>/...`), and detached file
  hooks inherit the same env and can outlive the runner.
- Successor agents (monitor/gate follow-ups, retries, continuations) get a fresh scratch
  key via `_managed_agent_scratch_env` with a new launch timestamp, so they are not at
  risk; the in-process `sase_pipe` successor finishes before `finalize`.
- Nothing inside `finalize_runner_shutdown` or after it in `main()` (only `sys.exit`
  plus the telemetry atexit flush) uses `TMPDIR`.

So the runner-exit cleanup needs an explicit liveness guard (step 2).

## Implementation

### 1. Reopen the falsely-closed phase

Run `sase bead open sase-10r.2`, then append a note with `sase bead note` stating: the
phase was closed as done but its changes never reached master (no commit references it;
`6a60ee5fb7` from `sase-zw.8.1` replaced the Python reaper it edited); it is being
completed by this plan with the pressure change in the Rust owner and a liveness-guarded
runner-exit cleanup.

### 2. Runner-exit scratch cleanup (sase repo, Python)

Launch side (`src/sase/agent/launch_spawn.py::_managed_agent_scratch_env`):

- Export the launch scratch key as a new env var (suggested `SASE_AGENT_SCRATCH_KEY`)
  next to `TMPDIR`/`CARGO_TARGET_DIR`. This lets the runner prove a directory is its own
  launch-assigned scratch rather than an `extra_env` override that happens to point at
  another child of a bucket.
- Check `src/sase/agent/env_hygiene.py` / `_remove_inherited_agent_identity_env`
  ordering: the variable must survive into the runner's `subprocess_env` (add or extend
  a launch-env test proving it), and it is fine (preferable) for proc/monitor spawn
  scrubbing to drop it. If the identity-family scrub would remove it before the runner
  starts, pick a name outside that family instead.

New helper module (keep `src/sase/axe/run_agent_runner.py` short; e.g.
`src/sase/axe/run_agent_runner_scratch.py`) with one public entry point, e.g.
`cleanup_launch_scratch(*, exec_outcome: str) -> None`:

- **Never raises.** Wrap everything; swallow `Exception` (not just `OSError`). Cleanup
  must never change the exit status or mask the original error.
- **Skip entirely** when `exec_outcome` is in `SHELL_HANDOFF_OUTCOMES`
  (`src/sase/core/dismissed_agent_completion.py`; monitor and gate handoffs), when the
  scratch-key env var is absent (runners launched before this change), or when the
  platform has no usable procfs (see liveness below). The reaper remains the backstop.
- **Candidates:** for `(bucket, env var)` in `("cargo-targets", "CARGO_TARGET_DIR")` and
  `("agent-tmp", "TMPDIR")`, the candidate is
  `managed_tmpdir_root() / bucket / <scratch key>`. Only consider it when the env var's
  value, normalized to an absolute path, is exactly that path (an overridden value means
  a user-supplied location: leave it alone). Require the candidate to be a **direct
  child** of the bucket, to exist as a real directory per `os.lstat` (not a symlink),
  and never touch the bucket itself or the managed root. Use
  `core.paths.managed_tmpdir_root` rather than `get_sase_managed_tmpdir`, which creates
  directories.
- **Liveness guard (Linux procfs):** before removing a candidate, scan `/proc/<pid>` for
  every pid other than `os.getpid()` whose `/proc/<pid>` is owned by the current uid. A
  candidate is live if any such process has, in `/proc/<pid>/environ` (NUL-separated),
  `TMPDIR`, `TMP`, `TEMP`, `CARGO_TARGET_DIR`, or `CARGO_BUILD_BUILD_DIR` equal to or
  under the candidate, or if its `/proc/<pid>/cwd` resolves under it. Tolerate
  `PermissionError`, `FileNotFoundError`, `ProcessLookupError`, and vanished pids per
  pid. If one candidate is live, keep both (a monitored build uses both). Make the proc
  root injectable for tests (compare `src/sase/axe/systemd_scope.py`'s `proc_root`
  parameter). If `/proc` is not a readable procfs (macOS), skip cleanup; do not guess.
- **Removal:** `shutil.rmtree` on the verified directory (no symlink following), with
  errors swallowed.

Runner wiring (`main()` in `src/sase/axe/run_agent_runner.py`): make the existing
`finally` run cleanup even if `finalize_runner_shutdown` raises, e.g.
`try: finalize_runner_shutdown(...) finally: cleanup_launch_scratch(exec_outcome=state.exec_outcome)`.
This also covers the user-kill `SystemExit` re-raise path and runs after every
in-process retry and pipe successor, since they share this process and env. SIGKILLed
runners keep relying on the reaper.

Tests (new file next to the other runner lifecycle tests, e.g.
`tests/test_run_agent_runner_scratch_cleanup.py`), using a `tmp_path` managed root and
fake proc root:

- launch-assigned `cargo-targets/<key>` and `agent-tmp/<key>` are removed at exit when
  no other process references them;
- both are kept when a fake process environ has `CARGO_TARGET_DIR` (or `TMPDIR`) under
  either one, and when a fake process cwd is under one;
- monitor and gate handoff outcomes skip cleanup;
- an overridden `CARGO_TARGET_DIR` / `TMPDIR` (outside the buckets, or a different child
  of the bucket), a symlinked candidate, and the bucket directories themselves survive;
- a missing scratch-key env var and a missing/unreadable proc root skip cleanup;
- an `rmtree` failure, and `finalize_runner_shutdown` raising, neither change the exit
  code nor hide the original error, and cleanup still runs in the latter case (drive
  `main()` with patched collaborators the way the existing runner tests do).

Check `tests/test_agent_artifact_directory_operation_audit.py` (it reviews
directory-deletion call sites) and register the new `rmtree` site if it requires that.

### 3. Low-free-space pressure min age (sase-core, Rust owner)

Open the core repo with `sase repo open sase-core` (fall back to
`sase repo open gh:sase-org/sase-core` if the linked slug fails; see task `sase-zo`) and
use only the printed path. Change `crates/sase_core/src/managed_tmp.rs`:

- Add an additive request field
  `#[serde(default)] pub pressure_low_free_space_min_age_seconds: Option<f64>` to
  `ManagedTmpReapRequestWire`. Keep `MANAGED_TMP_REAP_WIRE_SCHEMA_VERSION` at 1: an
  older binding ignores the unknown key (no relaxation, no crash) and older callers omit
  it, so neither side breaks and the published-floor install keeps working.
- Record whether the free-space floor is breached in `PressurePlan` (true for both
  `free_space` and `size_and_free_space`). In `reap_pressure_candidates`, compute the
  effective minimum age as `min(pressure_min_age_seconds, low_free_space_min_age)` when
  the floor is breached and the field is set, else `pressure_min_age_seconds`. Use that
  single cutoff both for candidate selection and for the `remove_if_stale`
  fresh-descendant recheck, so live build trees stay protected. Never let the low-space
  value lengthen the age.
- Add `#[serde(default)] pub pressure_effective_min_age_seconds: Option<f64>` to
  `ManagedTmpReapResultWire`: the cutoff age actually applied when a pressure plan ran,
  `None` otherwise. This makes the next emergency diagnosable from chop logs.
- Rust unit tests in the same module:
  - floor breached and root above `pressure_max_bytes` (trigger `size_and_free_space`,
    the exact apollo configuration): a 2h-old large `cargo-targets` entry with no fresh
    descendant is removed with low-space min age 1h and base 12h;
  - floor breached with only the `free_space` trigger: same entry removed;
  - ample free space under `size` pressure: the same 2h-old entry is kept, and removed
    once older than 12h;
  - a descendant modified within the last hour still protects the entry when the floor
    is breached;
  - field absent (`None`): behavior is unchanged (12h), proving compatibility;
  - the result reports the effective min age.
- If `crates/sase_core_py` has binding round-trip tests for this wire, extend them for
  the new optional fields. No new binding name is needed.
- Run core's prescribed check from its repo root (`just check`, per its `AGENTS.md`;
  never `cargo test -p sase_core` alone).

### 4. Python adapter, config, and docs (sase repo)

- `src/sase/config/_settings.py`: add
  `DEFAULT_MANAGED_TMP_PRESSURE_LOW_FREE_SPACE_MIN_AGE_SECONDS = 3600` and
  `get_managed_tmp_pressure_low_free_space_min_age_seconds()` (reuse
  `_managed_tmp_nonnegative_seconds`, config key
  `managed_tmp.pressure.low_free_space_min_age_seconds`). Export both through
  `src/sase/config/__init__.py` and `src/sase/config/core.py` following the existing
  `min_age_seconds` pattern.
- `src/sase/core/managed_tmp_reaper.py`: add a
  `pressure_low_free_space_min_age_seconds: float | None = None` parameter resolved from
  config like the others, send it on the wire, re-export the default constant (e.g. as
  `DEFAULT_PRESSURE_LOW_FREE_SPACE_MIN_AGE_SECONDS`) in `__all__`, and add
  `pressure_effective_min_age_seconds: float | None` to `_ManagedTmpReapResult`, read
  with `raw.get(...)` so an older binding without the key still works. No pressure
  decision logic in Python.
- `src/sase/scripts/sase_chop_managed_tmp_reap.py`: add `pressure_min_age_seconds` to
  the summary (the effective value when a pressure trigger fired, else `None`), and
  update `tests/test_axe_chop_output_contract.py` if it pins the summary keys.
- `src/sase/default_config.yml`: add `low_free_space_min_age_seconds: 3600` with a
  comment under `managed_tmp.pressure`, and update the `managed_tmp_reap` chop
  description to say pressure pruning waits for the minimum age (12h), or 1h once the
  free-space floor is breached, and that runners remove their own launch scratch at exit
  when nothing else still uses it.
- `src/sase/config/sase.schema.json`: add the property (integer, minimum 0, default
  3600, matching description).
- Docs: `docs/configuration.md` managed_tmp YAML example and table, and the
  `managed_tmp_reap` paragraph in `docs/axe.md` (pressure age plus runner-exit cleanup
  and its handoff/liveness exceptions). Note the new optional wire fields wherever
  `docs/rust_backend.md` documents this wire.
- Tests in `tests/test_managed_tmp_reaper.py`: the apollo configuration through the real
  binding (floor breached plus size over the max; 2h-old large target removed; result
  reports 1h effective age), ample free space keeps it, a fresh descendant still
  protects, and a config override of `low_free_space_min_age_seconds` is honored. Extend
  the config/schema parity tests if they enumerate keys.
- Do not edit `sase-core-revision.txt` in this change. Earlier Rust-owner phases
  (`6a60ee5fb7`, `6c433c14d1`, `347e53beab`) left the pin ratchet to the normal
  `just ratchet-core-revision` flow once the core commit is published. State in the
  closing note that the pin needs that ratchet.

### 5. Coordinate with sase-zw.8 before finishing

Append a note (`sase bead note`) to `sase-zw.8.5` and to `sase-zw.8`, roughly:
"COORDINATION from sase-10r: managed-tmp pressure now takes additive wire field
`pressure_low_free_space_min_age_seconds` (config
`managed_tmp.pressure.low_free_space_min_age_seconds`, default 1h) and reports
`pressure_effective_min_age_seconds`. Whenever the free-space floor is breached,
regardless of trigger, the effective pressure min age is min(base, low-space). When
unifying the pressure contract, preserve this and its tests." If `sase-zw.8.5` has
already landed Rust pressure changes by the time you implement, rebase onto them and
express the same rule in its new contract rather than reintroducing the old shape.

### 6. Verify

- Run `just install` first (this workspace's extension may be stale relative to the core
  checkout; sase-zw's audit saw stale-wheel failures misread as product bugs).
- Core: `just check` in the core repo.
- sase: focused runs of the new and touched test files, then `just check` (read the
  `lint_and_test` reference memory). If the scoped lane escalates to the full suite, run
  `just check-full` through `/sase_monitor`. Fix failures caused by this work; handle
  unrelated failures through the task policy (step 7) rather than silencing them.
- `sase bead epic-symbols sase-10r` for any Symvision whitelist entries to remove.

### 7. Disposition follow-ups and close the beads

The epic's land agent never dispositioned its phases' `PROPOSED FOLLOW-UP` notes. Using
`/sase_new_task` for each one (dedupe first; corroborate existing tasks with `+1`
instead of duplicating; decline anything already fixed on master, with evidence):

- `sase-10r.1` #1: ~146M root-owned `~/.local/share/Trash/files/_cacache` left on apollo
  (needs sudo). This is a one-off host issue and likely not SASE product work. Record
  the disposition in the closing note, and create a task only if the dedupe check says
  it is warranted.
- `sase-10r.2` #1: unrelated full-suite failures (stale monitor/start.py audit, gate
  singleton mount ordering, AF_UNIX socket path length, unquoted `sys.executable` in the
  commit-hook test, macOS `/tmp` symlink expectation). Re-check each on current master,
  since several may already be fixed.
- `sase-10r.3` #1: Symvision cannot see `monitor_records`/`project_records` consumers.
  Commits `d2ba89cb42` ("keep lookup helpers private") and `5024571a32` look like they
  resolved it; confirm and decline if so.
- If you confirm that monitor-handoff build scratch (`cargo-targets/<key>` of a runner
  that handed off to `/sase_monitor`) is left for the reaper with no earlier cleanup,
  consider a task for cleaning it up when the monitored proc finishes. It is out of
  scope here.

Then:

1. `sase bead close sase-10r.2 --note "<what landed, verification results, pin ratchet pending>"`.
2. Optionally (read-only) check `ssh apollo df -h /` (see the `tailnet` reference
   memory) and include the free-space figure in the epic note.
3. The user explicitly asked for the epic to be closed. Confirm no `sase-10r.land` agent
   is still active (`sase agent list -a`; the epic bead's assignee is `sase-10r.land`),
   then run
   `sase bead close sase-10r --note "<summary of all three phases, the sase-10r.2 correction, follow-up dispositions>"`.
   If a land agent is still active, leave the epic open and report that instead.
4. The epic plan file `plan:202609/apollo_disk_reclaim_1.md` still says `status: wip`.
   Update its status to done through the supported plan mechanism, if closing the epic
   does not do so automatically. Do not hand-edit other frontmatter.

## Out of scope

- Everything in `sase-zw.8`: inventory, doctor/`disk_pressure` threshold unification,
  proc/run/object owners, and host acceptance.
- Changing bucket horizons or the size-trigger min age (stays 12h).
- Scrubbing `TMPDIR`/`CARGO_*` from proc/monitor spawn environments.
- Any further deletion on apollo.
