---
create_time: 2026-07-11 17:49:42
status: done
prompt: 202607/prompts/linked_repo_clones_to_sase_repos.md
tier: tale
---
# Plan: Clone linked repos into `sase/repos/` instead of `.sase/workspaces/`

## Problem & Context

Host-scoped linked-repo workspaces (materialized by `sase workspace open` and by agent-launch linked-repo resolution)
are currently cloned to `<host_checkout>/.sase/workspaces/<linked_repo>`. We want them at
`<host_checkout>/sase/repos/<linked_repo>` instead, with the new directory ignored via a tracked `.gitignore` entry that
the `sase init` family auto-adds.

The clone path is constructed in exactly two places today:

- `src/sase/linked_repos.py` — `_resolve_workspace_dir()` builds `<host_workspace_dir>/.sase/workspaces/<name>` before
  calling `materialize_linked_repo_workspace()` (agent-launch resolution path).
- `src/sase/main/workspace_handler_list.py` — `resolve_checkout_path()` builds the same path for `sase workspace path` /
  `sase workspace open` on configured linked repos.

Supporting facts discovered while researching:

- `.sase/` is kept untracked per-clone via `.git/info/exclude` (written by `enter_agent_workspace()` in
  `src/sase/axe/run_agent_runner_setup.py`, helper in `src/sase/workspace_provider/git_exclude.py`). That exclude entry
  is what protects the nested linked clones from the `git clean` performed by `prepare_workspace()` /
  `run_sase_hg_clean` during workspace prep. A visible `sase/repos/` directory loses this protection unless we add
  equivalent ignore coverage — otherwise workspace cleaning would **delete the linked clones** (and any WIP inside
  them).
- `sase init` is a registry of plan/run pairs (`src/sase/main/init_registry.py`, specs: memory, sdd, skills) coordinated
  by `init_onboarding.py`. `sase init --check` and `sase validate` report drift from these specs. There is existing
  precedent for init-managed `.gitignore` content: `ensure_bead_store_gitignore()` in `src/sase/sdd/_bead_ignore.py`.
- Neither the Rust core (`sase-core`) nor the `sase-github` plugin references the `.sase/workspaces` path (verified by
  searching both linked repos). Linked-repo workspace dirs cross the process boundary only as already-resolved absolute
  strings (env vars, agent metadata, `opened_linked_workspaces.json`), so this change is fully contained in this repo
  and does not cross the Rust core backend boundary.
- Do not confuse the new in-checkout `sase/repos/` with the existing `~/.sase/repos/<name>.git` bare-repo storage or the
  managed workspace root (`.../state/sase/workspaces`); docs wording should keep these distinct.

## Goals

1. New linked-repo clones land at `<host_checkout>/sase/repos/<linked_repo>`.
2. Existing clones at the legacy `.sase/workspaces/` location keep working during a compatibility window and converge to
   the new location.
3. `sase/repos/` is ignored: immediately via per-clone `.git/info/exclude` (runtime-written, so clones are never exposed
   to `git clean`), and durably via a tracked `.gitignore` entry that `sase init` (bare onboarding + a new subcommand)
   auto-adds and reports drift for.

## Non-Goals

- No changes to `sase-core` or provider plugins (verified unnecessary).
- No change to the internal layout of a linked clone (its own `.sase/sdd` companion etc.).
- No change to primary-workspace materialization or the managed workspace root.

## Design

### 1. Single source of truth for the clone path (`src/sase/linked_repos.py`)

Add module-level constants and helpers so the path is defined once and both call sites share the legacy-fallback rules:

- `LINKED_REPO_CLONES_SUBDIR = ("sase", "repos")` and `LEGACY_LINKED_REPO_CLONES_SUBDIR = (".sase", "workspaces")`.
- `linked_repo_clone_dir(host_checkout, name)` → canonical new path.
- `resolve_linked_repo_clone_dir(host_checkout, name)` → resolution used by non-materializing callers: return the new
  path if it exists (or if neither exists); return the legacy path only when the legacy clone exists and the new one
  does not.

Migration on materialize (in `materialize_linked_repo_workspace()` or a small wrapper both call sites use): when the
legacy directory exists and the new one does not, `os.rename` the legacy clone into `sase/repos/<name>` (same filesystem
— cheap, preserves WIP), then proceed with the normal ensure-clone flow. If both exist, prefer the new path and emit a
non-fatal warning about the stale legacy directory. Clean up a now-empty legacy `.sase/workspaces/` directory after a
successful rename. Renaming at materialize time is safe because workspace prep runs against exclusively claimed
workspaces.

### 2. Update the two call sites + stale comments

- `_resolve_workspace_dir()` in `linked_repos.py`: build the target with the new helper (including legacy
  fallback/migration semantics via the shared wrapper).
- `resolve_checkout_path()` in `src/sase/main/workspace_handler_list.py`: use the same helpers for both the
  `materialize=True` and `materialize=False` (`sase workspace path`) branches so `path` reports where the clone actually
  is.
- Update the path comment in `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py` (matching logic itself is
  prefix-based on recorded absolute dirs and needs no behavior change).

