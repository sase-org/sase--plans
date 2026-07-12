---
create_time: 2026-07-08 13:11:52
status: wip
prompt: 202607/prompts/prepare_workspace_sdd_link.md
tier: tale
---
# Plan: Fix plan-chain SDD reference crash by linking + refreshing the workspace SDD store during prep

## Problem

A plan-chain agent (planner → auto-approve → coder) crashes with `SystemExit: 1` right after the plan is approved,
whenever the SDD store is configured as **`separate_repo`** (or `local`) and the agent runs from a **managed / ephemeral
workspace** rather than from the primary checkout. This is the failure that killed the sase agent named **"22"**: its
`22--plan` phase authored and committed `auto_commit_sdd_store.md`, then the coder follow-up died at hand-off.

Observed failure:

```
❌ ERROR: The following file(s) referenced in the prompt do not exist:
  - @.sase/sdd/tales/<YYYYMM>/<plan_name>.md
⚠️ File validation failed. Terminating workflow to prevent errors.
...
  File ".../xprompt/workflow_executor_steps_prompt.py", line 236, in _execute_prompt_step
    expanded_prompt = preprocess_prompt_late(expanded_prompt)
  File ".../llm_provider/preprocessing.py", line 162, in preprocess_prompt_late
    prompt = process_file_references(prompt, is_home_mode=is_home_mode)
  File ".../file_references.py", line 246, in process_file_references
    sys.exit(1)
SystemExit: 1
```

The coder follow-up prompt is:

```
#gh:<changespec> @.sase/sdd/tales/<YYYYMM>/<plan_name>.md

The above plan has been reviewed and approved. Implement it now.
```

The planner phase succeeds and the plan is committed; only the coder hand-off crashes, so all planning work is lost and
the implementation never starts.

## Root Cause (verified on disk)

For `separate_repo`/`local` storage the SDD store is a **single, shared clone** that lives under the **primary**
checkout at `<primary>/.sase/sdd`, resolved by `_sdd_dir_for_storage(...)` in `src/sase/sdd/store.py` via
`get_primary_workspace_dir(...)`. For this project the primary is the canonical checkout (recorded in each managed
workspace's `.sase/checkout.json` marker as `primary_workspace_dir`), **not** the ephemeral `sase_<N>` clone the agent
runs in.

Verified state:

- The authoritative store `<primary>/.sase/sdd` exists and is a git clone of the `sase-org/sdd` companion repo.
- The approval step writes and **synchronously commits** the plan into that store (`_archive_plan_for_approval` in
  `src/sase/plan_approval_actions.py` → `commit_sdd_store_files`); the push is asynchronous (`sdd.push_after_commit`
  defaults to `async`).
- Every managed `sase_<N>` workspace carries a `.sase/checkout.json` marker, but **almost none has its own `.sase/sdd`**
  — the ephemeral workspace that the coder runs in has no `.sase/sdd` directory at all.

The coder follow-up reference `@.sase/sdd/tales/<YYYYMM>/<plan>.md` is **relative** and is resolved by
`process_file_references` against the coder's working directory (the ephemeral workspace). Because that workspace has no
`.sase/sdd`, the reference cannot resolve, and `process_file_references` treats an unresolved `@`-reference as fatal
(`sys.exit(1)`), killing the whole agent. The reference itself is correct relative to the store; the store simply is not
reachable from the coder's CWD.

This is why the bug only appears for managed/ephemeral workspaces on non-in-tree storage. In-tree storage keeps the
store inside the workspace tree (`<workspace>/sdd`), so its `sdd/<rel>` reference already resolves; and running directly
in the primary checkout (`<primary> == <cwd>`) also resolves. The defect was latent until the project migrated to a
separate companion SDD repo.

## Relationship to the earlier plan (and the adjustment)

An earlier plan (`fix_plan_chain_sdd_ref_resolution.md`) fixed this by **changing the reference builders** to emit the
**absolute** committed-plan path when the relative `.sase/sdd/<rel>` form doesn't resolve. That works but produces
absolute, home-rooted references that `process_file_references` copies into `.sase/home/`.

Per the requested adjustment, this plan takes a different, complementary approach:

1. **Keep the relative `.sase/sdd/` reference unchanged** — the reference builders in
   `src/sase/axe/run_agent_exec_plan_sdd.py` are **not** modified. The existing `.sase/sdd/<rel>` output is exactly what
   we want.
2. **Make that relative path resolve from the coder's workspace** by ensuring, **during workspace preparation** (before
   any prompt validation runs), that the managed workspace's `.sase/sdd` resolves to the up-to-date primary store.

