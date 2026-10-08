---
tier: epic
title: 'just install / install-dev / install-venv: three honest install commands'
goal: '`just install` installs the latest sase release from PyPI as the user''s `sase`
  command, `just install-dev` installs this checkout plus its pin-paired sase-core
  (editable) in exactly the shape `sase update` maintains, and today''s venv recipes
  live on as `just install-venv*`. Every consumer is migrated: CI, the tool catalog,
  runtime remedies, docs, memory, sase-core strings, the plugin repos, and the chezmoi
  `acei`/`aceii` installers.

  '
decisions:
  lint_and_test_note:
    ask: Update lint_and_test.md to name `just install-venv` wherever it says `just
      install`?
    memory:
    - lint_and_test.md
    default: true
    requested: These commands are necessary but their names are not intuitive. Can
      we rename them to `just install-venv` / `just install-venv-*`? This will likely
      require updating some memory files.
    answer: true
  symvision_note:
    ask: Update symvision.md's verify step to run `just install-venv`?
    memory:
    - symvision.md
    default: true
    requested: These commands are necessary but their names are not intuitive. Can
      we rename them to `just install-venv` / `just install-venv-*`? This will likely
      require updating some memory files.
    answer: true
  human_only_record:
    ask: Add a decisions record that the global installers are human-only (agent refusal
      plus explicit bypass)?
    memory:
    - decisions
    default: true
    requested: Review the just_install_pypi_dev_venv_split.md file in the research
      sidecar repo for context and inspiration before planning. I agree with all of
      the requirements recommended in that research file.
    answer: true
phases:
- id: venv-rename
  title: Rename the venv recipes to install-venv and park bare install
  depends_on: []
  size: medium
  description: 'venv-rename: consolidate the three copy-pasted venv recipes into `_install-venv
    EXTRAS`, expose them as the `[install]` group''s `install-venv*`, park bare `install`
    behind a loud exit-2 placeholder, and migrate CI, the tool catalog, tests, docs,
    tools/ strings, the sase_monitor skill source, and the lint_and_test/symvision
    memory notes.'
- id: engine-core
  title: Installer engine foundation and dry-run planning
  depends_on: []
  size: medium
  description: 'engine-core: create the stdlib-only `tools/sase_install` engine with
    its CLI, agent and durable-source guards, receipt/env snapshot, plugin source
    policy, consequential-change classification, argv/overrides parity with `sase.uv_tool`,
    the plan panel and JSON renderers, and the confirmation policy, with `-n` working
    for both modes.'
- id: remedies
  title: Context-aware reinstall remedies in runtime code
  depends_on:
  - venv-rename
  size: small
  description: 'remedies: add one install-context helper that picks `sase update`,
    `just install-dev`, or `just install-venv` and route every src/ reinstall hint
    (core/rust.py, query facades, doctor, completion finalizer) through it; doctor''s
    import-root-drift warning stops suggesting a global reinstall.'
- id: rust-recipes
  title: Make the Rust dev-install recipes honest
  depends_on:
  - venv-rename
  size: small
  description: 'rust-recipes: make `rust-dev-install` write the core source stamp,
    make the `*-uv-tool` recipes fail when uv or the tool env is missing, give the
    rust recipes `[group(''rust'')]` and `[doc]`, and align `sase update --to dev`''s
    core step with the dev-update `rust-dev-install-uv-tool` shape.'
- id: engine-pypi
  title: Execution pipeline and the live `just install`
  depends_on:
  - venv-rename
  - engine-core
  size: medium
  description: 'engine-pypi: build the shared pipeline (progress renderer, log, code-swap
    lock, backup and restore command, uv swap, verification, scheduler restart, summary)
    and ship PyPI mode end to end by replacing the bare `install` placeholder.'
- id: dev-core-prep
  title: sase-core pairing and pre-swap preparation
  depends_on:
  - engine-core
  size: medium
  description: 'dev-core-prep: implement the core pairing rule (resolve, clone, fetch,
    fast-forward, pin containment), the `--sync` fatal sync gate, the version-window
    check, the SASE_ALLOW_STALE_CORE escape hatch, and the pre-swap build check, all
    surfaced as dev plan rows.'
- id: plugin-repos
  title: Rename install to install-venv in the plugin repos
  depends_on:
  - venv-rename
  size: medium
  description: 'plugin-repos: in sase-github, sase-telegram, sase-research-artifacts,
    and sase-listen, rename the venv recipe to a grouped, documented `install-venv`,
    keep `install` as a private forwarding alias that names the new recipe, and migrate
    each repo''s CI, tests, and docs.'
- id: engine-dev
  title: The live `just install-dev`
  depends_on:
  - engine-pypi
  - dev-core-prep
  - rust-recipes
  size: medium
  description: 'engine-dev: wire dev mode through the pipeline (dev swap argv and
    overrides, core plus LSP re-apply via `rust-dev-install-uv-tool`, stamp check,
    dev verification including `sase update -n -j` agreement, repeat-run no-op) and
    add the `install-dev` recipe.'
- id: core-strings
  title: Point sase-core's remedies at the new names
  depends_on:
  - engine-dev
  - plugin-repos
  size: small
  description: 'core-strings: in linked sase-core, change the triage environment remedies
    and verdict remedy to `just install-venv`, change the bead JSONL unknown-operation
    remedy to `sase update` / `just install-dev`, regenerate goldens and tests, and
    let the dual-repo commit move sase''s core pin.'
- id: chezmoi-acei
  title: Retire the chezmoi installers into install-dev
  depends_on:
  - engine-dev
  size: small
  description: 'chezmoi-acei: point `acei`/`aceii` at `just install-dev --sync -y`
    on the durable sase checkout and delete the drifted install_sase_github/install_sase_google
    scripts and their bash test.'
- id: install-docs
  title: Document the three commands and record the human-only rule
  depends_on:
  - engine-dev
  size: small
  description: 'install-docs: add INSTALL.md''s checkout section, a your-sase-versus-the-checkout''s-venv
    guide in docs/development.md, README/CONTRIBUTING pointers, and the new `global-install-is-human-only`
    decisions strand.'
proposed_by: bbugyi200.athena.0ym
decided_by: auto
create_time: 2026-10-08 18:23:25
status: wip
bead_id: sase-1ig
---

- **PROMPT:** [prompts/202610/just_install_pypi_dev_venv.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/just_install_pypi_dev_venv.md)
- **BEAD:** [sase-1ig](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ig/README.md)

# Plan: three honest install commands

## Why this shape

Today `just install` sets up this checkout's `.venv`. Elsewhere, `install` means "put
the program where I run it from": GNU make targets, `cargo install`, `pipx install`, and
`uv tool install` all use it that way. Even `just --list` gets it wrong: it shows
`install`'s description as `# distribution instead.`, because only the last comment line
is used. Nothing in the repo makes your `sase` run _this_ checkout with a matching
sase-core. Today that job is done by the drifted chezmoi `install_sase_github` /
`install_sase_google` scripts.

