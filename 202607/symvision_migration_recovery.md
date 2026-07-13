---
create_time: 2026-07-13 06:52:07
status: done
prompt: 202607/prompts/symvision_migration_recovery.md
tier: tale
---
# Recover and land the SASE pyvision → symvision migration

## Context and root cause

Epic `sase-5t` did produce the `symvision` package and release it successfully, but its SASE-repository phase was closed
against an ephemeral workspace rather than against a landed repository change:

- The first `sase-5t.5` attempt prepared the migration but its commit finalizer failed with 24 uncommitted files.
- A follow-up attempt rebased and committed the migration as `39451d036` in a numbered workspace, then correctly left
  the phase open because `symvision` was not yet available from PyPI.
- After `symvision 0.1.0` became available, the closeout run reused that migrated workspace, successfully ran registry
  installation, `just symvision`, and the test suite, and closed Phase 5 and the epic. Its only durable commit was
  `2903fa9`, which marked the companion-repository plan done.
- The SASE migration commit was never attached to a ChangeSpec or PR, never pushed to a remote branch, and is not an
  object reachable in the current repository. `origin/master` therefore still installs no `symvision` dependency,
  invokes `tools/pyvision-260708`, and contains the old pragmas and live documentation.

The recovery must both redo the small consumer migration on current `master` and repair the lifecycle invariant that
allowed local ephemeral state to be mistaken for landed work.

## Goals

1. Make the published `symvision` package the sole symbol-visibility linter used by SASE.
2. Remove the vendored pyvision executable and all obsolete live references in code, tests, configuration, and normal
   project documentation.
3. Preserve current SASE behavior, including external-repository pragmas and bead-backed epic-symbol support.
4. Correct the `sase-5t` bookkeeping and require durable repository evidence before the epic is considered complete.

## Plan

### 1. Reopen and baseline the incomplete phase

- Reopen `sase-5t` and `sase-5t.5` before implementation, recording that the previous closure verified an unlanded
  workspace commit. Leave the already-landed package/release phases and chezmoi cleanup (`sase-5t.1` through `.4` and
  `.6`) closed.
- Confirm the worktree is clean and based on current `origin/master`, then run the required `just install` baseline
  before other project commands.
- Inventory current live `pyvision` references so the migration is based on current code rather than attempting to
  recover the now-unreachable `39451d036` object. In particular, do not carry forward the transcript's temporary
  `sase-5u(get_max_running_agents)` allowance unless current Symvision output proves it is still necessary; that symbol
  now has real production consumers.

### 2. Switch dependency and command wiring to Symvision

- Add `symvision>=0.1.0,<0.2.0` to the development dependencies beside `toobig`, using the same published-package
  integration pattern. Avoid unrelated lockfile churn; refresh tracked dependency metadata only where the repository's
  actual install workflow requires it.
- Replace the `Justfile`'s `_lint-pyvision` and public `pyvision` recipes with `_lint-symvision` and `symvision`. Invoke
  the installed `{{ venv_bin }}/symvision` entry point with `BD_COMMAND=tools/sase_bead`, and update the aggregate
  `lint` recipe, `check` stage label, headers, and comments consistently.
- Delete `tools/pyvision-260708`. Keep `tools/sase_bead`, since `BD_COMMAND` remains Symvision's bead integration
  contract.

### 3. Migrate consumers and user-facing guidance

- Rename every live source pragma from `# pyvision:` to `# symvision:`. Preserve each existing local path or external
  repository URI exactly, then let Symvision validate that the target still references the annotated symbol.
- Update code comments and docstrings that describe pyvision-specific behavior to use Symvision terminology without
  changing the underlying symbol visibility boundaries.
- Update `README.md`, `docs/development.md`, `docs/integrations.md`, and `src/sase/default_config.yml` so commands,
  pragma examples, lint-stage names, and remediation guidance consistently refer to `symvision` / `just symvision`.

### 4. Strengthen regression coverage

- Update `tests/test_justfile_lint.py` to assert that both aggregate lint paths use `_lint-symvision`, that the public
  `symvision` target delegates to the same private stage, and that the command uses the installed executable rather than
  a Python-invoked vendored script.
- Update `tests/test_github_actions_ci.py` so CI continues to rely on the single aggregate `just lint` command and does
  not grow a redundant standalone `just symvision` step.
- Add or refine a focused repository-wiring assertion if needed so a future reintroduction of `tools/pyvision-YYmmdd` or
  `_lint-pyvision` fails tests, while allowing intentionally retained historical/protected prose.

### 5. Handle protected memory and generated instruction files explicitly

- The remaining files that teach agents how to fix symbol-visibility failures are `memory/pyvision.md`,
  `memory/README.md`, root and `tools/` `AGENTS.md`, and their generated provider shims. Repository policy requires
  explicit user permission in the implementation conversation before any of them may be edited.
- If that permission is granted during plan review, migrate the memory note to Symvision terminology and provenance,
  update the Tier 2 references and vendored-tool guidance, and regenerate the provider shims through the sanctioned
  memory initialization workflow rather than editing generated files independently.
- If permission is not granted, leave every protected file untouched, make the remaining stale references an explicit
  follow-up, and do not obscure the resulting known memory-freshness validation failure.

### 6. Verify behavior and durable completion

- Run `just install` and confirm it resolves `symvision 0.1.x` from the package registry with repository sources
  disabled, then run `just symvision` against `src/sase`.
- Run the focused Justfile/CI tests, audit `pyvision` references to ensure none remain in live unprotected integration
  surfaces, and run the required full `just check` gate. If `just check` stops only at a pre-existing protected-memory
  freshness failure, establish that against clean `origin/master` and run `just test` separately so test coverage is
  still complete.
- Review the final diff for a consumer-only migration: published dependency added, recipe/pragma/docs/tests renamed,
  vendored script removed, and no changes to Symvision semantics or unrelated SASE behavior.
- Record the verification results on `sase-5t.5`, but do not close Phase 5 or the epic merely because a numbered
  workspace is green. Completion evidence must name a durable SASE commit and its ChangeSpec/PR; final closure occurs
  only after the migration is reachable from `origin/master`. If landing is outside the implementation turn's authority,
  leave both beads open with an exact commit/PR handoff instead of repeating the original false closure.

## Acceptance criteria

- SASE installs `symvision>=0.1.0,<0.2.0` and all symbol-visibility lint entry points execute the installed `symvision`
  command.
- `tools/pyvision-260708`, `_lint-pyvision`, the public `pyvision` recipe, and live `# pyvision:` pragmas are gone.
- Source pragmas validate successfully, focused wiring tests pass, and the full repository gate passes apart from any
  separately demonstrated protected-memory baseline failure.
- Protected memory/instruction files are changed only with explicit user permission and through their sanctioned
  generation flow.
- `sase-5t.5` and `sase-5t` remain open until the SASE migration is durably landed on `origin/master`, with the landing
  commit/PR recorded in the bead notes.
