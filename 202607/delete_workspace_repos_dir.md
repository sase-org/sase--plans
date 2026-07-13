---
create_time: 2026-07-13 08:42:53
status: wip
prompt: 202607/prompts/delete_workspace_repos_dir.md
tier: tale
---
# Delete the entire `sase/repos/` directory during workspace prep

## Problem & Context

Workspace-directory preparation currently clears only `sase/repos/linked/` before launching a new agent (by renaming
linked clones into the internal `sase/repos/.linked-cache/`, introduced in commit `df60999b5`). Everything else under
`sase/repos/` survives across launches:

- `sase/repos/plans` and `sase/repos/research` — durable SDD companion clones.
- `sase/repos/.linked-cache/` — the rename-based stash of previously-opened linked clones.
- Legacy junk from older layouts (e.g. a stale `sase/repos/<project>--plans` dir from the pre-`df60999b5` naming) that
  nothing ever cleans up.

This is both an isolation hole and a hygiene problem:

- The `.linked-cache` restore path (`_restore_linked_repo_clone`) renames a previous agent's linked clone back into
  place on `sase workspace open` **without any clean/reset**, so a new agent can inherit the previous agent's
  uncommitted linked-repo state.
- Stale directories accumulate forever in `sase/repos/` and are visible to every new agent.

**Goal:** during workspace prep, delete the _entire_ `sase/repos/` directory, then re-establish the companion repos
configured with `auto_clone: true` (currently just the `plans` companion) before the new agent launches — without paying
a meaningful per-launch performance cost.

## Current Behavior (what the change touches)

Agent launch (`src/sase/axe/run_agent_runner.py` → `run_agent_runner_setup.py`):

1. `prepare_workspace_if_needed()` — `prepare_workspace()` (clean + checkout + sync), then `clear_linked_repo_clones()`
   (stash `linked/` → `.linked-cache/`), then `ensure_workspace_sdd_clone()` (for companion storage:
   `ensure_companion_sdd_clone()` on `sase/repos/plans` — pulls if present, **clones from the network remote if
   missing**).
2. `refresh_linked_repos_for_workspace()` — resolves linked repos (`materialize=False`).
3. `prepare_linked_repo_workspaces_if_needed()` — for each `auto_clone` repo (the injected `<project>--plans` default):
   `materialize_linked_repo_workspace()` → `ensure_git_clone_at()` (**local** clone from the durable primary-side
   checkout, hardlinked objects, then `remote set-url origin` + fetch) and `prepare_workspace()` on the clone.

Workflow launch (`src/sase/axe/run_workflow_runner.py`):

- `_prepare_workflow_workspace()` — `prepare_workspace()` + `clear_linked_repo_clones()` only. Workflows currently rely
  on the companion clones being durable; nothing re-creates them.

`sase workspace open` (`src/sase/main/workspace_handler_list.py::resolve_checkout_path`) →
`materialize_linked_repo_workspace()` — restores a linked clone from `.linked-cache/` when present, else local-clones
from the linked repo's primary dir via `ensure_git_clone_at()`.

## Design

### 1. Whole-directory teardown, made cheap via rename + deferred delete

Replace `clear_linked_repo_clones()` in `src/sase/linked_repos.py` with a new teardown helper (e.g.
`clear_workspace_repos(workspace_dir)`) that removes `sase/repos/` wholesale:

- **No-op** when `sase/repos/` does not exist.
- **Primary-checkout guard:** never delete when preparing the primary checkout (`workspace_num <= 1`). The primary's
  `sase/repos/plans`/`research` are the _durable_ companion clones (see `materialized_sdd_clone()`) and are also the
  local clone source for the fast path below. Callers pass `workspace_num` so the helper can enforce this.