The research report
`research:202610/just_install_pypi_dev_venv_split/just_install_pypi_dev_venv_split.md`
is the design basis, and the user accepted all of its requirements. This plan follows
it, with six deliberate refinements:

1. **No `--core PATH` flag.** `install-dev` pairs with exactly the checkout
   `sase update` rebuilds from: `$SASE_CORE_DIR`, else `<sase checkout>/../sase-core`
   (the same rule as `src/sase/dev_update/_plan_core_checkout.py::core_checkout_dir`).
   With one knob, the two tools agree by construction. The Justfile's workspace-scoped
   `sase_core_dir` lookup exists for `.venv` builds inside SASE workspaces, and the
   global installer does not use it.
2. **Non-interactive global installs need `-y`.** Any non-dry run with no TTY and no
   `-y` prints the plan and exits 2, not only the consequential ones. A stray
   `just install` in a script or CI job can therefore never silently replace somebody's
   `sase`. At a terminal, only consequential changes prompt.
3. **The variant recipes are renamed outright.** `install-visual` and
   `install-terminal-smoke` have no callers outside this repo, and a forwarding alias
   would be a backward-compatibility branch that sase's flag policy makes us flag. Bare
   `install` gets a loud placeholder only until its installer lands later in this epic.
4. **The core is built before the swap.** A sase-core that does not compile never
   reaches `uv tool install`, so a failed build leaves the old install untouched.
5. **The installer never consumes the Rust prebuild cache.** `uv tool install --force`
   recreates the tool environment, so the editable core must be re-created by
   `maturin develop` (`just rust-dev-install-uv-tool`). A prebuild copy would end up
   next to a PyPI wheel.
6. **Fresh installs suggest `plugins.required`; they never install it** (research open
   question 3).

**Non-goals:**

- exact-pin core builds
- making the global core non-editable (research open questions 2 and 4)
- `sase update --to dev` delegating to `just install-dev` (later convergence)
- per-plugin-repo `install-dev`
- `just uninstall`
- side-by-side `sase` and `sase-dev`
- shell, completion, or provider setup
- Windows

**No feature flags.** The bare-`install` placeholder is a loud error, not a reachable
old branch. `just install` and `just install-dev` each land complete in a single phase.

## Target command surface

```text
[install]
    install *args               # Install the latest sase release from PyPI as your `sase` command
    install-dev *args           # Install this checkout + its paired sase-core as your `sase` command
    install-venv                # Set up this checkout's .venv for tests, lint, and benchmarks
    install-venv-terminal-smoke # install-venv plus real-terminal smoke-test dependencies
    install-venv-visual         # install-venv plus visual-snapshot test dependencies
```

The group uses only `[group]`, `[doc]`, `[private]`, and `[positional-arguments]`. CI
pins just 1.50.0, so do not use `[arg(...)]` flags (added in 1.53). A short comment
block above the group states the three destinations:

```just
# ── Installing sase ───────────────────────────────────────────────────────────
#   just install       your `sase` command ← the latest PyPI release
#   just install-dev   your `sase` command ← this checkout + its paired sase-core
#   just install-venv  this checkout's .venv ← tests, lint, benchmarks (never your `sase`)
```

|            | `just install`                             | `just install-dev`                                                     | `just install-venv*`                             |
| ---------- | ------------------------------------------ | ---------------------------------------------------------------------- | ------------------------------------------------ |
| Target     | uv tool env (`$(uv tool dir)/sase`)        | uv tool env                                                            | `./.venv` (`venv_dir`)                           |
| sase from  | PyPI latest, or `--version`                | `-e` this checkout, as on disk                                         | `-e .[dev…]`                                     |
| Core from  | the release's dependency (wheel)           | `$SASE_CORE_DIR` or `../sase-core`, must contain the pin, **editable** | `SASE_CORE_WHEEL` or the local build (unchanged) |
| Plugins    | receipt set, moved to PyPI where published | receipt set, editable where a durable checkout exists                  | `plugins.required` (unchanged)                   |
| Needs      | `uv`                                       | `uv`, `git`, `cargo`                                                   | unchanged                                        |
| Repeat run | no-op when current and healthy             | no-op when receipt, core identity, and health all match                | unchanged                                        |
| Guards     | agent refusal; confirm                     | agent refusal; confirm; durable sase **and** core                      | none (never guarded, per `guarded-recipes`)      |

## Shared engine contracts (engine-core, engine-pypi, dev-core-prep, engine-dev)

**Location and invocation.**

- The entry point is `tools/sase_install`, an extensionless Python script, with helper
  modules `tools/_sase_install_*.py`. They follow the `smoke_sase_tool_runs` +
  `_smoke_tool_runs_*.py` pattern: the entry point inserts its own directory on
  `sys.path`.
