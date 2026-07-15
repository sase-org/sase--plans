---
tier: tale
goal: 'Every numbered workspace launch removes its launch-scoped sase/repos tree and
  recreates the required plans sidecar with fresh authoritative-remote clone semantics,
  without inheriting local commits or introducing rebase conflicts.

  '
create_time: 2026-07-15 11:04:29
status: wip
prompt: 202607/prompts/fresh_workspace_sidecar_clone.md
---

# Plan: Fresh workspace sidecar clones

## Context and root cause

Numbered workspace preparation already calls `clear_workspace_repos()` after the host checkout is cleaned and updated.
That helper makes the launch path disappear quickly by atomically renaming the entire `sase/repos/` tree into
`.sase/trash/`, then deleting the renamed tree and older trash entries in a detached process. Agent and workflow launch
paths both recreate the plans SDD clone after this teardown.

The clone recreated at `sase/repos/plans` is not currently guaranteed to have the same history as a normal clone of its
recorded remote. When the target is absent, `ensure_sidecar_sdd_clone()` prefers the durable primary-checkout plans
clone as a local clone source, changes the new clone's origin to the recorded remote, and runs `git pull --rebase`. A
matching origin only establishes repository identity; it does not prove that the primary clone has no local, unpushed,
or divergent commits. Those commits are copied into the supposedly fresh workspace clone and can conflict during the
immediate rebase. A second sidecar materialization path in `_materialize_remote_identified_sidecar()` can likewise clone
the primary checkout instead of the authoritative remote.

The cleanup optimization is sound and should remain. The unsafe optimization is reusing primary-checkout refs/history to
create a sidecar that is required to behave like a fresh remote clone.

## Implementation

1. Establish one launch-preparation invariant for numbered workspaces.
   - Keep primary workspaces (`workspace_num <= 1`), home-mode launches, and retry-spawn children protected by their
     existing guards; retry children must preserve the parent agent's in-progress repository work.
   - For ordinary agent and workflow launches, require the successful host-workspace clean/update to be followed
     synchronously by removal of the complete launch-visible `sase/repos/` path before any linked repo or sidecar is
     resolved or materialized.
   - Preserve the fast atomic rename into `.sase/trash/` and detached recursive deletion. Treat the rename/removal
     boundary as the synchronous guarantee: nothing from the prior `sase/repos/` tree may remain reachable at that path
     when rehydration begins. Keep safe handling for missing paths, files, and symlinks, and keep best-effort sweeping
     of older trash without following symlinks.
   - Keep the agent and workflow call order aligned through a shared launch helper or equivalent focused coverage so
     future launch paths cannot materialize plans before teardown.

2. Make a missing sidecar target a true clone of its authoritative remote.
   - In the SDD sidecar materializer, replace the local-primary-clone-then-rebase branch with a normal non-interactive
     `git clone <recorded-remote> <target>`. The primary clone may help identify or validate the configured remote, but
     its branches, working tree, and local commits must never seed the fresh target.
   - Apply the same rule to configured/auto-cloned sidecars in `_materialize_remote_identified_sidecar()` so there is no
     alternate launch path that recreates `sase/repos/<role>` from primary-checkout history.
   - Do not add shallow clones, shared clones, alternates, reference repositories, hard-link caches, or a local-clone
     fallback. Those techniques can add history, object-lifetime, corruption, or recovery behavior that a plain remote
     clone would not have. The plans repository is expected to be small, so the safe speed optimization is to avoid
     redundant work around one normal clone, not to change clone semantics.
   - Preserve existing in-place synchronization behavior for repositories that were not removed by launch preparation
     (for example, durable primary checkouts and explicitly opened repos). This change targets creation of an absent
     numbered workspace sidecar, not recovery of user-owned durable clones.

3. Make required launch rehydration transactional and fail closed.
   - Materialize the plans sidecar in strict mode after teardown. A clone, authentication, remote, or filesystem failure
     must stop workspace preparation instead of launching an agent with an absent or partially initialized plans
     repository.
   - Ensure a failed clone does not leave a target that later code can mistake for a valid repository. Clean clone
     staging/partial output and surface the original clone diagnostic. Do not fall back to the primary checkout, because
     that would reintroduce failures and conflicts that a fresh remote clone cannot produce.
   - Once the fresh plans clone exists, let linked-repo resolution reuse that exact checkout for environment metadata
     without cloning it again. Skip unnecessary clean/fetch/rebase preparation for a sidecar proven to have been freshly
     cloned during this launch; continue the existing preparation behavior for ordinary auto-cloned linked repositories.
     This removes an extra network round trip and the redundant rebase opportunity while retaining normal clone
     correctness.
   - Keep origin normalization and `.git/info/exclude` installation, but perform them without changing the freshly
     cloned branch or adding local commits.

4. Add regression coverage for the actual conflict scenario and lifecycle edges.
   - Build a plans remote plus a durable primary clone containing an unpushed commit, then advance the remote
     incompatibly. Prepare a numbered workspace and assert that its new `sase/repos/plans` HEAD and tracking branch
     match the remote, the primary-only commit is absent, the worktree is clean, and no rebase/conflict state exists.
   - Assert that the clone command targets the recorded remote and that neither SDD nor configured-sidecar
     materialization uses the primary checkout as its clone source. Replace the current test that requires the
     matching-local-source fast path with tests for authoritative-remote cloning.
   - Cover clone failure: preparation raises, no usable/partial target survives, and no local fallback is attempted.
     Cover a successful fresh clone without a redundant pull/rebase or second clone.
   - Retain and strengthen cleanup tests for the whole `sase/repos/` namespace, including plans, linked repos, external
     repos/strays, non-directory shapes, stale trash, asynchronous-delete spawn failure, and primary-workspace guards.
   - Verify both agent and workflow launch paths perform `prepare host -> clear repo tree -> strict plans clone`, while
     home mode, retry-spawn preservation, failed host preparation, and normal linked-repo preparation remain unchanged.

5. Update documentation to state the precise contract.
   - Replace claims that launch sidecars are recreated from a durable primary clone with the authoritative behavior: the
     numbered workspace's full `sase/repos/` tree is atomically evicted, required plans are cloned directly from their
     recorded remote, and other repos remain lazy unless configured for auto-clone.
   - Clarify that pull-with-rebase applies when synchronizing a retained existing clone, not when creating the freshly
     prepared plans checkout.

This remains Python launch/filesystem orchestration and does not introduce a new cross-frontend domain API or Rust wire
contract.

## Validation

Run the focused lifecycle and SDD tests first:

```bash
uv run pytest \
  tests/test_linked_repo_workspaces.py \
  tests/sdd_store/test_workspace_clone.py \
  tests/test_run_agent_runner_setup.py \
  tests/test_run_workflow_visibility.py \
  tests/test_linked_repo_resolution.py
```

Then install the current workspace dependencies and run the repository-required full validation:

```bash
just install
just check
```

Re-run `rg` over `docs/` for the old local-source/network-fallback wording so the documented launch contract matches the
tested behavior.