- **Rename, don't rmtree, on the critical path:** rename `sase/repos` to a unique name under `.sase/trash/` (e.g.
  `.sase/trash/repos-<token>`). `.sase/` is already covered by the per-clone `.git/info/exclude` entry that launch setup
  installs, so this needs **no gitignore/exclude pattern changes** and is never swept into clean backups. The rename is
  atomic and O(1) on the same filesystem, so the launch-visible directory disappears instantly and a crash mid-delete
  can never leave a partially-deleted `sase/repos/` that looks valid.
- **Deferred deletion:** after the rename, delete the trash entry off the critical path with a detached best-effort
  process (e.g. `rm -rf` via `Popen(start_new_session=True)`), falling back to synchronous `shutil.rmtree` if spawning
  fails. Each teardown also sweeps any leftover `.sase/trash/*` entries from prior crashed launches the same way.
- Handle the paranoid shapes the old code handled: `sase/repos` being a symlink or plain file is removed directly.

### 2. Smart auto-clone of companions: local-first `ensure_companion_sdd_clone`

After deletion, `sase/repos/plans` is _always_ missing at launch. Today's `ensure_companion_sdd_clone()`
(`src/sase/sdd/_store_link.py`) would fall through to `_clone_sdd_store()` — a full **network** clone every launch. That
is the main performance trap to avoid:

- Extend `ensure_companion_sdd_clone()` (plumbed from `ensure_sdd_kind_clone()` in `src/sase/sdd/store.py`) with an
  optional **local source**: the durable primary-side clone at `companion_repo_clone_dir(primary_workspace_dir, kind)`.
  Mirror the existing `_clone_sdd_store_from_primary()` pattern used by separate-repo storage.
- **Validate before trusting:** only use the local source when it is a git clone whose `origin` matches the recorded
  companion remote (reuse `is_matching_store_clone`/`_same_git_remote`).
- After a local clone: `git remote set-url origin <real remote>` and fast-forward from origin (existing
  `_pull_sdd_clone`) so freshness is identical to today — origin stays the source of truth; the local clone is purely a
  transport optimization (hardlinked objects, no network).
- **Fallback unchanged:** if the local source is missing/mismatched, clone from the remote exactly as today, so
  correctness never depends on host layout.

This also speeds up every lazy materialization path for free: the `research` companion (`auto_clone: false`) is
re-created on demand by `sase workspace open` / `ensure_sdd_kind_clone("research")`, which now local-clones instead of
network-cloning.

### 3. Launch integration

- **Agent runner** (`prepare_workspace_if_needed` in `run_agent_runner_setup.py`): swap
  `clear_linked_repo_clones(workspace_dir)` for the new teardown (passing `workspace_num`). Keep the existing
  `ensure_workspace_sdd_clone()` call — with (2) it becomes a fast local clone + one origin fast-forward, comparable to
  today's pull-on-existing-clone cost. The downstream `prepare_linked_repo_workspaces_if_needed()` then finds a valid
  clone (`ensure_git_clone_at` returns early) and just preps it, exactly as today. Both the normal and the
  deferred-workspace claim paths in `run_agent_runner.py` flow through these helpers, so no changes there.
- **Workflow runner** (`_prepare_workflow_workspace` in `run_workflow_runner.py`): swap the clear call the same way, and
  **add** an `ensure_workspace_sdd_clone(workspace_dir, workspace_num)` call after teardown. Workflows previously relied
  on companion clones being durable; after this change they must re-establish the plans clone themselves (cheap via the
  local-first path). `workspace_num` is already available in the runner's `main()`.
- **Unchanged skip semantics:** home mode, retry-spawn children (which intentionally preserve the parent's workspace,
  including opened linked repos), and empty `update_target` continue to skip prep entirely — same as the current
  cache-clear behavior.

### 4. Remove the `.linked-cache` machinery

With the whole directory deleted every launch, the one-day-old stash/restore cache is dead weight — and its restore path
is the isolation hole described above. Remove:

- `LINKED_REPO_CACHE_SUBDIR`, `_linked_repo_cache_dir()`, `_restore_linked_repo_clone()`, the stash logic in
  `clear_linked_repo_clones()`, and the restore hook inside `materialize_linked_repo_workspace()` (plus the `__all__`
  export).