- Everything is **stdlib-only and never imports `sase`**, because it must work when the
  installed `sase` is missing or broken. The one exception: it may import
  `tools/_sase_core_source_identity.py`, which is already stdlib-only.
- Recipes run the engine through uv so that it works on a machine that has only uv:
  `uv run --no-project --quiet --python '>=3.12' -- python -I <tools/sase_install> <mode> "$@"`.
  - Verify the exact flag set and record it in the recipe comment.
  - Requirements: Python ≥ 3.12, isolation from an activated `.venv` and from
    `PYTHONPATH`, and offline operation when a suitable interpreter exists.
  - The recipe first checks `command -v uv` and prints the uv install link with exit 2
    if uv is missing.
- argparse `prog` is `just install` / `just install-dev`, so `--help` reads naturally.
  The help includes an examples epilog.

**Flags.** These mirror `sase update`:

- `-n/--dry-run`, `-y/--yes`, `-q/--quiet`, `-v/--verbose`, `-j/--json`
- `--with PKG` (repeatable; adds to the plugin set)
- `--python PY` (explicitly recreate the env on PY)
- `--force` (run the full pipeline even when already current)

PyPI mode adds `--version X`. Dev mode adds `--keep-plugin-sources` and `--sync`.

**Exit codes.**

| Code | Meaning                                                                  |
| ---- | ------------------------------------------------------------------------ |
| 0    | success, no-op, or dry run                                               |
| 1    | failure, or the user declined the prompt ("cancelled — nothing changed") |
| 2    | refusal, usage error, missing prerequisite, or non-TTY without `-y`      |
| 130  | interrupted                                                              |

**Agent guard.** The engine refuses with exit 2 when `SASE_AGENT` or `SASE_MONITOR_ID`
is set, unless `SASE_GLOBAL_INSTALL_BYPASS` holds a non-empty reason. When the bypass is
used, it prints a one-line note that names the reason.

```text
✗ just install-dev won't run inside a SASE agent — it replaces the global `sase` that every agent and the scheduler run.
  repair this workspace's .venv      just install-venv
  the user asked for a reinstall     propose it with /sase_gate and SASE_GLOBAL_INSTALL_BYPASS='<reason>'
```

**Durable sources (dev only).** Refuse, with exit 2, when the sase root or the resolved
core root is ephemeral. Use the same classification as
`sase.uv_tool.preflight._is_ephemeral_plugin_path`: under `managed_workspace_root()`
(`SASE_WORKSPACE_ROOT`, else the `_default_state_root()` rules), or containing a
`sase/repos/linked` or `sase/repos/external` path segment. The refusal names the durable
command, using the receipt's current editable sase path when that path is durable:
`just -f <durable checkout>/Justfile install-dev`.

**Consequential changes.** Each of these is marked ⚠ in the plan:

- a mode flip (editable ↔ PyPI host)
- a host retarget (an editable path that differs)
- any downgrade (including a local core 0.37 → PyPI 0.35)
- a plugin removal or source conversion
- a Python change
- a dirty core
- an `SASE_ALLOW_STALE_CORE` pairing

Fresh installs, same-source upgrades, and no-ops are not consequential.

- At a TTY without `-y`: prompt `Proceed? [y/N]` only for consequential plans.
- Without a TTY: every non-dry run needs `-y` (refinement 2).
- `-n` prints the plan and exits 0. It has no side effects beyond `git fetch`, and in
  particular it never writes the overrides file.

**Parity with `sase.uv_tool`.** These are enforced by tests that import `sase` (tests
may; the engine may not).

- **Swap argv:** byte-for-byte equal to
  `sase.uv_tool.commands.build_reinstall_set(primary, plugins, color="never", overrides=…)`
  for the same inputs.
- **Overrides:** same content (`-e <path>` per editable, plus a bare `sase-core-rs` line
  when the host is editable) and same path (`$SASE_HOME/uv/editable-overrides.txt`) as
  `sase.uv_tool.overrides.write_editable_overrides`.
- **Receipt reading:** same requirements as `sase.uv_tool.receipt.parse_receipt` on
  fixture receipts, including the live shape: editable entries, top-level `overrides`,
  `entrypoints`, and `[tool.options]`.
- **Ephemeral classification:** same answers as
  `sase.uv_tool.preflight._is_ephemeral_plugin_path`.
- **Lock:** same lock path and holder-payload shape as `sase.dev_update.code_swap_lock`.
- **Pin containment:** same answers as
  `sase.dev_update.core_pin.core_contains_revision`.

**Never pass `--python` to an existing env** unless the user asked with `--python`. The
plan shows `python 3.14.7 (kept)`, and verification asserts that `pyvenv.cfg` reports
the same version after the swap. If the hermetic test shows that uv does not preserve
the interpreter on `--force --reinstall`, pass the existing env's own interpreter
instead, and record that finding in the phase notes.

**Visual language.** This follows `sase update` (`src/sase/update_progress/`) and is
hand-rolled with ANSI codes:

- rounded cyan panels titled `just install-dev · …`
- step glyphs: `○` pending, braille spinner while running, `✓` green, `⚠` yellow, `✗`
  red, `–` skipped
- dim detail column, and durations formatted `0.4s` / `1:02`
- the panel width tracks the terminal (60–100 columns)
- `NO_COLOR` and non-TTY are honored
- Without a TTY, output is plain append-only `[mm:ss] ✓ title — detail (dur)` lines.
- `-q` prints only the final summary line.
- `-j` prints one JSON document to stdout (`schema_version: 1`) and nothing decorative.
- `-v` forces plain mode and streams subprocess output indented under `│`.
- Progress goes to stderr; plans, summaries, and JSON go to stdout.

## venv-rename: Rename the venv recipes to install-venv and park bare install

> [!decision] lint_and_test_note [!decision] symvision_note

**Justfile.**

