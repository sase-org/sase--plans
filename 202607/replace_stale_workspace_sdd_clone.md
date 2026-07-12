---
create_time: 2026-07-08 14:13:33
status: wip
prompt: 202607/prompts/replace_stale_workspace_sdd_clone.md
tier: tale
---
# Plan: Fix plan-chain SDD crash by replacing a stale workspace SDD _clone_ (not just linking when absent)

## Problem

A plan-chain agent (planner → auto-approve → coder) still crashes with `SystemExit: 1` at the coder hand-off when the
SDD store is `separate_repo` and the agent runs from a managed/ephemeral `sase_<N>` workspace. This is the **same**
failure that killed agent "22" and, most recently, agent "29":

```
❌ ERROR: The following file(s) referenced in the prompt do not exist:
  - @.sase/sdd/tales/202607/agent_provider_setup_doc.md
⚠️ File validation failed. Terminating workflow to prevent errors.
...
  File ".../file_references.py", line 246, in process_file_references
    sys.exit(1)
SystemExit: 1
```

The coder follow-up prompt is:

```
#gh:<changespec> @.sase/sdd/tales/<YYYYMM>/<plan_name>.md

The above plan has been reviewed and approved. Implement it now.
```

The planner phase succeeds and commits the plan into the store; only the coder hand-off crashes, so all planning work is
lost and the implementation never starts.

## Why the previous fix did not work (verified on disk)

The previous change added `ensure_workspace_sdd_link(...)` and wired it into workspace prep. It was built on a **false
assumption** — that managed workspaces have _no_ `.sase/sdd` directory, so a symlink can simply be created. Reality:

- The managed workspace agent 29 ran in **already had a real `.sase/sdd` directory**: its own git clone of the companion
  `sdd` repo. The first line of agent 29's workflow log is the smoking gun:

  ```
  Refusing to overwrite real SDD directory at <workspace>/.sase/sdd
  ```

- That workspace clone is **stale** — checked out far behind the authoritative store and **missing the just-committed
  tale**. The authoritative store under the primary checkout (`<primary>/.sase/sdd`) _does_ have the tale.

- `ensure_workspace_sdd_link` correctly resolves the store to `<primary>/.sase/sdd`, refreshes it, then reaches its
  safety branch: it refuses to overwrite the pre-existing real directory, logs, and returns **without creating the
  symlink**. The coder's relative `@.sase/sdd/...` reference is then resolved against its CWD (the workspace), where the
  stale clone shadows the authoritative store, so the file is absent and `process_file_references` calls `sys.exit(1)`.

