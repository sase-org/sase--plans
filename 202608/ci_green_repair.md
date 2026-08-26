---
tier: tale
title: Repair the three deterministic master CI failure clusters
goal:
  "master CI is green again: the two pooled-alias reservation tests, the provider-drain
  end-to-end drill, and the six Artifacts Agent PNG snapshot tests all pass on CI, with
  no production behavior change."
size: medium
proposed_by: bbugyi200.athena.0e6
---

# Repair the three deterministic master CI failure clusters

## Goal

Return `sase-org/sase` master CI to green by fixing the three _deterministic_ test
failures that fail on every CI run, every Python version. All three are test-side
defects that pass on a typical developer machine and fail on CI, so each one is
reproduced below with an exact local command that turns the developer machine into the
CI environment.

No production (`src/`) behavior changes. All three root causes have been reproduced
locally and each proposed fix has been verified locally.

## Background: what is actually failing

The newest CI run with results is
<https://github.com/sase-org/sase/actions/runs/32959217774> (commit `e8de34fe0`). Its
failure set is identical in `test (3.12)`, `test (3.13)`, `test (3.14)` and
`coverage-contexts`:

```
FAILED tests/fakey/test_provider_drain_e2e.py::test_provider_drain_e2e_flag_on_relaunches_stranded_agent - assert None is not None
FAILED tests/test_pooled_alias_single_consumption.py::test_two_consecutive_default_launches_alternate_pool_members - AssertionError: assert ('claude', 'opus') == ('codex', 'gpt-5.6-sol')
FAILED tests/test_pooled_alias_single_consumption.py::test_explicit_large_directive_and_default_alias_share_pool_cursor - AssertionError: assert ('claude', 'opus') == ('codex', 'gpt-5.6-sol')
```

and the `visual-test` job fails with six `wait_for()` timeouts, all in
`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_agents.py`.

The nine PNG _pixel_ mismatches seen in the older run for `e391b1a2` are already gone:
`e8de34fe0` rebaselined those goldens and that run produced no mismatch artifacts. Do
not chase them.

## Task A: pooled-alias reservation redemption sees the host's real PATH

### Root cause

`tests/test_pooled_alias_single_consumption.py` has an autouse fixture
`_force_pool_availability` whose stated job is to "treat both frozen `@large` pool
members as always available". It does that by patching one seam only:

```python
monkeypatch.setattr(llm_config, "_resolved_target_is_available", lambda _t: True)
```

That seam is honoured by the alias _resolution_ path, which looks the callable up
indirectly (`config.__dict__.get("_resolved_target_is_available", ...)` in
`src/sase/llm_provider/model_alias_resolution_selector.py` and
`src/sase/llm_provider/model_alias_resolution_resolve.py`).

It is _not_ honoured by the reservation _redemption_ gate.
`launch_selection_from_reservation()` in `src/sase/llm_provider/launch_selection.py`
calls the module-level `resolved_target_is_available` imported at the top of that file.
That is deliberate, not a bug: the sibling test
`test_unavailable_reservation_falls_back_without_hanging` (same file) patches exactly
`sase.llm_provider.launch_selection.resolved_target_is_available` to prove the fallback
behaviour. So the redemption gate is a genuine "is this provider's CLI actually
installed" check.

On CI the autouse fixture `_default_test_llm_cli` in `tests/_conftest_runtime.py`
prepends a stub bin directory containing **only** a `claude` stub. `codex` is therefore
genuinely unavailable on CI. The two failing tests then run:

1. Launch 1 bootstrap reserves pool member 0 (`claude/opus`); cursor `0 -> 1`.
2. Launch 1 redemption checks `claude/opus` against the real PATH: the stub is there, so
   it redeems. `assert _pool_cursor() == 1` passes.
3. Launch 2 bootstrap reserves pool member 1 (`codex/gpt-5.6-sol`); cursor `1 -> 0`.
4. Launch 2 redemption checks `codex/gpt-5.6-sol` against the real PATH: no `codex`
   binary, so `MemberAvailability.UNAVAILABLE`, the reservation is rejected and spent,
   and the fallback consuming resolution runs with the _patched_ seam at cursor `0` --
   re-picking `claude/opus`.