- Replace the three copy-pasted bodies (`install`, `install-visual`,
  `install-terminal-smoke`) with one private `_install-venv EXTRAS: _venv` recipe.
  - It takes `SASE_CORE_WHEEL`, else builds the local core with `rust-install`. Detect
    the core the way `_setup` does: a checkout without `Cargo.toml` counts as absent.
  - It then runs `uv pip install … $(just _core-overrides-arg) -e ".[{{ EXTRAS }}]"`.
  - It then runs `_setup-required-plugins` for **every** variant. Today only `install`
    does, and that drift is a bug.
  - Use `[install-venv]` as the printf prefix.
- Add the public `[group('install')]` recipes `install-venv: (_install-venv "dev")`,
  `install-venv-visual: (_install-venv "dev,visual")`, and
  `install-venv-terminal-smoke: (_install-venv "dev,terminal-smoke")`, each with a
  one-line `[doc(...)]`.
- Add the group comment block from the surface section above.
- Bare `install` becomes `[group('install')]`,
  `[doc('Install the latest sase release from PyPI as your `sase` command')]`,
  `[positional-arguments]`, `install *args`. Until engine-pypi lands, it is a
  placeholder that exits 2 and prints:

  ```text
  ✗ `just install` is moving: it will install sase from PyPI as your `sase` command.
    setting up this checkout's .venv?   just install-venv
  ```

- Reword the remedies:
  - `_setup`'s "Then rerun 'just install'" becomes `'just install-venv'`.
  - `rust-install` and `rust-dev-install` become "Then rerun `just install-venv` (this
    checkout's .venv) or `sase update` (your installed `sase`)."
- Comments that say "install recipes" now say `install-venv`.

**CI.**

- `.github/actions/setup-sase/action.yml`:
  - `install-recipe` defaults to `install-venv`.
  - The allow-list becomes `install-venv|install-venv-visual`.
  - Update the input description.
- `ci.yml` (`visual-test`, `ace-page-group-isolation`) and `telemetry.yml`
  (`coverage-contexts`, `contention-test`) pass `install-recipe: install-venv-visual`.
- Jobs that rely on the default keep passing no input.
  `tests/test_github_actions_ci_master_gate.py` pins that, so leave it.

**Tool catalog.**

- In `sase/sase.yml`, rename the `install` tool to `install-venv` with argv
  `[just, install-venv]` and the description "Set up this checkout's .venv (editable
  sase, dev deps, local sase-core)".
- Fix the line-198 comment.
- Update `tools/_smoke_tool_runs_cases_basic.py` `FIVE_TOOLS` and
  `tests/test_tool_catalog.py`. ToolRun duration history restarts under the new name,
  which is expected.

**Tests.**

- `tests/test_justfile_sase_core_dir.py`: the parametrized recipe list becomes the new
  names.
- `tests/test_github_actions_setup_sase.py`: `INSTALL_RECIPE` values and the expected
  `just install-venv …` stdout.
- `tests/ace/tui/visual/renderer_env.py` remediation and `test_renderer_env.py`:
  `just install-venv-visual`.
- Assert-message strings that name the recipe (`tests/fakey/*`,
  `tests/test_sase_tool_runs_smoke.py`).
- Add a test that runs `just --list` and checks the `[install]` group's recipes and
  docs, and that bare `just install` exits 2 with the placeholder text. The engine-pypi
  phase replaces the second assertion.
- Leave incidental sample shell strings (`'just install && just check'` quoting
  regressions) and the `test_inline_memory.py` fixture untouched.

**tools/ strings** (all `.venv` contexts, so plain renames): `validate_sase_core_rs`
(both remedies), `check_bead_note_migration`, `setup_required_plugins` (docstring and
failure message), the `check_sase_core_rs_bindings` docstring, the
`validate_test_environment` comment, and the `src/sase/_linked_repo_workspaces.py`
docstring.

