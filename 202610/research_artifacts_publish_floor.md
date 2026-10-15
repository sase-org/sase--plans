---
tier: tale
title: Unblock sase-research-artifacts Publish
goal:
  The plugin's Publish lane targets the real sase 0.18.0 floor and one shared wheel
  smoke, so it publishes once sase 0.18.0 ships.
size: medium
decisions:
  core_pairing:
    ask:
      Should the smoke derive its core version from sase 0.18.0's own core floor instead
      of hard-coding it?
    choices:
      derived:
        lowest-direct resolve, assert core equals sase's floor; survives the core-window
        ratchet
      pin:
        Hard-code sase-core-rs==0.40.0; explicit, but stale if sase ratchets before
        0.18.0 ships
    default: derived
    why: sase owns the core window and has already ratcheted it from 0.34.71 to 0.40.0
    answer: derived
  bead:
    ask: Should the coder close bug bead sase-1ei once the fix lands?
    choices:
      close:
        Close as done; publish then waits only on sase 0.18.0 (sase-1kx) plus a
        release-PR merge
      keep: Leave it ready with a progress note until a 0.3.x release reaches PyPI
    default: close
    why: the defect it names is fixed; the remaining wait is already tracked by sase-1kx
    answer: close
proposed_by: bbugyi200.apollo.7t
decided_by: auto
create_time: 2026-10-15 10:06:57
status: wip
---

# Plan: Unblock sase-research-artifacts Publish: real sase floor, derived core pairing, one shared wheel smoke

## Diagnosis

`actstat` is not installed on this host (only `~/.config/actstat/config.yml` exists), so
the investigation used the equivalent `gh` queries.

- **Failing:** `Publish` run 38716910820 (merge of `chore(master): release 0.3.1`,
  2026-10-15), job `install-smoke-published-minimum`, step "Install built wheel with
  published minimum dependencies". The v0.3.0 Publish run 36936311149 (2026-10-01)
  failed identically. `CI` and `PR Title` are green. PyPI still serves
  `sase-research-artifacts` 0.2.0, so 0.3.0 and 0.3.1 were tagged but never published.
- **Error:** `uv pip install ... "sase==0.17.2" "sase-core-rs==0.35.0" dist/*.whl` →
  `there is no version of sase==0.17.2`.

There are three stacked root causes. The first masks the other two.

1. **The sase floor names a version that will never exist.** `pyproject.toml` declares
   `sase>=0.17.2`, and `publish.yml`, `tests/test_ci_install_contract.py` and
   `tests/test_wheel_contract.py` pin or assert 0.17.2. That value was a guess made on
   2026-09-10 (commit 8f00896, bead sase-z4.6.4). sase's `release-please-config.json`
   sets `bump-minor-pre-major` and `bump-patch-for-minor-pre-major`, and the pending
   release contains a breaking change. So sase's open release PR #299 is
   `chore(master): release 0.18.0`, and its branch `pyproject.toml` says
   `version = "0.18.0"`. PyPI's latest sase is still 0.17.1. That release cannot carry
   this plugin: it lacks weighted queue metadata, the `sase.macro` APIs and the
   mode-aware `provider_enabled`/`provider_disabled` filters. It is also uninstallable
   (sase-16p).
2. **The exact core pin cannot pair with the real floor.** sase's release branch
   requires `sase-core-rs>=0.40.0,<0.41.0`. That window is re-ratcheted on every
   scheduled sase Publish run by `sync-release-metadata` / `tools/ratchet_core_window`;
   it has already moved from 0.34.71 to 0.35 to 0.40. PyPI also prunes old
   `sase-core-rs` releases (sase-16p). So `sase==0.18.0` plus `sase-core-rs==0.35.0` is
   unsatisfiable, and any hard-coded core pin goes stale.
3. **The published-minimum smoke is a stale, release-only copy of the wheel-contract
   smoke.** The inline Python in `publish.yml` only runs when a release is cut, and it
   has drifted from the maintained script in `tests/test_wheel_contract.py`:
   - It imports `sase.xprompt.loader_sources` / `sase.xprompt.processor`. sase 0.18.0
     has no `sase.xprompt` package; the APIs are now `sase.macro.*`
     (`load_macros_from_plugins`, `expand_single_macro`).
   - It asserts `research_swarm.content.count("%q(1.5x, w=0.25") == 9`, but the swarm
     now routes through the `_queue` local macro and the count is 0.
   - It expects 4 segments; the maintained smoke expects 3, or 5 with `image=true`.
   - It runs without `SASE_HOME` isolation.

