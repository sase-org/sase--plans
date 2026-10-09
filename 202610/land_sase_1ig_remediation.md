---
tier: tale
size: small
title:
  "Finish landing sase-1ig: docs link, mixed-mode agreement, build-check interpreter,
  then close the epic"
goal:
  "Fix the three defects the sase-1ig landing found: the Deploy Docs build broken by a
  link to a file outside docs/, `just install-dev` failing verification when a plugin
  comes from PyPI, and the pre-swap build check compiling against the wrong interpreter.
  Then close epic sase-1ig."
proposed_by: bbugyi200.athena.sase-1ig.land
bead: sase-1ig
create_time: 2026-10-09 02:41:03
status: wip
---

- **PARENT:**
  [202610/just_install_pypi_dev_venv.md](https://github.com/sase-org/sase--plans/blob/main/202610/just_install_pypi_dev_venv.md)
- **BEAD:**
  [sase-1ig](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ig/README.md)

# Plan: finish landing sase-1ig

Epic **sase-1ig** ("just install / install-dev / install-venv: three honest install
commands") has all 11 phases closed. The land agent verified the phases and triaged
every follow-up. Its notes are recorded on the bead: read them with
`sase bead read sase-1ig -r "..."`. Three defects caused by the epic remain, and this
tale fixes them. This tale has no land agent and nothing resumes the landing afterwards,
so its **last step is the epic closeout** (step 4).

Do not run `just install` or `just install-dev` from the agent. They refuse inside
agents, and real global installs are human-only. Do not run `just check-full`.

## 1. Deploy Docs: replace the out-of-tree INSTALL.md link

The Deploy Docs workflow has failed on every master commit since 29724dc042
(sase-1ig.11), with this error:

```text
WARNING - Doc file 'development.md' contains a link '../INSTALL.md#installing-from-a-checkout',
but the target '../INSTALL.md' is not found among documentation files.
Aborted with 1 warnings in strict mode!
```

`mkdocs.yml` sets `docs_dir: docs`, and `INSTALL.md` lives at the repo root. In
`docs/development.md`, in the "Your `sase` versus this checkout's `.venv`" subsection
(the "See [Installing from a checkout](../INSTALL.md#installing-from-a-checkout)"
sentence), change the link target to the absolute GitHub URL
`https://github.com/sase-org/sase/blob/master/INSTALL.md#installing-from-a-checkout`.
The INSTALL.md heading is `## Installing from a checkout`, so the anchor matches. Leave
the link text unchanged.

- Check that no other file under `docs/` links outside `docs/`. Run
  `grep -rn '](\.\./' docs/*.md`; it should print nothing after the fix.
- Verify with `just docs-check`, which runs `mkdocs build --strict`. It must exit 0.

## 2. Accept `mixed` update mode when the plan intentionally sources plugins from PyPI

`just install-dev` uses this plugin source policy (engine-core): for each receipt
plugin, use its durable editable path, else a durable sibling checkout, else PyPI. With
any PyPI-sourced plugin (or any `--with` package that has no checkout), the installed
receipt has an editable host plus a non-editable plugin. `sase update` then classifies
that install as `mixed` (`src/sase/main/update_routing.py::update_mode`: `has_dev` and
`has_managed`). `mixed` is a supported mode: its dry run carries both a dev plan and a
managed upgrade argv (`src/sase/main/update_handler_dry_run.py`).

The bug: `tools/_sase_install_run.py::classify_update_agreement` returns
`agrees=False, warning_only=False` for any mode other than `"dev"`. That makes
`verify_dev_install` fail with exit 1 after the swap has already succeeded.

Fix:

- Give `classify_update_agreement` a keyword-only parameter `allow_mixed: bool = False`.
  - When `allow_mixed` is true, treat `mode == "mixed"` like `"dev"`: run the same dev
    section checks (published-wheel core, unhealthy reconcile step, fetch-error
    inconclusive).
  - On success, return a detail that names the mode, for example
    `sase update agrees: mode "mixed" (PyPI plugins stay managed), no repair planned`.
  - `"managed"` and any other mode still fail as today. `"mixed"` without `allow_mixed`
    still fails with the existing message.
- In `verify_dev_install`, compute
  `allow_mixed = any(row.role == "plugin" and row.target.kind == "pypi" and row.kind != "remove" for row in plan.rows)`
  (the `PlanRow`/`TargetSource` fields from `tools/_sase_install_plan.py`) and pass it
  through.
  - Before relying on `row.kind != "remove"`, check how removed plugins are represented
    in the plan rows.
  - If a `--with` package that resolves to PyPI is represented differently, include it
    too.
- Do the same for PyPI mode, where a mismatch is only a warning today. In the PyPI
  verify block (`verify_pypi_install`, the `agree_doc.get("mode") == "managed"` branch),
  also accept `"mixed"` when the plan intentionally keeps an editable plugin: the
  unpublished-plugin fallback, a plugin row whose `target.kind == "editable"`. This
  stops `just install` from printing a spurious "did not report managed (inconclusive)"
  ⚠ for that case. Keep every other non-managed result as the existing warning.

Tests, in `tests/sase_install/test_run_dev.py` next to the existing `test_agreement_*`
cases, plus a PyPI case in `tests/sase_install/test_run_pypi.py`:

- `mixed` with `allow_mixed=True` and a clean dev section agrees.
- `mixed` with `allow_mixed=True` but a published-wheel core row still fails.
- `mixed` without `allow_mixed` still fails (the current behavior stays pinned).
- An end-to-end dev run whose plan has a PyPI-sourced plugin and whose fake
  `sase update -n -j` reports `"mode": "mixed"` exits 0 with the success summary. Reuse
  the existing hermetic harness (`tests/_sase_install_testkit.py`, the fake `sase` used
  by `test_update_repair_fails_dev_verify`).
- A PyPI run that keeps an unpublished editable plugin and gets `mixed` from the fake
  update records an ok `update-agreement` check, not a warning.

## 3. Pin the tool interpreter in the pre-swap build check

The plan required `tools/_sase_install_core.py::pre_swap_build_check` to compile "with
the same `CARGO_TARGET_DIR`, environment, and **interpreter** that `rust-dev-install`
uses, so the post-swap `maturin develop` reuses the compiled artifacts". The target dir
and environment match, but the interpreter does not.
`uv run --no-project --with maturin` replaces `VIRTUAL_ENV` with a random ephemeral
environment. The check passes no `--interpreter`, so maturin builds against uv's
ephemeral Python. The land agent measured this against a scratch tool venv and sase-core
0.37.0:

| Step                                               | Today                                 | Interpreter pinned |
| -------------------------------------------------- | ------------------------------------- | ------------------ |
| cold build check                                   | 238 s                                 | (same cold cost)   |
| repeat build check (warm)                          | 44.6 s (rebuilds pyo3\*)              | 2 s                |
| post-swap `maturin develop` (rust-dev-install env) | 42 s (rebuilds pyo3\* + sase_core_py) | 1 s                |

The heavy `sase_core` crate was reused in both columns. The 40 s difference is the pyo3
configuration changing with the interpreter and `VIRTUAL_ENV`.

Fix, in the `uv`-available branch of `pre_swap_build_check`:

- When `tool_python` exists as a file, run
  `uv run --no-project --with maturin -- env VIRTUAL_ENV=<tool venv> maturin build --profile <RUST_DEV_PROFILE> --interpreter <tool_python> --out <outdir>`.
  - `<tool venv>` is `Path(tool_python).parent.parent`, which `_cargo_env` already
    computes.
  - Use the absolute `env` from `shutil.which("env", path=...)`, falling back to
    `/usr/bin/env`.
  - Keep `cwd`, the target dir, the timeout, and the result handling unchanged.
- When `tool_python` does not exist (a fresh install with no tool env yet), keep today's
  command. There is nothing to match, and the swap creates the env.
- Update the docstring to say the interpreter is pinned when the tool env exists.
- Leave the `cargo check` fallback unchanged.

Tests, in `tests/sase_install/test_build_check.py`, which already puts a fake
`uv`/`maturin` on PATH: make the fake `uv` record its argv. Then assert that:

1. With an existing `tool_python`, the argv contains `--interpreter <tool_python>` and
   the `env VIRTUAL_ENV=<venv>` wrapper.
2. With a missing `tool_python`, neither appears.

The existing pass and fail tests must keep passing.

A live measurement is optional. Do not run a real global install. If you do measure,
record the warm-path timings in your close note.

## 4. Verify, then close epic sase-1ig (final step)

Verification:

- `.venv/bin/python -m pytest -q tests/sase_install tests/test_justfile_sase_core_dir.py tests/test_justfile_lint.py`
  must be green. Use `.venv/bin/python -m pytest`, not `uv run pytest`, which re-syncs
  the published core.
- `just docs-check` must exit 0.
- `sase tool run check` (the guarded `just check`).
  - These failures are known on master 974d44aa9b and are not this tale's:
    `test_macro_string_literals_avoid_xprompt_terms` (sase-1hr),
    `test_directive_completion_includes_representative_descriptions`, and
    `test_tab_through_every_subcommand_reaches_the_last_with_a_highlight` (repairs
    pending on sase-1i5.9.1.2.1.7), and the `lint (test waits)` failure in
    `tests/ace/tui/test_plan_decision_ace_stale.py` (recorded on epic sase-1hi).
  - Confirm that any other failure reproduces on a clean base before treating it as
    pre-existing.

Closeout of epic **sase-1ig** (this epic has no `parent_bead`, so nothing above it needs
handling):

1. Run `sase bead epic-symbols sase-1ig`. It printed "No --epic-symbol entries" at
   landing time. If any entries appear now, resolve each one per the Symvision
   epic-whitelist policy: wire it up, privatize it, add a non-test pragma, or delete it.
   Re-key an entry only if a still-open bead needs the exemption. Do not leave entries
   behind; `sase bead close` refuses while any remain.
2. Close the epic normally. Never use `--force`; all 11 phases are already closed.

   ```bash
   sase bead close sase-1ig --note "<verification>"
   ```

   The note must cover:
   - The land agent verified all 11 phases against source and commits (6f240b3c96,
     e5c09c19f5, fa0de348ee, d0b6e2e99d, 62f256008b, 10f5e2169f, 33bc9ca36c, 29724dc042,
     sase-core 4ea91b95, the four plugin-repo rename commits, chezmoi 0ed26344).
   - 328 targeted epic tests passed, and the agent guard refused through both recipes.
   - Plugin-repo CI was green, and `just symvision` was clean.
   - Follow-up triage is recorded in the LAND TRIAGE note: sase-1ij was created, a
     DISCOVERED ISSUE went to sase-1hi, and the rest were declined with reasons.
   - The sase-core pin is left for the core-pin-ratchet workflow, because no sase change
     needs 4ea91b95.
   - This tale fixed the Deploy Docs link, mixed-mode agreement, and the build-check
     interpreter pin. Include the test and docs-check results you observed.

3. Run `just symvision` and confirm it reports clean.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. Today it says
   `status: wip`. The file is the PLAN path printed by
   `sase bead read sase-1ig -r "Need the plan path to mark it done"`
   (`plan:202610/just_install_pypi_dev_venv.md` in the plans repo). Change only that
   field.

Out of scope, left to the human after landing: redeploy the generated `sase_monitor`
skill with `sase skill init --force` from a clean merged tree (it writes the global
chezmoi source), then run `chezmoi apply`.
