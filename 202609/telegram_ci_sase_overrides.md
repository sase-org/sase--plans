---
tier: tale
title: Fix sase-telegram CI by routing sase through a uv overrides file
goal:
  sase-telegram CI (and the publish install-smoke job) install sase from the coordinated
  local checkout instead of an unresolvable PyPI release, and go green.
size: small
proposed_by: bbugyi200.athena.0p8
create_time: 2026-09-22 09:50:14
status: wip
---

# Fix sase-telegram CI: route `sase` to the local checkout with a uv overrides file

All changes are in the **`sase-telegram`** linked repo (open it with
`sase repo open sase-telegram`). No changes to the sase repo itself.

## Diagnosis

`actstat` shows every sase-telegram `CI` run failing since 2026-09-20 ~18:20 UTC (last
green: `chore(master): release 0.4.19`). Both matrix jobs die in step 9 "Install
dependencies", inside `just install`:

```
uv pip install --python '.venv/bin/python' -e ".[dev]"
error: No solution found when resolving dependencies
  cause: Because only sase-core-rs>=0.34.23 is available and sase>=0.17.0 depends on
         sase-core-rs>=0.32.16,<0.33.0, we can conclude that sase>=0.17.0 cannot be used.
         And because sase-telegram==0.4.19 depends on sase>=0.17.0 ...
```

Root cause:

- `pyproject.toml` declares `sase>=0.17.0`, and `just install` first resolves `.[dev]`
  **from PyPI**; only afterwards does it overlay the local sase checkout with
  `--no-deps -e`.
- Every `sase` release on PyPI pins an old core window (`0.17.0`/`0.17.1` →
  `sase-core-rs>=0.32.16,<0.33.0`; older releases pin even older windows).
- PyPI now only hosts `sase-core-rs` `0.34.23`–`0.34.72` (older releases are gone), so
  no published `sase` is installable and the PyPI resolve step fails before the local
  overlay ever runs. Nothing in sase-telegram changed; the index changed underneath it.

`sase-research-artifacts` already hit and solved this: its Justfile writes a
`.sase-overrides.txt` containing `-e <local sase checkout>` and passes `--overrides` to
every `uv pip install`, so the `sase` requirement resolves to the coordinated source
tree (whose own pin, `sase-core-rs>=0.34.71,<0.35.0`, is satisfiable on PyPI), and then
`maturin develop` overwrites `sase-core-rs` with the coordinated sase-core build. Port
that pattern.

The `Publish` workflow's `install-smoke` job has the same latent bug
(`uv pip install dist/*.whl` into a fresh venv resolves `sase` from PyPI) and will fail
on the next release (release PR `0.4.20` is pending), so fix it too, again mirroring
sase-research-artifacts' `publish.yml`.

## Changes

### 1. `Justfile`

- Add `sase_overrides_file := clean(repo_dir / ".sase-overrides.txt")` next to the other
  path variables.
- Add a `_write-sase-overrides: _validate-local-sase` recipe that writes
  `printf -- '-e %s\n' {{ quote(local_sase_source) }} > {{ quote(sase_overrides_file) }}`,
  with a short comment explaining why (route `sase` to the coordinated checkout;
  `sase-core-rs` resolves from PyPI then gets overwritten by
  `_install-local-sase-core`).
- `install`: depend on `_write-sase-overrides`; change the project install to
  `uv pip install --python {{ quote(venv_python) }} --overrides {{ quote(sase_overrides_file) }} -e ".[dev]"`,
  keep `just _install-local-sase-core`, and drop the now-redundant trailing
  `--no-deps -e <local sase>` line (sase is already installed editable from the local
  source via the override).
- `_setup`: depend on `_write-sase-overrides`; use the same `--overrides` flag in its
  fresh-venv install; drop the trailing `--no-deps -e` overlay the same way.
