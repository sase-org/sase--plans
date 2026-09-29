---
tier: tale
title: Make sase-research-artifacts core pin floor-only to fix CI
goal:
  sase-research-artifacts CI installs and passes again against sase master's
  sase-core-rs 0.35.x window, and future sase core-window ratchets can no longer break
  the plugin's dependency resolution.
size: small
proposed_by: bbugyi200.athena.0tx
create_time: 2026-09-29 08:31:21
status: wip
---

# Fix sase-research-artifacts CI: make the plugin's sase-core-rs pin floor-only

## Problem

GitHub Actions `CI` is red on `sase-org/sase-research-artifacts` master (run
36317037884, commit `7be5cae`) and on its open release-please PR (`release 0.3.0`). Both
matrix legs die in the `Install dependencies` step (`just install`) before lint or tests
run:

```
error: No solution found when resolving dependencies
  cause: Because only sase==0.17.1 is available and sase==0.17.1 depends on
  sase-core-rs>=0.35.0,<0.36.0, ... And because sase-research-artifacts==0.2.0 depends
  on sase and sase-core-rs>=0.34.23,<0.35.0, ... your requirements are unsatisfiable.
```

## Root cause

- CI checks out `sase-org/sase` **master** and installs it through `.sase-overrides.txt`
  (`-e <sase checkout>`), so the resolver takes sase master's own requirements.
- On 2026-09-27 sase master commit `bc71445748`
  (`chore(core): ratchet sase-core-rs floor to >=0.35.0,<0.36.0`) moved sase's core
  window from 0.34.x to 0.35.x.
- This plugin declares its own direct `sase-core-rs>=0.34.23,<0.35.0`. The overrides
  file only overrides `sase`, not `sase-core-rs`, so uv must satisfy both windows. They
  don't overlap, so resolution fails. (`just install` would overwrite the resolved core
  with a maturin build of sase-core master anyway, but resolution fails first.)
- This has happened before. Commit `1a7643f` (2026-09-13) fixed the same break by
  copying sase's then-current window (0.33.x → 0.34.x). Copying the window again only
  delays the next failure. sase's `publish.yml` `sync-release-metadata` job moves sase's
  window to the newest published stable core at each release, and `sase-core-rs 0.36.0`
  is already on PyPI (sase-core master is 0.36.0). The next sase release will move to
  0.36.x and break this plugin again.
- The plugin's upper bound does no work. `src/sase_research_artifacts/` never imports
  `sase_core_rs`: every core interaction goes through `sase`, and sase caps its own core
  version. Sibling plugins (`sase-github`, `sase-telegram`) declare no core requirement
  at all. The floor still matters: it is the published-minimum core that carries
  weighted-queue / runner-capacity support, and the release smoke checks it.

Verified locally: with sase master as the `-e` override, the current pin reproduces the
exact CI resolver error, and `sase-core-rs>=0.35.0` resolves cleanly (picking 0.35.1).
The PyPI `sase-core-rs==0.35.0` wheel exposes `runner_capacity_snapshot`, and
`runner_capacity_policy_schema_version()` returns 6, which meets the `>= 4` check.

## Fix

Change the plugin's requirement to a floor-only `sase-core-rs>=0.35.0`. The floor
matches sase master's current floor, which will also be the floor of the unreleased
`sase 0.17.2` that this plugin already requires. Drop the ceiling so future sase core
ratchets can't break this plugin again. Then update every place that encodes the old
window.

All work is in the linked `sase-research-artifacts` repo. Open it with
`sase repo open sase-research-artifacts -r "<why>"`, read its `AGENTS.md`, and work only
in the printed path. No changes to the sase repo itself.

### 1. `pyproject.toml`

- Replace `"sase-core-rs>=0.34.23,<0.35.0"` with `"sase-core-rs>=0.35.0"`.
- Rewrite the comment above `dependencies`. Keep the sase 0.17.2 weighted-queue
  sentence. Replace the "core window matches current SASE ... 0.34.x cohort" text with
  the new policy: the direct core requirement is a floor only. The floor is the
  published-minimum core with the runner-capacity APIs, aligned with sase's current core
  floor. The plugin never imports `sase_core_rs`, so sase owns the ceiling, and a plugin
  ceiling only breaks CI whenever sase moves its core window.

### 2. `tests/test_ci_install_contract.py`

- Constants: `PLUGIN_CORE_REQUIREMENT = "sase-core-rs>=0.35.0"`,
  `PUBLISHED_MINIMUM_CORE = "sase-core-rs==0.35.0"`. Change
  `INCOMPATIBLE_CORE = "sase-core-rs==0.34.73"`, the newest published core below the new
  floor.
