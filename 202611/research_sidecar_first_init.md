---
tier: tale
title: Create newly declared sidecars on the first repo init run
goal:
  Adding a project from the Projects tab initializes its research sidecar without a
  second manual sase init.
size: medium
proposed_by: bbugyi200.athena.1a5
proposed_with: claude/opus
create_time: 2026-11-08 13:38:59
status: wip
---

# Plan: Create newly declared sidecars (research) on the first `sase repo init` run

## Problem

Adding a project from the Admin Center **Projects** tab (New Project wizard) runs the
durable `sase project add …`, whose result hands off to `sase init -p <key>`: the wizard
runs `--check --json`, shows the init preview, and the terminal valve (`t`) runs the
interactive `sase init -p <key>`. For the new `dev` project, that run created the plans,
beads, agents, and attachments-private sidecar repos but **not** the research sidecar.
The user had to run `sase init` a second time to create it.

## Root cause (verified)

`sase repo init` (also the `repo` step of `sase init`) writes the standard sidecar
declarations into `sase/sase.yml`, then resolves the sidecar list from a stale
in-process cache:

1. `plan_repo_init()` runs before anything is written. The onboarding coordinator plans
   every step first, and `run_repo_init()` calls `plan_repo_init(args)` again before it
   writes. Planning calls `configured_sidecar_specs()` →
   `sase._linked_repo_config.resolution_config()` → `_resolution_config_cached`
   (`src/sase/_linked_repo_config_state.py`). That is an `lru_cache` that lives for the
   whole process, keyed only by the primary workspace path, so it stores the config as
   it was **before** the write.
2. `run_repo_init()` then writes `explicit_sidecar_config_update()`, which adds
   `repos.sidecar.custom.research` (plus explicit plans/beads/agents entries). Nothing
   resets the cache.
3. `_run_configured_sidecars()` calls `_configured_sidecar_specs()` again and gets the
   cached pre-write config. The only specs it sees are the ones
   `inject_default_linked_repos()` adds on its own: plans, beads, agents, attachments,
   and attachments-private. **research is the only standard role that exists only
   through the explicit declaration**, so it is never preflighted, never prompted for,
   and never created. The next process reads the file fresh, which is why a second
   `sase init` works.

There is a second, planner-side mismatch: `plan_repo_init()` builds its sidecar actions
from the config already on disk, not from the declarations the same run is about to add.
So the `--check` preview (what the TUI shows) leaves out "create or connect the provider
research sidecar repository". For a project where research is the only new sidecar, it
also reports `requires_tty: false`, even though the run will need a terminal.

Evidence:

- In the `dev` repo, GitHub created `dev--plans/--beads/--agents/--attachments-private`
  at 15:39:32–45Z. The `chore: initialize SASE repositories` commit, which declares
  research, landed at 15:39:57Z. `dev--research` was only created at 18:15Z, by the
  manual rerun.
- Reproduced in a scratch managed repo: `configured_sidecar_specs()` → write
  `explicit_sidecar_config_update()` → `configured_sidecar_specs()` returns no
  `research`. After `reset_linked_repo_config_caches()` it does.
- Two other registered projects (`tailnet`, `sase-spotify`) declare research in
  `sase/sase.yml` but have no research entry in `.sase/sdd-store.json`. This is
  consistent with the same bug.

## Changes

### 1. Runner: reset the config caches after the `sase.yml` write

In `src/sase/main/repo_init_handler.py::run_repo_init`, call
`reset_repo_identity_caches()` from `sase._linked_repo_identity`, imported lazily, right
after `config_update.path.write_text(...)` succeeds. It must run before
`_run_configured_sidecars` / `_run_materialized_sidecars`. This follows the existing
convention in `src/sase/sdd/_store_records.py::_reset_repo_identity_caches`, which keeps
process-lifetime repo derivations consistent after store changes. Add a short comment
explaining that later sidecar resolution in this process must see the declarations just
written.

With this change, the first run preflights research, asks the usual default-no
`Create private GitHub sidecar repository <owner>/<project>--research …? [y/N]`, and
creates it. That covers the wizard terminal valve, `sase project add`'s `_offer_init`,
bare `sase init`, and `sase repo init`.

### 2. Planner: plan against the pending declarations

Make the plan and the run agree on which sidecars exist:

- `src/sase/config/core.py::load_merged_config_for_local_path`: add a keyword-only
  `local_config_override: Mapping[str, Any] | None = None`. When it is set and
  `local_path` is not `None`, pass `merge_config_sources` a wrapper `yaml_loader` that
  returns a copy of the override for `path == local_path` and calls `_load_yaml_file`
  for every other layer. The default behavior stays the same, and no cache changes.
