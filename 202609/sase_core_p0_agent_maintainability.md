---
tier: epic
title: "sase-core P0: fast loop, instruction delivery, cross-repo truth"
goal: "Every agent that touches sase-core gets a concise, accurate guide without being
  told to look for it. A real sase_core edit re-checks in about 30 s on athena, and
  switching cargo scope no longer recompiles sase_core. sase stops producing false
  signals about the core: the pin bot opens its PRs, and `just check` rebuilds a dev
  extension that no longer matches the linked core source.

  "
phases:
  - id: core-fast-loop
    title: Pinned features, just fast, and a true MSRV in sase-core
    depends_on: []
    size: medium
    description:
      "core-fast-loop: in sase-core, unify dependency features with a cargo-hakari
      workspace-hack plus a tool-free check.sh features gate (also run in CI). Add `just
      fast`, and make rust-version true by deleting the incompatible_msrv allows.
      Measure a ≤5 s no-edit -p scope switch."
  - id: core-agent-guide
    title: sase-core agent guide, provider shims, module map, README
    depends_on:
      - core-fast-loop
    size: medium
    description:
      "core-agent-guide: rewrite sase-core AGENTS.md to the target content, add
      CLAUDE.md and GEMINI.md import shims, add `just modules` and fill the missing
      top-level `//!` summaries. Delete the stale sase_core_py binding manifest and cut
      the README down to current facts."
  - id: instruction-delivery
    title: repo-open AGENTS.md hint, sase_repo skill, core memory fix
    depends_on: []
    size: small
    description:
      "instruction-delivery: sase repo open names an opened repo's AGENTS.md on stderr,
      the sase_repo skill says the same, and the rust_core_backend_boundary core memory
      stops pointing at ../sase-core and points at docs/rust_backend.md (sase-15w). Also
      add a short cross-repo pointer in docs/rust_backend.md."
  - id: pin-ratchet-bot
    title: Core pin ratchet workflow opens its PR
    depends_on: []
    size: small
    description:
      "pin-ratchet-bot: make core-pin-ratchet.yml tolerate the apply path's exit 2 so it
      reaches the push and PR steps. Behavior-test the step script (sase-15v)."
  - id: extension-freshness
    title: Dev extension rebuilds when linked sase-core source changes
    depends_on: []
    size: medium
    description:
      "extension-freshness: rust-install stamps the source identity (HEAD plus a dirty
      digest) it built from. validate_test_environment compares it with the linked
      checkout and flags a rebuild through a new status bit and fingerprint bucket,
      _setup rebuilds on that bit, and the stale-extension flake-baseline entry is
      retired."
  - id: reaper-root
    title: Managed-tmp reaper covers the root agents actually use
    depends_on: []
    size: medium
    description:
      "reaper-root: capture SASE_TMPDIR (and sibling SASE path overrides) in the service
      env, and warn when the reaper's managed root differs from the launch root, so
      per-run cargo targets are really reaped (sase-15q). This is a prerequisite for
      larger incremental target dirs."
  - id: incremental-check
    title: Incremental check/clippy through the athena rustc wrapper
    depends_on:
      - reaper-root
    size: medium
    description:
      "incremental-check: sase-rustc-wrapper runs metadata-only incremental units
      directly and strips incremental from codegen units before sccache, and chezmoi
      drops incremental=false. A new sase config opt-in replaces the forced
      CARGO_INCREMENTAL=0 and athena enables it. Measure a ≤30 s edit→check."
proposed_by: bbugyi200.athena.0p2
create_time: 2026-09-22 08:03:49
status: wip
---

- **PROMPT:**
  [prompts/202609/sase_core_p0_agent_maintainability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_core_p0_agent_maintainability.md)

# Plan: sase-core P0: fast loop, instruction delivery, cross-repo truth

## Context

This implements the **P0** section of the research report
`research:202609/sase_core_agent_maintainability/sase_core_agent_maintainability.md`.
Read it with
`sase artifact read research:202609/sase_core_agent_maintainability/sase_core_agent_maintainability.md "<why>"`;
§4.1, §4.3, §4.4 and §7 P0 are the relevant parts. Its findings, re-verified while
planning:

- **sase-core's own instructions never reach agents.** sase-core is a _linked repo_
  opened with `sase repo open sase-core`.
  - Claude only loads `AGENTS.md` when no `CLAUDE.md` exists above the cwd. It does load
    a nested `CLAUDE.md` on demand, but sase-core has none.
  - Gemini skips gitignored `sase/repos/`.
  - `sase repo open` prints only the path.
  - The core memory `rust_core_backend_boundary` points at a nonexistent `../sase-core`.
  - The current `AGENTS.md` has 36 lines, mostly about release-plz and the Python
    interpreter. It has no crate map, no recipes and no fast loop.
- **The verify loop is slow and partly self-inflicted.**
  - A real `sase_core` edit costs 57–61 s to `cargo check`. Incremental without sccache
    costs 23–30 s.
  - Switching to `-p sase_core --lib` with _no edit_ costs 49 s. Ten dependencies
    resolve extra features in the workspace build: chrono, serde_json, regex-automata,
    syn, smallvec, hashbrown, memchr, once_cell, num-traits and serde_core. This was
    re-verified with `cargo tree` while planning.
  - There is no `just fast`.
  - `just check` takes ~281 s, which is longer than a 2-minute default tool timeout.
- **False signals across the repo boundary:**
  - The core-pin-ratchet workflow fails on every pending bump (`sase-15v`).
  - `tools/validate_test_environment` keys its cache on sase-core's `Cargo.toml` only. A
    stale dev extension therefore validates green, and its failure is baselined as a
    flake at `tests/reproducible_flake_baseline.txt` (~line 402).
- **The false MSRV and disk.**
  - `rust-version = "1.78"` is false. There are 8 `#[allow(clippy::incompatible_msrv)]`,
    and the dependencies already need 1.88.
  - Incremental caches cost ~2 GB per run, while athena's root disk is 89% full. The
    reaper also watches the wrong root (`sase-15q`): `~/.cache/sase/tmp/cargo-targets`
    holds ~105 GB across ~450 dirs on athena, and the captured service env lacks
    `SASE_TMPDIR`.

Repos: **sase-core** and **chezmoi** are linked repos. Open each with
`sase repo open <name> -r "<why>"` and work only in the printed path. **sase** is the
primary checkout. In each repo, read its `AGENTS.md` before editing.

## Decisions (deliberate deviations from the report)

1. **Flakes are a query, not a list.** `AGENTS.md` does not list open flake bead IDs,
   because those go stale as soon as P1 closes them. It gives a query instead:
   `sase bead list -T flake`, whose sase-core beads are titled `sase-core flake: …`.
2. **`just modules`, not a committed `docs/MODULES.md`.**
   - A recipe that prints each top-level module's first `//!` line is fresh by
     construction and needs no staleness gate.
   - P1's module-docs gate can decide later whether to commit a generated, layered map.
3. **Two small pull-forwards from P1/P2, both of which shrink the add-a-binding
   recipe:**
   - Delete the hand-kept `//!` binding manifest in `crates/sase_core_py/src/lib.rs`. It
     is ~660 of the file's 736 lines, misses 191 of 823 bindings, and nothing parses it.
     P1's generated inventory replaces it.
   - `AGENTS.md` tells new code to import core items by module path and to stop adding
     root `pub use` names or `core_*` prelude aliases. This is P2's end state, adopted
     now so the #1 churn hub stops growing.
4. **Incremental is a host opt-in in sase config, not simply "stop forcing
   `CARGO_INCREMENTAL=0`".** Only athena has the splitting rustc wrapper. On a host
   without it, dropping the override would make test builds incremental, at ~9 GB per
   run.
5. **Extension freshness compares a built-from stamp; it does not just add a fingerprint
   input.** A fingerprint alone only re-runs the binding validator. That still never
   rebuilds after a behavior-only Rust edit that adds no binding.
