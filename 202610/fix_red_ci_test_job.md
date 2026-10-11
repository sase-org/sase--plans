---
tier: tale
title: Fix the red test CI job
goal:
  The Master Gate and Full CI `test` jobs pass again on master, with no failure that
  depends on clone depth or wall-clock timing.
size: medium
decisions:
  flakes:
    ask:
      Also fix the two intermittent test-job races (terminate escalation, oracle mtime)
      and close their flake beads?
    choices:
      include:
        Fix both races with deterministic regression tests; close sase-1hx and sase-10g
      skip: Fix only the five deterministic failures; leave both flake beads open
    default: include
    why: both races reproduce deterministically and each fix is a few lines
    answer: include
proposed_by: bbugyi200.athena.0zm.w1
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 08:35:55
status: wip
---

# Fix the red `test` CI job

## Context

The `test` job has been red on every Master Gate push for over a day (40+ consecutive
runs), and the matching `test (3.12/3.13/3.14)` legs fail in the scheduled Full CI
workflow. The latest Master Gate run (master `1517ed288f`) fails shards 1, 3, and 8 with
exactly five deterministic failures. Full CI and earlier Master Gate runs also show two
intermittent failures. This plan fixes all seven. The other red jobs (`lint` and
`visual-test` in Full CI, `release-core-floor-smoke` on the release-please PR) are out
of scope.

Every root cause below was reproduced locally before this plan was written.

### Deterministic failures (fail on every run, every Python version)

1. `tests/completion/test_candidates_providers.py::test_flag_candidates_come_from_the_in_process_registry`:
   `assert {'agents_archive_view'} == set()`. The test was written when the flag
   registry was empty. The Agents Archive epic then added the legitimate beta flag
   `agents_archive_view` through `sase flag new` (flag bead `sase-1ju`), so the test now
   pins a registry state that no longer holds.
2. `tests/completion/test_snapshot.py::test_checked_in_snapshot_has_no_drift` and
   `::test_current_structural_view_matches_checked_in_snapshot`: the checked-in
   `tests/completion/snapshots/cli_spec.json` is stale. The only drift is
   `root.subcommands.agent.subcommands.search.description_digest` (`f75a3fcb2bbd4075` ->
   `d0bf3fe5c8158622`), because the `sase agent search` help text changed in the
   archive-search CLI parity work and nobody ran `just sync-completion-spec`.