`sase workspace open` for a normal linked repo after a launch now always creates a fresh local clone from the linked
repo's primary dir (`ensure_git_clone_at`: hardlinked local clone + `remote set-url` + fetch). That is a bounded,
mostly-local cost paid once per opened repo per launch — and it guarantees the opened workspace never carries a previous
agent's state. (Alternative considered: relocating the cache outside `sase/repos/`. Rejected — it preserves the
stale-state restore semantics the whole change is meant to kill, for a small saving over an already-cheap local clone.)

### 5. Documentation

- `docs/workspace.md` (lifecycle paragraph around line 253) — describe the new lifecycle: whole `sase/repos/` removed at
  every agent/workflow launch prep; `auto_clone` companions re-established from the durable primary-side clone (network
  fallback); other linked repos and the `research` companion re-created on demand by `sase workspace open`.
- `docs/configuration.md` (line ~578) — remove the `.linked-cache` description.
- Sweep `docs/commit_workflows.md` and `sase init workspace` help text (`main/parser_init.py`) for stale wording; the
  `/sase/repos/` gitignore/exclude story is unchanged.

### 6. Tests

- `tests/test_linked_repo_workspaces.py` — replace stash/restore-cache tests with teardown semantics: whole-dir removal
  via rename into `.sase/trash/`, stale-trash sweep, primary-guard no-op, symlink/file shapes, missing-root no-op.
- `tests/test_run_agent_runner_setup.py` — prep now removes companions + re-clones via the local-first path.
- `src/sase/sdd/_store_link.py` tests — local-source validation (used when origin matches, skipped when
  mismatched/missing, remote fallback, origin reset + fast-forward after local clone).
- Workflow-runner prep test — companion clone re-established after teardown.
- `tests/test_agent_artifact_directory_operation_audit.py` — update the audited directory-operation whitelist (its
  justification currently names `.linked-cache`; the new teardown's rename/detached-delete needs an equivalent audited
  entry).

## Performance Summary (per launch, steady state)

| Step                              | Before                  | After                                                              |
| --------------------------------- | ----------------------- | ------------------------------------------------------------------ |
| Clear previous linked clones      | O(n) renames into cache | one O(1) rename of `sase/repos` + detached delete                  |
| Plans companion                   | pull on existing clone  | local hardlink clone + origin fast-forward                         |
| Research companion / linked repos | untouched until opened  | re-created on open via local clone (fetch is the only network hop) |

Net: the critical path trades one pull for one local clone + one pull on a small repo — roughly cost-neutral — while the
deletion itself moves off the critical path entirely.

## Edge Cases & Risks

- **Primary workspace protection** is load-bearing: without the `workspace_num <= 1` guard, a prep against the primary
  checkout would destroy the durable companions that both the fast clone path and `materialized_sdd_clone()` depend on.
- **Stale/missing local source** on some hosts (e.g. sibling `<project>--plans` never cloned): remote fallback preserves
  today's correctness; only speed degrades.
- **Workflows that read plans/beads mid-run:** covered by the new ensure call in the workflow runner; anything else that
  lazily materializes goes through `ensure_sdd_kind_clone`.
- **TUI/history views** pointing at linked-clone paths from finished runs already tolerate the paths vanishing (the
  cache-stash made them vanish too); no new breakage expected, verify the prompt-panel agent-commits view degrades
  gracefully.
- **Rust core boundary:** this is Python launch/workspace orchestration (same layer as `df60999b5`); no `sase-core`
  changes required.

## Out of Scope

- Deduplicating the remaining double network sync of the plans clone in the agent path (`ensure_workspace_sdd_clone`
  pull + linked-repo `prepare_workspace` sync) — it exists today and is unchanged; a follow-up could collapse it to a
  single sync.
- Any change to the `/sase/repos/` gitignore/exclude rules or to `sase init workspace`.