**Docs** (setup sections become `just install-venv`; drop the now-redundant `uv venv` /
`source` lines where the recipe bootstraps `.venv`): CONTRIBUTING.md (both sites), the
README Development block, `docs/development.md` (setup, verification list, :1100,
:1184), `docs/rust_backend.md`, `docs/tool.md` (including "`test` and `install-venv` are
never guarded"), `docs/monitors.md`, `docs/perf_runbook.md`, `docs/mobile_gateway.md`,
`docs/mobile_mvp_runbook.md`, `demos/README.md`, `tests/ace/tui/repro/README.md`, and
`tests/perf/README.md`. Do not touch CHANGELOG.md, which is generated.

**Skill source.** In `src/sase/macros/skills/sase_monitor.md:113`, the example becomes
`'just install-venv && just check'`. Per `generated_skills.md`, do **not** deploy.
Redeploying with `sase skill init --force` happens after the epic lands.

**Memory** (use `/sase_memory_write`, then `sase memory init`):

- `lint_and_test.md`: the recipe block line, the known-long-commands list, and the "you
  MAY need to run `just install-venv` before `just check`" sentence.
- `symvision.md`: its Verify step.
- Leave `decisions:guarded-recipes` untouched. Accepted records are immutable, and its
  claim stays true of the renamed venv recipe.

**Verify:**

- `just --list` shows the group cleanly.
- `just --dry-run install-venv-visual` shows the wheel branch.
- `git grep -nE 'just install([^-]|$)'` leaves only intended hits.
- `sase tool run check` passes.

## engine-core: Installer engine foundation and dry-run planning

**Plugin source policy.** The receipt decides which plugins are installed; the mode
decides where each one comes from. This matches `sase update --to`, and it avoids a dev
host running released plugins, which is the mixed state most likely to break.

- **Dev mode** sources each receipt plugin from, in order: (1) its current editable
  path, if that path is durable; else (2) a durable sibling
  `<sase checkout>/../<dist-name>` whose `pyproject.toml` declares that name; else (3)
  PyPI.
- **PyPI mode** moves every plugin to PyPI, except one that is not published there
  (`bugyi-chops`): it stays editable, with a ⚠ note.
- **`--keep-plugin-sources`** keeps each plugin's current source for one run, unless
  that source is ephemeral or missing.

**Scope.** Build the read-only half of the engine. Recipes are not wired yet; tests
drive `tools/sase_install` directly.

**Modules:**

- `tools/sase_install`: entry point; argparse with `pypi` and `dev` subcommands,
  dispatch, exit codes, and KeyboardInterrupt → 130.
- `tools/_sase_install_env.py`: guards and environment.
  - agent guard and bypass
  - ephemeral classifier
  - prerequisite probes with versions: uv for both modes; git and cargo for dev, where a
    missing one prints "install rustup, or use `just install`" with exit 2
  - `uv tool dir` and `uv tool dir --bin`
  - TTY and color detection
  - `SASE_HOME`
- `tools/_sase_install_state.py`: an `InstallState` snapshot, read without running the
  installed `sase`:
  - tool env path, and whether it exists
  - receipt requirements (via `tomllib`)
  - Python version from `pyvenv.cfg`
  - installed distributions from `*.dist-info` (name, version, and editable source from
    `direct_url.json`)
  - the core source stamp
  - whether the LSP binary exists in the tool `bin/`
  - the current mode (`pypi`, `dev`, `mixed`, or `none`)
- `tools/_sase_install_pypi.py`: a tiny PyPI JSON client (`/pypi/<name>/json`, 5 s
  timeout) for latest versions and "is this published?". Network failure is a ⚠ during
  `-n` and a preflight failure for real runs.
- `tools/_sase_install_plan.py`:
  - desired package set per mode (the source policy above, plus `--with`)
  - per-package plan rows: role, current and target source/version, a change kind
    (`keep`, `add`, `upgrade`, `downgrade`, `to-editable`, `to-pypi`, `retarget`,
    `remove`), a consequential flag, and a note
  - no-op determination (healthy-probe hooks are filled in by later phases)
  - swap argv and overrides content (both parity-tested)
  - a fresh install (no receipt) installs only the host plus `--with`, and uses
    `--force` when a tool dir exists without a receipt
- `tools/_sase_install_ui.py`:
  - style primitives
  - the plan panel (the dev and PyPI-flip mockups below are the design contract)
  - the agent-refusal and non-TTY messages
  - the `[y/N]` prompt
  - the JSON document (`schema_version`, `command`, `mode`, `dry_run`, `outcome`,
    `packages[]`, `python`, `target`, `warnings[]`, `consequential`, `noop`,
    `commands[]`, `steps[]`, `error`, `log_path`)

**Dev plan panel (contract).** dev-core-prep fills in the core rows.

```text
╭─ just install-dev · your `sase` → this checkout (editable) ─────────────────────╮
│ sase           ~/projects/github/sase-org/sase        master @ 3412a9f · clean   │
│ sase-core-rs   ~/projects/github/sase-org/sase-core   master @ e411a39           │
│                contains pin e8606a5 (sase-core-revision.txt) · +1 commit         │
│ plugins        sase-github ✎  sase-telegram ✎  sase-research-artifacts ✎        │
│                bugyi-chops ✎                            ✎ editable checkout     │
│ python         3.14.7 (kept)                                                     │
│ target         ~/.local/share/uv/tools/sase             currently: dev @ 0ac86ad │
╰──────────────────────────────────────────────────────────────────────────────────╯
```

**PyPI flip panel (contract):**

```text
╭─ just install · your `sase` → PyPI ──────────────────────────────────────────╮
│ sase           dev 0.17.1 (editable)  →  0.17.1   PyPI                        │
│ sase-core-rs   0.37.0 local build     →  0.35.4   PyPI   ⚠ downgrade          │
│ sase-github    editable               →  0.4.2    PyPI                        │
│ bugyi-chops    editable               →  stays editable · not on PyPI         │
│ ⚠ Replaces your editable install from ~/projects/github/sase-org/sase.        │
╰───────────────────────────────────────────────────────────────────────────────╯
Proceed? [y/N]
```

**Tests** go in `tests/sase_install/`, which loads `tools/` the same way existing tool
tests do. Do not add `pytest.mark.contract`, because the contract manifest budget is
exact.

- guard matrix: env vars, bypass, and the ephemeral classifier
- every parity contract above except the lock and pin, which belong to later phases
- the plugin-policy table, with and without `--keep-plugin-sources`
- consequential classification
- the non-TTY policy
- fixed-width, no-color panel snapshots
- JSON shape

`tools/_lint-pyscripts` requires every `tools/` file to be referenced outside `tools/`,
so the tests must import each helper. Make sure the helper modules are type-checked:
extend `tools/typecheck_extensionless_tools` if `_*.py` helpers are not covered today.

## remedies: Context-aware reinstall remedies in runtime code

Add `src/sase/install_remedy.py`, a cheap module that only uses `importlib.metadata`,
`sys`, and the `uv_tool.detect` prefix rule. It has two functions.

`install_context()` returns one of:

- `uv_tool_dev`: `sys.prefix` is the uv tool env, and the host's `direct_url.json` is
  editable
- `uv_tool_release`
- `checkout_venv`: `sys.prefix` is `<dir>/.venv`, and `<dir>` has a Justfile and sase's
  `pyproject.toml`
- `other`

`reinstall_remedy()` returns the matching phrase:

| Context       | Remedy                                                                            |
| ------------- | --------------------------------------------------------------------------------- |
| dev           | "run `sase update` (or `just install-dev` from your sase checkout)"               |
| release       | "run `sase update` (or `uv tool install --force sase`)"                           |
| checkout venv | "run `just install-venv` in <dir>"                                                |
| other         | "reinstall sase in this environment (for example `uv tool install --force sase`)" |

Route every src/ hint through it:

- `core/rust.py` (`_PROJECT_INSTALL_HINT` / `_install_hint`). Keep the uv pip core
  repair command only for `uv_tool_release`: force-reinstalling a PyPI core clobbers a
  dev install.
- `core/query_corpus_facade.py:70` and `core/query_profile_corpus_facade.py:227`
- `doctor/checks_providers.py:209`
- `doctor/checks_runtime_environment.py` (`runtime.core` ERROR, Python < 3.12)
- `finalizers/_prepare_completion.py:226`

Doctor's "active sase import root differs from the current checkout root" warning
becomes: "To run this checkout's code, use its `.venv/bin/sase` (set it up with
`just install-venv`)." It never suggests a global reinstall, because that is the normal
state in agent workspaces.

