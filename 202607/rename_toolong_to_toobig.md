---
create_time: 2026-07-12 17:11:42
status: done
prompt: 202607/prompts/rename_toolong_to_toobig.md
tier: tale
---
# Rename toolong → toobig (PyPI, GitHub repo, project dir, sase integration)

## Context

Epic **sase-5r** factored the old in-repo `pylimit` linter into a standalone project `toolong`: GitHub repo
`bbugyi200/toolong`, local checkout `~/projects/github/bbugyi200/toolong/`, PyPI distribution `bbugyi-toolong`, import
package `bbugyi_toolong`, console script `toolong`.

Current state:

- Phases sase-5r.1 (port) and sase-5r.2 (CI/release automation/README) are **closed**.
- Phase sase-5r.3 (first release, v0.1.0 on PyPI) is **stuck**: release-please created the v0.1.0 tag + GitHub release,
  but the publish job (run 29207782700) failed during the trusted-publishing token exchange with `invalid-publisher` —
  the `toolong` name on PyPI is owned by an unrelated project, so no matching trusted publisher exists. **Nothing was
  ever published to PyPI.**
- Phase sase-5r.4 (migrate sase) already landed on sase master (`feat: Migrate from pylimit to toolong`), so sase
  depends on `bbugyi-toolong>=0.1.0,<0.2.0`, which 404s on PyPI → `just install` / `just check` are currently **broken**
  on sase master.
- Bryan has registered the new PyPI project name **`toobig`** as a pending (trusted-publisher) project awaiting its
  first release.

Goal: rename everything to `toobig`, complete the first release under the new name, unbreak the sase integration, and
close beads sase-5r.3, sase-5r.4, and epic sase-5r.

## Naming decisions

| Artifact          | Old                                    | New                                                            |
| ----------------- | -------------------------------------- | -------------------------------------------------------------- |
| PyPI distribution | `bbugyi-toolong` (never published)     | `toobig` (plain name is registered, drop the `bbugyi-` prefix) |
| Import package    | `bbugyi_toolong`                       | `toobig`                                                       |
| Console script    | `toolong`                              | `toobig`                                                       |
| GitHub repo       | `bbugyi200/toolong`                    | `bbugyi200/toobig`                                             |
| Local checkout    | `~/projects/github/bbugyi200/toolong/` | `~/projects/github/bbugyi200/toobig/`                          |