- `test_plugin_core_window_accepts_installed_sase_floor`: replace the 0.34 membership
  asserts with:
  - `0.35.0`, `0.35.1`, and `0.36.0` are in the plugin specifier. `0.36.0` stands in for
    a future sase core ratchet.
  - `0.34.73` is not.
  - No specifier item uses `<`, `<=`, `==`, `~=`, or `!=`, which proves the requirement
    is floor-only.

  Keep the existing `sase_floor in plugin_core` / `sase_floor in sase_core` checks. Add
  a short comment explaining why there is deliberately no ceiling.

- Add a small test that no file under `src/sase_research_artifacts/` contains
  `sase_core_rs` or `sase-core-rs`. That invariant is what justifies dropping the
  ceiling. The failure message should say to revisit the no-ceiling policy if the plugin
  starts calling the core directly.
- `test_wheel_contract_is_source_coordination_not_published_minimum`: the literal
  `assert 'startswith("0.34.")' in wheel_test` must go. Assert that the wheel test
  contains the new floor-check expression from step 3, and that it contains no
  `startswith("0.3` version pin.

### 3. `tests/test_wheel_contract.py`

- `MINIMUM_SASE_CORE_RS_VERSION = "0.35.0"`. Delete `MAXIMUM_SASE_CORE_RS_VERSION`.
- In `test_distribution_artifacts_use_renamed_identity`, the `sase-core-rs`
  `Requires-Dist` assertion should require `>=0.35.0` and no `<` in that requirement.
- In the installed-venv `smoke_script` string (a plain, non-f-string), replace
  `assert version("sase-core-rs").startswith("0.34.")` with a floor-only check that
  works without `packaging` in the fresh venv. For example, compare
  `tuple(int(p) for p in version("sase-core-rs").split(".")[:2]) >= (0, 35)`. This lane
  maturin-builds sase-core **master** (currently 0.36.0), so the check must accept any
  version at or above 0.35. Keep the `runner_capacity_*` asserts.

### 4. `.github/workflows/publish.yml` (`install-smoke-published-minimum`)

- Install step: `"sase-core-rs==0.34.23"` → `"sase-core-rs==0.35.0"`.
- Smoke script: `== "0.34.23"` → `== "0.35.0"`.
- `Refuse older published core` step: `"sase-core-rs==0.33.0"` →
  `"sase-core-rs==0.34.73"`. This must stay in sync with `INCOMPATIBLE_CORE`, because
  the contract test checks for it.
- Don't touch `ci.yml`. The coordinated-source CI wiring stays as it is.

### 5. Docs

- `README.md` (Installation), `docs/configuration.md` (Requirements), and `AGENTS.md`
  (Architecture "Depends on ..." bullet) all say `sase-core-rs>=0.34.23,<0.35.0`. Change
  them to `sase-core-rs>=0.35.0`. In `AGENTS.md`, replace "the core window matches
  current SASE" with one clause on the floor-only policy: sase owns the core ceiling,
  and this plugin only raises the floor when it needs newer core capabilities.
- Leave `CHANGELOG.md` alone. release-please owns it.

## Verification

1. From the opened repo root, run `sase tool run check` (lint + tests; `check` is
   guarded, so don't run bare `just check`). It must pass on both the lint and test
   legs. It exercises the real `just install` resolution against the linked sase
   checkout and sase-core checkout, which is the step that failed in CI.
2. `grep -rn '0\.34\.23\|<0\.35\.0\|startswith("0\.34' .` (excluding `.git`/`.venv`)
   returns nothing.
3. Optional: `just test-wheel` builds a real wheel plus a maturin core build and takes
   minutes. If it runs, hand it to `/sase_monitor` instead of blocking. It should pass
   against the 0.36.0 source-built core.
4. Commit with a Conventional Commit message such as
   `fix(compat): make sase-core-rs requirement floor-only at 0.35.0`, through the normal
   host-owned completion. The GitHub CI run on the pushed commit is the final
   confirmation.

## Out of scope (note in the final report, don't fix)

- `install-smoke-published-minimum` pins `sase==0.17.2`, which is not on PyPI yet (the
  latest is 0.17.1), so the plugin's release PR can't publish until sase 0.17.2 ships.
  If sase 0.17.2 ships with a 0.36.x floor, that lane's exact `sase-core-rs==0.35.0` pin
  (and this plugin's floor) must then move to sase 0.17.2's real floor. That is a
  release-time decision, not a CI fix.
- CI tracks sase-core `master` rather than sase's `sase-core-revision.txt` pin, and it
  logs a Node.js 20 deprecation warning for `actions/checkout@v4`, `setup-uv@v4`, and
  `setup-just@v2`. Neither caused this failure.
