---
bead_id: sase-5r
create_time: 2026-07-11 19:59:31
status: wip
prompt: 202607/prompts/toolong_extraction.md
tier: epic
---
# Plan: Factor `pylimit` into `bbugyi200/toolong` and Migrate sase to It

## Context

`pylimit` is a bash script that enforces per-file line-count limits with three thresholds (hard limit / warning / info).
It lives in the chezmoi dotfiles repo (`home/bin/executable_pylimit`, plus the `home/bin/executable_pylimit_files`
wrapper that prints only offending file paths) and is vendored into the sase repo as `tools/pylimit-260221` and
`tools/pylimit_files-260227` (both source the vendored `lib/bugyi-260221.sh`).

Goal: extract this tool into a dedicated, language-agnostic Python project published on PyPI, then migrate sase to
consume the published package. The chezmoi originals are left behind untouched (no chezmoi changes anywhere in this
plan).

**The GitHub repo has already been created (empty): <https://github.com/bbugyi200/toolong>.**

## Decisions Already Made (flag at review if you disagree)

1. **PyPI distribution name: `bbugyi-toolong`.** The name `toolong` is taken on PyPI (Textualize's log viewer). The
   GitHub repo stays `bbugyi200/toolong` as requested. Import package is `bbugyi_toolong` (avoids a site-packages
   collision with Textualize's `toolong` import package if both are ever installed). The console script is still
   `toolong` (Textualize's tool installs `tl`, so no entry-point conflict).
2. **Language-agnostic via `--include GLOB` (repeatable), defaulting to `*.py`.** The default preserves drop-in parity
   with `pylimit` (same args → same behavior); `--include '*.rs'` makes it work on a Rust project.
3. **`pylimit_files` is subsumed by a `--files-only` flag** instead of being a second script.
4. **Release automation: release-please** (python release-type, manifest mode), modeled on sase's `publish.yml`, using
   the default `GITHUB_TOKEN` (simpler than sase's PAT; caveat: CI does not auto-run on release-please PRs, which is
   acceptable on a fresh repo with no required checks).
5. **Publishing: PyPI trusted publishing (OIDC)** via `pypa/gh-action-pypi-publish` with a `pypi` GitHub environment,
   like sase.
6. **The sase xprompt keeps its `pylimit_split` name.** It is referenced as `#!sase/pylimit_split` by external config
   (chezmoi `sase_athena.yml`); only its internal scanner command changes.

## Human Prerequisites (Bryan — needed before Phase 3 can publish)

- On pypi.org: add a **pending trusted publisher** for the new project name `bbugyi-toolong` (owner `bbugyi200`, repo
  `toolong`, workflow `publish.yml`, environment `pypi`). Without this the first publish job will fail with a
  trusted-publisher error; the Phase 3 agent should detect that and notify you rather than retrying blindly.
- Nothing else: the `pypi` GitHub environment can be created by the Phase 2 agent via `gh api`.

## Parity Contract (spec for the new tool — self-contained, no chezmoi access needed)

Source of truth for behavior: `tools/pylimit-260221` and `tools/pylimit_files-260227` in the sase repo (identical to the
chezmoi originals except for the vendoring header and lib path).

### CLI

```
toolong [-v]... [--include GLOB]... [--files-only] <directory> <line_limit> <warning_limit> <info_limit>
toolong -h | --help
```

- `directory`: scanned recursively (hidden files included, symlinks not followed, no gitignore awareness — same as
  `find <dir> -type f`).
- `line_limit` > `warning_limit` > `info_limit`, all positive integers.
- `-v/--verbose` repeatable: `-v` reveals per-file debug `OK:` lines; `-vv` adds extra trace output (the bash `set -x`
  equivalent is just more verbose diagnostics — document the difference).
- Line counting must match `wc -l` semantics: count of newline characters (a file without a trailing newline reports one
  less than its visual line count). Write a parity test for this.

### Exit codes

| Condition                                                                             | Exit                               |
| ------------------------------------------------------------------------------------- | ---------------------------------- |
| Missing positional argument                                                           | 2 (usage error, message to stderr) |
| Directory does not exist / limit not a positive integer / threshold ordering violated | 1                                  |
| One or more files exceed `line_limit`                                                 | 1                                  |
| Only warnings/FYIs, or all files within limits                                        | 0                                  |

### Output (stderr, log-style)

Classification lines — the `LEVEL: <path> has N lines ...` core must be byte-identical to pylimit (downstream consumers
grep `(VIOLATION|WARNING|FYI): ` and sed out the path before `has`):

- `VIOLATION: {file} has {n} lines (limit: {line_limit})` — error level
- `WARNING: {file} has {n} lines (warning: {warning_limit}, limit: {line_limit})` — warn level
- `FYI: {file} has {n} lines (info: {info_limit}, warning: {warning_limit}) - will trigger warning soon` — info level
- `OK: {file} has {n} lines` — debug level (only with `-v`)

Summary lines (same shape as pylimit; the word "Python" is generalized to "files" since the tool is language-agnostic —
intentional, documented deviation):

- violations > 0 → error `Found {v} file(s) exceeding line limit of {line_limit}`, exit 1
- else warnings > 0 → warn `Found {w} file(s) exceeding warning limit of {warning_limit}`
- else infos > 0 → info `Found {i} file(s) exceeding info limit of {info_limit}`
- else → info `All files are within the info limit of {info_limit}`

A startup info line describing directory, patterns, and thresholds is kept but may be reworded. Log lines carry a simple
level-colored prefix (bugyi.sh's PID/caller metadata is NOT reproduced); color is suppressed when stderr is not a TTY or
`NO_COLOR` is set.

### `--files-only` mode (replaces `pylimit_files`)

Prints only the unique, sorted paths of files that triggered VIOLATION/WARNING/FYI, one per line, to **stdout** (nothing
when there are none), and exits with the same code the normal run would.

## Target Repo Shape (`bbugyi200/toolong`)

Modeled on sase (hatchling + uv + Justfile + ruff + mypy + pytest), scaled down:

- `pyproject.toml` — name `bbugyi-toolong`, `requires-python >= 3.10`, **zero runtime deps** (stdlib only), console
  script `toolong = bbugyi_toolong.cli:main`, dev extras (ruff, mypy, pytest, pytest-cov, build, twine), MIT license,
  full metadata/classifiers/urls.
- `src/bbugyi_toolong/` — `__init__.py` with `__version__ = "0.1.0"  # x-release-please-version`, `cli.py` (argparse),
  small pure modules for scanning/classification (keep logic testable without subprocess), `py.typed`.
- `tests/` — pytest suite: classification/threshold edge cases (exactly-at-limit is OK — pylimit uses strict `>`),
  `wc -l` parity, exit codes, `--files-only` output, `--include` globs (Rust fixture), validation errors, missing-arg
  usage errors, NO_COLOR.
- `Justfile` — `install`, `fmt`, `fmt-check`, `lint` (ruff check + mypy), `test`, `check` (all of the above), mirroring
  sase's recipe naming but without sase-specific machinery.
- `.github/workflows/ci.yml` — lint job (ruff check, ruff format --check, mypy) + test job with a Python version matrix
  (3.10 → 3.14), driven through `just` + `uv` like sase's CI.
- `.github/workflows/pr-title.yml` — Conventional Commits title check (copy sase's).
- `.github/workflows/publish.yml` — release-please → build (`uv build` + `twine check`) → install-smoke (install the
  wheel in a fresh venv, run `toolong --help` and a real scan against a fixture tree, assert exit codes) → publish
  (OIDC, `environment: pypi`, `skip-existing: true`). Single release-please invocation (skip sase's triple-retry).
- `release-please-config.json` + `.release-please-manifest.json` — python release-type, `include-v-in-tag`,
  `bump-minor-pre-major` + `bump-patch-for-minor-pre-major`, generic extra-file updater for
  `src/bbugyi_toolong/__init__.py`. Manifest starts at `0.0.0` so the first release PR lands `0.1.0` for the initial
  `feat:` commits.
- `README.md` — excellent, top-tier: badges (CI, PyPI version, Python versions, license); pitch (why line limits matter
  — for humans _and_ for coding agents/LLM context); quick start (`uv tool install bbugyi-toolong`, pipx, pip); usage
  examples for Python **and** Rust projects; threshold semantics table; exit-code table; `--files-only` scripting
  example; CI integration snippet (GitHub Actions + Justfile); development section (`just check`); release process
  section (release-please + Conventional Commits, PyPI trusted publishing).
- `LICENSE` (MIT), `.gitignore`.

## Phases

Each phase is completed by a distinct agent instance. Phases 1–3 operate on a plain clone of the new repo:
`gh repo clone bbugyi200/toolong ~/projects/github/bbugyi200/toolong` (clone only if the directory does not already
exist; otherwise pull). Push directly to `master` (fresh personal repo, no branch protection). **Every commit message
must follow Conventional Commits** — release-please derives versions from them (the initial port is `feat: ...`). Phase
4 operates on the sase repo through the normal sase workspace/commit flow.

### Phase 1 — Port the tool: code, tests, dev tooling

Deliverable: `bbugyi200/toolong` master contains a working, fully-tested `toolong` CLI.

1. Clone the (empty) repo as described above.
2. Scaffold `pyproject.toml`, `src/bbugyi_toolong/`, `tests/`, `Justfile`, `LICENSE`, `.gitignore`, and a minimal
   placeholder `README.md` (one paragraph + install/usage stub; Phase 2 completes it).
3. Implement the CLI per the Parity Contract above (stdlib only; argparse; pure-function core).
4. Write the pytest suite per the Target Repo Shape section; configure ruff + mypy (strict) in `pyproject.toml`
   following sase's conventions (sase's `[tool.ruff]`/`[tool.mypy]` sections are available in the sase repo for
   reference).
5. Verify locally: `just check` passes (fmt-check + lint + tests) in the toolong clone.
6. Sanity-check parity by hand: run `toolong` and sase's `tools/pylimit-260221` against the sase repo's `src/` tree with
   thresholds `1000 850 700` and confirm identical classification results and exit codes; likewise compare
   `--files-only` against `tools/pylimit_files-260227`.
7. Commit (e.g. `feat: port pylimit to a language-agnostic Python CLI`) and push to master.

### Phase 2 — CI, release automation, README

Deliverable: green CI on master; release automation in place; polished README.

1. In the toolong clone: add `ci.yml`, `pr-title.yml`, `publish.yml`, `release-please-config.json`,
   `.release-please-manifest.json` per the Target Repo Shape section (sase's `.github/workflows/*.yml` and
   release-please files are the reference material).
2. Create the `pypi` GitHub environment: `gh api -X PUT repos/bbugyi200/toolong/environments/pypi`.
3. Write the full README (see the README spec above — treat "excellent" as a hard requirement, not filler).
4. Commit with Conventional Commit messages (workflow/CI changes as `ci:`/`chore:`; README as `docs:`) and push; then
   watch the CI run to green with `gh run watch` (fix forward if red).
5. Note: the push will also trigger `publish.yml`'s release-please job, which opens a release PR for `v0.1.0`. Leave
   that PR open — merging it is Phase 3's job.

### Phase 3 — First release: v0.1.0 on PyPI

Deliverable: `bbugyi-toolong==0.1.0` installable from PyPI.

Blocked on the human prerequisite: PyPI pending trusted publisher for `bbugyi-toolong` (see above).

1. Confirm the release-please PR exists and proposes `0.1.0` (title like `chore(master): release 0.1.0`); confirm
   CHANGELOG contents look right; merge it.
2. Watch the resulting `publish.yml` run: release created → build → install-smoke → publish. If the publish step fails
   on trusted-publisher configuration, notify Bryan with the exact error and stop (do not switch to token-based auth on
   your own).
3. Verify end-to-end: in a scratch venv, `pip install bbugyi-toolong==0.1.0`, run `toolong --help`, and run a real scan
   of some directory with `--include '*.rs'` to prove the language-agnostic path works from the published artifact.
4. Confirm the GitHub release `v0.1.0` and tag exist.

### Phase 4 — Migrate sase to the published package

Deliverable: sase uses `bbugyi-toolong` from PyPI; vendored pylimit scripts are gone; `just check` passes.

Pre-reading for this phase: use `/sase_memory_read` on `memory/xprompts.md` before touching
`xprompts/pylimit_split.yml`.

1. `pyproject.toml`: add `bbugyi-toolong>=0.1.0,<0.2.0` to the `dev` extras (it is a lint-time tool, like ruff/mypy).
2. `Justfile`:
   - Rename `_lint-pylimit` → `_lint-toolong`, invoking `{{ venv_bin }}/toolong src ...` and
     `{{ venv_bin }}/toolong tests ...` with the same default `1000 850 700` / `*args` override behavior (the `*.py`
     default preserves current scan scope).
   - Update the `lint` recipe (stage banner + comment), the `check` recipe line
     (`run_silent "lint (toolong)" just _lint-toolong`), and rename the public `pylimit` recipe to `toolong`.
3. `xprompts/pylimit_split.yml`: replace the `["tools/pylimit_files-260227", tree, *limits]` subprocess invocation with
   the venv's `toolong --files-only` (resolve the binary as `Path(sys.executable).parent / "toolong"` inside the python
   step so it works regardless of PATH). Keep the workflow name, structure, and launch behavior identical.
4. Delete `tools/pylimit-260221` and `tools/pylimit_files-260227`. Then grep for remaining `bugyi-260221.sh` references;
   if the only consumers were those two scripts, delete `lib/bugyi-260221.sh` as well — but note the reference to it in
   `tools/CLAUDE.md` / `tools/AGENTS.md` (and sibling provider shims): those are protected memory files, so either get
   explicit user permission in-conversation before updating them, or leave both the lib file and the shims untouched and
   report that follow-up to the user.
5. Update tests: `tests/test_justfile_lint.py` (recipe names/labels), `tests/test_github_actions_ci.py` (the
   no-`just pylimit` assertion → toolong equivalent), `tests/test_xprompt_pylimit_split.py` (structural assertions over
   the xprompt body and its subprocess shim). Then do a repo-wide `pylimit` grep sweep and fix stragglers in `README.md`
   (lint description), `docs/development.md`, and other docs — EXCEPT: keep the `pylimit_split` xprompt name and its
   test/module references, keep `CHANGELOG.md` history untouched, and never touch anything under a linked-repo checkout.
6. Run `just install` (dep set changed) then `just check`; fix anything red.
7. Commit via the sase commit skill / normal sase CL flow.

## Explicit Non-Goals

- No changes to the chezmoi repo — `home/bin/executable_pylimit` and `home/bin/executable_pylimit_files` stay where they
  are ("leave the old copy behind").
- No rename of the `pylimit_split` xprompt (external config references it).
- No gitignore-awareness, exclude patterns, or config-file support in `toolong` v0.1.0 — parity first; extensions can
  come later.

## Risks / Watch-outs

- **PyPI trusted publisher not configured** → Phase 3 publish fails; surfaced as a human prerequisite above.
- **`wc -l` semantics** — off-by-one vs naive Python line counting on files without trailing newlines; covered by an
  explicit parity test.
- **release-please first-release mechanics** — manifest must start below `0.1.0` and commits must be Conventional; Phase
  2 verifies the release PR proposes `0.1.0` before Phase 3 merges it.
- **Protected memory shims in `tools/`** — Phase 4 must not edit them without explicit user permission (plan approval
  does not count as that permission).