- `src/sase/main/_repo_init_config.py::configured_sidecar_specs`: add keyword-only
  `pending_config_text: str | None = None`.
  - `None`: behavior is unchanged (`resolution_config` + `read_project_local_config`).
  - Text: parse it with `sase._yaml_safe.yaml_safe_load` (a non-mapping becomes `{}`).
    Use local path `resolve_project_config_read_path(primary)`, falling back to the
    project config write path. Build the config with
    `load_merged_config_for_local_path(local_path, linked_repo_origin_key=LINKED_REPO_DECLARED_BY_KEY, local_config_override=parsed)`.
    This is not cached and must not fill or read `_resolution_config_cached`. Pass
    `parsed` as `local_config` to `inject_default_linked_repos`. Everything after that
    (entry filtering, `full_github_repo_name`, spec construction) stays shared.
- `src/sase/main/repo_init_handler.py`:
  - The `_configured_sidecar_specs` seam gains the same keyword-only
    `pending_config_text=None` and forwards it.
  - `plan_repo_init` passes
    `pending_config_text=config_update.updated_text if config_update.changed else None`
    (and `None` when `config_update.error` is set).
  - The runner call sites keep calling the seam without the keyword. After change 1 they
    read the written file.
- Leave `src/sase/sdd/_auto_init.py` alone. It writes no config.

Result: on a brand-new managed project, the `--check`/preview lists the research
creation action and its guide-file actions, and `requires_tty` is true whenever the run
will prompt to create any newly declared sidecar.

### 3. Tests

- **Mechanical seam update.** Every monkeypatch of
  `sase.main.repo_init_handler._configured_sidecar_specs` currently uses
  `lambda _root: …`; there are 25, in `tests/main/test_repo_init_plan.py`,
  `test_repo_init_handler.py`, `_test_repo_init_handler_creation_sidecars.py`,
  `_test_repo_init_handler_creation_confirmation.py`,
  `test_init_onboarding_interactive.py`, and `test_init_onboarding_sidecar_defer.py`.
  Change each to `lambda _root, **_kwargs: …`. Also fix any test that patched
  `sase._linked_repo_config.resolution_config` and relied on `plan_repo_init` reaching
  it while an update is pending.
- **Runner regression** (the bug itself), in `tests/main/test_repo_init_handler.py` or a
  sibling file. Use the real `_configured_sidecar_specs`, a tmp git repo with a GitHub
  `origin` and `sase/sase.yml` = `is_sase_managed: true`, and the helpers in
  `tests/main/_repo_init_handler_creation_helpers.py`. Patch
  `_project_provider_sdd_policy` → `"separate_repo"`, `_primary_repo_visibility`
  (verified private or `None`), `sase.sdd._sidecar_init.preflight_sidecars` (record the
  roles; return `found` for each), and `sase.sdd._sidecar_init.initialize_sidecars`
  (record the selected roles). Call `plan_repo_init(args)` first, which fills the cache
  the way the coordinator does, then `run_repo_init(args)`. Assert `research` was
  preflighted and initialized. Confirm the test fails without change 1.
- **Planner, new project**, in `tests/main/test_repo_init_plan.py`: with the same setup,
  `plan_repo_init` includes
  `create or connect the provider research sidecar repository`, reports
  `requires_tty is True`, and leaves `sase/sase.yml` byte-for-byte unchanged.
- **Planner, upgrade case**: `sase/sase.yml` already declares plans/beads/agents (as
  `explicit_sidecar_config_update` would) but not research, and no research clone
  exists. The plan contains both the `sase.yml` action declaring research and the
  research creation action, with `requires_tty is True`.
- **Loader unit test**, next to the existing `load_merged_config_for_local_path` tests:
  `local_config_override` replaces only the local layer, never reads that file, and
  leaves later default calls unaffected.

### 4. Docs

Read the `docs_writing` reference memory first. Then, in
`docs/reference/agents/initialization.md` § "Repository Initialization", add a sentence
or two saying that the same run initializes the sidecars it just declared, and that
`--check`/`--diff` already preview creating a newly declared sidecar such as research.
Do not add an edit if the existing wording already says this clearly.

### 5. Verification

Read the `lint_and_test` reference memory and follow it, including running `just check`
the way it prescribes. Before that, run the targeted suites: `tests/main/` repo-init and
init-onboarding tests, `tests/sdd_store/test_auto_init.py`, and the config loader tests.

## Out of scope

- Repairing the existing `tailnet` and `sase-spotify` projects. Creating their research
  repos needs interactive human confirmation by design, so the user runs
  `sase init -p <key>` (or Projects tab init → terminal). The implementation must not
  create remote repositories.
- Making `_resolution_config_cached` invalidate itself on file mtime. This plan follows
  the explicit-reset convention instead, so it adds no per-call stat to hot TUI paths.
- Changing sidecar-creation authorization. `--yes` and non-interactive stdin still
  cannot create a remote. The wizard's terminal valve stays the creation path.

## Acceptance criteria

- On a managed project whose `sase/sase.yml` has only `is_sase_managed: true`, one
  interactive `sase init -p <key>` (or `sase repo init`) declares research and also
  prompts for and creates/connects the research sidecar.
- `sase init -p <key> --check --json` on such a project lists the research sidecar
  creation action and requires a TTY.
- The new regression tests pass, and the runner regression fails on the pre-fix code.
- All existing repo-init and onboarding tests pass after the seam lambda update.
  `just check` is green.