- The prior test suite **asserted the buggy behavior** as correct ("an existing real `.sase/sdd` directory is not
  clobbered — logged, skipped"). That test passed while the real scenario crashed, which is why the regression slipped
  through.

### Where these workspace clones come from, and why replacing them is safe

- They are **historical leftovers**. They were materialized when `get_primary_workspace_dir(...)` used to mis-resolve
  the primary to the workspace itself (before the recent checkout-marker resolution fixes). Every older `sase_<N>`
  workspace tends to carry one.
- **No current code path recreates them.** All SDD materialization/clone targets now compute `<primary>/.sase/sdd` (both
  the core `_sdd_dir_for_storage(...)` and the GitHub provider's materialize hook). So a one-time replacement per
  workspace is durable; nothing re-creates a real clone afterward.
- The stale clone is **regenerable and safe to discard**: it is a clean working tree with **zero commits ahead of its
  upstream** (everything in it is already on the remote), and the authoritative store is a strict superset. Nothing
  unique lives only in the workspace clone.

### Why a symlink is durable once placed

- `.sase/` is added to the workspace's `.git/info/exclude`, so `git clean -fd` (used by `vcs_clean_workspace`, no `-x`)
  and the `#gh`/bare-git pre-steps (which use `git stash --include-untracked --exclude-standard`, **not** `git clean`)
  all leave the symlink untouched.
- `ensure_workspace_sdd_link` already re-points an existing symlink to the primary store on every launch (idempotent).
- The only flow that removes it is the corrupt-workspace `shutil.rmtree` re-clone path; after that, the next agent setup
  simply re-links, so the system self-heals.

## Root cause (one sentence)

The launch-time helper only handles the _absent_ case; when a workspace carries a **stale real SDD store clone**, the
helper refuses to touch it, so the coder's relative `@.sase/sdd/...` reference resolves against stale, tale-less content
and the workflow aborts.

## Design

Keep the overall approach from the prior plan — the relative `.sase/sdd/<rel>` reference is the desired output, made
resolvable by a workspace-local view of the authoritative store — but make the helper **replace a regenerable stale
store clone** instead of refusing it. This stays Python-side: `store.py` already owns SDD path/placement policy and the
Rust core only receives resolved paths, so no `sase-core` change is needed (consistent with the accepted boundary
reasoning from the prior plan).

### 1. Teach `ensure_workspace_sdd_link(...)` to replace a safe, regenerable store clone

In `src/sase/sdd/store.py`, change the branch that currently handles a pre-existing **real** `<workspace>/.sase/sdd`
(the "refuse to overwrite" branch). Instead of unconditionally refusing, classify the existing real directory:

1. **Regenerable, safe-to-discard SDD store clone** → discard it and create the symlink to `<primary>/.sase/sdd`. "Safe
   to discard" means all of:
   - it is a git working clone (`.git` present), and
   - its working tree is clean (no uncommitted or untracked-non-ignored changes), and
   - it has **no commits ahead of its upstream** (nothing unique lives only here), and
   - (defensively) its `origin` remote matches the store's `remote_url` when that is known — i.e. it really is a clone
     of the SDD companion repo, not unrelated content.

   When safe, move the stale clone aside to a sibling backup path (e.g. `<workspace>/.sase/sdd.stale-backup`, replacing
   any prior backup) rather than hard-deleting, then create the symlink. Move-aside keeps the operation reversible; once
   the path is a symlink, later launches see a symlink and just repoint, so no repeated backups accumulate.

2. **Genuine/unexpected content, or a store clone carrying local work** (dirty tree, commits ahead, no upstream, or a
   non-matching remote) → **do not destroy it.** Preserve the current refuse-and-log behavior for non-store content. For
   a store clone that merely lags but can't be blindly discarded, best-effort **fast-forward it from the _local_ primary
   store** (fetch/merge `--ff-only` from `<primary>/.sase/sdd`, which is race-free because the plan is already committed
   there synchronously) so the reference can still resolve; if even that isn't safe, log loudly and return. The helper
   must never raise into the launch path.

The rest of the helper is unchanged: in-tree storage no-ops; the co-located case (`<primary> == <workspace>`) only
refreshes; the absent case still creates the symlink; the primary store is still refreshed via `git pull --ff-only`
before linking.

### 2. Wiring is already correct

`prepare_workspace_if_needed(...)` in `src/sase/axe/run_agent_runner_setup.py` already calls the helper after
`prepare_workspace` succeeds (non-home, non-retry), before any prompt-step file validation runs. No new wiring is
needed; this plan only changes the helper's behavior on the pre-existing-real-directory branch.

### Files to change

- `src/sase/sdd/store.py` — replace the "refuse real dir" branch in `ensure_workspace_sdd_link(...)` with the
  classify-and-replace logic above; add a small helper to decide "safe-to-discard regenerable store clone" (clean tree +
  no commits ahead of upstream + matching remote).
- `tests/test_sdd_store.py` — fix the inverted test and add coverage for the real scenarios (below).

### Explicitly unchanged

- The reference builders in `src/sase/axe/run_agent_exec_plan_sdd.py` keep emitting the relative `.sase/sdd/<rel>` form.
- `resolve_sdd_dir` / `resolve_sdd_store` / `_sdd_dir_for_storage` — the authoritative store stays at
  `<primary>/.sase/sdd`; we only fix the workspace-local view of it.
- The vestigial workspace-level `.sase/sdd-store.json` is ignored by resolution (the record is read from the primary),
  so it is harmless and left alone.

## Tests

Replace the inverted safety test and add the missing production-scenario coverage in `tests/test_sdd_store.py`:

- **Stale clean clone (the agent-29 scenario, the key regression):** a managed workspace whose `.sase/sdd` is a real,
  clean git clone with no commits ahead of upstream, distinct from the primary store → the helper **replaces** it with a
  symlink to the primary store, the stale content is moved aside to a backup, and a tale that exists only in the primary
  store is now reachable through the workspace path.
- **No pre-existing `.sase/sdd`:** symlink created (unchanged behavior).
- **Pre-existing stale symlink:** repointed to the primary store; idempotent on re-run.
- **Store clone with local work is protected:** a real `.sase/sdd` clone with uncommitted changes (or commits ahead of
  upstream) is **not** discarded; the helper degrades without raising and (best-effort) fast-forwards from the local
  primary; no data loss.
- **Non-store real directory is protected:** an unrelated real `.sase/sdd` directory (no `.git`, or a non-matching
  remote) is preserved, logged, and skipped.
- **In-tree:** no-op (no symlink).
- **Co-located (`<primary> == <workspace>`):** refresh only, no symlink, no error.
- **Best-effort refresh:** a failing `git pull` does not raise and the symlink is still created.
- **Regression / integration:** compose the coder follow-up prompt for a `separate_repo` store in a managed workspace
  that starts with a stale real clone, run the prep helper, and assert the relative `@.sase/sdd/...` reference now
  **resolves** from the workspace CWD (so the `process_file_references` `sys.exit(1)` path cannot re-trigger).
- Keep the existing wiring test that `prepare_workspace_if_needed` invokes the helper for a non-home, non-retry launch
  and skips it for home-mode / retry-handoff launches.

## Validation

- `just install` then `just check` (lint + mypy + targeted tests).
- Run the changed modules directly with `pytest`.
- Manual sanity: in a managed workspace with `sdd.storage: separate_repo` that currently has a stale real `.sase/sdd`
  clone, run a plan through auto-approval and confirm the coder follow-up launches (resolves `@.sase/sdd/...`) instead
  of crashing; confirm `<workspace>/.sase/sdd` is now a symlink to the primary store and the stale clone was moved
  aside.
- Confirm the symlink survives a `#gh` pre-step (stash-based, no `git clean`) and a `git clean -fd` (no `-x`).

## Risks & Mitigations

- **Destroying content that isn't regenerable.** Only replace a directory that is verified to be a git clone of the SDD
  store with a clean tree and no commits ahead of upstream; anything else is preserved. Move-aside (not delete) keeps
  the operation reversible.
- **A clone with genuine local commits.** The "no commits ahead of upstream" guard blocks discarding it; the fallback
  fast-forwards from the local primary or logs loudly, never silently dropping work.
- **Concurrency on the shared store.** The primary refresh is best-effort with a timeout and existing index-lock retry
  handling; correctness does not depend on it because the plan is already committed locally in the shared store.
- **Never crash the launch.** The helper remains fully best-effort and must never raise into the launch path — a
  link/replace/refresh failure degrades gracefully.

## Out of Scope

- Implementing the feature agent 29 was planning (auto-commit/push of the SDD store), tracked separately.
- Changing the reference builders or relocating the SDD store to be per-workspace.
- A one-shot bulk migration of every existing workspace's stale clone — unnecessary, because each workspace self-heals
  the next time it is prepared for an agent launch.
- Fixing the stale/outdated comment in `src/sase/axe/run_agent_exec_plan_sdd.py` about `#gh` running `git clean` (it is
  now stash-based); optional, unrelated cleanup.