Tests: unit-test `install_context()` with faked prefix, metadata, and filesystem; update
`tests/test_core_rust.py` and `tests/doctor/test_checks_runtime.py`.

## rust-recipes: Make the Rust dev-install recipes honest

- **Source stamp.** `rust-dev-install VENV` writes `<VENV>/.sase-core-rs-source.json`
  exactly as `rust-install` does.
  - Capture the identity with `tools/_sase_core_source_identity.py` after the checkout
    refresh and before the build.
  - Write it only after both the extension and the LSP install succeed.
  - Add a test next to the existing `rust-install` stamp coverage in
    `tests/test_justfile_lint.py`.
- **Fail closed.** `rust-install-uv-tool`, `rust-dev-install-uv-tool`, and
  `rust-lsp-install-uv-tool` exit 1 (not 0) when uv is missing or
  `$(uv tool dir)/sase/bin/python` does not exist, keeping their messages.
  - Callers (`sase update`, the mode switch, and later the installer) only run them
    after an install exists.
  - Recipe names are unchanged, because installed sase versions call them by name.
- **List hygiene.** Give every `rust-*` recipe `[group('rust')]` and a one-line `[doc]`,
  and fix the garbled multi-line descriptions.
- **Mode switch.** `src/sase/mode_switch/plan.py`'s core step becomes
  `just rust-dev-install-uv-tool`, run with the dev-update environment
  (`SASE_RUST_DEV_PROFILE=dev-update`) and the 3600 s build timeout. Mirror
  `dev_update/_plan_reconcile.py`'s `rust_dev_install` step, threading the timeout
  through `mode_switch/execute.py`. This way `sase update --to dev` leaves the editable
  core that the next `sase update` expects (research finding 5). Update
  `tests/mode_switch/`.

## engine-pypi: Execution pipeline and the live `just install`

Add `tools/_sase_install_run.py` (pipeline) and extend `_sase_install_ui.py` with the
live/plain/quiet progress renderers and summaries. PyPI mode then runs:

```text
preflight ─► plan ─► confirm ─► lock ─► swap ─► verify ─► restart ─► summary
```

- **Log.** Write every command and its output to
  `$SASE_HOME/logs/install/install-<UTC>-<pid>.log`. Failures print the log path and a
  20-line tail of the failed step.