### Correction the implementation must account for

The literal directive — "run `git pull` in `.sase/sdd/` during workspace preparation" — is **necessary but not
sufficient on its own**: the ephemeral workspace has **no `.sase/sdd` directory to pull**. So preparation must first
make `<workspace>/.sase/sdd` point at the store, then refresh it. The chosen mechanism is a **symlink**
`<workspace>/.sase/sdd → <primary>/.sase/sdd`, plus a `git pull --ff-only` of the store:

- The **symlink** is what actually fixes the crash: it makes the relative `.sase/sdd/<rel>` reference resolve from the
  ephemeral CWD, and the plan is already committed into the shared store synchronously at approval time — so the file is
  present the moment the symlink exists, with **no dependency on the async push landing**.
- The **`git pull --ff-only`** is a freshness guarantee (remote/cross-machine updates, catching up async pushes). It is
  best-effort: even if it fails or a remote commit hasn't arrived, the locally-committed plan is still visible through
  the symlink, so the coder never crashes.

A symlink is preferred over giving each workspace its own separate clone: a per-workspace clone would have to `git pull`
the plan from the **remote**, which races the **async** push and would reintroduce the intermittent crash; it also
duplicates the store on disk. The symlink shares the single authoritative store (already the design intent) and is
race-free for the local plan-chain. (Per-workspace local clone considered and rejected.)

## Design

Add a workspace-preparation step that, for non-in-tree SDD storage where the store lives outside the workspace, links
and refreshes the store before the agent's prompt is validated.

### 1. A store-link/refresh helper (SDD layer)

Add a helper (e.g. `ensure_workspace_sdd_link(workspace_dir, workspace_num)`) near the SDD storage policy in
`src/sase/sdd/store.py` (Python owns SDD path policy; the Rust core only receives resolved paths, so no `sase-core`
change is needed). Behavior:

1. Resolve the store via `resolve_sdd_store(workspace_dir, workspace_num)`.
2. If `store.is_in_tree`: **no-op** (the `sdd/<rel>` reference already resolves inside the workspace tree).
3. If the store already lives at `<workspace>/.sase/sdd` (i.e. `<primary> == <workspace>`, same-file): just refresh
   (`git pull --ff-only`) and return — no symlink needed.
4. Otherwise (managed workspace; store under `<primary>/.sase/sdd`):
   - Ensure the store is materialized when missing (reuse `materialize_sdd_store`); if it cannot be materialized, log
     and return without raising (do not convert a freshness problem into a launch failure).
   - `git pull --ff-only` the store dir (reuse the existing best-effort `_refresh_materialized_store`), so it is
     up-to-date before validation.
   - Idempotently create/repair the symlink `<workspace>/.sase/sdd → <primary>/.sase/sdd`: create parent `.sase/` if
     absent; if a symlink already exists, repoint it only if stale; **refuse to overwrite a real directory** at that
     path (log and skip), mirroring the existing symlink-safety pattern in
     `src/sase/main/workspace_handler_migration.py`.

The helper must be best-effort and must never raise into the launch path — a link/refresh failure should degrade
gracefully, not kill the agent.

### 2. Wire it into workspace preparation (axe runner)

Invoke the helper from the agent workspace-prep path so it runs **before** the prompt step validates file references:

- `src/sase/axe/run_agent_runner_setup.py` — `prepare_workspace_if_needed(...)` is the natural call site; thread
  `workspace_num` into it (already available at both call sites in `src/sase/axe/run_agent_runner.py`, lines ~300 and
  ~440) and call the helper after `prepare_workspace` succeeds, for non-home, non-retry launches.
- This runs during agent setup, ahead of `WorkflowExecutor` prompt-step execution (`workflow_executor_steps_prompt.py` →
  `preprocess_prompt_late` → `process_file_references`), satisfying "before any prompt validation is performed."

`.sase/` is gitignored, so the symlink survives the `git checkout . && git clean -fd` performed by `prepare_workspace`
and by any `#gh` pre-step (as long as clean is not run with `-x`). Creating the symlink at the **end** of prep (after
the clean) also protects it from prep's own clean.