6. **Provider shims are one-line `@` imports, not byte copies.** sase's memory renderer
   does not manage linked repos, so copies would drift.
7. **The ratchet fix lives in the workflow.** `tools/ratchet_core_revision` keeps its
   exit-2-on-apply contract, which mirrors `tools/ratchet_core_window`. `publish.yml`
   already tolerates the same contract.
8. **`decisions:rust-core-required` is not edited.** Its quoted version window is
   illustrative, and accepted decision records are immutable.

The **memory change is in scope**: phase `instruction-delivery` edits
`sase/memory/rust_core_backend_boundary.md`. This is P0 item 4 and bead `sase-15w`.

## Target content for sase-core `AGENTS.md`

Principles:

- Every line must either prevent a mistake the compiler and gate do not catch, or save a
  search the agent would otherwise make.
- Nothing the gate already reports.
- No volatile lists, counts or bead IDs. The one timing is the gate duration, because it
  sets tool timeouts.
- Hard cap 150 lines; aim for about 90.

The file is loaded for every agent that reads any sase-core file, including sase agents
who only glance at a binding. So each line is paid many times.

The `core-agent-guide` phase starts from this text. It must verify every claim against
the tree after `core-fast-loop` lands, and it may tighten wording, but it must not grow
the file with anything the principles exclude.