- **Lock.**
  - Take the code-swap writer lock (`$SASE_HOME/locks/code-swap-v2.lock`,
    `flock LOCK_EX|LOCK_NB`, holder JSON with `op: "install.pypi"` / `"install.dev"`).
  - Retry for up to 120 s, showing the holder in the spinner detail. On timeout, exit 1
    and name the holder.
  - Honor `SASE_DISABLE_CODE_SWAP_LOCK=1`.
  - Advisory agent-runner holders become a ⚠ row ("N agent runners are running from this
    install").
  - Add a parity test against `sase.dev_update.code_swap_lock`.
- **Backup.** Before the swap, write `$SASE_HOME/install/last-install.json`. It holds
  the previous receipt requirements and a restore argv: `build_reinstall_set` of the
  previous receipt, with overrides if it was editable, plus "then
  `just rust-dev-install-uv-tool`" when the previous core was editable. Failures print
  this restore command. Never claim a rollback that did not happen.
- **Swap.** Write the overrides file only if editables remain, then run the parity argv
  (600 s timeout).
- **Verify.** Run from a neutral cwd, with `PYTHONPATH`, `PYTHONHOME`, and `VIRTUAL_ENV`
  removed, using the tool's own executables:
  - `sase` and `sase_core_rs` import from tool site-packages
  - `sase core health -j` reports `ok`
  - `sase version -j` reports a wheel host at the target version
  - the interpreter was kept
  - `sase update -n -j` reports `mode == "managed"` (120 s; a timeout or error is ⚠
    inconclusive, not ✗)
  - ⚠ when `command -v sase` is not the managed executable (for example, an active
    `.venv` shadows it, or the uv bin dir is not on PATH). Print the managed path, and
    never edit shell files.
- **Restart.** Before the swap, probe whether the scheduler is running, mirroring
  `axe/_process_probe.py` with stdlib (the `orchestrator.lock` flock, then pid-file
  liveness). If it was running, run `<bin>/sase scheduler restart` afterwards. A restart
  problem is ⚠ and never changes the exit code.
- **No-op.** If the host and every PyPI plugin are already at their target versions and
  a quick health probe passes, print
  `✓ sase 0.17.1 from PyPI is already installed — nothing to do (--force reinstalls)`
  and exit 0.
- **Fresh-install suggestion.** After a fresh install, probe the new install for missing
  required plugins, best-effort: run `<tool python> -I -c …` against
  `sase.plugins.required`, and print nothing on any error. If plugins are missing, print
  `Next: sase plugin install <names>`.
- **Summary.**

  ```text
  ✓ sase 0.17.1 from PyPI is installed (sase-core-rs 0.35.4 · 2 plugins)
    update later: sase update · develop on this checkout: just install-dev
  ```

  The pointer to `just install-dev` appears once that recipe exists. Until engine-dev
  lands, print only the `sase update` hint.

**Justfile.** Replace the placeholder body with the engine call, and update the
venv-rename test assertion.

**Tests:**

- Hermetic: fake `uv`/`sase` executables on PATH, plus temporary `HOME`, `SASE_HOME`,
  `UV_TOOL_DIR`, and `UV_TOOL_BIN_DIR`. Cover step sequencing and argv, exit codes, lock
  contention (a test holds the flock), scheduler-restart branches, failure output with
  the restore command, plain-renderer output, quiet and JSON modes, and no-op.
- A recipe-level test: `SASE_AGENT=1 just install` exits 2 with the refusal text.

## dev-core-prep: sase-core pairing and pre-swap preparation

Add `tools/_sase_install_core.py`, and feed its results into the dev plan rows.

**The pairing rule:**

```text
pin  := <sase checkout>/sase-core-revision.txt   (working tree; missing/malformed 40-hex → ✗)
core := $SASE_CORE_DIR | <sase checkout>/../sase-core      (must not be ephemeral)
1. core missing              → plan row "clone sase-org/sase-core → ../sase-core" (HTTPS, default branch)
2. pin object missing        → git fetch; still missing → ✗ "pin <sha> is not on the core's remote"
3. pin ⊄ HEAD, clean+behind  → plan row "fast-forward sase-core <a> → <b>"
4. pin ⊄ HEAD otherwise      → ✗ with exact remedies (pull/switch the core checkout, or SASE_CORE_DIR=…)
5. pin ⊆ HEAD                → ✓ "sase-core e411a39 (pin e8606a5 + 1)"; dirty → ⚠ row (consequential, allowed)
then: tools/validate_sase_core_rs_version --sase-core-dir <core> --pyproject <checkout>/pyproject.toml
      (exit 3 = behind the floor → ✗; exit 4 = ahead → dim note)
```

- Use plain `git` subprocesses. Rule 3 mirrors `refresh_clean_linked_checkout`: clean,
  not detached, has an upstream, strictly behind → `merge --ff-only`.
- Test rule 3 and the containment check with parity tests against
  `dev_update/core_pin.py`.
- The clone URL is overridable through `SASE_INSTALL_CORE_REMOTE`, so tests can use
  local bare repos.
- `SASE_ALLOW_STALE_CORE=1` turns rule-4 and floor failures into ⚠ consequential rows.
  Such a pair never gets the healthy success line.

**`--sync`.** This is acei's fatal gate. Before planning, process the sase checkout, the
core, and every editable plugin checkout the plan will use:

1. Require each one to be clean, attached, and not diverged.
2. Fetch, then `merge --ff-only @{u}`.
3. If any checkout fails, abort before any change: exit 1, with a boxed per-repo failure
   list and per-repo diffstats on success.

`-n --sync` fetches and reports, but never merges. Without `--sync`, nothing is ever
pulled except the rule-3 core fast-forward.

**Pre-swap build check.** Real runs only, and before the lock is taken. Compile the core
so that a broken tree fails while the old install is untouched (`✗ Prepare sase-core`,
exit 1).

- Preferred approach: `maturin build --profile dev-update` into a temp dir, with the
  same `CARGO_TARGET_DIR` (`<core>/target/uv-tool-py`), environment, and interpreter
  that `rust-dev-install` uses. Get maturin through
  `uv run --no-project --with maturin`. Then the post-swap `maturin develop` reuses the
  compiled artifacts.
- Fallback: `cargo check` of the same package, if the measured reuse does not hold.
- Record the cold and warm timings in the phase notes. `maturin build` writes nothing
  in-tree, so live installs are untouched.

**Tests:**

- real temporary git repos for rules 1–5, `--sync` failure aggregation, and dirty,
  behind, and diverged states
- a fake `cargo`/`maturin` on PATH for build-check pass and fail
- plan-row snapshots

## plugin-repos: Rename install to install-venv in the plugin repos

Open each repo with `/sase_repo`: `sase-github`, `sase-telegram`,
`sase-research-artifacts`, and `sase-listen`. sase-nvim has no Justfile, and sase-core
has no install recipe.

In each repo:

- Rename the venv recipe to `install-venv` with `[group('install')]` and a one-line
  `[doc]`.
- Keep `[private] install` as a forwarding alias. It prints "`just install` is now
  `just install-venv`" to stderr, then runs it. This is not the sase project, so sase's
  flag policy does not apply. Record removal after one release as each repo's follow-up.
- Update internal callers, for example sase-github's `_setup`, which runs
  `just install`.
- Update CI `run: just install` steps.
- Update tests that pin the text: sase-github `tests/test_ci_install_contract.py`,
  sase-telegram `tests/test_justfile.py`, and sase-research-artifacts
  `tests/test_ci_install_contract.py`.
- Update docs: README, AGENTS.md/CLAUDE.md, and CONTRIBUTING.
- Leave `install-source-sase` unchanged.
- Run each repo's `sase tool run check` (sase-listen: its own lint and test recipes).

## engine-dev: The live `just install-dev`

Wire dev mode into the pipeline:

```text
preflight ─► plan ─► confirm ─► prepare (dev-core-prep) ─► lock ─► swap ─► re-apply ─► verify ─► restart ─► summary
```

- **Swap.** Editable host = this checkout, with plugins per engine-core's source policy.
  The overrides file carries `-e` lines plus a bare `sase-core-rs`. Use the parity argv.
- **Re-apply.**
  1. Capture the core identity (`_sase_core_source_identity.compute_identity`).
  2. Run `just -f <checkout>/Justfile rust-dev-install-uv-tool` with
     `SASE_RUST_DEV_PROFILE=dev-update`. This rebuilds the editable core (warm from the
     build check) and installs `sase-macro-lsp` into the tool `bin/`.
  3. Require the stamp that rust-recipes now writes to equal the captured identity. A
     mismatch is ⚠ "sase-core changed during the build; rerun just install-dev".
  4. A failure here is ✗. Print the restore command, and say that the env now holds a
     PyPI core.
- **Verify** (dev):
  - `sase` imports from `<checkout>/src/sase`, and `sase_core_rs` from
    `<core>/crates/sase_core_py/python`
  - `sase core health -j` is ok
  - `tools/check_sase_core_rs_bindings --src <checkout>/src/sase` passes under the tool
    interpreter
  - `sase version -j` reports an editable host whose `source_root` is this checkout
  - `sase lsp --version` works
  - the interpreter was kept
  - `sase update -n -j` agrees. It reports `mode == "dev"` and plans **no repair**: no
    `role: core` package that would be restored from a published wheel, and no reconcile
    step that exists only because the install is unhealthy. Pull work from roots that
    are behind their upstream is fine. Study `dev_update/_plan_status.py` and
    `_plan_reconcile.py` and pin the rule with fixture JSON. A timeout or fetch error is
    ⚠ inconclusive.
  - ⚠ when a stale `~/.cargo/bin/sase-macro-lsp` (from the old chezmoi installer) is on
    PATH ahead of the managed one, with `cargo uninstall sase_macro_lsp` as the fix.
- **No-op.** A run does nothing when all of these hold:
  - the receipt already equals the desired set
  - the installed core is editable from the resolved core
  - the stamp equals the current identity (so a dirty Rust edit at the same HEAD
    rebuilds)
  - the LSP exists
  - a quick health probe passes

  In that case, print
  `✓ sase already runs this checkout (3412a9f) with sase-core e411a39 — nothing to do (--force reinstalls)`.

- **Summary.**

  ```text
  ✓ sase now runs this checkout (3412a9f) with sase-core e411a39 (pin e8606a5 + 1)
    Python edits are live · after Rust edits: just install-dev · back to the release: just install
  ```

  Also add the `install-dev` hint to the PyPI summary.

**Justfile.** Add
`[group('install')] [doc('Install this checkout + its paired sase-core as your `sase` command')] [positional-arguments] install-dev *args`.

**Tests:**

- Hermetic: fake `uv`/`just`/`sase` and real temporary git repos. Cover the full dev
  sequence, re-apply failure messaging, stamp mismatch, `sase update` agreement parsing
  (pass, repair, and inconclusive), repeat-run no-op, a dirty-core rebuild, a
  workspace-path refusal through the recipe, and paths with spaces.
- Document a manual live smoke in the phase notes: `just install-dev -n`, then a real
  run from the durable checkout, then `sase update -n`. Do not run a real global install
  from an agent.

## core-strings: Point sase-core's remedies at the new names

Open `sase-core` with `/sase_repo`.

- `crates/sase_core/src/tool_run/triage/extractors/environment.rs`: the `MARKERS`
  remedies and `remedy_for()` for `missing_binding`, `core_import`, `core_wheel`, and
  `keep_sorted_missing` become `just install-venv`. These are `_setup` failures in a
  checkout's `.venv`. Regenerate the triage goldens with `UPDATE_TRIAGE_GOLDENS=1`.
- `triage/verdict.rs::environment_remedy`: "run `just install-venv` or `sase update` and
  retry".
- `bead/jsonl.rs:920`: "…; run `sase update` (or `just install-dev` from your sase
  checkout) to update sase-core". Update
  `unknown_event_operation_names_the_upgrade_remedy`.