- Add an `install-source-sase python` recipe (copy of sase-research-artifacts' version)
  that, for an arbitrary interpreter, runs `uv pip install --no-deps -e <local sase>`,
  installs `maturin`, and runs `maturin develop --release` in the local
  `crates/sase_core_py` with `VIRTUAL_ENV` pointed at that interpreter's venv and
  `PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1`. Used by the publish smoke test.

### 2. `.gitignore`

Add `.sase-overrides.txt`.

### 3. `.github/workflows/ci.yml`

The "Install dependencies" step currently runs `just install` and then repeats the
maturin build and the `--no-deps -e .sase-deps/sase` overlay by hand. Reduce it to just
`run: just install` (the Justfile now does all of it). Keep the `.sase-deps/sase` /
`.sase-deps/sase-core` checkouts and the pinned `extractions/setup-just@v2` block
unchanged (`tests/test_justfile.py::test_ci_pins_authenticated_just_setup` asserts it).

### 4. `.github/workflows/publish.yml` (`install-smoke` job)

Mirror sase-research-artifacts:

- Add `actions/checkout@v4` (for the Justfile), checkouts of `sase-org/sase` →
  `.sase-deps/sase` and `sase-org/sase-core` → `.sase-deps/sase-core` (token
  `${{ secrets.SASE_RELEASE_TOKEN || github.token }}`), `extractions/setup-just@v2`
  (same pinned `just-version: "1.58.0"` + `github-token` as ci.yml), and
  `dtolnay/rust-toolchain@stable`.
- Install step becomes:
  ```
  uv venv --python 3.12 /tmp/smoke-venv
  printf -- '-e %s\n' "$(realpath .sase-deps/sase)" > /tmp/sase-overrides.txt
  uv pip install --python /tmp/smoke-venv/bin/python --overrides /tmp/sase-overrides.txt dist/*.whl
  just install-source-sase /tmp/smoke-venv/bin/python
  ```
- Leave the smoke-check commands unchanged.

### 5. `tests/test_justfile.py`

- Update `test_install_dry_run_installs_project_before_local_sase` (rename to something
  like `test_install_dry_run_routes_sase_through_overrides`): assert the dry-run output
  contains the overrides write (`-e` + the `.sase-deps/sase` path, targeting
  `.sase-overrides.txt`), the
  `uv pip install --python '.venv/bin/python' --overrides '<repo>/.sase-overrides.txt' -e ".[dev]"`
  line, and `just _install-local-sase-core`, in that order; assert the old
  `--no-deps -e '<source>'` overlay is gone.
- Update `test_setup_dry_run_overlays_local_sase` likewise (overrides written, core
  installed).
- Add a test that actually runs `just _write-sase-overrides` in a temp repo with a fake
  `.sase-deps/sase` and asserts `.sase-overrides.txt` contains exactly
  `-e <resolved source>\n`.
- Add a dry-run test for `install-source-sase /tmp/x/bin/python` asserting the
  `--no-deps -e <source>` install and the `maturin' develop --release` /
  `PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1` lines.
- Optionally add a workflow assertion that the `publish.yml` `install-smoke` job uses
  `--overrides` and `just install-source-sase`.

Use `just --dry-run` exactly as the existing tests do; check the exact quoting just
prints before hard-coding expected strings.

## Verification

In the sase-telegram checkout:

1. Reproduce the failure first:
   `rm -rf .venv && uv venv .venv && uv pip install --python .venv/bin/python -e ".[dev]"`
   should fail with the same "No solution found" error.
2. After the changes, from a fresh venv: `rm -rf .venv && just install` succeeds, then
   `just check` (lint + tests) passes. Confirm
   `.venv/bin/python -c "import sase, sase_core_rs, sase_telegram"` works and that
   `sase` resolves to the local checkout (`uv pip show --python .venv/bin/python sase`
   shows an editable location).
3. Simulate the publish smoke locally: `uv build`, then run the four install-smoke
   commands above against a throwaway venv (with the overrides file pointing at the
   local sase checkout that `just _local-sase-source` prints) and run the smoke-check
   commands.
4. `git status` should not show `.sase-overrides.txt`.

After landing, re-run `actstat` (or `gh run list -R sase-org/sase-telegram -w CI`) to
confirm the master CI run and the pending `release 0.4.20` PR run go green.

## Out of scope / follow-up

Published `sase` on PyPI (latest `0.17.1`) is itself uninstallable for end users because
its `sase-core-rs<0.33.0` pin no longer resolves. That is a sase release problem, not a
sase-telegram one; the implementer should file a task bead (via `/sase_new_task`) if one
does not already exist, rather than touching sase-telegram's `sase>=0.17.0` floor.