### Files to change

- `src/sase/sdd/store.py` — add `ensure_workspace_sdd_link(...)` (symlink + best-effort refresh; reuse
  `materialize_sdd_store` / `_refresh_materialized_store`).
- `src/sase/axe/run_agent_runner_setup.py` — call the helper from `prepare_workspace_if_needed`; add a `workspace_num`
  parameter.
- `src/sase/axe/run_agent_runner.py` — pass `workspace_num` to `prepare_workspace_if_needed` at both call sites.

### Explicitly unchanged

- `src/sase/axe/run_agent_exec_plan_sdd.py` — the reference builders (`build_saved_plan_ref`, `build_sdd_plan_ref`,
  `build_epic_plan_ref`) keep emitting the relative `.sase/sdd/<rel>` form. No behavior change, so existing tests such
  as `test_epic_prompt_uses_non_vc_sdd_ref_from_google_sibling_workspace` (which asserts the relative ref) stay green —
  under this approach that relative ref is the **desired** output, now made resolvable by the symlink.
- `resolve_sdd_dir` / `_sdd_dir_for_storage` — the store stays authoritatively at `<primary>/.sase/sdd`; we only add a
  workspace-local view of it.

## Tests

- Unit tests for `ensure_workspace_sdd_link(...)`:
  - **Managed separate-repo:** `<workspace>` distinct from `<primary>`, store under `<primary>/.sase/sdd` → symlink
    `<workspace>/.sase/sdd` is created pointing at the store; a file under the store is reachable via the workspace
    path.
  - **In-tree:** no-op (no symlink created).
  - **Co-located (`<primary> == <workspace>`):** no symlink; refresh attempted; no error.
  - **Safety:** an existing **real** `.sase/sdd` directory in the workspace is not clobbered (logged, skipped); a stale
    existing symlink is repointed; re-running is idempotent.
  - Refresh is best-effort: a failing `git pull` does not raise and the symlink is still created.
- A regression test that composes the coder follow-up prompt for a `separate_repo` store in a managed workspace, runs
  the prep helper, and asserts the relative `@.sase/sdd/...` reference **resolves** from the workspace CWD (so the
  `process_file_references` `sys.exit(1)` path cannot re-trigger). Mirror the existing epic-ref test fixtures.
- A wiring test that `prepare_workspace_if_needed` invokes the helper for a non-home, non-retry managed launch and skips
  it for home-mode / retry-handoff launches.

## Validation

- `just install` then `just check` (lint + mypy + targeted tests).
- Run the new/updated modules directly with `pytest`.
- Manual sanity: in a managed workspace with `sdd.storage: separate_repo`, run a plan through the `Tale` auto-approval
  and confirm the coder follow-up launches (resolves `@.sase/sdd/...`) instead of crashing at hand-off; confirm
  `<workspace>/.sase/sdd` is a symlink to the primary store.
- Confirm no `#gh` pre-step clean removes the symlink (clean is run without `-x`; `.sase/` is gitignored).

## Risks & Mitigations

- **Symlink clobbering real content.** Only ever replace an existing _symlink_; refuse (log + skip) a real directory.
- **Store not yet materialized in a fresh clone.** Materialize on demand; if that fails, degrade gracefully (best-effort
  link/refresh, never raise into launch).
- **Concurrency on the shared store.** `git pull --ff-only` is best-effort with a timeout and existing index-lock retry
  handling; correctness does not depend on the pull because the plan is already committed locally in the shared store.
- **`git clean -fdx` in a pre-step could remove the symlink.** `.sase/` is gitignored and standard cleans omit `-x`;
  validate this holds for the coder's `#gh` pre-steps. Creating the link at the end of prep limits exposure.
- **Ordering.** The helper must run in the same workspace the prompt validates against, before validation; the
  `prepare_workspace_if_needed` call site guarantees this.

## Out of Scope

- Implementing the feature the failing agent was planning (auto-commit/push of the SDD store from mutating bead commands
  - finalizer fallback), archived separately as `auto_commit_sdd_store.md`. This plan only removes the infrastructure
    crash that blocked that (and any other) plan-chain hand-off.
- Changing the reference builders or the absolute-path fallback from the earlier plan.
- Relocating the SDD store to be per-workspace, or giving each workspace an independent clone.