Hence `assert ('claude', 'opus') == ('codex', 'gpt-5.6-sol')`.

These tests pass on a developer machine only because a real `codex` CLI happens to be
installed there.

### Reproduce locally

From the repo root, hide `codex` from `PATH` (adjust the `grep -v` filter to whatever
directory supplies `codex` on your machine; `command -v codex` shows it):

```bash
CLEANPATH=$(echo "$PATH" | tr ':' '\n' | grep -v "$(dirname "$(command -v codex)")" | paste -sd: -)
PATH="$CLEANPATH" .venv/bin/python -m pytest tests/test_pooled_alias_single_consumption.py -p no:randomly -q
```

Expected before the fix: `2 failed, 9 passed`, with exactly the CI assertion text.

### Fix

In `tests/test_pooled_alias_single_consumption.py`, extend `_force_pool_availability` so
it also neutralises the redemption gate, using the same seam the sibling test in this
file already uses:

```python
monkeypatch.setattr(
    "sase.llm_provider.launch_selection.resolved_target_is_available",
    lambda _target, **_kwargs: True,
)
```

Keep the existing `llm_config` patch -- both seams are needed. Update the fixture
docstring to say that _two_ seams are patched and why: resolution reads the callable
indirectly through `config`, redemption imports it directly, and a pool member whose CLI
is not installed on the host must not change which member these tests observe.

`test_unavailable_reservation_falls_back_without_hanging` applies its own
`monkeypatch.setattr` on the same symbol after the autouse fixture runs, so its narrower
stub still wins. This was verified: all 11 tests in the file pass.

### Verified

With the fixture patch applied via an out-of-tree plugin and `codex` removed from
`PATH`: `11 passed`.

An alternative fix --
`monkeypatch.setenv(provider_path_env_var("codex"), <an existing executable>)` plus
`_provider_cli_available.cache_clear()`, the idiom
`tests/fakey/test_provider_drain_e2e.py` already uses -- was also verified
(`11 passed`). Prefer the seam patch: it is hermetic and needs no binary to exist. Do
not do both.

## Task B: the provider-drain e2e test needs `sase` on PATH

### Root cause

`tests/fakey/test_provider_drain_e2e.py::test_provider_drain_e2e_flag_on_relaunches_stranded_agent`
submits a real durable proc whose argv is asserted by the test itself:

```python
assert request.argv == ["sase", "agent", "drain", "fakey", "--yes", "--json", "--limit", "20"]
```

The proc supervisor runs that argv verbatim -- `subprocess.Popen(argv, ...)` in
`_run_command` (`src/sase/procs/supervisor.py`) -- so `argv[0]` is resolved through the
child's `PATH`.

CI never puts `.venv/bin` on `PATH`: every `Justfile` test recipe invokes
`{{ venv_bin }}/python tools/run_pytest ...` directly. So on CI the drain child cannot
start at all. The supervisor records `status="error"` /
`"could not start command: ..."`, the durable operation result is written with
`payload: null`, and `assert payload is not None` fires at
`tests/fakey/test_provider_drain_e2e.py:235`.

The test passes on a developer machine only because a globally installed `sase` (for
example `~/.local/bin/sase`) is on `PATH`. That is worse than a CI-only failure: it
means the drill has been silently exercising a _different_ sase build than the workspace
under test.

Note the test deliberately expects the drain to _fail_ later on
(`assert result.success is False`) because of a known, documented relaunch bug. That
expected failure still carries a `payload`. The CI failure is the earlier, different
one: no payload at all.

### Reproduce locally

Remove whatever directory supplies the global `sase` from `PATH`:

```bash
CLEANPATH=$(echo "$PATH" | tr ':' '\n' | grep -v "$(dirname "$(command -v sase)")" | paste -sd: -)
PATH="$CLEANPATH" .venv/bin/python -m pytest tests/fakey/test_provider_drain_e2e.py -p no:randomly -q
```