### 3. Runtime ignore protection (`.git/info/exclude`)

Because the tracked `.gitignore` entry only helps once each project commits it, runtime code must self-protect the host
clone:

- `enter_agent_workspace()` (`src/sase/axe/run_agent_runner_setup.py`): in addition to the existing `.sase/` entry, add
  `/sase/repos/` via `ensure_git_info_exclude_entry()`.
- The shared materialize wrapper (Design §1): before cloning/renaming into `<host_checkout>/sase/repos/`, write the same
  `/sase/repos/` exclude entry into the host checkout, so clones created by `sase workspace open` (outside an agent-run
  setup) are protected too.
- Refresh the `git_exclude.py` module docstring to mention both managed entries.

Use the anchored pattern `/sase/repos/` so an unrelated nested `sase/repos` path elsewhere in a project tree is
unaffected.

### 4. Tracked `.gitignore` entry via a new `sase init` spec

Add a fourth `InitCommandSpec` named `workspace` (label "Workspace"):

- **Plan** (`plan_init_workspace`): if the current project root is a git checkout and its root `.gitignore` lacks a
  `/sase/repos/` line, report a single `create`/`update` `InitAction` for `.gitignore` with the full proposed content
  (so `sase init --check`, `--diff`, and `sase validate` all work). No-op (empty plan) for non-git project roots and
  non-project directories.
- **Run** (`run_init_workspace`): idempotently append the entry, modeled on `ensure_bead_store_gitignore()` (preserve
  existing content, add trailing newline handling, treat an already-present line as up to date). Commit the `.gitignore`
  change through the same project-commit helper pattern the SDD init uses for `sdd/.gitignore`, with a `--no-commit`
  escape hatch consistent with `sase init memory`.
- **Registration**: add the spec to `iter_init_command_specs()` and filter it alongside `sdd` when not in a project
  directory (`_active_onboarding_specs`). Register a `sase init workspace` subparser in `src/sase/main/parser_init.py`
  with `-c/--check` and `-d/--diff` flags matching the sibling subcommands, and wire dispatch in `entry.py` like the
  existing aliases.
- **CLI conventions** (from `memory/cli_rules.md`): keep subcommand and option listings alphabetical, give every public
  long option a short alias, and make the help text match the quality of the existing init subcommands.

With this, bare `sase init` onboarding prompts to add the entry, `sase init --check` / `sase validate` report drift, and
`sase init workspace` applies it directly — covering "the proper sase init commands auto-add this entry".

### 5. Documentation updates

Update the stated layout from `<host_workspace>/.sase/workspaces/<linked_repo>` to
`<host_workspace>/sase/repos/<linked_repo>` (and mention the legacy fallback + gitignore management) in:

- `README.md` (Configured linked repos bullet)
- `docs/configuration.md` (linked_repos section)
- `docs/commit_workflows.md` (linked-repo example path)
- `docs/workspace.md` — add/adjust any linked-workspace layout description and document the new `sase init workspace`
  behavior wherever init subcommands are enumerated.

## Compatibility & Rollout

- **Legacy fallback window**: non-materializing resolution honors existing `.sase/workspaces/` clones; materialization
  migrates them by rename. After the window (follow-up change), drop the legacy constants/fallback like the
  sibling→linked compat pattern already in this module.
- Historical records (`opened_linked_workspaces.json`, agent metadata) store absolute paths and remain valid read-only
  history; TUI consumers compare recorded absolute paths and need no changes.
- Old sase versions running concurrently would still write to the legacy path; the rename-based migration converges each
  workspace the next time a new-version process materializes it.

## Testing

- `tests/test_linked_repos.py`, `tests/test_sibling_repos.py`: update expected paths to `sase/repos/`; add cases for
  legacy fallback (legacy-only → legacy path returned when not materializing) and rename migration (legacy clone moved
  to new path on materialize; both-exist prefers new + warns).
- `tests/main/test_workspace_handler_list_path.py`: new expected path for `sase workspace path`, plus a legacy-fallback
  case.
- `tests/test_run_agent_runner_setup.py`, `tests/test_axe_run_agent_runner_deferred_workspace.py`: expect both `.sase/`
  and `/sase/repos/` exclude entries; update fixture paths.
- `tests/llm_provider/test_commit_finalizer_siblings_advisory.py`,
  `tests/ace/tui/widgets/test_agent_display_step_metadata.py`, shared fixtures/helpers
  (`tests/_mobile_agents_fixtures.py`, `tests/perf/bench_agent_launch.py`,
  `tests/ace/tui/modals/project_management_modal_test_helpers.py`): mechanical path updates.
- New tests for the `workspace` init spec: plan drift detection (missing entry / present entry / non-git root),
  idempotent apply, `--check` exit codes, and inclusion in bare `sase init` onboarding output.

## Notes for the Implementer

- The `.gitignore`-entry work touches init behavior only; if any further CLI options are added beyond those planned
  here, re-read `memory/cli_rules.md` first.
- No `memory/*.md` or generated instruction-shim changes are needed (verified no path references); if any turn up during
  implementation, they require explicit user permission.
- Run `just install` then `just check` before finishing.
