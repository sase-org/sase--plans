---
create_time: 2026-07-12 19:19:53
status: done
prompt: 202607/prompts/basher_extraction.md
tier: epic
bead_id: sase-5v
---
# Factor pyvendor + bugyi.sh into basher and migrate sase + chezmoi

## Summary

Extract the `pyvendor` vendoring script (chezmoi `home/bin/executable_pyvendor`) **and** the `bugyi.sh` bash library
(chezmoi `home/lib/bugyi.sh`) into the new **public** GitHub repository `bbugyi200/basher`, which publishes a **basher**
package to PyPI via release-please. The basher package becomes the canonical home of the bugyi.sh library (shipped as
package data) and of a modern, user-friendly, configurable `basher` CLI that replaces pyvendor. Then migrate sase off
its vendored copy and retire the chezmoi scripts.

This mirrors the completed `sase-5r` ("Factor pylimit into toolong" → `toobig`) and `sase-5t` ("Factor pyvision into
symvision") epics. The `toobig` repo (`~/projects/github/bbugyi200/toobig`) and `symvision` repo
(`~/projects/github/bbugyi200/symvision`) are the quality bar: basher's packaging, linting, testing, CI, and release
automation must match them — and exceed them where noted below.

### Pre-work already completed (do not redo)

- The old 2021-era `bbugyi200/basher` GitHub repo ("basic package manager for shell libraries") was **renamed to
  `bbugyi200/pybasher`**. Its dormant local checkout was moved to `~/projects/github/bbugyi200/pybasher/` and its
  `origin` remote retargeted to the new name (this matters: GitHub's rename redirect died the moment the new `basher`
  repo was created, so any leftover `basher.git` remote would have silently pointed at the new repo).
- A brand-new, empty, **public** `bbugyi200/basher` repo was created: <https://github.com/bbugyi200/basher>.
- PyPI already has a `basher` project owned by Bryan (the 2021 predecessor, version **0.1.0**, homepage already pointing
  at `bbugyi200/basher`). The new package reuses this PyPI project; consequently the **first release of the rewrite is
  v0.2.0** (see "Release numbering" below).

## Product context

### What pyvendor does today (behavior that must survive the port)

`pyvendor SCRIPT_PATH PROJECT_PATH` (with `-d/--tools-dir`, `-l/--lib-dir`, `-v/--verbose`):

1. Copies SCRIPT into `PROJECT/<tools_dir>/<name>-<yymmdd>` (default `tools/`). When SCRIPT lives under the chezmoi
   source root (`CHEZMOI_SOURCE_ROOT` env override, default `~/.local/share/chezmoi`), a leading `executable_` prefix is
   stripped from the vendored basename.
2. Removes stale date-stamped copies of the same script (both prefixed and unprefixed basenames) from the tools dir.
3. Inserts a provenance comment after the shebang.
4. If the script contains `source ~/lib/bugyi.sh`, also vendors the library into `PROJECT/<lib_dir>/bugyi-<yymmdd>.sh`
   (default `lib/`), removes old `bugyi[_-]*.sh` copies, adds provenance, and rewrites the script's source line to a
   `BASH_SOURCE`-relative path.
5. Restores the executable bit and rewrites **all references** to the removed old filenames throughout the project
   (grep + replace, excluding `.git`).

### What bugyi.sh provides

A ~300-line general-purpose bash library: `die` (printf-style, `-x N` exit codes), `log::debug/info/warn/error`
(colored, caller metadata via `caller`, tees to stderr + syslog via `logger`, honors `DEBUG`/`VERBOSE`/
`DISABLE_LOG_COLOR`), `pyprintf`, `urlencode`, `usage`/`USAGE_GRAMMAR`, plus globals (`SCRIPTNAME`, `COLOR_*`,
`MY_SHELL`, XDG dirs) behind a `BUGYI_HAS_BEEN_SOURCED` double-source guard, and `export TZ="America/New_York"`.

### Consumers (verified by GitHub code search and local greps)

1. **chezmoi** — the two source files; bashunit suites `tests/bash/pyvendor_test.sh` (3 tests) and
   `tests/bash/bugyi_test.sh` (3 tests) run by `just test`; **36 scripts** under `home/bin/` do `source ~/lib/bugyi.sh`;
   `xfiles/bugyi_bash_lib.txt` and `.claude/commands/bugyi_sh.md` reference the library.
2. **sase** — `lib/bugyi-260221.sh` is vendored but **nothing sources it** (only `tools/AGENTS.md` and the generated
   provider shims mention it); `tools/pyscripts-260619` is a pyvendor-vendored Python script (stays vendored; carries
   the legacy provenance format).
3. No other repo contains vendored `bugyi-*.sh` copies. Older repos (`scripts`, `zorg`, `python-lib`, …) source
   `~/lib/bugyi.sh` by convention and keep working as long as that file exists — which the chezmoi migration phase
   guarantees.

## Design

### Package shape

- **Repo/package/CLI**: `basher`, Python `requires-python = ">=3.11"` (needs stdlib `tomllib` for config), CI matrix
  3.11–3.14.
- **One runtime dependency: `rich`.** This is a deliberate divergence from toobig/symvision's zero-dep policy: basher is
  a user-facing workflow tool where beautiful terminal output (tables, diffs, styled status) is an explicit product
  requirement, and rich provides TTY detection, `NO_COLOR`, and consistent rendering for free. Everything else stays
  stdlib.
- **Layout**: `src/basher/` split into focused modules — suggested seams: `cli.py` (argparse tree + rendering entry),
  `engine.py` (vendoring operations), `config.py` (layered config), `provenance.py` (write/parse provenance lines,
  legacy pyvendor format included), `render.py` (rich console helpers), `data/bugyi.sh` (the packaged library),
  `py.typed`. Keep every file comfortably under toobig thresholds.
- **bugyi.sh parity**: the library content is ported byte-identical, with exactly one addition — a
  `readonly BUGYI_VERSION="X.Y.Z"` line inside the double-source guard, maintained by release-please
  (`# x-release-please-version` annotation). Behavior otherwise must not drift.

### CLI surface

```
basher [--version] [-v | -q] [--color {auto,always,never}] <command> ...

  vendor SCRIPT [PROJECT]   Vendor a script (and bugyi.sh if it sources it) into a project
  lib [PROJECT]             Vendor or refresh only the bugyi.sh library
  update [PROJECT]          Refresh every vendored artifact basher can find in the project
  status [PROJECT]          Show vendored artifacts + staleness (rich table; --json for machines)
  cat                       Print the packaged bugyi.sh to stdout (raw, no provenance)
  path                      Print the packaged bugyi.sh path — enables `source "$(basher path)"`
  export [DEST_DIR]         Write an *unversioned* bugyi.sh (with provenance) into DEST_DIR
                            (default ~/lib) — this is how machines materialize ~/lib/bugyi.sh
```

- `PROJECT` defaults to the enclosing git repo root of the cwd (via `git rev-parse --show-toplevel`), falling back to
  the cwd — friendlier than pyvendor's mandatory positional.
- `vendor`/`lib`/`update` options: `-t/--tools-dir` (default `tools`), `-l/--lib-dir` (default `lib`), `--no-lib`
  (vendor only: skip library bundling), `-n/--dry-run` (full preview with rich-rendered diffs, zero writes), `--force`
  (skip the confirmation that guards deleting files lacking a recognizable provenance line).
- Exit codes (documented in README): `0` success, `1` runtime error, `2` CLI usage error, `3` (`status` only) stale
  artifacts found — so CI and scripts can gate on freshness.

### Filename suffix scheme (the headline improvement)

- The vendored **library** is named `bugyi-<basher package version>.sh` (e.g. `bugyi-0.2.0.sh`) — derived from
  `importlib.metadata.version("basher")` — instead of the current date. The suffix now means something: which library
  version a project has.
- Vendored **scripts** keep the `-%y%m%d` date suffix (they carry no version; the date is their provenance). `--suffix`
  overrides either when needed.
- Stale-copy cleanup and project-wide reference rewriting must recognize **both** old date-stamped names
  (`bugyi-260221.sh`, `foo-260619`) and new version-stamped names, so `basher lib` / `basher update` cleanly migrates
  projects vendored by pyvendor.

### Configuration (layered, lowest to highest precedence)

1. Built-in defaults (`tools_dir="tools"`, `lib_dir="lib"`, `color="auto"`).
2. User config: `${XDG_CONFIG_HOME:-~/.config}/basher/config.toml`.
3. Project config: `[tool.basher]` in the project root's `pyproject.toml`, overridden by a `.basher.toml` at the project
   root if present.
4. Environment: `BASHER_TOOLS_DIR`, `BASHER_LIB_DIR`, `BASHER_COLOR` (and the standard `NO_COLOR` is honored).
5. CLI flags.

Config keys are intentionally few (`tools_dir`, `lib_dir`, `color`); per-project configs mean repos like sase never need
flags at all.

### Provenance line (a documented, machine-parseable contract)

Written after the shebang (or as line 1 when there is none):

```
# Vendored by basher v<version> from <source> on <YYYY-MM-DD>. Run 'basher update' to refresh.
```

- Scripts: `<source>` is the absolute source path with `$HOME` abbreviated to `~` (this is what lets `update`
  re-vendor).
- Library: `<source>` is `https://github.com/bbugyi200/basher`.
- `status`/`update` must **also parse the legacy pyvendor format**
  (`# Vendored from https://github.com/bbugyi200/dotfiles via pyvendor on <date>`): legacy library copies are
  refreshable (the source is the packaged lib); legacy scripts lack a source path, so `status` lists them as "legacy —
  re-vendor manually" and `update` skips them with a warning (sase's `tools/pyscripts-260619` is the live example).

### Reliability improvements over pyvendor

- Reference rewriting uses **literal** string replacement (no sed/regex injection), skips binary files, and reports
  every rewritten file.
- Refuses to vendor a file onto itself; validates paths up front with clear errors.
- Idempotency: re-running against an up-to-date project is a colored "already up to date" no-op.
- `--dry-run` previews every copy/delete/rewrite before touching anything.

### Beautiful output

- rich console throughout: `✓`/`↻`/`−` glyph lines for vendored/refreshed/removed files, dim provenance details, a
  proper `status` table (artifact, kind, vendored version/date, latest, colored state), unified diffs in dry-run mode.
- `-q` silences everything but errors; `-v` adds debug detail; `--color`/`NO_COLOR`/non-TTY degrade to plain text.

### Quality bar (parity with toobig/symvision, or better)

Match: hatchling packaging with `py.typed`; uv-based Justfile (`install`/`fmt`/`fmt-check`/`lint`/`test`/`check`,
`BASHER_PYTHON` override); ruff (format + lint) and **strict** mypy; pytest with `--strict-config --strict-markers`;
`.github/workflows/ci.yml` (lint job + per-Python test matrix), `pr-title.yml`, `publish.yml` (release-please v5 →
`uv build` → `twine check` → fresh-venv wheel install smoke test exercising the real CLI → PyPI OIDC publish gated on
the `pypi` GitHub environment); `release-please-config.json` + `.release-please-manifest.json`; README with
CI/PyPI/pyversions/license badges; MIT LICENSE.

Exceed:

- **Dogfooding**: `just lint` runs `toobig` (sase's `1000 850 700` thresholds) and `symvision` (both as dev deps)
  against basher itself.
- **Shellcheck**: `just lint` shellchecks `src/basher/data/bugyi.sh` via the `shellcheck-py` dev dep.
- **Coverage gate**: CI enforces ≥90% (set to what the suite honestly supports, never below toobig's effective bar).
- **Docs depth**: README documents every subcommand and flag, the config layers and precedence, the provenance contract
  (both formats), the suffix scheme, exit codes, and a function-by-function bugyi.sh library reference (`die`, `log::*`,
  `pyprintf`, `urlencode`, `usage`, exported globals) — today those docs exist nowhere.

### Release numbering

`pyproject.toml` starts at `version = "0.1.0"` and `.release-please-manifest.json` is bootstrapped at `"0.1.0"` —
matching the version already on PyPI from the 2021 predecessor — so the first `feat:` commits make release-please
propose **v0.2.0**, which is publishable. Never propose 0.1.0; PyPI will reject the duplicate.

## Phases

Each phase is completed by a distinct agent instance. Phases 1–5 work in `~/projects/github/bbugyi200/basher/`
(initialized in Phase 1 against the existing empty GitHub repo — use that directory directly; do not create another
checkout). Phase 6 works in a sase workspace. Phase 7 works in the chezmoi repo opened via
`sase workspace open -p chezmoi -r "<reason>" <workspace_num>`. All commits use conventional-commit messages
(release-please derives versions from them) and go through the sanctioned sase commit workflow. During bootstrap (Phases
1–4) commits may land directly on `master` (set the default branch to `master`; all workflows and release-please target
it, as in toobig/symvision); after v0.2.0, use the normal PR flow.

### Phase 1 — Initialize the repo; port the engine and core commands

1. Initialize a fresh git repo at `~/projects/github/bbugyi200/basher/`, default branch `master`, SSH remote
   `git@github.com:bbugyi200/basher.git` (the GitHub repo already exists and is empty), and push the initial commit.
2. Scaffold packaging and dev tooling to the toobig/symvision template (adapted per "Quality bar"): `pyproject.toml`
   (name `basher`, version `0.1.0`, hatchling, MIT, `requires-python >=3.11`, runtime dep `rich`, dev extras
   `build/mypy/pytest/pytest-cov/ruff/shellcheck-py/symvision/toobig/twine`,
   `[project.scripts] basher = "basher.cli:main"`, URLs, classifiers, ruff/strict-mypy/pytest/coverage config),
   `LICENSE`, `.gitignore`, `Justfile` (uv venv pattern, `BASHER_PYTHON` override), `py.typed`.
3. Port the sources (read them from the chezmoi repo, opened via
   `sase workspace open -p chezmoi -r "<reason>" <workspace_num>`): copy `home/lib/bugyi.sh` to
   `src/basher/data/bugyi.sh` (byte-identical + the `BUGYI_VERSION` line), and reimplement pyvendor's behavior in Python
   per the Design section — module seams, suffix scheme, provenance v2 + legacy parsing, reliability improvements, rich
   output.
4. Implement the core commands this phase: `vendor`, `lib`, `cat`, `path`, `export`, `--version` (leave
   `status`/`update`/config layers/`--dry-run` for Phase 2, but structure `engine.py` so they bolt on cleanly).
5. Add a pytest smoke suite (CLI invocation, exit codes, one vendor happy path with library bundling, prefix stripping
   via `CHEZMOI_SOURCE_ROOT`) so `just check` is meaningful; the full suite is Phase 3.
6. Verify: `just check` green. Additionally run a **parity check** in a scratch project: run chezmoi's `pyvendor` and
   `basher vendor` against the same script + fake `~/lib/bugyi.sh` home, and diff the resulting trees — they must agree
   modulo the documented suffix and provenance changes.

### Phase 2 — Config layers, status/update, and UX polish

1. Implement the layered config system exactly per Design (defaults → user config → `[tool.basher]`/`.basher.toml` →
   `BASHER_*` env → flags), with `tomllib`.
2. Implement `status` (rich table, `--json`, exit 3 on staleness) and `update` (refresh library + re-vendorable scripts;
   legacy handling per Design).
3. Implement `-n/--dry-run` (rendered previews + diffs, zero writes), `--force`, `-q`/`-v`, `--color`/`NO_COLOR`
   handling, and the "already up to date" idempotent path.
4. Extend the smoke tests to cover each new command and at least one config-precedence case.
5. Verify: `just check` green; manually exercise `basher status`/`basher update` against a scratch project vendored by
   old pyvendor (date-stamped names) and confirm the migration path (cleanup + reference rewrite) works.

### Phase 3 — Full test suite + coverage gate

1. Port the semantics of chezmoi's bashunit suites into pytest: all 3 `pyvendor_test.sh` cases (prefix stripping,
   old-reference cleanup, non-chezmoi basename preservation) and all 3 `bugyi_test.sh` cases (`pyprintf`, `log::info`,
   `die` exit codes — run real `bash` subprocesses that source `src/basher/data/bugyi.sh`).
2. Add focused unit tests: config precedence matrix, provenance write/parse (both formats), suffix/cleanup/reference
   rewriting edge cases (binary files skipped, literal replacement), `status --json` schema, dry-run writes nothing,
   `update` legacy-script skip warning, `export` unversioned output, project-root discovery.
3. Wire the coverage gate (pyproject coverage config + Justfile recipe used by CI) at the highest honest threshold
   (target ≥90%).
4. Verify: `just check` green with the coverage gate enforced.

### Phase 4 — CI, release automation, README

1. Add the three workflows adapted from toobig/symvision, Python matrix 3.11–3.14, `BASHER_PYTHON` plumbed through:
   `ci.yml`, `pr-title.yml`, `publish.yml` (release-please v5, build + `twine check`, fresh-venv wheel install smoke
   test running `basher --version`, `basher cat | bash -n`, and a real `basher vendor` scenario asserting exit codes,
   then OIDC publish under the `pypi` environment).
2. Add `release-please-config.json` (python release-type, `include-v-in-tag`, pre-major bump settings, `extra-files` →
   `src/basher/__init__.py` **and** `src/basher/data/bugyi.sh`) and `.release-please-manifest.json` at `"0.1.0"` (see
   "Release numbering" — first proposed release must be v0.2.0).
3. Create the `pypi` environment on the GitHub repo (`gh api`), matching toobig/symvision.
4. Write the README to the documented quality bar (badges, motivation, quick start via `uv tool install basher`, full
   CLI/config/provenance/suffix/exit-code docs, bugyi.sh library reference, development section). Ensure at least one
   `feat:` commit exists so release-please proposes **v0.2.0**.
5. Verify: CI green on GitHub (`gh run watch`); PR-title workflow behaves; release-please opens its release PR proposing
   v0.2.0. Leave the release PR open for Phase 5.

### Phase 5 — First release: v0.2.0 on PyPI

**USER CHECKPOINT (before this phase runs):** on pypi.org, add a **trusted publisher to the existing `basher` project**
(Manage → Publishing): owner `bbugyi200`, repository `basher`, workflow `publish.yml`, environment `pypi`. Note this is
"add a publisher" on an existing project (Bryan owns basher 0.1.0 from 2021), _not_ a pending publisher as
toobig/symvision needed. Requires a pypi.org login; the phase agent must stop with a clear notification if the publisher
is missing.

1. Merge the release-please PR (squash, conventional title).
2. Watch the Publish workflow end-to-end: GitHub release + `v0.2.0` tag; build, twine check, install-smoke pass; PyPI
   publish succeeds.
3. Verify independently: PyPI JSON API reports `basher 0.2.0`; in a clean environment `uv tool install basher` yields a
   working CLI (`basher --version`, `basher cat`, `basher export` into a temp dir,
   `bash -c 'source <exported file> && log::info smoke'`).

### Phase 6 — Migrate sase to the published package

Work in a sase workspace (run `just install` first; ephemeral workspaces may have drifted deps).

1. Confirm the current state of the symvision migration (the sase-side commit may or may not have merged by now); rebase
   awareness only — do not reintroduce anything it removed.
2. Delete `lib/bugyi-260221.sh` (and the then-empty `lib/` dir) after re-verifying with a repo-wide grep that nothing
   sources it.
3. **Memory-file edits require explicit user permission** granted in that agent's own conversation: update
   `tools/AGENTS.md` (remove the `../lib/bugyi-260221.sh` sentence; state that vendored tools are refreshed with
   `basher vendor`/`basher update` from the `basher` PyPI package instead of pyvendor) and regenerate/update the
   provider shims (`tools/CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). The launch prompt for this phase should
   explicitly grant permission for exactly these files; if it does not, leave them untouched and flag the follow-up.
4. Sweep remaining references: `grep -rn 'pyvendor\|bugyi'` outside memory files and archived plans; update or remove
   stragglers (e.g. any memory file that mentions pyvendor is covered by the same permission grant above).
5. Do NOT touch `tools/pyscripts-260619` itself — it stays vendored with its legacy provenance (now recognized by
   `basher status`).
6. Verify: `just install`, full `just check`. Confirm any failures are pre-existing at origin/master before treating
   them as regressions.

### Phase 7 — Migrate chezmoi and retire the scripts

Open the chezmoi repo via `sase workspace open -p chezmoi -r "<reason>" <workspace_num>`. This phase must keep
`~/lib/bugyi.sh` existing on the machine at all times — 36 `home/bin` scripts source it.

1. Add `home/.chezmoiscripts/run_onchange_install_basher.tmpl` modeled on the existing
   `run_onchange_install_bashunit.tmpl` (including its monthly `{{ output "date" "+%Y-%m" }}` refresh trick and
   `chez::log` usage): `uv tool install --upgrade basher` then `basher export ~/lib` so chezmoi materializes
   `~/lib/bugyi.sh` from the package instead of managing the file itself. The 36 consumer scripts keep their
   `source ~/lib/bugyi.sh` lines completely unchanged.
2. Delete `home/lib/bugyi.sh`, `home/bin/executable_pyvendor`, `tests/bash/pyvendor_test.sh`, and
   `tests/bash/bugyi_test.sh`.
3. Update the two referencing docs: `xfiles/bugyi_bash_lib.txt` (point it at the basher repo / `basher cat` as the
   library's home) and `.claude/commands/bugyi_sh.md` (reference the library via basher instead of `home/lib/bugyi.sh`).
4. Sweep `grep -rn 'pyvendor\|bugyi\.sh'` for remaining live references (historical plan archives under
   `sase/repos/plans/` are archives — leave them).
5. Verify chezmoi's `just test` passes, then commit via the sanctioned commit workflow. Per the chezmoi repo
   instructions, run `chezmoi update -a --force` after committing, then verify on the machine: `~/lib/bugyi.sh` exists
   (freshly exported, carrying the basher provenance line and `BUGYI_VERSION`), and
   `bash -c 'source ~/lib/bugyi.sh && log::info smoke'` plus one real consumer (e.g. `copando --help` or `helphelp -h`)
   still work.

## Dependencies and bead structure

Create an epic bead mirroring `sase-5r`/`sase-5t`, with seven phase children in a strict chain (1 → 2 → 3 → 4 → 5 → 6 →
7). Each child bead's description must state its repository/worktree requirement (Phases 1–5:
`~/projects/github/bbugyi200/basher/`; Phase 6: the sase repo; Phase 7: chezmoi via `sase workspace open`). Phase 5
additionally carries the PyPI trusted-publisher user checkpoint.

## Risks / notes for implementers

- **`~/lib/bugyi.sh` availability is the crown jewel.** 36 scripts break if it disappears. Phase 7's install script must
  be applied and verified in the same change that deletes chezmoi's copy, and the post-apply smoke checks are mandatory.
- **Behavior parity is the contract** for both the library (byte-identical + version line) and the vendoring flow (Phase
  1 parity diff, Phase 3 ported suites). The suffix and provenance changes are the only intended diffs.
- **PyPI version collision**: `basher 0.1.0` exists (2021). The manifest bootstrap at `0.1.0` is what makes
  release-please propose v0.2.0 — do not "simplify" it to 0.0.0.
- **Old repo redirect is gone**: `git@github.com:bbugyi200/basher.git` now resolves to the _new_ repo. The dormant
  checkout was already moved/retargeted to `pybasher`; nothing else is known to reference the old URL (PyPI 0.1.0's
  homepage link now lands on the new repo, which is acceptable since the project continues there).
- **rich dependency** is a deliberate, documented divergence from the zero-dep siblings; do not add further runtime
  deps.
- **Legacy provenance parsing** is load-bearing for sase's `tools/pyscripts-260619` and for migrating any date-stamped
  `bugyi-*.sh` copies; don't skip it.
- **Memory files**: no agent may edit `memory/*.md`, `AGENTS.md`, or generated provider shims without explicit user
  permission in its own conversation (Phase 6, step 3). Chezmoi's memory files contain no pyvendor/bugyi references, so
  Phase 7 needs no such grant.
- **Known sase-side flakiness**: verify `just check` failures against a clean baseline before attributing them to the
  migration; if the sandbox kills the full suite (exit 144), fall back to static gates plus targeted pytest subsets.