Expected before the fix: `1 failed, 1 passed`, failing with `assert None is not None` at
line 235.

### Fix

In `_configure_reroute_environment`, put the workspace's own entry-point directory on
`PATH` for the drain child (and its grandchild relaunch), next to the existing `fakey`
binary lookup, which already derives from `sys.executable`:

```python
venv_bin = Path(sys.executable).parent
sase_binary = venv_bin / "sase"
assert sase_binary.is_file(), "just install must register the sase binary"
monkeypatch.setenv("PATH", f"{venv_bin}{os.pathsep}{os.environ.get('PATH', '')}")
```

Add `import os` if it is not already imported. Extend the function docstring's list of
env knobs with a fourth bullet explaining that `PATH` is prepended so the proc
supervisor's bare `"sase"` argv resolves to _this_ workspace's entry point, not to
whatever `sase` happens to be installed on the developer's machine.

Also make the failure legible when it does happen. At line 235 change:

```python
assert payload is not None
```

to:

```python
assert payload is not None, f"drain reported no payload: {result.error}"
```

so a future CI failure of this shape prints the supervisor's reason instead of
`assert None is not None`.

### Verified

`2 passed` with the workspace `.venv/bin` on `PATH` and no other `sase` available;
`1 failed, 1 passed` without it.

## Task C: the six Artifacts Agent PNG snapshot tests

### Root cause

`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_agents.py` was added by
`e8de34fe0`. All six of its tests fail -- **on CI and locally** -- at `_open_agents`.
Two independent defects:

1. **Stub arity.** `_install_agents_fixture` installs

   ```python
   monkeypatch.setattr(agents_pane_module, "load_agents_snapshot", lambda _project: snapshot)
   ```

   but since `6ffdfb0a9` ("feat(artifacts): load agent pane in two stages") the pane
   calls it with two positional arguments:
   `load_agents_snapshot(request.project, None if request.full else AGENTS_FIRST_PAGE_LIMIT)`
   in `ArtifactsAgentsPane._build_snapshot`
   (`src/sase/ace/tui/widgets/artifacts/agents_pane.py`). The stub raises `TypeError`,
   the load lands in `_on_snapshot_error`, and `pane.snapshot` is never set.

   The correct idiom already exists in the tree:
   `tests/ace/tui/test_artifacts_list_navigation.py` stubs it as
   `lambda _project, _limit=None: snapshot`.

2. **Unsatisfiable predicate.** `_open_agents` waits on

   ```python
   lambda _state: pane.snapshot is snapshot and pane._query_index is not None
   ```

   `_query_index` is only populated from a `full=True` request (`_build_snapshot` builds
   the index only when `request.full`), and the full second stage is only scheduled when
   `not result.snapshot.complete` (`_apply_snapshot`). The fixture's `AgentsSnapshot`
   uses the default `complete=True`, so the second stage never runs and `_query_index`
   stays `None` forever. The working navigation test waits on
   `pane.snapshot is snapshot` alone.