3. `tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed`:
   the reviewed-site key `src/sase/axe/run_agent_directive_metadata.py:session_root_tab`
   is stale. Commit `eff003d6a9` ("split run_agent_directive_metadata into focused
   modules") moved `session_root_tab` unchanged into
   `src/sase/axe/run_agent_directive_metadata_preserved.py`. The function is still a
   read-only `agent_meta.json` read, so the existing exemption text stays accurate.
4. `tests/test_agent_auto_restart_incident_replay.py::test_incident_replay_end_to_end`:
   `assert '9fd8a08' in '↻ Restarted research.46.final after sase update 9c5000f-<HEAD>'`.
   This test has never passed in CI. It depends on the host checkout's real git history.
   The W3 file-proof witness (`sase.agent.auto_restart.witnesses._collect_file_proof`)
   runs bounded `git` probes against the live sase source root, using the incident's
   real commits `BOOT_REV=9c5000f…` and `UPDATE_REV=9fd8a08…`.
   - In a full local clone those commits exist, so W3 names culprit `9fd8a08…` and the
     Rust episode id becomes `sase@9fd8a08`.
   - CI's `actions/checkout` is a depth-1 shallow clone that lacks both commits. W3
     returns `None`, and `derive_auto_restart_episode` falls back to the
     boot-identity-to-current-HEAD range `sase@9c5000f-<HEAD>`.
   - Forcing `_collect_file_proof` to return `None` locally reproduces the CI assertion
     exactly. Locally W3 returns
     `AutoRestartFileProofWire(symbol='auto_launch_prefix', module='sase.monitor.continuation_delivery', boot_has=True, head_has=False, culprit_commit='9fd8a081f45689655249b7bf6ed7b561de65b8bf', culprit_subject='feat(autonomy): structural inheritance of live record and truthful A toggle')`.

   The product behavior is correct. The test is not hermetic.

### Intermittent failures (decision `flakes`)

5. `tests/test_agent_terminate_processes.py::test_terminates_every_process_shape_and_verifies_death`:
   `assert 'killed' == 'force_killed'`. Seen in Master Gate run 38130053371, shard 2.
   Already tracked as flake bead `sase-1hx`, which had no root cause. The root cause is
   a real race in `_run_termination` (`src/sase/agent/_user_kill_tree.py`):
   - When stragglers outlive SIGTERM, the code first sends
     `termination.signal_group(signal.SIGKILL)` and then calls
     `termination.signal(target, SIGKILL)` for each straggler. `escalated` is computed
     only from the per-target calls.
   - `signal()` returns `False` when `alive(target)` already reports the process dead.
     If the group SIGKILL has already killed the straggler (here the same-group
     `ignores_term` child), `escalated` stays `False` and the result reads `killed`,
     even though a SIGKILL escalation happened.
   - Reproduced 100% by injecting a `killpg` that waits for the `ignores_term` child to
     die after a group SIGKILL. Result:
     `status='killed', escalated=False, survivors=()`.
6. `tests/test_agent_load_tiering_production_oracle.py::test_production_machine_query_oracle_repairs_owner_after_index`:
   `assert '20260912135847' not in {'20260912135847'}`. Seen in Full CI run 38116223239,
   3.12 leg. Already tracked as flake bead `sase-10g`, which suspected leaking
   process-global state. The real cause is a wall-clock-second boundary race in the
   fixture:
   - Full-history reconcile (sase-core `agent_scan/index/refresh.rs`,
     `reconcile_source_directories` and `artifact_dir_is_dirty`) re-checks a row's
     marker signatures only when the artifact directory's mtime is newer than the row's
     `indexed_at`. `indexed_at` is a second-resolution SQLite `CURRENT_TIMESTAMP`.
   - The fixture helper `set_artifact_machine_provenance` (in
     `tests/perf/_agent_load_tiering_fixture_mutations.py`) rewrites markers in place
     with `Path.write_text`. An in-place rewrite never bumps the directory mtime.
   - So the repair is detected only when the artifact directory was created in the same
     wall-clock second as the index rebuild. When `rebuild_index` crosses a second
     boundary, the change is missed.
   - Production marker writes are atomic (temp file plus rename), and a rename does bump
     the directory mtime. So production is fine and the fixture is not
     production-shaped.
   - Reproduced 100% by back-dating the artifact directory mtimes 2s right after
     `rebuild_index`. With the provenance helper switched to temp-file plus
     `os.replace`, all 12 oracle tests pass under the same back-dating.

> [!decision] flakes = skip Do steps 1-4 and the verification only. Skip steps 5 and 6
> and the bead closes in step 7, and leave `sase-1hx` and `sase-10g` open. With
> `include` (default), do every step.

## Steps

### 1. Flag candidates test tracks the live registry

In `tests/completion/test_candidates_providers.py`,
`test_flag_candidates_come_from_the_in_process_registry`:

- Replace `assert keys == set()` with `assert keys == set(feature_flag_definitions())`.
  Import `feature_flag_definitions` from `sase.feature_flags.registry`, which returns a
  mapping keyed by flag string.
- Keep every existing `assert "<retired flag>" not in keys` line.
- Do not hard-code `agents_archive_view`. Its epic removes it before landing, and the
  test must stay green across flag additions and removals.

### 2. Regenerate the completion spec snapshot

Run `just sync-completion-spec`. If a raw agent invocation of that recipe is refused,
run it through `sase tool run` or use
`.venv/bin/python tools/sync_completion_spec --write`.

Confirm that `git diff tests/completion/snapshots/cli_spec.json` changes only the
`agent search` `description_digest`. If more nodes changed, the tree moved since this
plan was written. Still accept the regenerated file, but check that each changed node
matches a real CLI change. Then confirm `tools/sync_completion_spec --check` exits 0.

### 3. Re-key the moved marker-path audit site

In `tests/test_agent_artifact_marker_path_passing_audit.py`, rename the
`_REVIEWED_PATH_PASSING_CONTEXTS` key
`"src/sase/axe/run_agent_directive_metadata.py:session_root_tab"` to
`"src/sase/axe/run_agent_directive_metadata_preserved.py:session_root_tab"`. Keep the
existing exemption text unchanged, since the function body is unchanged and still
read-only. If the audit reports any other new or missing site, review it the same way:
re-key a pure move, and write a real review entry for genuinely new code.

### 4. Make the incident replay hermetic

In `tests/test_agent_auto_restart_incident_replay.py::test_incident_replay_end_to_end`,
stub W3 so the replay never reads the host checkout's git history:

- Add a module-level constant for the recorded incident proof. Import
  `AutoRestartFileProofWire` from `sase.core.agent_auto_restart_wire`:
  `AutoRestartFileProofWire(symbol="auto_launch_prefix", module="sase.monitor.continuation_delivery", boot_has=True, head_has=False, culprit_commit=UPDATE_REV, culprit_subject="feat(autonomy): structural inheritance of live record and truthful A toggle")`.
- In the test,
  `monkeypatch.setattr("sase.agent.auto_restart.witnesses._collect_file_proof", fake)`.
  The keyword-only `fake(*, module, symbol, root, boot_rev)`:
  - appends its call to a local list;
  - asserts `module == "sase.monitor.continuation_delivery"`,
    `symbol == "auto_launch_prefix"`, `root.name == "sase"`, and `boot_rev == BOOT_REV`;
  - returns the recorded proof.

  The signature mirrors `_collect_file_proof` in
  `src/sase/agent/auto_restart/witnesses.py`.

- After `heal_one(...)` returns, assert the stub was called at least once. If the
  witness path ever stops consulting W3, the test then fails loudly instead of silently
  skipping the culprit path.
- Update the module docstring's "Fake only …" sentence to also name the W3 git file
  proof, recorded from the real incident so the replay does not depend on the clone's
  history depth.
- Do not loosen any title or note assertion. `9fd8a08` must still appear.

Verify that the test passes. The stub-was-called assertion is what proves history
independence: W3 no longer runs `git` against the checkout, so the clone's depth cannot
change the episode id.

### 5. Count the group SIGKILL toward escalation (decision `flakes`)

In `src/sase/agent/_user_kill_tree.py`:

- Make `_TreeTermination.signal_group` return `bool`: `True` when `killpg` was called
  and did not raise, and `False` when the group is not signalable or `killpg` raised
  `ProcessLookupError` or `PermissionError`.
- In `_run_termination`'s straggler branch, seed `escalated` from the group SIGKILL and
  keep the per-target OR:

  ```python
  if stragglers:
      escalated = termination.signal_group(signal.SIGKILL)
      for target in stragglers:
          escalated = termination.signal(target, signal.SIGKILL) or escalated
  ```

- The SIGTERM call site ignores the return value, so it needs no change.

Add a deterministic regression test to `tests/test_agent_terminate_processes.py`, next
to `test_terminates_every_process_shape_and_verifies_death`:

- Launch the same fake runner with `launch_runner`, `write_meta`, and `no_registry`.
- Pass `terminate_agent_processes(..., killpg=killpg)`, where the injected `killpg`
  calls `os.killpg(pgid, sig)` and, for `signal.SIGKILL`, then calls
  `wait_until_gone(pids["ignores_term"])`. This forces the "group kill already reaped
  the straggler" ordering.
- Assert `result.success is True`, `result.survivors == ()`,
  `result.status == "force_killed"`, `result.escalated is True`, and
  `assert_dead(pids)`. Then call `proc.wait(timeout=5)`.

This test failed with `status='killed', escalated=False` on the unfixed code. Confirm it
fails before your source change and passes after it. Also confirm that
`tests/test_agent_user_kill.py`, `tests/test_kill_durable_termination.py`,
`tests/test_agent_name_wipe_live.py`, and the rest of
`tests/test_agent_terminate_processes.py` still pass. Their `killed` expectations cover
paths with no stragglers, so they should not change.

### 6. Make post-index fixture mutations production-shaped (decision `flakes`)

In `tests/perf/_agent_load_tiering_fixture_mutations.py`:

- Add a small private helper `_replace_json(path, payload)`. It writes
  `json.dumps(payload, sort_keys=True)` to a sibling temp file in the same directory,
  then calls `os.replace` onto `path`, which bumps the directory mtime the way
  production's atomic marker writes do. Use a temp name such as `.<name>.tmp` that never
  matches a marker glob.
- Use it in `set_artifact_machine_provenance` instead of `path.write_text(...)`, and in
  `set_artifact_hidden` instead of `_write_json(...)`. Mention atomicity in both
  docstrings: an in-place rewrite does not bump the directory mtime, so revalidation's
  directory-dirty check could miss the change.
- Leave `_write_json` and the initial-build writers alone. Creating a new file already
  bumps the directory mtime.

In
`tests/test_agent_load_tiering_production_oracle.py::test_production_machine_query_oracle_repairs_owner_after_index`,
make the worst-case ordering deterministic. Right after `rebuild_index(fixture)` and
before `set_artifact_machine_provenance(...)`, back-date `target_dir`'s mtime by 2
seconds with `os.utime`. This simulates an index stamped in a later wall-clock second
than the artifact directory's creation. Add a one-line comment explaining why. With the
old in-place helper this fails every time. With step 6's helper it passes every time.

If the oracle tests fail locally with
`ModuleNotFoundError: No module named 'sase_github'`, the workspace's editable
`sase-github` plugin checkout is not materialized yet. Run
`sase repo open sase-github -r "<reason>"` once and rerun. This is local setup only and
unrelated to CI.

### 7. Close the flake beads (decision `flakes`)

After steps 5 and 6 pass verification, close both beads, each with a note naming the
root cause and the regression coverage:

- `sase bead close sase-1hx --note "<root cause + fix + regression test>"`
- `sase bead close sase-10g --note "<root cause + fix + regression test>"`

The `sase-10g` note should state that the cause was the fixture's in-place marker
rewrite racing the second-resolution `indexed_at`, not leaking process-global state.

## Verification

1. Targeted run of every touched or related test module. It must pass serially and under
   xdist:
   `.venv/bin/python -m pytest -q tests/completion/test_candidates_providers.py tests/completion/test_snapshot.py tests/test_agent_artifact_marker_path_passing_audit.py tests/test_agent_auto_restart_incident_replay.py tests/test_agent_terminate_processes.py tests/test_agent_user_kill.py tests/test_kill_durable_termination.py tests/test_agent_name_wipe_live.py tests/test_agent_load_tiering_production_oracle.py`.
   Then run it again with `-n 4`. Run the two new deterministic regressions from steps 5
   and 6 five times each in a shell loop to confirm they are stable.
2. Run `just fix` (or at least `just fmt`) inline, then run the repo gate
   `sase tool run check`. Follow the lint-and-test memory for routing a long run through
   a monitor. If the workspace venv is stale, run `just install-venv` first. Do not run
   `just check-full`.
3. Do not report success from local results alone. In the final summary, list each of
   the seven failures with its fix. Note that the next Master Gate push is the real
   confirmation for the shallow-clone behavior in step 4.

## Out of scope

- Full CI `lint` and `visual-test`, and the release-please PR's
  `release-core-floor-smoke` job. These are separate CI jobs, not the `test` job.
- Any change to sase-core. The reconcile directory-dirty heuristic is correct for
  production's atomic writes, and only the fixture was wrong.
- Changing CI checkout depth. Tests must not depend on clone history.
