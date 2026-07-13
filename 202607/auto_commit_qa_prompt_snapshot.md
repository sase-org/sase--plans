---
create_time: 2026-07-13 07:44:14
status: wip
prompt: 202607/prompts/auto_commit_qa_prompt_snapshot.md
tier: tale
---
# Auto-commit SDD Q&A prompt-snapshot writes to the external plans repo

## Problem

The `sase-5v` epic-finisher run failed with:

> Commit finalizer failed: uncommitted changes remain after 2 finalizer pass(es) in
> sase-org/sase--plans=<workspace>/sase/repos/plans: 202607/prompts/finish_sase_5v_epic.md

The uncommitted change was **not made by the agent** — it was made automatically by sase itself. When the user answered
the agent's permission question, the questions flow appended the merged `### Questions and Answers` block to the SDD
prompt snapshot in the external plans companion repo and never committed it. Bryan's suspicion is confirmed.

Principle to enforce: **any time sase automatically writes to the plans companion repo, that change must be
auto-committed and auto-pushed** — machine-made changes must never be left for agents (or the user) to clean up.

## Confirmed root cause

Two defects compound:

1. **The Q&A snapshot write is the only machine-made external-SDD-store write with no auto-commit.**
   `handle_questions_marker()` in `src/sase/axe/run_agent_exec_questions.py` calls
   `set_prompt_qa(Path(state.sdd_spec_path), merged_qa_text)` inside a bare `try/except: pass` and stops there. Every
   sibling flow commits its writes:
   - Plan accept (`src/sase/axe/run_agent_exec_plan_accept.py`) writes prompt+plan files, then calls
     `commit_sdd_store_files(..., push_after_commit=True)` (the observed `Add SDD files for finish_sase_5v_epic`
     commit).
   - Bead-state writes are committed by the sync flows, plus a finalizer safety net
     (`_auto_commit_separate_sdd_store_if_possible` in `src/sase/llm_provider/commit_finalizer.py`) that is hard-scoped
     to `paths=[store.kind_root("beads")]`.
   - In-tree `status: wip → done` flips have their own proven-narrow finalizer auto-commit
     (`auto_commit_done_sdd_plan_status` in `src/sase/llm_provider/commit_finalizer_git.py`), but it only applies to the
     **main** repo (`kind == "main"`, `sdd/plans/` prefix), never to external SDD repos.

2. **The commit finalizer then guarantees a failure.** Its instructions (`build_commit_instruction_message()` in
   `src/sase/commit_instructions.py`) tell the agent: "If you did NOT make these changes, ignore this warning." The
   follow-up agent obeyed — its finalizer pass responses explicitly say the prompt edit "predates this work and will
   remain uncommitted" — but `run_commit_finalizer()` requires every repo to be clean and raises `CommitFinalizerError`
   (`reason: dirty_after_max_passes`) after max passes. For external SDD repos there is no ignore/baseline mechanism, so
   a machine-made uncommitted change plus an honest agent is an unavoidable run failure.

### Evidence (from the failed run's artifacts)

- `commit_finalizer_result.json`: `status: failed`, `reason: dirty_after_max_passes`, changed file
  `sase-org/sase--plans:202607/prompts/finish_sase_5v_epic.md`.
- `commit_finalizer_pass_{1,2}_response.md`: the agent committed all of its own work (main repo, plan `status: done`,
  bead bookkeeping — all pushed) and deliberately left only the sase-made Q&A edit.
- Plans repo history: the snapshot was created by the auto-commit `Add SDD files for finish_sase_5v_epic`; the leftover
  Q&A diff was later committed manually by Bryan (`chore: Add q/a to finish_sase_5v_epic.md prompt` — no such message
  template exists anywhere in the codebase, and no run artifact claims it).

## Fix design

### 1. Primary fix: commit + push the Q&A snapshot at write time

In `handle_questions_marker()`, after a successful `set_prompt_qa()`:

- Resolve the store with `resolve_sdd_store(ctx.workspace_dir, ctx.workspace_num or 1)`.
- When the store is external (`not store.is_in_tree`), call
  `commit_sdd_store_files(store, message, auto_commit_type="sdd", paths=[state.sdd_spec_path], artifacts_dir=state.current_artifacts_dir)`.
  - `sdd_commit_targets()` already routes absolute paths to the right companion repo (plans vs research).
  - Leave `push_after_commit=None` so the configured `sdd.push_after_commit` mode applies (default `async`); nothing
    later in the run re-reads the remote, so sync push is unnecessary.
  - Commit message: `Add Q&A to <plan_name> prompt` (name = snapshot filename stem), tagged via the normal
    `auto_commit_type="sdd"` runtime-tag path so it reads as machine-made in history.
- Extract the write+commit into a small helper so it is unit-testable, and replace the silent `except Exception: pass`
  with a logged warning (the questions flow must stay best-effort and never crash the run loop).
- In-tree stores keep current behavior (the snapshot lives inside the workspace repo and belongs to the agent's normal
  commit flow); note this explicitly in the helper docstring.

### 2. Defense in depth: finalizer safety net for proven Q&A-only leftovers

Even with the write-time fix, a failed commit/push (transient git error) or a question answered under an older sase
build can still leave the plans repo dirty and hard-fail a later run. Mirror the existing
`auto_commit_done_sdd_plan_status` precedent for external SDD repos:

- New proven-narrow check in the finalizer: for a dirty external SDD repo (`kind == "sdd"`), consider only tracked,
  modified files matching `<yyyymm>/prompts/*.md`. Prove the diff is machine-made by verifying
  `strip_qa_block(worktree_text).rstrip("\n") == strip_qa_block(head_text).rstrip("\n")` (reusing `strip_qa_block()`
  from `sase.sdd`) — i.e. the only difference is the Q&A block. This covers first-round appends (including the
  newline-at-EOF normalization `set_prompt_qa` performs) and later-round block replacements, while refusing anything an
  agent hand-edited.
- On proof, commit those paths via `commit_sdd_store_files` (which pushes per config) and re-collect dirty state — wired
  into `run_commit_finalizer()` alongside `_auto_commit_done_plan_status_if_possible`, both before the pass loop and
  after each pass, so a Q&A-only leftover never reaches the agent at all.

### 3. Tests

- Questions flow (extend `tests/test_axe_run_agent_exec_plan_followup_questions.py` or a focused new module):
  - External store: answering a question round updates the snapshot **and** commits it (temp git repo or mocked
    `commit_sdd_store_files`; assert message, paths, and `auto_commit_type`).
  - In-tree store: no store commit is attempted.
  - Commit raises: warning logged, loop continues (returns `None`).
- Finalizer safety net (new module next to `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py`):
  - Q&A-only modification in an external plans repo → auto-committed; finalizer finishes clean without prompting the
    agent.
  - Mixed diff (Q&A block plus any other edit), untracked file, or non-prompts path → not auto-committed; normal
    finalizer prompting proceeds.
  - Unit tests for the Q&A-only prover: append with/without trailing newline, multi-round replacement, frontmatter edits
    rejected.

### 4. Verification

`just install`, then `just check`. No Rust-core changes: everything touched is Python agent-runtime glue (axe loop +
llm_provider finalizer), which stays in this repo per the core-boundary rule.

## Out of scope (noted as follow-ups)

- In-tree SDD stores can hit the same contradiction (sase-made Q&A edit in the main workspace repo that the agent
  rightly disclaims). Committing to the agent's working branch mid-run has different trade-offs; not reported and not
  changed here.
- The finalizer instruction text promises "it will not appear again" after an agent disclaims a change, which is untrue
  for external SDD repos. With the safety net above, machine-made Q&A changes never reach the agent, but a general
  disclaim/baseline mechanism for external repos is a separate design question.