3. **Stale goldens.** With both defects fixed the six tests run and then fail on pixel
   comparison, ~0.29% changed pixels each. The differing region is a single band --
   bounding box `(267, 91, 1046, 116)` in a 1482x1026 image -- which is the Artifacts
   subtab strip row. That is the direct consequence of `4dd299502` ("feat(artifacts):
   put agents first in subtabs"): that commit rebaselined 35 goldens but could not
   regenerate these six, because their tests error out before ever reaching the snapshot
   write. So the six `artifacts_agents_*` goldens are stale and must be regenerated.

### Reproduce locally

```bash
.venv/bin/python -m pytest tests/ace/tui/visual/test_ace_png_snapshots_artifacts_agents.py -p no:randomly -q -m visual
```

The `-m visual` is required; without it all six deselect. Expected before the fix:
`6 failed`, each `AssertionError: wait_for() timed out after 5.0s`.

### Fix

In `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_agents.py`:

1. `_install_agents_fixture`: change the stub to
   `lambda _project, _limit=None: snapshot`, matching
   `tests/ace/tui/test_artifacts_list_navigation.py`.
2. `_open_agents`: drop the `and pane._query_index is not None` conjunct, leaving
   `lambda _state: pane.snapshot is snapshot`.
3. Rebaseline the six goldens:

   ```bash
   just test-visual tests/ace/tui/visual/test_ace_png_snapshots_artifacts_agents.py --sase-update-visual-snapshots
   ```

   Then confirm `git status` shows exactly these six PNGs changed and nothing else:
   - `tests/ace/tui/visual/snapshots/png/artifacts_agents_empty_120x40.png`
   - `tests/ace/tui/visual/snapshots/png/artifacts_agents_family_grouped_120x40.png`
   - `tests/ace/tui/visual/snapshots/png/artifacts_agents_filter_completion_120x40.png`
   - `tests/ace/tui/visual/snapshots/png/artifacts_agents_filter_parse_error_120x40.png`
   - `tests/ace/tui/visual/snapshots/png/artifacts_agents_narrow_80x24.png`
   - `tests/ace/tui/visual/snapshots/png/artifacts_agents_populated_120x40.png`

   If any _other_ golden also turns up changed, stop and report it rather than accepting
   it: that would mean a second, unrelated rendering change is in flight.

4. Re-run the six tests without the update flag and confirm they pass.

### Verified

With both test defects corrected via an out-of-tree plugin, all six tests get past
`wait_for` and reach the PNG comparison, failing only on the stale-golden pixel diff
described above.

## Verification

1. Targeted, in CI-like conditions -- both clusters at once, with `codex` and the global
   `sase` hidden:

   ```bash
   CLEANPATH=$(echo "$PATH" | tr ':' '\n' \
     | grep -v "$(dirname "$(command -v codex)")" \
     | grep -v "$(dirname "$(command -v sase)")" | paste -sd: -)
   PATH="$CLEANPATH" .venv/bin/python -m pytest \
     tests/test_pooled_alias_single_consumption.py \
     tests/fakey/test_provider_drain_e2e.py -p no:randomly -q
   ```

   Must be all-pass. This is the check that actually proves the CI failures are gone; a
   plain run on a fully-equipped developer machine passes even with the bugs present and
   proves nothing.

2. Visual suite: `just test-visual`, all green.

3. `just check-full` through your `/sase_monitor` skill (the `TESTING` / `TESTED` status
   pair), never inline.

## Out of scope

Do not fix these here. File each as a task bead through `/sase_new_task` if it is not
already tracked:

- `tests/test_models_panel_edit_outcomes.py::test_on_alias_edited_offers_commit_when_in_repo`
  failed once on `test (3.14)` only, with
  `textual.css.query.NoMatches: No nodes match '#confirm-btn' on ConfirmActionModal`.
  Single job, single run -- flake, not part of this repair.
- `tests/fakey/test_pipe_e2e.py::test_default_pipe_creates_family_member_with_fork_and_shared_workspace`
  failed once on `test (3.12)` with `[Errno 32] Broken pipe`. Flake.
- `perf-floors` failed once on run `32923010097` at
  `scan_agent_artifacts.synthetic_6p_200pp.scan_facade`: rust median 285269.89us against
  a 281973.47us absolute ceiling -- 1.2% over. It passed on both following runs.
  Hosted-runner variance against a floor that already carries a documented per-anchor
  exemption; note it, do not retune it here.
- Broader local/CI parity hardening, worth its own bead: `_test_llm_bin` in
  `tests/_conftest_runtime.py` stubs only `claude`, and durable-proc argv relies on a
  bare `"sase"` resolving through `PATH`. Both invite the exact class of bug fixed in
  Tasks A and B. Fixing either one properly is a separate change.

## Non-goals

- No changes under `src/`. Every root cause here is in test code or in committed test
  goldens.
- Do not re-tune the Phase 7E perf floors.
- Do not rebaseline any golden outside the six named in Task C.
