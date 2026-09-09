---
tier: epic
create_time: 2026-09-09 19:53:24
status: wip
---

- **PROMPT:**
  [prompts/202607/symvision_extraction.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/symvision_extraction.md)

# Plan: Factor `pyvision` into `bbugyi200/symvision` and Migrate sase to It

## Context

`pyvision` is the unused/misused-symbol linter for Python: it flags public symbols with
no non-test consumer, private symbols imported across files, and private symbols unused
in their own file, with a pragma/epic-symbol whitelist system (including cross-repo URI
pragmas). It lives in the chezmoi dotfiles repo and is vendored into sase as
`tools/pyvision-260708` (1251 lines, stdlib-only Python). It runs as the `pyvision`
stage of `just lint` / `just check` (`just _lint-pyvision`, `just pyvision`).

Goal: extract this tool into a dedicated Python project published on PyPI, then migrate
sase to consume the published package — the same treatment `pylimit` got in the
`toolong` epic (sase-5r), and the new repo should be just as nice or nicer (top-tier
README, CI, release automation). The chezmoi originals are left behind untouched (no
chezmoi/dotfiles changes anywhere in this plan).

Unlike toolong, the GitHub repo does NOT exist yet — creating it is part of Phase 1.

## Decisions Already Made (flag at review if you disagree)

1. **Project name: `symvision`** ("symbol vision") — GitHub repo `bbugyi200/symvision`,
   PyPI distribution `symvision`, import package `symvision`. All three verified
   available (PyPI 404; no `bbugyi200/symvision` repo). `pyvision` is confirmed taken on
   PyPI (a computer-vision toolkit). Free alternatives if you prefer: `symsight`,
   `deadsight`, `deadsym`, `oculint`; taken: `pysight`, `codesight`, `hawkeye`, `vigil`,
   `oversight`, `farsight`, `sightline`.
2. **The console script stays `pyvision`** (`pyvision = symvision.cli:main`), per
   Bryan's explicit requirement. The pragma prefix stays `# pyvision:` and the env vars
   stay `PYVISION_EXTERNAL_REPO_PATHS` / `PYVISION_EXTERNAL_REPO_CACHE` — sase's
   pragmas, memory, and muscle memory all depend on them. (PyPI's `pyvision` package is
   a library; a console-script collision only matters if both are installed in one env —
   acceptable.)
3. **Behavior parity is byte-exact**, streams and exit codes included.
   `memory/pyvision.md` and agent workflows quote exact error strings; golden tests must
   lock them. The only additive change allowed in v0.1.0 is a `--version` flag.
4. **`requires-python >= 3.11`** (the script uses `tomllib`); zero runtime dependencies
   (stdlib only). CI matrix 3.11 → 3.14.
5. **`BD_COMMAND` integration is kept as-is** (generic `<cmd> show <bead_id>`, default
   `bd`, "CLOSED" substring check). README documents it as a pluggable
   issue/bead-tracker hook; sase keeps passing `BD_COMMAND=tools/sase_bead`.
6. **Release automation mirrors the toolong repo pattern**: release-please (python
   release-type, manifest mode, default `GITHUB_TOKEN`, single invocation), PyPI trusted
   publishing (OIDC) via `pypa/gh-action-pypi-publish` with a `pypi` GitHub environment.
7. **Internals stay private.** The package is a CLI tool, not a library: split the
   script into focused private modules, ship `py.typed`, but document that the CLI is
   the only supported interface.
8. **Dogfooding**: the symvision repo lints itself with its own `pyvision` binary in
   `just lint` and CI.

## Human Prerequisites (Bryan — needed before Phase 4 can publish)

- On pypi.org: add a **pending trusted publisher** for the new project name `symvision`
  (owner `bbugyi200`, repo `symvision`, workflow `publish.yml`, environment `pypi`).
  Without this the first publish job fails with a trusted-publisher error; the Phase 4
  agent should detect that, notify you, and stop rather than retry.
- Nothing else: Phase 1 creates the GitHub repo via `gh repo create`, and Phase 3
  creates the `pypi` environment via `gh api`.

## Parity Contract

