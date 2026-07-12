---
create_time: 2026-07-12 09:36:10
status: done
prompt: 202607/prompts/fix_agent_deltas_commit_provenance.md
tier: tale
---
# Fix Agents-Tab Deltas Commit Provenance

## Problem and root cause

The `Deltas:` section for agent `6l` showed two files from the plans companion repository even though the agent's active
primary workspace had three modified source files. The artifact timestamps and metadata establish the sequence:

- The agent committed the plans-companion repair at 09:18, producing a two-file persisted commit diff.
- The screenshot was taken at 09:23 while the primary workspace still had the three uncommitted source changes shown by
  `git status`.
- The primary source commit diff was not captured until 09:24.

Every successful `sase commit` currently writes its latest diff path into `agent_meta.json` as `commit_diff_path`,
regardless of which repository was committed. Active-agent enrichment then promotes that field into the generic
`Agent.diff_path`. The Agents detail code treats any `diff_path` as authoritative before considering the live workspace,
and a root Plan/Tale row redirects to its active coder child. Consequently, the latest companion-repository commit diff
was mistaken for the coder's primary diff and masked the live primary-workspace changes.

This is a provenance bug compounded by precedence: one field represents both workflow/primary diffs and arbitrary
per-commit diffs, while persisted data is preferred even when the workspace is active and safe to inspect.

## Proposed implementation

1. **Keep the primary commit fallback repository-aware.**
   - When commit tracking updates `agent_meta.json`, compare the repository for the commit invocation with the agent's
     recorded primary workspace repository.
   - Only publish that diff as the generic `commit_diff_path` fallback when it belongs to the primary repository.
     Commits from linked, companion, temporary, or otherwise external repositories remain recorded in
     `commit_results.json` for their repo-aware history/delta presentation, but they must neither seed nor overwrite the
     primary fallback.
   - Preserve a conservative compatibility path when old/incomplete metadata cannot identify the primary repository,
     rather than dropping historical diff information silently.

2. **Use live-primary-first precedence for active agents.**
   - For an active resolved diff source, probe the existing primary workspace through the current cached VCS fast path
     first. If it has changes, use that live diff for the file panel, `Deltas:` section, and matching row hint.
   - Fall back to a valid persisted primary diff when the active worktree is clean or cannot be resolved, retaining
     visibility after an agent commits its changes.
   - Keep terminal agents persistence-only so a released/reused workspace can never leak another agent's changes.
   - Preserve the existing root Plan/Tale-to-active-coder source resolution and the current cache, background-worker,
     and failure-sentinel behavior; do not add synchronous VCS work to the Textual event loop.

3. **Align badge/detail behavior and document the provenance contract.**
   - Ensure the pencil/live-file-change classification follows the same active/terminal precedence as `get_agent_diff`,
     including probe-failure fallback behavior.
   - Clarify in code comments that workflow/finalized primary diffs are authoritative for terminal agents, whereas
     active workspaces are fresh and must win when dirty.

4. **Add focused regression coverage.**
   - Reproduce the reported case: an active coder has a persisted commit diff from an external companion checkout while
     its primary workspace has a different live diff; assert the primary files render and the companion diff does not
     masquerade as primary.
   - Verify an external commit does not seed or overwrite a previously recorded primary `commit_diff_path`.
   - Verify an active dirty workspace wins over a persisted primary fallback, while an active clean/unresolvable
     workspace can still use that fallback.
   - Verify DONE/FAILED agents never probe or fall back to a live workspace and continue rendering their persisted diff.
   - Cover redirected root Plan/Tale rows so the selected family row uses the active coder workspace.
   - Retain tests for cache/failure behavior so a failed VCS probe does not destabilize an existing hint.

5. **Validate the complete change.**
   - Run the focused commit-tracking, active-diff, agent-delta, and live-hint tests during development.
   - Run `just install` followed by the repository-required `just check` before handoff.
   - If the visual surface changes materially, run the dedicated Agents-tab visual snapshot coverage and inspect any
     diff rather than accepting snapshots automatically.

## Expected outcome

While an agent is active, `Deltas:` reflects its current primary workspace whenever that workspace is dirty. A commit
made in a companion or temporary repository cannot replace the primary diff merely because it was the latest commit.
Once the primary workspace is clean—or the agent is terminal—the panel retains the appropriate persisted primary diff
without probing potentially reused workspaces. Non-primary commits remain available through their repository-aware
commit/delta metadata instead of being mislabeled as primary changes.
