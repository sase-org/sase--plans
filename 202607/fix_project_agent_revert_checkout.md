---
create_time: 2026-07-14 08:55:43
status: wip
prompt: 202607/prompts/fix_project_agent_revert_checkout.md
tier: tale
---
# Fix: Agents-tab revert fails for project-scoped agents ("Could not check out branch '<project-name>'")

## Problem

Reverting the commits of the sase agent `8e` from the `sase ace` Agents tab failed with:

```
Could not check out branch 'gh_bobs-org__bob-cli' in revert workspace:
git checkout failed: error: pathspec 'gh_bobs-org__bob-cli' did not match any file(s) known to git
```

Agent `8e` was a **project-scoped** agent: it was launched with a `#gh` VCS xprompt against the `gh:bobs-org/bob-cli`
project (a `#tale` run), not against a ChangeSpec. Its commits (primary repo `bob-cli` plus the `bob-cli--plans`
sidecar) landed on each repo's **default branch**, and no branch named after the project exists anywhere.

## Root cause

For project-scoped agent rows, the RUNNING-field `cl_name` is the _project name_ (the project spec basename, e.g.
`gh_bobs-org__bob-cli`), not a ChangeSpec name. This is an established convention — `Agent.is_project_agent`
(`src/sase/ace/tui/models/agent.py:634-639`) detects exactly this case by comparing `cl_name` to the project directory
name.

The Agents-tab revert flow ignores that convention:

1. `build_revert_intent()` / `build_bulk_revert_intent()` (`src/sase/ace/revert_agent_resolution.py`) copy
   `agent.cl_name` into the immutable `RevertIntent` / `BulkRevertIntent` unconditionally.
2. `_prepare_revert_workspace()` → `_prepare_branch()` (`src/sase/ace/revert_agent_workspace.py:310-320`) treats
   `intent.cl_name` as a ChangeSpec branch: `stash_and_clean` → `resolve_revision(cl_name, ...)` → `checkout(...)`.
3. The git provider's `vcs_resolve_revision` (`src/sase/vcs_provider/plugins/_git_query_ops.py:149-201`) finds no local
   or remote branch matching the project name and falls back to the derived name unchanged, so
   `git checkout gh_bobs-org__bob-cli` fails with the pathspec error.
4. For the **primary** repo this raises `RevertWorkspaceError` (`src/sase/ace/revert_agent_workspace.py:233-238`),
   aborting the entire revert before preview. For **linked** repos the same checkout failure would mark the repo
   `prep_blocked`, silently blocking e.g. the plans-sidecar commits even if the primary prep were fixed.

The launch path already handles this scope correctly, and is the model for the fix: project-scoped launches set
`update_target = VCS_DEFAULT_REVISION` (`src/sase/ace/tui/actions/agent_workflow/_launch_body_single.py:127`), and
`prepare_workspace()` (`src/sase/axe/runner_utils.py:184-208`) then checks out
`provider.get_default_parent_revision(workspace_dir)` (git: `origin/<default-branch>`, which `vcs_checkout` DWIMs to the
local default branch) followed by `provider.sync_workspace()` to pick up remote commits. Linked-repo launch prep uses
`VCS_DEFAULT_REVISION` as well (`src/sase/axe/run_agent_runner_setup.py:118-123`).

## Fix design

Teach the revert intent about agent scope and mirror the launch-path checkout for project-scoped reverts. All changes
are in existing Python-side revert orchestration in this repo; no Rust core (`sase-core`) changes are needed (no shared
wire/API or domain model changes — this is glue around the existing VCS provider interface).

### 1. Carry scope on the intents (`src/sase/ace/revert_agent_models.py`)

Add `is_project_scoped: bool = False` to both `RevertIntent` and `BulkRevertIntent`, documented as "the agent ran
against the project (its `cl_name` is the project name), so revert preparation must target the default branch rather
than a ChangeSpec branch".

### 2. Populate it in all four intent builders (`src/sase/ace/revert_agent_resolution.py`)