**Source of truth: `tools/pyvision-260708` in the sase repo** (identical to the dotfiles
original except the vendoring header). Phase agents run from a sase workspace and must
read/copy it directly — this is a port-and-restructure of existing working code, not a
reimplementation. Preserve behavior exactly; the invariants below are the ones most at
risk during restructuring:

- **CLI**:
  `pyvision [--exclude-file PATH]... [--epic-symbol '<bead_id>(<symbol>)']... [--exclude-decorator NAME]... [-E|--external-repo-path PATH]... <directory>`,
  plus `-h/--help` and (new) `--version`. Set `prog="pyvision"`.
- **Exit codes**: 0 = clean (also "No Python files found" / "No public functions or
  classes found!"); 1 = any violation, pragma/epic/validation error, or bad directory; 2
  = argparse usage error.
- **Streams**: warnings, pragma/epic/private-symbol errors, and no-files notices go to
  **stderr**; the unused-public report and the final success line
  (`All public/private classes/functions are used properly!`) go to **stdout**.
- **Check ordering and short-circuiting**: epic-symbol validation errors abort before
  pragma validation; pragma errors abort before private-symbol checks; private-imported
  errors abort before private-unused; private-unused aborts before the unused-public
  report. Each error class prints its full batch before exiting.
- **Exact message strings** for every error/report class (they are documented verbatim
  in sase's `memory/pyvision.md` and `tools/CLAUDE.md`).
- **Scanning rules**: top-level defs only; `main` exempt; entry-point functions from the
  nearest `pyproject.toml` (walking up from the target dir; `[project.scripts]`,
  `[project.gui-scripts]`, `[project.entry-points.*]`) exempt; test-support paths (any
  `test`/`tests`/`testing` path component or `test_*.py`) excluded from definitions and
  from public-usage evidence; `.venv`/`venv` excluded; usage-only files = git-tracked
  `*.py` outside the target tree.
- **Pragmas**: contiguous `# pyvision: <ref>` lines directly above the def (above
  decorators), stackable; local targets must exist, not be markdown, not be
  test-support, not live under `src/`, and contain `\b<symbol>\b`; stale-pragma
  detection when the symbol is already imported; URI targets (scheme `://` or scp-style
  `user@host:path`) resolved via explicit `-E` paths + `PYVISION_EXTERNAL_REPO_PATHS`
  (os.pathsep-separated) + git-root siblings matched by normalized origin (with the
  exact-name / other / `name_<N>` candidate ranking), falling back to a depth-1 clone
  under `PYVISION_EXTERNAL_REPO_CACHE` (default `~/.cache/pyvision/external-repos`,
  slug+sha256 dir naming); `file://` URIs resolve directly. External reference check
  scans non-test tracked files: `.py` via AST usage, everything else via word-boundary
  text match. Externally-proven symbols seed the public-API dependency graph
  (annotations, bases, dataclass fields, called public names) so transitively-required
  publics count as used.
- **Epic symbols**: the five validations (format, non-private, bead exists / not CLOSED
  via `BD_COMMAND show`, symbol exists as a public def, symbol not already used) with
  their exact self-cleaning error messages.
- **Unparseable files**: `Warning: Could not parse {file}: {e}` on stderr, file skipped,
  run continues.

## Target Repo Shape (`bbugyi200/symvision`)

Modeled on sase and the toolong repo (hatchling + uv + Justfile + ruff + mypy + pytest),
scaled to a single-tool repo:

- `pyproject.toml` — name `symvision`, `requires-python >= 3.11`, zero runtime deps,
  console script `pyvision = symvision.cli:main`, dev extras (ruff, mypy, pytest,
  pytest-cov, build, twine), MIT license, full metadata/classifiers/urls.
- `src/symvision/` — `__init__.py` with
  `__version__ = "0.1.0"  # x-release-please-version`; `cli.py` (argparse +
  orchestration); focused internal modules (suggested: file-info extraction, usage
  search, pragma validation, external repo resolution, epic symbols, entry points, git
  helpers — exact boundaries are the Phase 1 agent's call); `py.typed`.
- `tests/` — pytest suite per the Phase 2 test matrix below.
- `Justfile` — `install`, `fmt`, `fmt-check`, `lint` (ruff check + mypy + self-lint via
  its own `pyvision` binary), `test`, `check`.
- `.github/workflows/ci.yml` — lint job + test job with Python matrix 3.11 → 3.14,
  driven through `just` + `uv` (sase's `ci.yml` is the reference).
- `.github/workflows/pr-title.yml` — Conventional Commits title check (copy sase's).
- `.github/workflows/publish.yml` — release-please → build (`uv build` + `twine check`)
  → install-smoke (install the wheel in a fresh venv, run `pyvision --help` and a real
  scan of a fixture tree, assert exit codes) → publish (OIDC, `environment: pypi`,
  `skip-existing: true`). Single release-please invocation.
- `release-please-config.json` + `.release-please-manifest.json` — python release-type,
  `include-v-in-tag`, `bump-minor-pre-major` + `bump-patch-for-minor-pre-major`, generic
  extra-file updater for `src/symvision/__init__.py`. Manifest starts at `0.0.0` so the
  first release PR lands `0.1.0`.
- `README.md` — excellent, top-tier: badges (CI, PyPI version, Python versions,
  license); pitch (why unused-symbol hygiene matters for humans _and_ for coding agents
  — dead code wastes LLM context and misleads agents); quick start
  (`uv tool install symvision`, pipx, pip — note the package installs a `pyvision`
  executable); the rule set explained (public/private/`main`/entry-point exemptions,
  test references don't count); the decision hierarchy for fixing findings (delete →
  make private → pragma → epic symbol); full pragma guide (local paths, forbidden
  targets, URI pragmas, cache env vars, `-E`/`PYVISION_EXTERNAL_REPO_PATHS`);
  epic-symbol + `BD_COMMAND` integration; `--exclude-decorator`/`--exclude-file`;
  exit-code table; CI integration snippet (GitHub Actions + Justfile); development
  section (`just check`); release process section (release-please + Conventional
  Commits, PyPI trusted publishing).
- `LICENSE` (MIT), `.gitignore`.

## Phases

Each phase is completed by a distinct agent instance. Phases 1–4 operate on a plain
clone: `gh repo clone bbugyi200/symvision ~/projects/github/bbugyi200/symvision` (clone
only if the directory does not already exist; otherwise pull). Push directly to `master`
(fresh personal repo, no branch protection). **Every commit message must follow
Conventional Commits** — release-please derives versions from them (the initial port is
`feat: ...`). Phase 5 operates on the sase repo through the normal sase workspace/commit
flow.

### Phase 1 — Create the repo and port the tool

Deliverable: `bbugyi200/symvision` exists; master contains a working, modularized
`pyvision` CLI with a focused test suite; `just check` green.

1. Create the repo:
   `gh repo create bbugyi200/symvision --public --description "Unused/misused Python symbol linter (installs the pyvision CLI)"`,
   then clone as described above. If creation fails on auth scopes, notify Bryan and
   stop.
2. Scaffold `pyproject.toml`, `src/symvision/`, `tests/`, `Justfile`, `LICENSE`,
   `.gitignore`, and a minimal placeholder `README.md` (one paragraph + install/usage
   stub; Phase 3 completes it).
3. Port `tools/pyvision-260708` (read it from the sase workspace this agent starts in)
   into the module layout per the Target Repo Shape — a restructuring of working code,
   not a rewrite. Preserve every behavior in the Parity Contract; add `--version`.
4. Write a focused first test suite: CLI smoke tests (clean tree, unused-public,
   private-import, private-unused, exit codes, stdout/stderr split), plus unit tests for
   symbol extraction and usage search. Configure ruff + mypy (strict) in
   `pyproject.toml` following sase's conventions.
5. Verify: `just check` green in the symvision clone, including the self-lint
   (`pyvision src/symvision` passes — internal cross-module imports keep shared symbols
   alive; everything else stays private).
6. Sanity-check parity by hand from the sase workspace: run the vendored script and the
   new CLI against a small synthetic fixture tree and against `src/sase`
   (`BD_COMMAND=tools/sase_bead`, venv python) and confirm identical stdout, stderr, and
   exit codes.
7. Commit (`feat: port pyvision to a standalone symvision package`) and push to master.

### Phase 2 — Exhaustive test suite

Deliverable: a comprehensive, network-free pytest suite locking the Parity Contract;
`just check` green.

Cover every feature area (git fixture repos built in `tmp_path` with `git init`; a stub
`BD_COMMAND` executable script; no network anywhere):

1. **Golden message tests**: exact-string assertions for every error/report class quoted
   in the Parity Contract, including stream (stdout vs stderr) and exit code.
2. **Scanning rules**: test-support exclusions (`test`/`tests`/`testing` components,
   `test_*.py`), `testing/` defs ignored, `.venv` exclusion, `main` + entry-point
   exemptions (scripts, gui-scripts, entry-points groups; nearest pyproject.toml wins),
   `--exclude-file`, `--exclude-decorator` (plain, dotted, call, decorated-class forms).
3. **Usage detection**: `from X import name`, module aliases (`import a.b as c` +
   `c.d.Symbol` chains), tracked-module suffix resolution, `__init__` collapsing,
   usage-only files outside the target tree, test files not counting toward public usage
   but private-import behavior matching the script exactly.
4. **Pragmas (local)**: stacked pragmas, pragma above decorators, private-symbol pragma
   error, stale pragma, missing file, markdown forbidden, test-support forbidden,
   inside-src/ forbidden, no-symbol-reference error, no-git-root error.
5. **Pragmas (external)**: `file://` direct resolution; origin-URL normalization (https
   vs scp-style vs `.git` suffix vs case); sibling-checkout discovery and candidate
   ranking (exact name / other / `name_<N>`); `-E` and `PYVISION_EXTERNAL_REPO_PATHS`;
   cache-hit and cache-mismatch paths via a pre-seeded `PYVISION_EXTERNAL_REPO_CACHE`;
   the clone-fallback branch via a monkeypatched `subprocess.run` (the only
   untestable-offline branch — do not hit the network); Python-vs-text reference
   detection in external repos; non-test filtering there too; API dependency-graph
   reachability from externally-proven roots (annotations, bases, dataclass fields,
   called names).
6. **Epic symbols**: all five validation errors byte-exact, `BD_COMMAND` env override,
   open-bead pass-through, multiple entries, error batching.
7. **Robustness**: unparseable file warning + continue, empty directory, not-a-directory
   error, files without trailing newlines, non-UTF-8 external text files skipped.
8. Add a coverage gate to `just test` (pytest-cov; pick a threshold ≥ 90% and enforce
   it).
9. Commit (`test: ...`) and push; `just check` green.

### Phase 3 — CI, release automation, README

Deliverable: green CI on master; release automation in place; polished README.

1. In the symvision clone: add `ci.yml`, `pr-title.yml`, `publish.yml`,
   `release-please-config.json`, `.release-please-manifest.json` per the Target Repo
   Shape (sase's `.github/workflows/*.yml` and release-please files are the reference
   material; the toolong repo, if its epic has landed by now, is an even closer
   reference).
2. Create the `pypi` GitHub environment:
   `gh api -X PUT repos/bbugyi200/symvision/environments/pypi`.
3. Write the full README (see the README spec above — treat "excellent" as a hard
   requirement, not filler).
4. Commit with Conventional Commit messages (workflows as `ci:`; README as `docs:`) and
   push; watch CI to green with `gh run watch` (fix forward if red).
5. The push also triggers `publish.yml`'s release-please job, which opens a release PR
   for `v0.1.0`. Leave it open — merging it is Phase 4's job.

### Phase 4 — First release: v0.1.0 on PyPI

Deliverable: `symvision==0.1.0` installable from PyPI.

Blocked on the human prerequisite: PyPI pending trusted publisher for `symvision` (see
above).

1. Confirm the release-please PR exists and proposes `0.1.0` (title like
   `chore(master): release 0.1.0`); confirm the CHANGELOG contents; merge it.
2. Watch the resulting `publish.yml` run: release created → build → install-smoke →
   publish. If the publish step fails on trusted-publisher configuration, notify Bryan
   with the exact error and stop (do not switch to token-based auth).
3. Verify end-to-end: in a scratch venv, `pip install symvision==0.1.0`, run
   `pyvision --help` and `pyvision --version`, and run a real scan of a fixture tree
   from the published artifact, checking exit codes.
4. Confirm the GitHub release `v0.1.0` and tag exist.

### Phase 5 — Migrate sase to the published package

Deliverable: sase uses `symvision` from PyPI; the vendored script is gone; `just check`
passes.

Pre-reading for this phase: use `/sase_memory_read` on `memory/pyvision.md` before
starting.

1. `pyproject.toml`: add `symvision>=0.1.0,<0.2.0` to the `dev` extras (a lint-time
   tool, like ruff/mypy).
2. `Justfile`: in `_lint-pyvision` and the public `pyvision` recipe, replace
   `{{ venv_bin }}/python tools/pyvision-260708` with `{{ venv_bin }}/pyvision`, keeping
   `BD_COMMAND=tools/sase_bead`, the `src/sase` argument, and `{{ args }}` pass-through.
   Recipe names, the lint/check stage labels, and the `run_silent "lint (pyvision)"`
   line all stay as-is (the tool is still called pyvision).
3. Live parity check before deleting anything: run the vendored script and the venv
   `pyvision` binary against `src/sase` and confirm identical stdout, stderr, and exit
   codes.
4. Delete `tools/pyvision-260708`.
5. Grep-sweep `pyvision-260708` and `pyvision`: the remaining vendored-path references
   live in `tools/CLAUDE.md` / `tools/AGENTS.md` (and sibling provider shims) and in
   `memory/pyvision.md` — **all protected memory files**. Ask Bryan for permission via
   `/sase_questions` to update them (replace the vendored-script framing with the
   installed-package reality; the rule content itself stays valid). Instructions in this
   plan do NOT count as that permission. If permission is not granted in-conversation,
   leave them untouched and report the follow-up. Everything else found by the sweep
   (`README.md`, `docs/development.md`, `docs/integrations.md`, `src/` pragma comments,
   `tests/test_github_actions_ci.py`, `src/sase/default_config.yml`) stays correct as-is
   because the tool name is unchanged — verify, don't churn.
6. Run `just install` (dep set changed) then `just check`; fix anything red.
7. Commit via the sase commit skill / normal sase CL flow.

## Explicit Non-Goals

- No changes to the chezmoi/dotfiles repo — the original `pyvision` source stays where
  it is ("leave the old copy behind").
- No renames: the executable, pragma prefix (`# pyvision:`), env vars (`PYVISION_*`),
  and sase Justfile recipe names all stay `pyvision`.
- No behavior changes or feature extensions in v0.1.0 beyond `--version` — no config
  file, no JSON output, no new lint rules; parity first.
- No edits to sase's `# pyvision:` pragma comments in `src/`.
- No edits to protected memory files without explicit in-conversation user permission
  (plan approval does not count).

## Risks / Watch-outs

- **PyPI trusted publisher not configured** → Phase 4 publish fails; surfaced as a human
  prerequisite above.
- **Byte-parity of messages and streams** — sase's memory and agent workflows quote
  exact strings; the golden tests in Phase 2 are the enforcement mechanism, and Phases 1
  and 5 each do a live old-vs-new comparison against `src/sase`.
- **Concurrent toolong epic (sase-5r)** — its Phase 4 edits the same sase
  `Justfile`/`pyproject.toml`/docs regions. Whichever migration lands second must rebase
  and re-verify `just check`, and Phase 3 here should prefer the landed toolong repo as
  reference material when available.
- **Network-free tests** — the external-repo clone fallback shells out to `git clone`;
  tests must cover it with a monkeypatched subprocess or pre-seeded cache, never a real
  network clone.
- **`tomllib` floor** — requires Python ≥ 3.11; the CI matrix and `requires-python` must
  agree.
- **release-please first-release mechanics** — manifest starts at `0.0.0`, commits must
  be Conventional; Phase 3 verifies the release PR proposes `0.1.0` before Phase 4
  merges it.
- **Protected memory files** (`tools/CLAUDE.md`/`AGENTS.md` + shims,
  `memory/pyvision.md`) — Phase 5 must ask for permission in-conversation or leave them
  and report the follow-up.
