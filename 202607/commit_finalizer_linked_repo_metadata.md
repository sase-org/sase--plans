---
create_time: 2026-07-14 05:57:10
status: wip
prompt: 202607/prompts/commit_finalizer_linked_repo_metadata.md
tier: tale
---
# Fix: Commit finalizer fails on SASE-generated metadata inside linked repo checkouts (sase-5y.3 failure)

## Background: what happened

The agent `sase-5y.3` (workflow `ace(run)-260714_045600`, workspace #10) failed with:

```
CommitFinalizerError: Commit finalizer failed: uncommitted changes remain after 2 finalizer
pass(es) in sase-github=<ws10>/sase/repos/linked/sase-github: sase-github:sase/repos/plans/.
```

where `<ws10>` = `/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_10`.

Root-cause chain (all verified against workspace #10 state, run artifacts, and transcripts):

1. **Yesterday (2026-07-13 ~18:18)**, during the original sase-5y.3 run, the `sase-github` linked repo was materialized
   for workspace #10. `materialize_linked_repo_workspace()` (`src/sase/linked_repos.py`) then called
   `ensure_workspace_sdd_clone(checkout_dir, ...)` with the **linked clone dir** as the anchor. SDD storage resolution
   walked up to the host sase project's sidecar record and materialized a full clone of `sase-org/sase--plans` at
   `<linked sase-github clone>/sase/repos/plans/` — i.e. the host project's SDD sidecar nested _inside_ the linked
   repo's working tree (`resolve_sdd_store()` anchors `sidecar_repo_clone_dir()` at whatever `workspace_dir` it is
   passed: `src/sase/sdd/store.py:128`).
2. The linked repo checkout has **no `.git/info/exclude` protection**, unlike primary workspace clones, which get
   `.sase/` and `/sase/repos/` exclude entries written by `src/sase/axe/run_agent_runner_setup.py:276-277`. So the
   nested clone shows up as untracked dirt: `git status` in the linked checkout reports `?? sase/`.
3. Yesterday's run failed its commit finalizer on exactly this dirt (transcript `...tmp_260713_181104...main_ERROR`),
   _after_ the real work had already been committed and pushed (`b644bab27`, bead closed). The workflow was retried
   today.
4. **Today (04:56)**, the retry found the bead already closed, verified the work, and had nothing to commit. But the
   commit finalizer scans **all configured linked repos** (`_configured_sibling_targets` /
   `_blocking_sibling_candidates` in `src/sase/llm_provider/commit_finalizer_state.py`) — opened or not — and found the
   stale `sase/repos/plans/` dirt from yesterday.
5. The finalizer prompt told the agent: _"If you did NOT make these changes, ignore this warning for the session; it
   will not appear again."_ The agent (correctly) inspected the path, identified it as workspace-generated metadata, and
   declined to commit — twice. The finalizer re-detected the same dirt each pass and hard-failed after `max_passes=2`
   (`run_commit_finalizer`, `src/sase/llm_provider/commit_finalizer.py:306-318`), contradicting its own prompt promise.

Two structural bugs compound here:

- **Pollution bug**: SDD sidecar/store clones can be materialized anchored at a linked repo clone dir instead of a
  primary workspace root.
- **Robustness bug**: the finalizer treats SASE-reserved subtrees (`sase/repos/`, `.sase/`) inside scanned repos as
  agent-authored dirt and hard-fails on it, even though no agent may legitimately commit those paths, and even though
  managed clones are supposed to be git-blind to them.

Note: `.sase/sdd/` clones inside linked checkouts (e.g. `sase-core--sdd`, `sase-github--sdd`) are _current systemic
behavior_ — every linked clone on this machine has one, including ones freshly materialized today. They are only
invisible to git because Bryan's global gitignore happens to cover `.sase/`. They are NOT the fatal artifact and should
not be deleted; but the exclude-entry fix below makes them robustly invisible without relying on the user's global
gitignore.

## Goals

1. The commit finalizer must never fail (or even prompt) because of SASE-generated metadata (`sase/repos/`, `.sase/`)
   inside a scanned repository.
2. SASE-managed non-primary clones (linked, external, SDD sidecar) must get the same `.git/info/exclude` protection
   primary workspace clones already get.
3. SDD sidecar clones for a project must never be materialized under a linked/external repo clone dir.
4. Remediate workspace #10 so the held workspace can be released and future runs in it pass finalization.

## Non-goals

- Redesigning the finalizer "disclaimed changes" semantics (the prompt's "it will not appear again" promise for
  genuinely agent-relevant changes). With fix #1 the observed contradiction disappears because metadata never surfaces;
  a general acknowledgement mechanism is a separate follow-up.
- Changing whether the finalizer scans configured-but-unopened linked repos.
- Removing or redesigning the `.sase/sdd` per-linked-clone SDD initialization behavior.
- Rust core changes: everything touched is Python agent-runtime/workspace glue in this repo (commit finalizer,
  linked-repo materialization, CLI open path); no `sase-core` boundary crossing.

## Design

### 1. Finalizer: ignore SASE-reserved subtrees when computing dirty files

In `src/sase/llm_provider/commit_finalizer_git.py`, filter SASE-reserved paths out of `git_changed_files()` results (or
add a filtered wrapper used by the finalizer scanners). Reserved prefixes (after normalizing the porcelain rename/quote
forms already handled by `_changed_files_from_git_status`):

- `.sase/` — SASE workspace scratch/marker/SDD state
- `sase/repos/` — SASE-managed repo clone root (source the subtree components from the existing
  `LINKED_REPO_CLONES_SUBDIR` / `SIDECAR_REPO_CLONES_SUBDIR` / `EXTERNAL_REPO_CLONES_SUBDIR` constants in
  `src/sase/linked_repos.py` rather than hardcoding, so a future layout change stays in sync)

This makes every finalizer consumer — main repo details, sibling/linked scan (`_dirty_configured_sibling_repos`),
external scan (`_dirty_opened_external_repos`), and SDD store scan (`_dirty_sdd_store_repos`) in
`src/sase/llm_provider/commit_finalizer_state.py` — blind to metadata dirt. A repo whose only dirt is reserved-path
metadata must be treated as clean (no finalizer pass prompted, no failure).

This fix alone would have prevented both yesterday's and today's failures.

### 2. Write git exclude entries into managed non-primary clones

Primary workspace clones already get `.sase/` + `/sase/repos/` written to `.git/info/exclude`
(`ensure_git_info_exclude_entry` from `src/sase/workspace_provider/git_exclude.py`, called in
`src/sase/axe/run_agent_runner_setup.py:276-277`). Extend the same protection to managed clones:

- In `materialize_linked_repo_workspace()` (`src/sase/linked_repos.py`), after `ensure_git_clone_at(...)` returns the
  checkout dir, write both entries into the materialized clone's own `.git/info/exclude`. (Today only the _host_
  checkout gets a `/sase/repos/` entry there, at `src/sase/linked_repos.py:465`.)