- `build_revert_intent(agent, ...)` and `build_revert_execute_intent(agent, ...)`:
  `is_project_scoped=agent.is_project_agent`.
- `build_bulk_revert_intent(..., representative)` and `build_bulk_revert_execute_intent(representative, ...)`:
  `is_project_scoped=representative.is_project_agent`. The bulk action layer already rejects marked agents spanning more
  than one `(project_file, cl_name)` pair (`src/sase/ace/tui/actions/agents/_revert.py:284-291`), so the
  representative's scope holds for every target — mixed-scope bulk reverts cannot reach the backend.

### 3. Branch preparation honors scope (`src/sase/ace/revert_agent_workspace.py`)

In `_prepare_branch()`:

- **ChangeSpec-scoped (unchanged)**: `stash_and_clean` → `resolve_revision(cl_name)` → `checkout`.
- **Project-scoped (new)**: `stash_and_clean` → `checkout(provider.get_default_parent_revision(dir))` → best-effort
  `provider.sync_workspace(dir)`:
  - `get_default_parent_revision` returns `origin/<default-branch>` for git providers; `vcs_checkout` strips the
    `origin/` prefix and lands on the local default branch (pushable, so `push_revert_commit` keeps working).
  - The sync step matters because a _reused_ claimed workspace can hold a stale default branch that predates the agent's
    pushed commits; commit discovery scans `git log` of the prepared HEAD (`src/sase/ace/revert_agent_discovery.py`), so
    without a sync the revert could falsely report "No commits tagged for agent ... found". Mirror the launch path
    (`runner_utils.py:201-208`) but make sync failure **non-fatal** here (tolerate `NotImplementedError` and error
    results): repos without an `origin` remote must still prepare, and a failed sync degrades to an honest "no commits
    found" / "commit(s) no longer exist" preview rather than a hard abort.

This helper is shared by primary and linked-repo prep, so the plans-sidecar checkout for project-scoped reverts is fixed
by the same change. ChangeSpec-scoped linked-repo behavior (blocked when the ChangeSpec branch is missing) is
intentionally preserved.

### 4. Tests (`tests/ace/test_revert_agent_workspace.py`)

Extend the existing suite (it already builds throwaway git repos and fakes claim/release):

- **Project-scoped preview succeeds**: primary claimed checkout has AGENT-tagged commits on its default branch only; an
  intent with `is_project_scoped=True` and `cl_name` set to the project name prepares without error and the preview
  discovers the commits (regression test for this bug).
- **Project-scoped linked repo prepares on default branch**: linked checkout with tagged commits on the default branch
  is included in `repos` instead of `prep_blocked`.
- **Stale reused checkout**: claimed clone whose default branch is behind its origin picks up the agent's pushed commit
  after prep (validates the sync step).
- **No-origin tolerance**: project-scoped prep succeeds when the repo has no `origin` remote (sync failure is
  non-fatal).
- **ChangeSpec scope unchanged**: existing tests (`test_prepare_raises_when_primary_branch_checkout_fails`,
  `test_prepare_blocks_linked_repo_missing_branch`, etc.) keep passing without modification.
- Builder coverage in `tests/ace/test_revert_agent_repos.py` (or wherever the intent builders are tested):
  `build_revert_intent` / `build_bulk_revert_intent` and both execute-intent builders set `is_project_scoped` from the
  agent/representative row.

## Verification

- `just install` (fresh workspace) then `just check` must pass.
- Manual sanity path (optional, described for the reviewer): from `sase ace`, select a done project-scoped agent with
  commits (like `8e`) and trigger revert; the preview modal should list the agent's commits across the primary repo and
  the plans sidecar instead of failing with the pathspec error.

## Out of scope

- Changing ChangeSpec-scoped linked-repo prep to fall back to the default branch when the ChangeSpec branch is missing
  (current blocked behavior is by design and covered by existing tests).
- Any TUI/rendering, keymap, or help-modal changes — the fix is entirely in the revert backend, so no footer/help
  updates are triggered.
- Rust core (`sase-core`) changes.
