---
create_time: 2026-07-08 15:14:33
status: done
prompt: 202607/prompts/per_workspace_sdd_clone.md
tier: tale
---
# Plan: Per-workspace SDD clones so coder hand-offs resolve `@.sase/sdd/...`

## Problem

Coder agents launched to implement an approved plan fail immediately with `SystemExit: 1`, losing all planning work. The
failure surfaces while the coder agent's prompt is being processed:

```
File ".../src/sase/axe/run_agent_runner.py", line 498, in main
    exec_result = run_execution_loop(ctx, prompt)
File ".../src/sase/axe/run_agent_exec.py", line 314, in run_execution_loop
    ...
SystemExit: 1
```

The coder's prompt is built by the plan-approval hand-off and looks like:

```
#gh:<project> @.sase/sdd/tales/<yyyymm>/<plan_name>.md

The above plan has been reviewed and approved. Implement it now.
```

`process_file_references` validates the relative `@.sase/sdd/tales/...` path against the coder's current working
directory, does not find the file, and calls `sys.exit(1)`.

## Root cause

SDD storage for this project is `separate_repo` (`sase.yml`: `sdd.storage: separate_repo`). The SDD store is a real git
clone of the companion repo `git@github.com:sase-org/sdd.git`.

1. **The store is shared, not per-workspace.** `sdd/store.py::_sdd_dir_for_storage` resolves the SDD directory for
   _every_ workspace to the single shared `<primary>/.sase/sdd` clone. Numbered agent workspaces (`sase_<N>`) get a
   **symlink** `<sase_N>/.sase/sdd → <primary>/.sase/sdd`, created by `ensure_workspace_sdd_link`.

2. **The symlink is only created on the planner's prep path.** `ensure_workspace_sdd_link` is called from exactly one
   place — `run_agent_runner_setup.py::prepare_workspace_if_needed`, i.e. the runner's own workspace preparation for the
   _planner_. The recent stale-clone fix (`_replace_stale_workspace_sdd_clone`) also lives inside
   `ensure_workspace_sdd_link`, so it too only ever runs on that one path.

3. **The plan is written to the primary store and pushed asynchronously.** On approval,
   `run_agent_exec_plan_accept.py::handle_accepted_plan` writes the spec + plan into `<primary>/.sase/sdd`, commits
   them, and pushes with the default `sdd.push_after_commit: async` mode (a detached background push).

4. **The coder runs in a _different_, unsynced workspace.** The coder follow-up prompt carries the VCS workflow tag
   (`#gh`). The `#gh` workflow (`xprompts/git.yml`) `setup` step runs `scripts/git_setup.py::main`. Because
   `SASE_GIT_PRE_ALLOCATED` is not set on the plan-chain hand-off, it calls `claim_next_axe_workspace`, which claims a
   _different_ workspace than the planner's, then materializes it with `ensure_workspace_checkout`.
   `ensure_workspace_checkout` clones/prepares only the **main** repo — it never creates a symlink, clone, or pull for
   `.sase/sdd`. The `git.yml` `prepare` and `checkout` steps `git fetch` / `git pull --rebase` only the **main** repo.

Net effect: the coder's workspace has no `.sase/sdd` view (or a stale one from an unrelated prior run), so the relative
`@.sase/sdd/tales/...` reference cannot resolve and the workflow aborts with `SystemExit(1)`. Because the outcome
depends on _which_ workspace the coder happens to claim (and on an async push that may not have landed), the failure is
intermittent — matching "still failing".

## Why the symlink approach is the wrong foundation

Even when the symlink _is_ present and correct, every workspace shares one physical `<primary>/.sase/sdd` working tree.
When multiple agents in different workspaces mutate SDD files (plans, beads, epics) concurrently, they race on the same
files and the same git index — the likely source of the recent "stale index lock" and "stale clone shadowing" fixes
(`fix: retry auto-commit after stale index lock`, `fix(sdd): replace stale workspace SDD clone instead of refusing`). A
shared mutable working tree has no natural conflict-resolution mechanism.

## Proposed design: per-workspace clones coordinated through the remote

Give each workspace its **own git clone** of the SDD companion repo and use the companion repo's **remote as the single
coordination point**. This is the user's proposed direction and it also resolves the concurrency question: two agents in
two workspaces each write to their own working tree and reconcile through normal `git pull --rebase` / `git push`
against the shared remote, instead of stomping on one shared directory.

We continue to use the relative `.sase/sdd/...` prompt reference (a hard requirement): with a real per-workspace clone
at `<workspace>/.sase/sdd`, the relative reference resolves from the coder's own CWD with no symlink indirection.

### Core changes

1. **Resolve the SDD dir per workspace (separate_repo only).** Change `_sdd_dir_for_storage` (and the `resolve_sdd_dir`
   / `resolve_sdd_store` callers) so that under `separate_repo` the SDD directory is `<workspace>/.sase/sdd` — the
   workspace's own clone — rather than the shared `<primary>/.sase/sdd`. `in_tree` and `local` storage are unchanged.
   The primary checkout simply becomes "the clone that happens to live in workspace 0/1". The store record
   (`sdd-store.json`, which carries `remote_url`) stays a single source of truth at `<primary>/.sase`; per-workspace
   resolution reads it from there.

2. **Replace `ensure_workspace_sdd_link` with `ensure_workspace_sdd_clone`.** New helper that, for `separate_repo`
   storage:
   - **clones** `origin` (the `remote_url` from the store record) into `<workspace>/.sase/sdd` when that directory is
     absent; and
   - **syncs** an existing clone with `git -C .sase/sdd pull --ff-only` (or `--rebase`) when present.
   - Falls back to fast-forwarding from the on-disk primary store when the remote is unreachable (same-machine
     robustness, no network dependency in the launch hot path) — reusing the existing
     `_fast_forward_workspace_clone_from_primary` logic.
   - Is idempotent, best-effort, and never raises into the launch path. Retire the symlink branch and the
     `_replace_stale_workspace_sdd_clone` / stale-backup machinery, which existed only to paper over shared-symlink
     breakage.