- Run `sase tool run check` in sase-core. Committing both repos lets the host move
  sase's `sase-core-revision.txt` pin automatically.

## chezmoi-acei: Retire the chezmoi installers into install-dev

Open `chezmoi` with `/sase_repo`.

- In `home/dot_config/aliases.sh`:
  - `acei() { just -f ~/projects/github/sase-org/sase/Justfile install-dev --sync -y "$@"; }`
  - `aceii` is the same, plus `--with sase-google --with sase-gchat`. The receipt
    carries those plugins afterwards.

  Drop the `sase tui --restart-axe` suffix, because the installer restarts a running
  scheduler.

- Delete `home/bin/executable_install_sase_github`,
  `home/bin/executable_install_sase_google`, and
  `tests/bash/install_sase_github_test.sh`, after `git grep` confirms that nothing else
  calls them.
- Do not run `chezmoi apply`. The user applies dotfiles.

## install-docs: Document the three commands and record the human-only rule

> [!decision] human_only_record

- **INSTALL.md.** Add "Installing from a checkout", with the three-line command table,
  what each one touches, prerequisites (`cargo` for `install-dev`), how it relates to
  `sase update` (bootstrap and repair versus day-to-day updates), and `SASE_CORE_DIR`.
  `uv tool install sase` stays the primary path for people who do not have a checkout.
- **docs/development.md.** Add a "Your `sase` versus this checkout's `.venv`" subsection
  that explains `install-dev`, `install-venv`, and the agent guard and its bypass. Then
  point README's Development block, CONTRIBUTING, and `docs/rust_backend.md` at it.
- **Memory.** Use `/sase_memory_write` to add the strand
  `sase/memory/decisions/global-install-is-human-only.md`, then run `sase memory init`.
  The strand covers:
  - Claim: `just install` and `just install-dev` refuse inside SASE agents and monitors
    unless `SASE_GLOBAL_INSTALL_BYPASS` holds a reason, and agents propose global
    reinstalls through `/sase_gate`.
  - Why: they replace the `sase` that every agent and the scheduler import. The rejected
    alternatives are a banner-only warning and guarding `install-venv`.
  - Cost: the guard is a guardrail, not a security boundary.
  - Reopens when: a sandboxed per-agent `sase` exists.

  Link `[[decisions/guarded-recipes]]` to explain that guarded-recipes routes runs while
  this guard protects the host. Do not edit `guarded-recipes`.

- After the epic lands, someone working from a clean, merged tree runs
  `sase skill init --force` to redeploy the `sase_monitor` skill change.
