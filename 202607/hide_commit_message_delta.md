---
create_time: 2026-07-09 13:27:02
status: done
prompt: 202607/prompts/hide_commit_message_delta.md
tier: tale
---
# Hide `commit_message.md` from Agent Deltas

## Problem

The screenshot shows the Agents-tab detail panel rendering a `Deltas:` entry for root-level `commit_message.md`. That
file is temporary bookkeeping created for `sase_git_commit -M commit_message.md` / `sase commit -M commit_message.md`.
The commit CLI deletes it after a successful workflow, but the commit workflow captures the pre-commit diff with
untracked files before deletion, so persisted per-commit diff artifacts can still contain it.

The affected surface is the Agents-tab agent detail `Deltas:` section built from
`src/sase/ace/tui/widgets/prompt_panel/_agent_deltas.py`. This is separate from ChangeSpec `DELTAS:` refresh, which is
computed from VCS state after commit tracking and should not need a behavioral change for this issue.

## Approach

1. Add a narrow agent-delta visibility helper that treats only root-level `commit_message.md` as SASE commit
   bookkeeping. Normalize path separators, but do not hide nested paths like `docs/commit_message.md` because those
   could be real project files.

2. Keep the raw unified-diff parser behavior focused on parsing. Filter the parsed `DeltaEntry` list at the agent
   display boundary instead of changing generic parser semantics.

3. Apply the filter anywhere parsed entries feed the Agents-tab `Deltas:` display:
   - persisted primary commit diffs used by completed agents,
   - persisted linked-repo commit diffs,
   - live primary workspace diffs for active agents,
   - live linked-repo delta groups computed off-thread and cached.

4. Preserve existing rendering, folding, line-stat, sorting, and file-hint behavior by filtering before entries reach
   `build_delta_entries_section`. If filtering removes every entry, omit the `Deltas:` section/group as today.

5. Do not change commit workflow capture, message-file deletion, or raw commit-diff modal behavior in this fix. The
   captured diff remains a faithful pre-commit artifact; this change only prevents commit-message bookkeeping from being
   presented as a user-facing delta entry.

## Tests

Add focused regression coverage in existing TUI delta tests:

1. A completed agent with persisted commit diffs containing both `commit_message.md` and a real file renders only the
   real file under `Deltas:`.

2. A completed agent whose only parsed delta is `commit_message.md` omits the `Deltas:` section.

3. A live linked-repo delta computation with `commit_message.md` plus a real file caches/renders only the real file,
   guarding the active-agent path as well as completed persisted commit diffs.

Run targeted tests first, then follow repository instructions:

```bash
just install
pytest tests/ace/tui/widgets/test_agent_deltas.py tests/ace/tui/widgets/test_linked_deltas.py
just check
```

## Boundaries

This is presentation-only TUI behavior, so it stays in the Python TUI layer rather than the Rust core backend. It does
not add keybindings or configuration, so `default_config.yml` should not change.
