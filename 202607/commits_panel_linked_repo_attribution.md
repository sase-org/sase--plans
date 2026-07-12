---
create_time: 2026-07-11 08:32:44
status: done
prompt: 202607/prompts/commits_panel_linked_repo_attribution.md
tier: tale
---
# Fix Agents-tab COMMITS panel misattributing linked-repo commits to the primary repo

## Problem

The Agents-tab COMMITS metadata panel shows commits made in a _linked_ repo workspace under the _primary_ repo's group.
Observed concretely: an agent running in a `bob-cli` project workspace committed
`11d08d9 fix(task-status-cycler): toggle Next tasks to Done` in the `bob-plugins` linked repo, but the panel renders
that commit under the `bob-cli` group. The agent's SDD commit in the same run rendered correctly (under
`bobs-org/bob-cli--sdd`) only because SDD commit records carry an explicit `repo_name` field.

## Root cause (diagnosed, confirmed against persisted artifacts)

Linked repo workspaces are physically **nested inside** the primary workspace directory at
`<primary_workspace>/.sase/workspaces/<repo>`. This was confirmed from the affected agent's persisted artifacts
(`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/.../20260711075745/`):

- `done.json` → `step_output.meta_commits[0].cwd` = `<primary_workspace>/.sase/workspaces/bob-plugins` (the linked
  workspace),
- `agent_meta.json` → `linked_repos[]` records `bob-plugins` with that exact `workspace_dir`,
- `done.json` → `workspace_dir` = `<primary_workspace>` (the bob-cli workspace root).

The commit `cwd` is recorded correctly at commit time (`write_result_marker` in
`src/sase/workflows/commit/commit_tracking.py` records `os.getcwd()`, and `sase commit` runs from the repo being
committed). The bug is purely in the **display-side attribution** in
`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`:

1. `_repo_name_for_commit_cwd()` checks whether the commit `cwd` is inside the **primary** workspace dir _first_, and
   only falls through to the linked-repo workspace dirs afterwards. Because every nested linked workspace path is also
   inside the primary workspace path, the primary check always wins and the linked-repo loop is unreachable for real
   linked commits.
2. `_commit_cwd_is_primary()` has the same primary-only containment check, so `CommitViewSpec.is_primary` and
   `CommitDiffInfo.is_primary` (via `agent_commit_diffs()`) are also wrong for linked-repo commits.

Existing tests in `tests/ace/tui/widgets/test_agent_display_step_metadata.py` never caught this because they model
linked workspaces as tmp_path **siblings** of the primary workspace (e.g. `tmp_path/"sase_7"` vs
`tmp_path/"sase-core_7"`), a layout that does not occur in practice. The PNG visual-snapshot fixtures use the same
sibling layout, so they are unaffected by the fix.

## Fix

All changes are in `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py` (presentation-side grouping logic for the
TUI panel; no cross-frontend/core behavior changes, so the Rust core boundary is not crossed — attribution inputs are
already persisted per-record and this module is the only consumer of the containment matching, verified by repo-wide
search):

1. **`_repo_name_for_commit_cwd()`** — match the **most specific** workspace first: check the linked repo workspace dirs
   _before_ the primary workspace dir. Since linked workspaces are always nested inside the primary workspace,
   linked-first ordering is exactly most-specific matching. Keep the existing fallbacks unchanged (missing cwd →
   primary; unmatched cwd → basename-derived repo name).
2. **`_commit_cwd_is_primary()`** — return `False` when the cwd is inside any `agent.linked_repos[*].workspace_dir`;
   otherwise keep the current behavior (missing cwd → `True`; containment in `agent.workspace_dir` → `True`). This
   corrects `is_primary` on both `CommitViewSpec` (persisted single-commit path) and `CommitDiffInfo` (commit-diff
   surfaces).
3. Add a short comment stating the nesting constraint (`<primary>/.sase/workspaces/<repo>` lives inside the primary
   tree), since that is the non-obvious fact the ordering depends on.

No persistence-format change is needed: the fix is computed at render time from already-persisted `cwd` +
`agent_meta.json` `linked_repos`, so it retroactively corrects the panel for existing completed agents (including the
observed one).

## Tests

Add regression coverage in `tests/ace/tui/widgets/test_agent_display_step_metadata.py` using the **real nested layout**
(primary at `tmp_path/"bob-cli_10"`, linked at `<primary>/.sase/workspaces/bob-plugins`):

1. Single persisted commit (`meta_commit_message`/`meta_new_commit`/`meta_commit_cwd` pointing at the nested linked
   workspace) renders under the linked repo group, not the primary group, and its `CommitViewSpec.is_primary` is
   `False`.
2. `meta_commits` records: a nested-linked-cwd record groups under the linked repo; a record with cwd at the primary
   workspace root still groups under the primary repo and orders first.
3. `agent_commit_diffs()` returns `is_primary=False` and the linked repo name for a nested linked-cwd record with a diff
   path.
4. Existing tests (sibling layout, explicit `repo_name` override, SDD basename fallback, unmatched cwd fallback) must
   keep passing unchanged — they pin the fallback behaviors the fix must not regress.

## Alternatives considered

- **Persist explicit `repo_name` at commit time** (mirror what SDD commits do) by resolving the linked repo from the
  environment in `write_result_marker`. Rejected as the primary fix: it would not repair already-persisted records, and
  the display side still needs correct cwd matching for legacy records. Could be a separate hardening follow-up.
- **Generic longest-prefix matching across all candidate roots**: equivalent to linked-first ordering given the
  guaranteed nesting direction; the simple reorder is clearer.

## Verification

- Run the targeted test module, then `just check` (full lint + tests, including PNG visual snapshots, which are expected
  to be unaffected since their fixtures use sibling layouts).
- Manual spot check: with the fix applied, the affected completed agent's COMMITS panel should show `11d08d9` under a
  `bob-plugins` group (attribution is recomputed from persisted metadata at render time).