- Do the equivalent for external repo clones (the `sase repo open <external>` path added in sase-5y.2,
  `src/sase/main/repo_open_external.py`) and SDD sidecar clones if they don't already have it.

Because materialization is "ensure"-style and re-runs on every agent setup / `sase repo open`, this heals existing
clones over time and removes the dependency on the user's global gitignore for `.sase/`.

### 3. Never anchor SDD sidecar materialization at a non-primary clone

Establish the invariant: _SDD sidecar/store clones are only materialized under a marker-bearing primary workspace root,
never under a linked/external clone dir._

- In `ensure_workspace_sdd_clone()` / `ensure_sdd_kind_clone()` (`src/sase/sdd/store.py:400-450`): when the resolved
  store's record does not belong to the repo at `workspace_dir` (i.e. resolution walked up past the passed dir to a
  different project's record, as happened here: linked `sase-github` clone → host `sase` sidecar record), do not
  materialize a sidecar clone anchored at `workspace_dir`. Either skip (the host workspace already has its own clone at
  `<workspace root>/sase/repos/plans`) or anchor at the workspace root that owns the record. The implementer should
  trace `_resolve_sdd_storage()` to find the cleanest place to detect "record owner ≠ passed anchor".
- In `_repo_target_context()` (`src/sase/main/repo_handler.py:478`): the context for a non-primary repo sets
  `primary_workspace_dir=repo.path`, which is the _workspace-scoped_ clone path. `prepare_opened_checkout()`
  (`src/sase/main/workspace_handler_list.py:311-333`) then runs
  `ensure_bare_git_sdd_initialized(ctx.primary_workspace_dir, ..., push=True)` against that ephemeral path. Align with
  the legacy linked-repo context (`_materialize_sibling_project_context` in
  `src/sase/main/workspace_handler_context.py`, which uses the durable `linked_repo.primary_dir`): anchor
  `primary_workspace_dir` (and the `WorkspaceStore`) at the repo's durable workspace-0 clone. Verify
  `resolve_checkout(ctx, workspace_num)` still yields the workspace-scoped linked clone path after the change (the
  legacy flow proves the machinery supports it).

