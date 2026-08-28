---
tier: tale
title: Restore the chat-name fallback chain
goal:
  Chat history names prefer the helper, then the git branch, then the current workspace
  directory without breaking resume or fork resolution.
size: small
proposed_by: bbugyi200.athena.sase-um.5.1.land--2
bead: sase-um.5.1
---

- **BEAD:**
  [sase-um.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.5.1.md)

# Restore the chat-name fallback chain

## Problem

Epic phase `sase-um.5.1.1` fixed chat history naming on machines without Bryan's
`branch_or_workspace_name` dotfile helper. Commit `30f384324` established the intended
fallback chain: prefer the helper, otherwise use the current git branch, and only then
use the current working directory's name. Later, unrelated commit `be63e9c7d` replaced
the last two steps with `Path.cwd().parent.name`. That silently changes chat basenames
for ordinary repositories and weakens the phase's resume/fork regression test so it no
longer exercises the git fallback.

The rest of the landing audit found no implementation work to combine with this fix. The
exact-tip visual lane is green (`842 passed, 1 skipped`); the inherited shard and
temporary-path failures were fixed by `95444f868` and `8690fe23a`; and the unrelated
flake proposals are already routed to `sase-uy`, `sase-uz`, `sase-ux`, `sase-u4`,
`sase-t7`, `sase-qr`, and `sase-oz`.

## Implementation

1. Restore an in-process fallback in `src/sase/history/chat.py` with this explicit
   precedence:
   - use a successful, non-empty `branch_or_workspace_name` helper result;
   - otherwise query `git branch --show-current` and use its successful, non-empty
     result;
   - if git is unavailable, fails, or reports no branch (for example, detached HEAD or a
     non-repository directory), use `Path.cwd().resolve(strict=False).name`, falling
     back to the literal `workspace` only if that name is empty.

   Keep helper failures non-fatal so gate-shell settlement still works without the
   personal helper. Keep the git probe non-throwing by handling process-launch errors
   and non-zero/empty results. Apply `strip_reverted_suffix` to the final selected label
   so helper, branch, and directory values all preserve the established suffix
   normalization.

2. Repair and extend `tests/history/test_chat_paths.py` to pin the full precedence
   contract:
   - helper output wins and has its reverted suffix stripped;
   - a missing, failed, or empty helper falls through to the current git branch;
   - a failed or empty git lookup falls through to the current directory name;
   - the directory fallback also has a reverted suffix stripped;
   - a fallback-derived chat saved through `save_chat_history` still resolves through
     `load_chat_for_resume` and renders through `build_fork_injected_history`.

   Make the mocks distinguish helper execution from git execution so the tests cannot
   pass while skipping a precedence level. Retain the gate-shell settlement regression
   in `tests/gate_shell/test_settlement_chat.py`; adjust it only if the restored git
   probe requires an explicit deterministic mock.

## Verification

Run the focused history and settlement tests first:

```bash
.venv/bin/python -m pytest -q \
  tests/history/test_chat_paths.py \
  tests/gate_shell/test_settlement_chat.py
```

Then run the repository-required verification:

```bash
just check
```

## Scope boundary

Do not change visual goldens, close `sase-um.5.1`, run its symbol cleanup, or update the
epic plan status in this tale. Do not absorb the already-routed flake tasks listed
above. Those are landing-agent work or independent task-bead work after this focused
regression fix lands.