**Evidence from a local simulation of the published-minimum lane.** A sase 0.18.0 wheel
was built from `origin/release-please--branches--master` and the plugin wheel from
master. Then
`uv pip install --resolution lowest-direct --find-links <wheels> "sase==0.18.0" "sase-core-rs>=0.35.0" <plugin wheel>`
resolved sase 0.18.0 plus `sase-core-rs` 0.40.0 from real PyPI; 0.40.0 is exactly sase's
own declared floor. Against that environment:

- The current `publish.yml` smoke failed with
  `ModuleNotFoundError: No module named 'sase.xprompt'`.
- With only its imports patched, it failed on the `== 9` count assertion.
- The wheel-contract smoke script passed (`WHEEL_CONTRACT_OK`), including under
  `python -I` from the plugin checkout root.

**Tracking.**

- Bug bead **sase-1ei** already describes cause 1 (status READY).
- Publishing sase 0.18.0 is owned by the in-progress epic **sase-1kx** through ci_watch.
  Never hand-merge sase PR #299.
- The workflow annotations are not causal and out of scope: Node 20 deprecation,
  `ubuntu-latest` moving to Ubuntu 26 on Oct 19, and a transient cache 400.

## Outcome and scope

After this lands:

- The fix commit's own Publish run on master is green: release-please opens a 0.3.2
  release PR, but no release is created, so the build and smoke jobs are skipped.
- The published-minimum lane becomes correct, and it passes once sase 0.18.0 is on PyPI.
- The plugin reaches PyPI when a human merges the plugin's release PR after sase 0.18.0
  is published. Merging earlier makes Publish fail with `no version of sase==0.18.0`;
  that is the gate working, not a bug.

Out of scope:

- Publishing sase (sase-1kx).
- Any `gh workflow run`, release dispatch or PR merge.
- `CHANGELOG.md` and `.release-please-manifest.json` (both release-please-owned).
- `tests/sase_macro_compat.py`.
- Moving `xprompts/` to `macros/` (sase-1gq).

There is no regression for source installs. sase master's own version is 0.17.1, so the
current `>=0.17.2` floor already excludes it; coordinated-source CI and local lanes use
`--overrides`.

## Implementation

Open the plugin with
`sase repo open sase-research-artifacts -r "Fix the published-minimum Publish lane"`,
read its `AGENTS.md`, and work only in the printed path. All paths below are relative to
that checkout.

### 1. Retarget the sase floor to 0.18.0

- `pyproject.toml`:
  - Change the dependency to `"sase>=0.18.0"`.
  - Rewrite the comment. sase 0.18.0 is the first published release with weighted queue
    support, the `sase.macro` APIs and the mode-aware provider filters. sase's
    release-please config made the pending release 0.18.0, not 0.17.2.
  - Keep `"sase-core-rs>=0.35.0"` and its floor-only rationale unchanged.
- `AGENTS.md`:
  - Update the "Depends on" bullet to `sase>=0.18.0`.
  - Resolve the "revisit the `sase` floor ... when a release actually carries them"
    caveat: 0.18.0 carries `provider_enabled("hard")` / `provider_disabled` (see
    `sase/macro/jinja_filters.py` on the release branch).
- `README.md` (Installation) and `docs/configuration.md` (Requirements): change
  `sase>=0.17.2` to `sase>=0.18.0`.
- In the `docs/configuration.md` paragraph about the publish workflow, say it installs
  the exact published-minimum SASE wheel paired with that release's own published core
  floor (or, under `core_pairing = pin`, an exact core wheel). Also say that both lanes
  run the same `tests/wheel_smoke.py`.

### 2. One shared wheel smoke script

