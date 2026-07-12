---
create_time: 2026-07-12 17:43:01
status: done
prompt: 202607/prompts/symvision_extraction_1.md
tier: epic
bead_id: sase-5t
---
# Factor pyvision into symvision and migrate sase + chezmoi

## Summary

Extract the `pyvision` linter (currently a single chezmoi-managed script at `home/bin/executable_pyvision` in the
chezmoi repo, vendored into sase as `tools/pyvision-260708`) into a new **public** GitHub repository
`bbugyi200/symvision` that publishes a **symvision** package to PyPI via release-please. Then migrate sase off the
vendored copy onto the published package, and retire the chezmoi script.

This mirrors the completed `sase-5r` epic ("Factor pylimit into toolong and migrate sase", which produced the `toobig`
package). The `toobig` repo (`~/projects/github/bbugyi200/toobig`) is the explicit quality bar: symvision's
documentation, linting, testing, CI, and release automation must match it — and exceed it where noted below.

## Product context

`pyvision` finds unused public Python functions/classes and misused private symbols. Key features that must survive the
port unchanged in behavior:

- Scans `src/`-style definition trees; tracked usage outside the tree counts, but test-support paths
  (`test`/`tests`/`testing` components, `test_*.py`) cannot keep a public symbol alive; defs under `testing/` are
  ignored.
- Pragma comments (`# pyvision: <repo-root-relative-path>` or `# pyvision: <repo-uri>`) mark consumers the scanner
  cannot see; local path pragmas are validated to exist and actually reference the symbol; URI pragmas resolve against
  local checkouts (`--external-repo-path` / `PYVISION_EXTERNAL_REPO_PATHS`, sibling dirs) with a deterministic cache
  clone fallback (`PYVISION_EXTERNAL_REPO_CACHE`); test/markdown pragma targets are rejected; stale pragmas are errors.
- `--epic-symbol <bead_id>(<symbol>)` whitelisting validated against a bead tracker via the `BD_COMMAND` env var
  (default `bd`; sase sets `BD_COMMAND=tools/sase_bead`).
- `--exclude-file`, `--exclude-decorator`, pyproject.toml entry-point discovery (`[project.scripts]`, etc. keep
  entry-point functions "used").

Consumers today (verified by GitHub code search and local greps):

1. **sase** — `just _lint-pyvision` runs `BD_COMMAND=tools/sase_bead .venv/bin/python tools/pyvision-260708 src/sase`;
   plus a `just pyvision` passthrough recipe, the `run_silent "lint (pyvision)"` stage in `just check`, ~10
   `# pyvision:` pragmas in `src/sase/`, several prose comments, docs (`README.md`, `docs/development.md`,
   `docs/integrations.md`), `src/sase/default_config.yml` (mentions `just pyvision`), and tests
   (`tests/test_github_actions_ci.py`, `tests/test_justfile_lint.py`).