```markdown
# sase-core

Rust core for [sase](https://github.com/sase-org/sase). Domain logic and its serde
`*Wire` contracts live in `sase_core`. sase (Python) calls them through the
`sase_core_rs` extension, looking each binding up by its Python name with
`require_rust_binding("<name>")`.

| Crate                     | Owns                                                                                                                                              |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `crates/sase_core`        | All domain logic, with no PyO3. Flat top-level modules, one per domain; `just modules` prints a one-line summary of each                          |
| `crates/sase_core_py`     | The `sase_core_rs` extension (PyPI `sase-core-rs`). One binding domain per `src/<domain>/`, often named differently from the core module it binds |
| `crates/sase_gateway`     | The mobile and fleet HTTP gateway plus the `sase_sudo_runner` and `sase_federation_worker` binaries. They ship inside the wheel                   |
| `crates/sase_xprompt_lsp` | The `sase-xprompt-lsp` language server                                                                                                            |

## Build and verify

Never run bare `cargo`. `sase_core_py` needs a Python >= 3.12 interpreter to build and
libpython to test, and `scripts/check.sh`, which every `just` recipe calls, resolves
both.

- `just fast` is the inner loop (`cargo check --workspace --all-targets`). Any
  `sase_core` edit recompiles the whole crate, so batch your edits between runs.
- `just test -p <crate> [<filter>]` runs targeted tests while you iterate.
- `just fmt` applies formatting.
- `just check` is the gate and runs the same steps as CI: fmt-check, clippy with
  `-D warnings`, every test, and the script tests. Pass it before you finish. It takes
  about 5 minutes, so give it an explicit tool timeout of 10 minutes or more.

A targeted run never replaces `just check`. For example, `-p sase_core` alone skips the
`sase_core_py` binding tests, and that gap let stale schema fixtures reach master in
`a509dcc`. `master` is unprotected, and a red commit also fails every release-plz run.

A test that fails under `just check` but passes when rerun alone is probably a load
flake. Before you treat it as yours, look for its `sase-core flake:` bead in
`sase bead list -T flake`. Never weaken an assertion to get a green run.

## Conventions

- Write free functions over `*Wire` serde structs, with errors as `thiserror` enums.
- Do not use `macro_rules!`. A generated item is invisible to grep and to the binding
  checks, so use a generic helper instead.
- A multi-file module's `mod.rs` is a facade that holds only `mod` and `pub use` lines.
  Tests sit beside the code in `tests.rs` or `tests/`. Keep new files at or under 1,500
  lines.
- Import core items by module path (`sase_core::<module>::Item`). Do not add names to
  the root `pub use` list in `crates/sase_core/src/lib.rs` or new `core_*` aliases to
  `crates/sase_core_py/src/prelude.rs`. Both are being retired.
- release-plz owns versions and changelogs:
  - Never edit a `version` field, a path-dependency version pin, or a `CHANGELOG.md`. A
    deliberate release recovery needs user approval and the `manual-version` PR label.
  - Commit subjects are Conventional Commits.
  - Mark a breaking change with `feat!:`/`fix!:` or a `BREAKING CHANGE:` footer. That
    covers removing or renaming a Python binding, and any wire change that old readers
    reject, because released sase accepts every core in its `sase-core-rs` minor window.
- Canonicalize both sides of a path comparison, or neither side. On macOS, `/tmp` and
  `/var` are symlinks into `/private`, so a symlinked ancestor is not evidence of an
  attack. CI also runs on macOS.

## Recipes

**Add a core function and expose it to Python.** This is the most common change.

1. Implement the function and its request/response `*Wire` types in the owning
   `sase_core` module, with tests. Export them from the module facade.
2. Put the binding in the domain that already binds that module:
   `rg -l '<sibling fn>' crates/sase_core_py/src/*/`.
3. Model the binding on a neighbour:
   - Use `#[pyfunction]`, `#[pyo3(name = "<python name>")]` and `fn py_<python name>`.
   - Parse the input dict into the request `*Wire`.
   - Map the core error to a Python exception.
   - Return `serialize_to_py(py, &out)`.
4. Register it in that domain's `register_<domain>` with
   `m.add_function(wrap_pyfunction!(py_<name>, m)?)?`. The compiler does not check this,
   so a missed registration only surfaces as an `AttributeError` in sase.
5. Add a round-trip test in the domain's `tests.rs`.
6. sase CI builds the core at the commit named in sase's `sase-core-revision.txt`. Any
   sase code that calls the new binding stays red until that pin moves past your commit.
   Move it with `just ratchet-core-revision` in sase, or wait for the six-hourly ratchet
   PR. See sase's `docs/rust_backend.md`.

**Change a wire schema version.** `*_WIRE_SCHEMA_VERSION` values are copied by hand.

1. Search both repos for the constant, its `*_wire_schema_version` getter and the old
   number.
2. Update every copy together:
   - in this repo, the Rust tests and fixtures
   - in sase, the Python mirror constant, `tools/validate_sase_core_rs`, and their tests

**Add a gateway route.**

1. Write the handler in `crates/sase_gateway/src/routes/`.
2. Register it in `routes/router.rs` and declare it in `contract.rs`. No test ties those
   two together, so do both.
3. Regenerate the snapshot with
   `UPDATE_MOBILE_CONTRACT=1 just test -p sase_gateway committed_`. For fleet routes,
   use `UPDATE_FLEET_CONTRACT=1` instead.
4. List the route in `crates/sase_gateway/README.md`.
```

## Phase `core-fast-loop` (sase-core)

1. **Feature unification.** Add a cargo-hakari workspace-hack crate (e.g.
   `crates/sase_workspace_hack`) and a `.config/hakari.toml`:
   - Use resolver 2. Cover the platforms CI and the release workflow build wheels for.
   - Generate it with `cargo hakari generate && cargo hakari manage-deps`. Install the
     tool with `cargo install cargo-hakari --locked`. Only regeneration needs the tool;
     verification does not.
   - Give the new crate `publish = false` and `rust-version.workspace = true`.
   - Add a `[[package]] name = … release = false` entry to `release-plz.toml`.
   - Confirm that `.github/scripts/check_cargo_version_edits.py` (the Cargo version
     guard) accepts the new crate's version line. If it does not, adjust the crate, not
     the guard's rules for existing crates.
   - Confirm the wheel still builds: run the maturin build/sdist command the release
     workflow uses, or `maturin build -m crates/sase_core_py/Cargo.toml`.
   - _Fallback:_ if hakari cannot coexist with the wheel or release tooling, hand-pin
     the union features with `[workspace.dependencies]` and member dependencies. Step
     2's gate stays the same.
2. **Tool-free drift gate:** `./scripts/check.sh features`.
   - For each workspace member, compare the feature set of every package in its normal,
     build and dev closure
     (`cargo tree -p <member> -e normal,build,dev --prefix none -f '{p} {f}'`) with the
     workspace-wide resolution.
   - On a difference, fail with a message that names the package and the feature delta,
     and that says to run `cargo hakari generate && cargo hakari manage-deps`.
   - Run it in `all`, and add a step for it to `.github/workflows/ci.yml` on both OS
     legs, since CI calls subcommands individually.
   - It must not need `PYO3_PYTHON` or a compile.
   - Tune the compared edge set to what `just fast` and `just test -p <crate>` actually
     build. The measured switch time in step 5 is the ground truth, not the diff.
3. **`just fast`:** a `fast` recipe that runs `./scripts/check.sh check [args]`.
   - The new `check` subcommand runs `cargo check --workspace --all-targets [args]`. It
     reuses `configure_pyo3_python` and `default_scope`.
   - Update `usage`. Rewrite the `default_scope` comment, which claims `-p` keeps
     single-crate runs cheap. That claim becomes true only with step 1, so state that
     dependency.
4. **True MSRV:**
   - Delete the 8 `#[allow(clippy::incompatible_msrv)]` (in `procs/store.rs`,
     `notifications/store.rs`, `prompt_stash/store.rs` and
     `notifications/pending_actions.rs`).
   - Set `[workspace.package].rust-version` to the lowest version that meets both
     conditions:
     - `just check` passes with the allows gone. Clippy's `incompatible_msrv` then
       enforces it.
     - It is at least the highest `rust_version` among dependencies in `cargo metadata`.
       That is 1.88 today; expect 1.89 because of `File::lock`.
   - If that toolchain installs cheaply
     (`rustup toolchain install <ver> --profile minimal`), also confirm that a workspace
     check under it passes.
5. **Measure the exit criteria** and record them with `sase bead note` on this phase
   bead, together with the load average. Use a fresh `CARGO_TARGET_DIR`.
   - After `just fast`, time `./scripts/check.sh check -p sase_core`, then
     `./scripts/check.sh check -p sase_gateway`, then `just fast` again. Each must be
     **≤5 s**; the report measured 48.8 s for the first switch.
   - Confirm that `just test -p sase_gateway --no-run` after `just fast` does not
     recompile `sase_core`.
6. Leave `AGENTS.md` and `README.md` alone; the next phase owns them. Verify with
   `just check`.

## Phase `core-agent-guide` (sase-core)

1. **Rewrite `AGENTS.md`** from the target content above.
   - Check every command and path against the tree: `just fast`, `just test`,
     `just fmt`, `just modules`, `routes/router.rs`, `contract.rs`, the `committed_`
     snapshot tests, the `UPDATE_*_CONTRACT` variables and `serialize_to_py`.
   - Dry-run the add-a-binding recipe against commit `45a966c`. Its only extra edit
     sites should be the ones this plan retires.
   - Keep the file under the 150-line cap and hold to the principles.
2. **Provider shims.** Add a `CLAUDE.md` whose whole content is `@AGENTS.md`, and a
   `GEMINI.md` whose whole content is `@./AGENTS.md`. These are the import forms Claude
   Code and Gemini CLI document. Add no other shim files, because the `sase repo open`
   hint covers the other providers.
3. **`just modules`:** a `modules` recipe that runs `./scripts/check.sh modules`.
   - It prints `<module>: <first //! line>`, sorted, for every top-level module of
     `crates/sase_core/src`. The module root is `<m>.rs` or `<m>/mod.rs`; skip `lib.rs`.
   - Add a one-line `//!` summary to every top-level module that lacks one. About 15 do
     today, including `agent_cleanup`, `agent_stats`, `content_layout`, `editor`,
     `managed_origin`, `model_completion`, `notifications`, `perf_logs`, `procs`,
     `prompt_stash`, `service`, `snippet_session`, `telemetry`, `text_tail` and
     `xprompt_catalog`.
   - Do not make this a gate; P1 adds that.
4. **Delete the binding manifest** (the `//! - name(...) -> ...` list) from
   `crates/sase_core_py/src/lib.rs`.
   - Keep a short crate `//!` saying the Python surface is the `#[pyo3(name = …)]`
     functions that each domain's `register_<domain>` registers.
   - Keep any still-accurate explanatory prose, such as the `QueryErrorWire` →
     `ValueError` note, if it is short.
5. **Cut `README.md` to current facts**, about 80 lines:
   - Keep:
     - what the repo is and how sase consumes it (PyPI `sase-core-rs`, pinned in sase)
     - the crate table, including the LSP crate
     - "Development: see `AGENTS.md`"
     - the release-plz, Cargo version guard and `manual-version` summary
     - the `bench_parse` example
     - the license
   - Delete:
     - "Phase status" and the Phase 1E packaging history
     - every `sase_100` and `SASE_CORE_BACKEND` reference
     - the `../sase-core` sibling instructions and the stale `just rust-*` list
     - the partial wire, parser and scanner tables
6. Verify with `just check`. Record the final `AGENTS.md` line count in a bead note.

## Phase `instruction-delivery` (sase)

1. **`sase repo open` hint** (`src/sase/main/repo_handler_open.py`, in both the
   inventory and the external-repo branches).
   - After a successful open, if `<path>/AGENTS.md` exists and `<path>` is not the
     invoking workspace's own checkout root, print one stderr line:
     `Read <path>/AGENTS.md before working in this repo; it is not loaded automatically from here.`
   - stdout stays exactly the path.
   - Tests in `tests/main/` cover four cases:
     - a linked repo with `AGENTS.md` (hint present, stdout unchanged)
     - a linked repo without one (stderr empty;
       `test_repo_open_direct_linked_name_has_path_only_stdout` must keep passing)
     - an external repo with one
     - the caller's own primary checkout (no hint)
2. **Skill source** `src/sase/xprompts/skills/sase_repo.md`: add one sentence to "Open A
   Repository" saying that when the opened repo has an `AGENTS.md`, the command names it
   on stderr and you must read it before editing. Preview with `sase skill init --diff`.
   Do **not** deploy from an unlanded tree; see Landing.
3. **Core memory** `sase/memory/rust_core_backend_boundary.md` (keep `type: core`, and
   keep it within about its current length):
   - Replace both `../sase-core` paths with: sase-core is a linked repo; open it with
     `sase repo open sase-core` and work in the printed path. Do not add a "read its
     `AGENTS.md`" line. The step 1 hint delivers that at open time, and this note is
     paid on every sase turn.
   - End the boundary paragraph with one sentence: a binding that sase calls also needs
     sase's `sase-core-revision.txt` CI pin moved past its sase-core commit (see
     `docs/rust_backend.md`).
   - Keep the litmus test and the presentation-only paragraph.
   - Run `sase memory init`. Then close `sase-15w` with
     `sase bead close sase-15w --note …`.
4. **`docs/rust_backend.md`**: add a short subsection under "Source / development
   workflow" on changing sase-core from a sase workspace. It covers opening it with
   `sase repo open sase-core`, following sase-core's `AGENTS.md` recipes, and a link to
   "The CI source revision pin" section. Do not duplicate the recipes.
5. Verify per the sase `lint_and_test` memory.

## Phase `pin-ratchet-bot` (sase)

1. In `.github/workflows/core-pin-ratchet.yml`, run the apply step so that exit 2
   ("applied") continues and any other non-zero code fails, e.g.
   `python3 tools/ratchet_core_revision || [ "$?" -eq 2 ]`. Keep the tool's exit
   contract.
2. Audit the rest of the step for the next failure. `git push` needs the
   `SASE_RELEASE_TOKEN` checkout credentials, and `gh pr create` needs `GH_TOKEN`; add
   whatever env is missing.
3. Behavior-test it in `tests/test_github_actions_ci_master_gate.py`:
   - Extract the step's `run` script and execute it with stub `python3`, `git`, `gh` and
     `tr` on `PATH`.
   - Assert that a pending bump reaches `gh pr create`, and that exit codes 0 and 3
     behave as today.
   - Keep the existing string assertions passing.
4. Close `sase-15v` with a note naming the fix. Verify per `lint_and_test`.

## Phase `extension-freshness` (sase)

1. **Built-from stamp.**
   - `rust-install` (Justfile) computes the source identity of `{{ sase_core_dir }}`
     after its checkout refresh and before it builds. Only after the install succeeds,
     whether from a wheel-cache hit or `maturin develop`, does it write that identity
     into the venv, e.g. `<venv>/.sase-core-rs-source.json`. An edit made during the
     build therefore still reads as stale.
   - The identity is `git rev-parse HEAD` plus a digest of uncommitted and untracked
     changes under the crate input paths that `tools/sase_core_wheel_cache` already
     hashes. Factor that path list and the identity helper into one shared place rather
     than copying them.
   - The identity must cost well under a second, using only git plumbing and never
     hashing `target/`.
2. **`tools/validate_test_environment --check-core`.**
   - Compute the current identity and compare it with the stamp. A missing or different
     stamp sets a new status bit (`CORE_SOURCE_STALE = 32`).
   - Add the identity as a `core-source` bucket in `_fingerprint_inputs`, so cached
     verdicts cannot mask a change.
   - Skip the check when `SASE_CORE_WHEEL` supplies a prebuilt wheel.
   - Do not add `core-source` to `ENVIRONMENT_ESCALATING_INPUTS` in
     `tests/_test_selection_manifest.py`. The rebuilt extension's `extension` bucket
     already escalates. Keep that module and its schema history consistent.
3. **`_setup`, `_setup-visual` and `_setup-terminal-smoke`**: treat bit 32 like bit 2.
   Rebuild with `rust-install`, then re-run `validate_sase_core_rs`, and print a
   one-line reason such as "linked sase-core source changed since the extension was
   built".
4. Retire
   `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`
   from `tests/reproducible_flake_baseline.txt`, following the file's header format.
5. Add tests in `tests/test_validate_test_environment_tool.py`:
   - identical identity sets no bit
   - a new commit sets the bit
   - a dirty edit under `crates/` sets the bit
   - a missing stamp sets the bit
   - `SASE_CORE_WHEEL` skips the check
6. Add a sentence to the dev-workflow section of `docs/rust_backend.md`. Verify per
   `lint_and_test`.

## Phase `reaper-root` (sase)

1. Add `SASE_TMPDIR`, plus any other `SASE_*` managed-root override that
   `sase.core.paths` honors, to the names `capture_service_environment` keeps
   (`src/sase/service/env.py`, next to `PATH` and `SASE_FEATURE_FLAGS`).
2. Make a root mismatch visible. When the reaper's `managed_tmpdir_root()` differs from
   the root the launcher gives agents, the housekeeping reap result and `sase doctor`,
   or the disk-pressure attribution, must say so and name the fix. Also check that
   disk-pressure attribution counts `cargo-targets/` under the effective root.
3. Add tests for the env capture and the mismatch warning. Name the exact one-time host
   command that refreshes the captured env, for Landing.
4. Close `sase-15q` with a note that says what was verified. Verify per `lint_and_test`.

## Phase `incremental-check` (chezmoi + sase)

**chezmoi** (read its `AGENTS.md`; tests: `just test-bash`):

1. Change `home/bin/executable_sase-rustc-wrapper` for any unit whose args carry an
   incremental flag (either the `-C incremental=X` or the `-Cincremental=X` form):
   - If the unit is metadata-only (its `--emit=` list lacks `link`), exec the compiler
     directly and skip sccache. The compiler may be `clippy-driver` followed by `rustc`.
   - Otherwise, strip the incremental flag and continue through the existing
     Poseidon/sccache path.
   - Units without the flag are unchanged. The Poseidon bypass rules still apply.
2. Extend `tests/bash/sase_rustc_wrapper_test.sh` to cover the metadata-only direct
   path, the stripped codegen path, the unchanged path, and a clippy-driver invocation.
3. Delete `incremental = false` from `home/dot_cargo/config.toml.tmpl`. Update
   `docs/poseidon_cargo_cache.md`.
4. Opt athena in through `home/dot_config/sase/sase_athena.yml` (step 5's key).

**sase:**

5. Add `managed_tmp.agent_cargo_incremental` (bool, default `false`) in
   `src/sase/config/_settings.py` and `src/sase/default_config.yml`, and document it in
   `docs/configuration.md`.
   - `_managed_agent_scratch_env` in `src/sase/agent/launch_spawn.py` exports
     `CARGO_INCREMENTAL=0` when it is false (today's behavior) and `CARGO_INCREMENTAL=1`
     when it is true. An explicit `extra_env` still wins.
   - Extend `tests/test_axe_chop_agents_env.py`.
   - Check whether sase's config loader rejects unknown keys. If it does, Landing must
     order the sase update before the chezmoi apply.
6. Update `src/sase/core/disk_footprint_inventory.py`'s incremental horizon text
   ("CARGO_INCREMENTAL=0 prevents return") and the `CARGO_INCREMENTAL=0` sentence in
   `docs/axe.md`.

**Measure before deploying**, and record the results with `sase bead note`:

- Work in the sase-core checkout with a fresh `CARGO_TARGET_DIR`,
  `RUSTC_WRAPPER=<the new wrapper in the chezmoi checkout>`, `CARGO_INCREMENTAL=1` and
  `PYO3_PYTHON` set.
- Run a cold `cargo check --workspace --all-targets`.
- Append a comment to `crates/sase_core/src/host_liveness.rs` and re-check. The target
  is **≤30 s**; record the load average. Restore the file with a trap.
- Record the incremental dir size after `check` and after `clippy`.
- Confirm that a `cargo test --no-run` sends no `-C incremental` unit to sccache.

## Landing (land agent, or the user where noted)

- After the `instruction-delivery` template lands, run `sase skill init --force` from
  the landed, clean tree.
- On each host, refresh the captured service env with the command `reaper-root` names.
  Then confirm that the next `managed_tmp_reap` result scans `~/.cache/sase/tmp`.
- **User:** run `chezmoi apply` on athena for the wrapper, the cargo config and the
  athena opt-in. Do this only after the installed sase includes
  `agent_cargo_incremental` if unknown keys are rejected.
- Confirm that the next `core-pin-ratchet` run, or a manual
  `gh workflow run core-pin-ratchet.yml`, either opens a pin PR or exits 0 with nothing
  pending.
- Re-measure a real agent's first sase-core check after rollout. The report flags an
  unmeasured cold-start risk.

## Exit criteria

| KPI                                                  | Today                       | Target                                                          |
| ---------------------------------------------------- | --------------------------- | --------------------------------------------------------------- |
| Core leaf real edit → workspace check (athena)       | 57–61 s                     | ≤30 s                                                           |
| No-edit `-p` scope switch                            | 49 s                        | ≤5 s                                                            |
| sase-core guide in context when working in sase-core | ~never                      | Claude and Gemini via shims; every provider via the stderr hint |
| sase-core `AGENTS.md`                                | 36 lines, no recipes        | ≤150 lines (aim ~90), 3 recipes, verified claims                |
| Pin bot                                              | fails on every pending bump | opens a PR                                                      |
| Dev extension after a Rust edit                      | silently stale              | rebuilt by `_setup`                                             |
| `rust-version`                                       | false (1.78, 8 allows)      | true, 0 allows                                                  |

## Out of scope

- P1: the size ratchet, the registration and schema-registry gates, the generated
  binding inventory, the module-doc gate, and the flake burn-down.
- P2: deleting the root prelude and alias layer, and renaming the binding domains.
- P3: the crate split.

The one exception is the two pull-forwards in Decision 3.