- Create `tests/wheel_smoke.py`. It is deliberately not `test_`-prefixed, so pytest does
  not collect it. Move the body of the `smoke_script` string out of
  `tests/test_wheel_contract.py`, turning the `"\\n"` escapes back into real `"\n"`
  literals. Then:
  - Import `load_macros_from_plugins` and `expand_single_macro` directly from
    `sase.macro.loader_sources` / `sase.macro.processor`, and delete the `sase.xprompt`
    fallback. The 0.18.0 floor guarantees the new names, and a fallback to removed APIs
    only hides breakage.
  - Add a module docstring. It should say that both the source-coordination lane
    (`tests/test_wheel_contract.py`) and the published-minimum lane (`publish.yml`) run
    this script, so it may only use APIs present at the sase floor and must stay
    version-agnostic, with no version pins.
  - Keep every existing assertion unchanged, including the final
    `print("WHEEL_CONTRACT_OK")`.
- `tests/test_wheel_contract.py`:
  - Run the script as `[str(venv_python), "-I", str(ROOT / "tests" / "wheel_smoke.py")]`
    with the existing isolated `SASE_HOME` env, and keep the `returncode` and
    `WHEEL_CONTRACT_OK` assertions.
  - Remove the inline string.
  - Set `MINIMUM_SASE_VERSION = "0.18.0"`.
  - Why `-I`: the script's directory is not put on `sys.path` and `PYTHON*` variables
    are ignored, so only the installed wheel can satisfy the imports.

### 3. Rewrite the `install-smoke-published-minimum` job in `.github/workflows/publish.yml`

1. Add `- uses: actions/checkout@v4` as the first step. It checks out only this repo,
   for `tests/wheel_smoke.py`; do not add any sase or sase-core checkout.
2. Install step:

   ```bash
   uv venv --python 3.12 /tmp/smoke-minimum-venv
   uv pip install --python /tmp/smoke-minimum-venv/bin/python --resolution lowest-direct "sase==0.18.0" "sase-core-rs>=0.35.0" dist/*.whl
   ```

   Add a short comment: `lowest-direct` pairs the floor sase with the lowest core
   allowed by both sase's own published window and this plugin's floor, which is exactly
   sase's published core floor.

   > [!decision] core_pairing = pin Instead install
   > `"sase==0.18.0" "sase-core-rs==0.40.0" dist/*.whl` without `--resolution`, and
   > assert both exact versions in step 3.
   >
   > Before landing, recheck `sase-core-rs` in `pyproject.toml` on sase's
   > `release-please--branches--master`; match the pin to its floor if it moved.
   >
   > Record in the `pyproject.toml` comment that the pin must be rechecked before the
   > plugin release PR is merged.

3. New step "Assert published floors" (inline Python,
   `/tmp/smoke-minimum-venv/bin/python`):
   - Assert that `importlib.metadata.version("sase") == "0.18.0"`.
   - Parse sase's own `sase-core-rs` requirement from
     `importlib.metadata.requires("sase")` using `packaging`, which sase depends on
     directly, and skip marker-gated entries. Assert that the installed `sase-core-rs`
     version equals the inclusive lower bound of that specifier.
   - Assert that the installed core version satisfies this plugin's own `sase-core-rs`
     requirement from `requires("sase-research-artifacts")`.
4. Replace the "Smoke weighted research swarm on published floors" step body with
   `SASE_HOME="$(mktemp -d)" /tmp/smoke-minimum-venv/bin/python -I tests/wheel_smoke.py`.
5. "Refuse older published SASE": keep `sase==0.17.1`. Change the grep alternation from
   `sase>=0\\.17\\.2` to `sase>=0\\.18\\.0`.
6. Leave these unchanged:
   - "Refuse older published core" (`sase-core-rs==0.34.73`).
   - The `install-smoke` source-coordination job.
   - The `publish` job's `needs`.

### 4. Update `tests/test_ci_install_contract.py`

- Set `PUBLISHED_MINIMUM_SASE = "sase==0.18.0"`. Assert `'"sase>=0.18.0"'` in
  `test_pyproject_floor_matches_expected_first_supporting_release`.