2. **chezmoi** — the source script itself plus a 766-line / 24-case bash regression suite at
   `tests/bash/pyvision_test.sh` (run by chezmoi's `just test` and CI).

No other repo (including the sase-org plugin repos) runs pyvision directly.

## Naming and renaming decisions

- Package, repo, CLI command, and lint stage are all **symvision** ("symbol-visibility" linter).
- Pragma keyword becomes `# symvision: <ref>`. The tool recognizes **only** the new keyword — no legacy `# pyvision:`
  compatibility. Each consuming repo renames its pragmas in the same change that switches linters, so no transition
  window is needed.
- Env vars become `SYMVISION_EXTERNAL_REPO_PATHS` / `SYMVISION_EXTERNAL_REPO_CACHE`.
- `BD_COMMAND` keeps its current name — it is an existing external contract with sase's `tools/sase_bead` wrapper.
- `requires-python = ">=3.11"` (the script uses stdlib `tomllib`; staying dependency-free beats supporting 3.10 via a
  `tomli` backport for a dev tool). CI matrix: 3.11, 3.12, 3.13, 3.14.

## Quality bar (parity with toobig, or better)

Match toobig exactly on: hatchling packaging with `py.typed`; uv-based Justfile (`install` / `fmt` / `fmt-check` /
`lint` / `test` / `check`); ruff (format + `B,C4,E,F,UP,W` at minimum) and **strict** mypy; pytest with
`--strict-config --strict-markers`; `.github/workflows/ci.yml` (lint job + per-Python test matrix), `pr-title.yml`
(Conventional Commits squash-title gate), `publish.yml` (release-please v5 → `uv build` → `twine check` → fresh-venv
wheel install smoke test that exercises the real CLI → PyPI OIDC publish gated on the `pypi` GitHub environment);
`release-please-config.json` + `.release-please-manifest.json` with an `extra-files` version bump of
`src/symvision/__init__.py`; README with CI/PyPI/pyversions/license badges, motivation, quick start, and usage docs; MIT
LICENSE.

Exceed toobig on:

- **Dogfooding**: symvision's own `just lint` runs symvision against `src/symvision`, and runs `toobig` (dev dep) with
  sase's `1000 850 700` thresholds on `src` and `tests`.
- **Coverage gate**: `just test` collects coverage and CI enforces a threshold (target ≥90%; set to what the ported
  suite honestly supports, never below toobig's effective coverage).
- **Docs depth**: README (or `docs/` if it outgrows README) must document every flag, both pragma forms, epic-symbol
  workflow + `BD_COMMAND`, external-repo resolution order + env vars, and exit codes — pyvision has far more surface
  than toobig, and today its only docs live in sase's memory files.

## Phases

Each phase is completed by a distinct agent instance. Phases 1–4 work in `~/projects/github/bbugyi200/symvision/`
(created in Phase 1, sibling of the existing `~/projects/github/bbugyi200/toobig/` checkout — use it directly, do not
create another checkout). Phase 5 works in a sase workspace. Phase 6 works in the chezmoi repo opened via
`sase workspace open`. All commits use conventional-commit messages (release-please derives versions from them) and go
through the sanctioned sase commit workflow. During bootstrap (Phases 1–3) commits may land directly on `master`,
matching how toobig was built; after v0.1.0, use the normal PR flow.

### Phase 1 — Create the repo and port the tool (code + dev tooling)

1. Create the **public** GitHub repo: `gh repo create bbugyi200/symvision --public` with a one-line description (e.g.
   "Symbol-visibility linter: flag unused public and misused private Python symbols"). Initialize a fresh git repo at
   `~/projects/github/bbugyi200/symvision/`, set the default branch to `master` (all workflows and release-please target
   `master`, as in toobig), add the SSH remote (`git@github.com:bbugyi200/symvision.git`), and push the initial commit.
2. Scaffold packaging and dev tooling to the toobig template: `pyproject.toml` (name `symvision`, version `0.0.0`,
   hatchling, MIT, `requires-python >=3.11`, no runtime deps, dev extras
   `build/mypy/pytest/pytest-cov/ruff/toobig/ twine`, `[project.scripts] symvision = "symvision.cli:main"`, URLs,
   classifiers, ruff/mypy-strict/pytest/coverage config), `LICENSE`, `.gitignore`, `Justfile` (uv venv pattern,
   `SYMVISION_PYTHON` override), `py.typed`.
3. Port the tool: read the source from the chezmoi repo (open it with
   `sase workspace open -p chezmoi -r "<reason>" <workspace_num>`; the script is `home/bin/executable_pyvision`). Split
   the 1,250-line script into focused modules under `src/symvision/` (suggested seams, following the script's existing
   structure: `cli.py`, file discovery/scanning, symbol extraction (`_FileInfo` pass), usage analysis, pragma parsing +
   validation, external-repo resolution, epic-symbol validation). Keep every file comfortably under the toobig
   thresholds. Apply the renames from "Naming and renaming decisions"; behavior must otherwise be identical (diagnostic
   wording may say "symvision").
4. Add an initial pytest smoke suite (CLI invocation, exit codes, one happy-path and one violation scenario) so
   `just check` is meaningful; the full regression port is Phase 2.
5. Verify: `just check` green. Additionally run a **parity check**: in a scratch copy of a real consumer tree (e.g. a
   clone of sase with `# pyvision:` pragmas sed-renamed to `# symvision:` and env vars mapped), run the vendored
   `tools/pyvision-260708` and the new `symvision` CLI and diff their normalized output — they must agree.

### Phase 2 — Port the regression suite to pytest + coverage

1. Translate all 24 bash test functions from chezmoi's `tests/bash/pyvision_test.sh` (open chezmoi via
   `sase workspace open` as above) into pytest under `tests/`, using fixtures that build temp git repos, tracked/
   untracked files, external repos with origin remotes, and a fake `BD_COMMAND` executable. Organize test modules to
   mirror the `src/symvision/` module split.
2. Add focused unit tests for internals the bash suite only exercised end-to-end (pragma parsing edge cases, entry-point
   extraction, external-repo resolution ordering).
3. Wire the coverage gate (pyproject coverage config + `just` recipe used by CI) at the highest honest threshold (target
   ≥90%).
4. Optional aid: the original bash suite can be run against the new CLI (sed the suite's hardcoded `PYVISION_SCRIPT`
   assignment, pragma keyword, and `PYVISION_*` env names in a temp copy) as a one-off equivalence check before deleting
   nothing — chezmoi cleanup happens in Phase 6, not here.
5. Verify: `just check` green with the coverage gate enforced.

### Phase 3 — CI, release automation, README

1. Add the three workflows, adapted from toobig with the Python matrix set to 3.11–3.14 and `SYMVISION_PYTHON` plumbed
   through: `ci.yml`, `pr-title.yml`, `publish.yml` (release-please v5, build + `twine check`, fresh-venv wheel install
   smoke test that runs `symvision --help` plus a real scan fixture asserting both success and violation exit codes,
   then OIDC publish under the `pypi` environment).
2. Add `release-please-config.json` (python release-type, `include-v-in-tag`, pre-major bump settings, `extra-files` →
   `src/symvision/__init__.py`) and `.release-please-manifest.json` at `0.0.0`.
3. Create the `pypi` environment on the GitHub repo (`gh api`), matching toobig's setup.
4. Write the README to the documented quality bar (badges, motivation, quick start via `uv tool install symvision`, full
   CLI/pragma/epic-symbol/external-repo/exit-code docs, development section). Ensure at least one `feat:` commit exists
   in history so release-please proposes **v0.1.0**.
5. Verify: CI run green on GitHub (`gh run watch`); PR-title workflow behaves; release-please opens/updates its release
   PR proposing v0.1.0. Leave the release PR open for Phase 4.

### Phase 4 — First release: v0.1.0 on PyPI

**USER CHECKPOINT (before this phase runs):** register PyPI **trusted publishing** for the new project on pypi.org — a
_pending publisher_ for project `symvision`: owner `bbugyi200`, repository `symvision`, workflow `publish.yml`,
environment `pypi` (same as was done for toobig). This requires a pypi.org login and must be done by Bryan; the phase
agent must verify publishing works and stop with a clear notification if the publisher is missing.

1. Merge the release-please PR (squash, conventional title).
2. Watch the Publish workflow end-to-end: release job creates the GitHub release + `v0.1.0` tag; build, twine check, and
   install-smoke pass; PyPI publish succeeds.
3. Verify independently: PyPI JSON API reports `symvision 0.1.0`; `uv tool install symvision` in a clean environment
   yields a working `symvision` CLI.

### Phase 5 — Migrate sase to the published package

Work in a sase workspace (run `just install` first; ephemeral workspaces may have drifted deps).

1. `pyproject.toml`: add `symvision>=0.1.0,<0.2.0` to the dev dependency group (next to `toobig`).
2. `Justfile`: replace `_lint-pyvision` / `pyvision` recipes with `_lint-symvision` / `symvision` running
   `BD_COMMAND=tools/sase_bead {{ venv_bin }}/symvision src/sase`; update the `lint` aggregate, the
   `run_silent "lint (pyvision)"` stage name, and the lint doc comment. Delete `tools/pyvision-260708`.
3. Rename all `# pyvision:` pragmas in `src/sase/` (10 files) to `# symvision:`, and update prose comments/docstrings
   that mention pyvision (e.g. the "Keep ... active for pyvision" comments in `src/sase/core/*_facade.py`,
   `src/sase/ace/query/parser.py`, etc.).
4. Update non-memory references: `README.md`, `docs/development.md`, `docs/integrations.md`,
   `src/sase/default_config.yml` (the `just pyvision` guidance), and tests (`tests/test_github_actions_ci.py`,
   `tests/test_justfile_lint.py`, plus anything else `grep -rn pyvision` still finds outside memory files).
5. **Memory-file edits require explicit user permission** granted in that agent's own conversation: `memory/pyvision.md`
   (rewrite/rename to cover symvision: new pragma keyword, `SYMVISION_EXTERNAL_REPO_PATHS`, `just _lint-symvision`, PyPI
   provenance replacing "vendored from dotfiles"), the `memory/pyvision.md` reference in the repo's Tier 2 memory
   listing, and the `pyvision-260708` section in `tools/AGENTS.md` + generated provider shims. The launch prompt for
   this phase should explicitly grant permission for exactly these files; if it does not, the agent must leave them
   untouched and flag the follow-up. Note the `sase validate` memory-init freshness gate may also require the sanctioned
   regeneration flow after these edits.
6. Verify: `just install`, `just symvision` (clean run against `src/sase`), full `just check`. Confirm any failures are
   pre-existing at origin/master before treating them as regressions.

### Phase 6 — Retire the chezmoi script

Open the chezmoi repo via `sase workspace open -p chezmoi -r "<reason>" <workspace_num>`.

1. Delete `home/bin/executable_pyvision` and `tests/bash/pyvision_test.sh`.
2. `grep -rn pyvision` the repo and clean any remaining live references (historical plan files under `sase/repos/plans/`
   are archives — leave them).
3. Do NOT touch `pyvendor` or the other vendored tools (`pyscripts-260619`, `bugyi-260221.sh` remain vendored).
4. Verify chezmoi's `just test` still passes, then commit via the sanctioned commit workflow. Optionally note for the
   user: run `uv tool install symvision` on machines where the old `~/bin/pyvision` command was used interactively.

## Dependencies and bead structure

Create an epic bead mirroring `sase-5r`, with six phase children in a strict chain (1 → 2 → 3 → 4 → 5 → 6). Each child
bead's description must state its repository/worktree requirement (Phases 1–4: `~/projects/github/bbugyi200/symvision/`;
Phase 5: the sase repo; Phase 6: chezmoi via `sase workspace open`). Phase 4 additionally carries the PyPI
trusted-publisher user checkpoint.

## Risks / notes for implementers

- **Behavior parity is the contract.** The sase migration should be a pure rename at the consumer level; any behavioral
  drift in the port will surface as new lint errors in sase. The Phase 1 parity diff and Phase 2 full test port are the
  guards.
- **`# pyvision:` pragma sweep must be complete** in Phase 5 — symvision will not honor the old keyword, and a missed
  pragma shows up as an unused-symbol failure.
- **Known sase-side flakiness**: verify `just check` failures against a clean baseline before attributing them to the
  migration; the full test suite may be killed by the sandbox (exit 144) — fall back to static gates plus targeted
  pytest subsets when that happens.
- **PyPI trusted publishing** cannot be configured by an agent; Phase 4 blocks on the user checkpoint above.
- **Memory files**: no agent may edit `memory/*.md`, `AGENTS.md`, or provider shims without explicit user permission in
  its own conversation (Phase 5, step 5).