3. **Sync `.sase/sdd` on _every_ workspace-preparation path — including the coder's.** The gap is that the `#gh` coder
   path never syncs SDD. Wire `ensure_workspace_sdd_clone` into both:
   - `prepare_workspace_if_needed` (planner / runner prep), replacing the current `ensure_workspace_sdd_link` call; and
   - the coder's `#gh` preparation — add an explicit SDD clone/pull step to `xprompts/git.yml` (alongside the existing
     main-repo `git fetch` / `git pull --rebase` in the `prepare` step), and/or fold the sync into
     `ensure_workspace_checkout` so every claimed workspace is covered from one chokepoint. Prefer the single chokepoint
     so future non-`#gh` workspace claimers are covered too.

4. **Commit _and push the plan synchronously_ before the coder's workspace is prepared.** In `handle_accepted_plan`, the
   plan-hand-off SDD commit must push with `push_after_commit=True` (synchronous), not the default `async`, so the plan
   is guaranteed to be on the remote before the coder's `#gh` prep pulls. This is the ordering guarantee the whole
   design rests on, and it maps directly to the user's requirement: "commit and push the plan before we prepare the
   coder's workspace." (Ordering is naturally satisfied because `handle_accepted_plan` runs before the loop hands
   control to the coder iteration — we only need the push to be synchronous rather than detached.)

5. **Concurrency-safe write path for all agents.** With per-workspace clones, any agent that mutates SDD files (planner
   writing plans, coder creating beads, epic/legend creators) commits to its own clone and reconciles with the remote
   via `pull --rebase` + `push`, retrying on non-fast-forward. The existing auto-commit + push helpers
   (`commit_sdd_store_files`, `push_bead_work_launch`) already push separate-repo mutations; the change is that
   `repo_root` now points at the per-workspace clone, and pushes should pull-rebase-retry on rejection so concurrent
   writers converge.

### Milestones

- **Milestone 1 — fix the coder hand-off (urgent).** Add per-workspace clone sync to the coder
  `#gh`/`ensure_workspace_checkout` prep path and make the plan-hand-off push synchronous. This alone stops the
  `SystemExit(1)` and recovers lost planning work, even before the store-dir resolution is fully migrated. Validate
  end-to-end: approve a tale, confirm the coder resolves `@.sase/sdd/tales/...` in a freshly-claimed workspace.
- **Milestone 2 — migrate the store dir to per-workspace and retire the symlink.** Change `_sdd_dir_for_storage`,
  replace `ensure_workspace_sdd_link` with `ensure_workspace_sdd_clone`, remove the stale-clone/symlink machinery, and
  make all SDD writers push-with-retry. Clean up any existing workspace symlinks/`sdd.stale-backup` directories
  idempotently on next prep.

## Risks & considerations

- **Network in the launch hot path.** Every workspace prep now performs a `git clone`/`pull`. Mitigate with the on-disk
  primary-store fast-forward fallback, keep the sync best-effort with clear warnings, and reuse the bounded git timeout
  helpers (`run_sdd_git`, `network_git_timeout`) already used by the SDD commit/push code.
- **Keep `.sase/` git-excluded.** The `#gh` pre-step runs `git clean`/`git checkout`; the per-clone `.git/info/exclude`
  entry for `.sase/` (added by `enter_agent_workspace`) must remain so the new clone is never wiped mid-run. Verify the
  coder's `#gh` path establishes that exclude before any clean.
- **Rust core boundary.** SDD _path-resolution policy_ already lives Python-side (`sdd/store.py`), with the Rust core
  receiving fully resolved paths. The per-workspace resolution and clone/pull glue can stay Python-side, but the
  implementer must confirm no Rust binding or adapter hard-codes the shared `<primary>/.sase/sdd` assumption; if it
  does, update the wire/API and tests in the sibling core repo.
- **Store record location.** Keep `sdd-store.json` (and thus `remote_url`) authoritative at `<primary>/.sase`;
  per-workspace clones read it from there and set their own `origin` at clone time. Do not fan the record out per
  workspace.
- **`SASE_SDD_DIR` env.** `enter_agent_workspace` / `set_sdd_dir_env` must point at the per-workspace clone so
  downstream SDD/bead commands operate on the right tree.
- **Backwards compatibility / migration.** Existing numbered workspaces currently hold symlinks or stale clones. The new
  `ensure_workspace_sdd_clone` must handle each case idempotently: replace a symlink with a fresh clone, fast-forward a
  matching clone, and preserve/repair anything carrying un-pushed local work before discarding.

## Out of scope

- Changing `in_tree` or `local` SDD storage behavior.
- Changing the relative `@.sase/sdd/...` prompt reference format (explicitly retained).
- Redesigning the `#gh` workflow beyond adding the SDD sync step.

## Validation

- Reproduce: launch a planning agent that auto-approves a tale (`#tale`) and confirm the coder follow-up currently fails
  with `SystemExit(1)` when it lands in a different workspace.
- After the fix: the coder's freshly-claimed workspace contains `.sase/sdd/tales/<yyyymm>/<plan>.md` (pulled from the
  remote), the relative reference resolves, and the coder proceeds to implement.
- Unit/integration coverage for: clone-when-absent, pull-when-present, remote-unreachable primary fallback, synchronous
  plan push ordering, and concurrent-writer push-with-retry convergence.
- `just check` before hand-off (code changes in the sase repo).