- `test_release_smoke_requires_clean_published_minimum_wheels`:
  - Under `derived`, drop `PUBLISHED_MINIMUM_CORE`. Assert that the job contains the
    exact install line from step 3.2, with `--resolution lowest-direct`,
    `f'"{PUBLISHED_MINIMUM_SASE}"'` and `f'"{PLUGIN_CORE_REQUIREMENT}"'`. Also assert
    that it contains the floor-equality check, `requires("sase")`.
  - Under `pin`, set `PUBLISHED_MINIMUM_CORE = "sase-core-rs==0.40.0"` and assert it.
  - Assert that it contains `uses: actions/checkout@v4` and
    `python -I tests/wheel_smoke.py`, plus `SASE_HOME=`.
  - Assert that it does not contain `repository: sase-org/sase`, `--overrides`,
    `install-source-sase` or `maturin`.
  - Keep the `INCOMPATIBLE_SASE`, `INCOMPATIBLE_CORE` and step-name assertions.
- Move the swarm-contract marker assertions to read `tests/wheel_smoke.py`:
  - `load_macros_from_plugins`, `expand_single_macro`, `plan_typed_launch_units`
  - `selected_project="sase"`, `directives.queue_capacity`, `capacity=`, `at least 1`
  - `{"runners": "0"}`, `{"runners": "1"}`, `runner_capacity_policy_schema_version`
- Assert that `"sase.xprompt"` and any `sase==` pin are absent from the shared script.
- `test_wheel_contract_is_source_coordination_not_published_minimum`:
  - The `(0, 35)` core-tuple check now lives in `tests/wheel_smoke.py`.
  - The wheel test must still contain `SASE_RESEARCH_ARTIFACTS_RESOLVED_SASE_SOURCE`,
    `maturin` and `wheel_smoke.py`, and must contain no inline `smoke_script = """`.
  - The published-minimum pins stay absent from both the wheel test and the shared
    script.
- Add a drift guard: both `publish.yml`'s published-minimum job and
  `tests/test_wheel_contract.py` reference `tests/wheel_smoke.py`.

### 5. Validate (all must pass)

1. `sase tool run check` in the plugin checkout (lint plus fast tests; `check` is
   guarded).
2. `just test-wheel`. It builds sase-core with maturin and takes several minutes, so run
   it in the foreground with a long timeout. It proves the source-coordination lane
   still passes through the shared script.
3. Simulate the published-minimum lane. CI cannot exercise it until sase 0.18.0 is on
   PyPI. Use a fresh `mktemp -d` directory and never write build output into the plugin
   checkout.
   1. Get a sase 0.18.0 wheel. If `curl -s https://pypi.org/pypi/sase/json` already
      lists 0.18.0, use PyPI directly and skip the build. Otherwise, in the sase
      workspace checkout:
      - Run `git fetch origin release-please--branches--master`.
      - Run
        `git archive origin/release-please--branches--master | tar -x -C <tmp>/sase-src`.
      - Run `uv build --wheel -o <tmp>/wheels <tmp>/sase-src`.
      - Confirm the wheel is `sase-0.18.0-*.whl`.
   2. Run `uv build -o <tmp>/dist .` in the plugin checkout.
   3. From the plugin checkout root, run the job's `run:` blocks as written in
      `publish.yml`. Extract them from the workflow so the committed text is what gets
      tested. Make only these substitutions:
      - Add `--find-links <tmp>/wheels` (only when the wheel was built locally).
      - Replace `dist/*.whl` with `<tmp>/dist/*.whl`.
      - Use `<tmp>`-scoped venv paths.
   4. Expect all of the following:
      - sase 0.18.0, and `sase-core-rs` equal to the release branch's floor (0.40.0 at
        planning time).
      - "Assert published floors" passes.
      - `WHEEL_CONTRACT_OK` is printed.
      - Both refusal steps pass.
   5. Delete the temp dir.
4. Run
   `python3 -c 'import yaml; yaml.safe_load(open(".github/workflows/publish.yml"))'`. If
   `actionlint` is unavailable, note that rather than installing it globally.

### 6. Commit and bead

- Commit the plugin repo through the normal `/sase_final` declaration. Use a
  conventional `fix(release): ...` subject so release-please cuts 0.3.2.
- Bead sase-1ei: note the fix commit and the simulation result, and state that the
  remaining publish waits on sase 0.18.0 (sase-1kx) and then a human merge of the plugin
  release PR.

  > [!decision] bead = close `sase bead close sase-1ei --note "<what you verified>"`.

  > [!decision] bead = keep `sase bead note sase-1ei "<progress>"` only; leave it ready.

- Do not create new beads. Report any unrelated discovery in the final response.