### 4. Remediate workspace #10 (one-time cleanup)

- Delete `<ws10>/sase/repos/linked/sase-github/sase/` (the nested `sase--plans` clone). Verified safe: `git status -sb`
  shows it in sync with `origin/main`, no unpushed commits, and the host workspace already has the real clone at
  `<ws10>/sase/repos/plans`.
- Leave `<ws10>/sase/repos/linked/sase-github/.sase/` alone (systemic, gitignored, harmless).
- After cleanup, Bryan dismisses the failed `sase-5y.3` agent in the `sase ace` TUI to release held workspace #10 (there
  is no CLI for dismissal). No re-run is needed: the sase-5y.3 work itself landed as `b644bab27` and the bead is closed.

## Test plan

- `commit_finalizer_git`: unit tests that reserved-path entries (`sase/repos/plans/`, `.sase/sdd/...`, and a rename
  record touching a reserved path) are filtered from changed-file results, and that non-reserved paths (including e.g.
  `sasefoo/` and `src/sase/repos/`-like lookalikes that are _not_ at the repo root) are kept.
- `commit_finalizer_state`: a linked/external repo whose only dirt is reserved metadata is not reported as a
  `DirtyRepo`; mixed dirt reports only the non-reserved files.
- `linked_repos`: materializing a linked clone writes both exclude entries into the clone's `.git/info/exclude`;
  re-materializing is idempotent.
- `sdd/store`: `ensure_workspace_sdd_clone` on a dir whose store record resolves to a different project does not create
  `sase/repos/` under that dir.
- `repo open`: opening a linked repo does not initialize SDD state anchored at the workspace-scoped clone path; existing
  `sase repo open` tests (`tests/main/test_repo_handler.py`) keep passing.
- Full `just check` before committing.

## Risks / notes

- The reserved-path filter must only match at the scanned repo's root (`sase/repos/`, not `**/sase/repos/`) to avoid
  hiding real source dirs coincidentally named `sase/repos/` in arbitrary depth; the sase repo itself has `src/sase/...`
  source paths that must never be filtered.
- Changing `_repo_target_context` anchoring touches the sase-5y.2/5y.4 "open registered and external repositories"
  behavior that just landed; the implementer should re-run the sase-5y regression suites
  (`tests/llm_provider/test_commit_finalizer_external_repos.py`, `tests/main/test_repo_handler.py`) and keep the
  recorded open path (`opened_linked_workspaces.json`) semantics unchanged.
- The finalizer runs from the _host_ install (`/home/bryan/projects/github/sase-org/sase`), so the fix only takes effect
  for agent runs after it lands on master and the host checkout is updated.