**Re-cut v0.1.0 instead of releasing 0.2.0.** The existing v0.1.0 tag/GitHub release never reached PyPI and the project
has no users, so delete them, reset release-please state, and let release-please re-cut v0.1.0 under the new name. This
keeps "first release = v0.1.0" (sase-5r.3's definition of done) and keeps sase's `>=0.1.0,<0.2.0` pin natural.

**Assumption to verify at release time:** the PyPI pending publisher for `toobig` must match the publish workflow's OIDC
claims — owner `bbugyi200`, repo `toobig`, workflow `publish.yml`, environment `pypi`. Agents cannot log into PyPI, so
if the publish job fails again with `invalid-publisher`, stop and notify Bryan to fix the pending-publisher config on
PyPI (the debugging claims printed by the failed job show the exact values to use).

## Phase 1 — Rename the repo and project (in `~/projects/github/bbugyi200/toolong/`)

1. **GitHub + directory rename.** Run `gh repo rename toobig` from inside the checkout (this also updates the `origin`
   remote URL; GitHub redirects the old URLs, so nothing breaks mid-rename). Then
   `mv ~/projects/github/bbugyi200/toolong ~/projects/github/bbugyi200/toobig` and confirm `origin` points at
   `bbugyi200/toobig`.
2. **Reset the never-published release state.**
   - `gh release delete v0.1.0 --cleanup-tag --yes`; delete the local `v0.1.0` tag too.
   - Reset `.release-please-manifest.json` to `{}` (bootstrap mode — this exact config already produced 0.1.0 from full
     history once, so it will again).
   - Trim `CHANGELOG.md` back to just the `# Changelog` header (the 0.1.0 section will be regenerated with `toobig`
     URLs).
   - Set versions back to `0.0.0` in `pyproject.toml` and the package `__init__.py` (`x-release-please-version` line).
3. **In-tree rename.**
   - `git mv src/bbugyi_toolong src/toobig`; update all imports in `src/` and `tests/`.
   - `pyproject.toml`: `name = "toobig"`, `[project.scripts] toobig = "toobig.cli:main"`, `[project.urls]` →
     `bbugyi200/toobig`, hatch wheel `packages`, coverage `source_pkgs`.
   - `release-please-config.json`: extra-files path → `src/toobig/__init__.py`.
   - `cli.py`: `prog="toobig"`; update module docstrings; `tests/test_cli.py` `usage: toobig` expectation.
   - `README.md`: title, CI/PyPI badge URLs, install commands (`uv tool install toobig`, `pipx`/`pip install toobig`),
     all usage examples, the sample `just` recipe (`uvx --from toobig toobig ...`), the pin-guidance and
     trusted-publishing paragraphs.
   - `.github/workflows/ci.yml` and `publish.yml`: smoke-test venv paths and binary names (`/tmp/toolong-smoke`,
     `bin/toolong`, output file names) → `toobig`.
   - `Justfile` header comment; delete stale `dist/` artifacts built under the old name.
4. **Local verification.** Run the repo's own Justfile lint + tests; `uv build` and install the wheel into a scratch
   venv: `toobig --help` works and the violation exit-code smoke passes. Final sweep: `grep -ri 'toolong\|bbugyi'` over
   tracked files comes back empty.
5. **Commit and push** to master as `feat: rename the project to toobig` (a conventional `feat` commit so
   release-please's bootstrap version math lands on 0.1.0).

## Phase 2 — Release toobig v0.1.0 on PyPI (completes sase-5r.3)

1. The push triggers the Publish workflow; release-please opens a `release 0.1.0` PR. Merge it.
2. The follow-up run tags `v0.1.0`, creates the GitHub release, builds, runs the install smoke, and publishes to PyPI
   via trusted publishing (environment `pypi` — repo environments survive the rename).
3. Verify: the workflow is green and `https://pypi.org/pypi/toobig/json` reports version 0.1.0. On `invalid-publisher`,
   stop and notify Bryan (see the assumption above).

## Phase 3 — Migrate the sase integration (in the sase repo)

All references below are in the sase repo checkout:

1. `pyproject.toml`: `bbugyi-toolong>=0.1.0,<0.2.0` → `toobig>=0.1.0,<0.2.0`.
2. `Justfile`: rename `_lint-toolong` → `_lint-toobig` (binary `{{ venv_bin }}/toobig`), update the `lint` recipe
   wiring, the `check` stage label (`tools/run_silent "lint (toobig)"`), the public `toolong` target → `toobig`, and the
   lint-stage comment.
3. Docs: the lint-stage lists in `README.md` and `docs/development.md`.
4. Tests: `tests/test_justfile_lint.py` (recipe/target/label assertions), `tests/test_github_actions_ci.py` (the "CI
   must not run `just toolong`" guard → `just toobig`), `tests/test_xprompt_pylimit_split.py` (expected binary name).
5. `xprompts/pylimit_split.yml`: the python step's binary path `toolong` → `toobig`. The xprompt keeps its
   `pylimit_split` name (see Out of scope).
6. Verify: `just install` (required — fresh workspaces need it, and it resolves the new dep now that toobig is on PyPI),
   then `just check`; confirm `.venv/bin/toobig` exists and `just toobig` passes. `git grep -i toolong` should only hit
   historical plan/bead records, which must not be edited.
7. Commit via the standard sase commit workflow.

## Phase 4 — Bead bookkeeping

1. Close `sase-5r.3` with a note that toobig v0.1.0 is live on PyPI (released under the renamed project).
2. Close `sase-5r.4` with a note that sase now installs `toobig` from PyPI and `just check` passes.
3. Close epic `sase-5r`.

## Out of scope

- Renaming the `pylimit_split` xprompt (and its test module). Its name references `pylimit`, not `toolong`; changing the
  user-facing `#pylimit_split` trigger is a separate decision.
- chezmoi nvim `highlight_toolong*` variables — a coincidental substring match, unrelated to this project.
- Historical plan files and bead event records that mention toolong — they are records, not code.
