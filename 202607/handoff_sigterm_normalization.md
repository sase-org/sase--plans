---
tier: tale
title: Normalize plan/question handoff interruptions without matching provider error text
goal:
  An agent that submits a plan with `sase plan propose` is recorded as DONE regardless of which LLM provider CLI it ran
  under, instead of being permanently marked FAILED when the provider reports the handoff SIGTERM with a non-standard
  exit code.
proposed_by: bbugyi200.athena.qs
create_time: 2026-07-31 16:36:29
status: wip
---

- **PROMPT:** [202607/prompts/handoff_sigterm_normalization.md](prompts/handoff_sigterm_normalization.md)

# Plan: Normalize handoff interruptions structurally, not by parsing provider error text

## Problem

An agent (`qr`, running the `agy` / Antigravity provider) submitted a plan successfully via the `/sase_plan` skill, but
ACE then displayed it as **FAILED** with an `LLMInvocationError` traceback — even though the plan was archived, the gate
was created, the plan was approved, the SDD files were committed, and the `--code` follow-up agent launched normally.

The handoff itself worked. Only the planner's durable status record is wrong, and because the planner's artifacts
directory is never rewritten after the follow-up moves to its own directory, the bogus FAILED record is **permanent**.

### Root cause

`sase plan propose` writes `.sase_plan_pending` and then SIGTERMs the agent runner's whole process group
(`src/sase/main/plan_propose_handler.py:175-202` → `src/sase/main/utils.py:kill_agent_runner_group`). That SIGTERM
intentionally kills the in-flight provider CLI, so the runner always observes a provider failure on the way to the
handoff. The runner then repairs that self-inflicted failure in
`src/sase/axe/run_agent_helpers_handoff.py:normalize_handoff_interruption_state`.

That repair only fires when the recorded error string matches one of three substrings
(`src/sase/axe/run_agent_helpers_handoff.py:16-24`):

```python
return (
    "exit code -15" in lowered
    or "exit code 143" in lowered
    or "sigterm" in lowered
)
```

`claude` and `codex` die from SIGTERM with `returncode == -15`, so their error text contains `exit code -15` and the
repair fires. `agy` does **not**: it traps the signal and exits `1` with its own message. The observed error was

```
LLMInvocationError: Error running LLM provider command (exit code 1)
stderr: Error: timeout waiting for response
```

which matches none of the three substrings. `normalize_handoff_interruption_state` therefore left `workflow_state.json`
at `status: "failed"` with the full traceback, and `finalize_handoff_artifacts_as_completed` (same file, lines 96-162)
could not help because it only rewrites the _non-terminal_ statuses `running`, `waiting_hitl`, and `in_progress` —
`failed` is terminal.

The surviving `failed` status is what the user sees: `src/sase/ace/tui/models/_loaders/_workflow_loaders.py` maps
`workflow_state.json`'s `status: "failed"` straight to the display status `FAILED`, and
`src/sase/ace/tui/models/_dedup.py:dedup_workflow_entries` explicitly prefers that non-RUNNING status over the live
RUNNING entry ("Prefer non-RUNNING status from workflow_state.json (accurate status)").
`src/sase/ace/tui/models/_loaders/_workflow_step_loaders.py:190-199` additionally propagates the parent error and
traceback onto any still-RUNNING child step.

The substring test is a fragile proxy for something the runner already knows authoritatively. Both call sites reach
`normalize_handoff_interruption_state` only from `_handle_killed_iteration` (`src/sase/axe/run_agent_exec.py:125-156`),
which requires all of:

1. `was_killed()` is true (the runner's own SIGTERM handler fired),
2. `has_user_kill_intent()` is false (this was not a user kill), and
3. a `.sase_plan_pending` / `.sase_questions_pending` marker exists whose timestamp predates the kill
   (`_marker_predates_kill`).

When those hold, a `failed` agent step in that artifacts directory is by construction the handoff SIGTERM's doing.
Re-deriving that fact from provider stderr is both unnecessary and provider-specific, which also conflicts with the
repo's **Uniform Agent Runtimes** convention.

## Approach

Make the repair structural. Drop the error-text heuristic and normalize the interrupted agent step directly, keeping a
real (and better-targeted) safety property: failures recorded by non-agent embedded steps are still preserved, because a
`bash`/`python` post-step can fail for reasons unrelated to the SIGTERM.

## Changes

### 1. `src/sase/axe/run_agent_helpers_handoff.py`

- Delete the `_is_sigterm_error` helper and every use of it.
- In `normalize_handoff_interruption_state`:
  - Rewrite any `steps[]` entry whose `status == "failed"` to `completed`, clearing `error` and `traceback`. `ace-run`
    builds a single-step anonymous workflow (`create_anonymous_workflow` in `src/sase/axe/run_agent_exec.py`), so these
    entries are the interrupted agent step.
  - Rewrite the top-level `status` from `failed` to `completed` (clearing `error` / `traceback`) whenever any step was
    normalized **or** the top-level status is `failed`. Do not keep gating this on the error text.
  - For `prompt_step_*.json` markers, rewrite `status == "failed"` to `completed` (clearing `error` / `traceback`)
    **only** when the marker's `step_type` is `agent`. Treat a missing `step_type` as `agent`, matching the default in
    `src/sase/ace/tui/models/_loaders/_workflow_step_loaders.py` (`data.get("step_type", "agent")`). Leave `bash` /
    `python` embedded-step markers untouched.
  - Preserve the existing single-refresh behavior: keep the `index_dirty` accumulator and the single
    `update_agent_artifact_index_for_marker_mutation(artifacts_dir)` call at the end, so state-file and marker rewrites
    still coalesce into exactly one Tier 1 index update. The current early `return` that skips marker scanning when no
    SIGTERM failure was seen must go away with the heuristic; make sure the function still performs at most one index
    refresh and none at all when nothing changed.
- Update the module/function docstring to state the precondition (called only after a validated handoff SIGTERM) and to
  record why the text heuristic was removed: provider CLIs surface the same SIGTERM inconsistently — `claude` and
  `codex` exit `-15`, while `agy` exits `1` with `Error: timeout waiting for response`.

`finalize_handoff_artifacts_as_completed` and `update_step_marker_chat_path` need no changes.

### 2. `tests/test_axe_run_agent_helpers_handoff.py`

- Keep `test_normalize_handoff_interruption_state_rewrites_sigterm_failures` and `..._rewrites_exit_code_143` as-is;
  they must still pass.
- Add a regression test named for this bug that uses the exact `agy` shape: state and marker `error` of
  `"LLMInvocationError: Error running LLM provider command (exit code 1)\nstderr: Error: timeout waiting for response"`,
  marker `step_type: "agent"`. Assert `workflow_state.json` `status` and the `steps[0]` entry both become `completed`
  with `error`/`traceback` cleared, and that the marker becomes `completed`.
- Replace `test_normalize_handoff_interruption_state_keeps_real_failures` with a test of the new, narrower safety
  property: given a failed `agent` marker plus a failed embedded marker (e.g. `prompt_step_gh__diff.json` with
  `step_type: "bash"`), the agent marker is normalized to `completed` while the `bash` marker keeps its `failed` status
  and its `error`. Name it so the intent is obvious (e.g.
  `test_normalize_handoff_interruption_state_keeps_embedded_step_failures`).
- Add a case asserting a marker with no `step_type` key is treated as an agent step and normalized.
- Keep `test_normalize_handoff_interruption_state_coalesces_index_update` passing (exactly one index refresh).
- Add a no-op case: an artifacts dir whose state and markers are already `completed` triggers **zero**
  `update_agent_artifact_index_for_marker_mutation` calls.

### 3. `tests/test_agent_artifact_marker_mutation_audit.py`

This AST audit pins the mutation-call shape for
`src/sase/axe/run_agent_helpers_handoff.py:normalize_handoff_interruption_state` as
`mutation_calls=("open", "dump", "open", "dump")` with `lifecycle_calls=(_UPDATE_INDEX,)`. If the refactor changes the
number or order of `open`/`dump` calls in that function, update this entry to match the new shape — do not weaken the
audit or add an exemption.

## Out of scope

- Changing the `agy` provider's SIGTERM handling or its `Error: timeout waiting for response` message. That is an
  Antigravity CLI behavior; the fix above makes SASE robust to it and to any other provider that exits non-`-15`.
- Backfilling already-written artifacts. This fix is forward-only; existing runs keep their stale `failed`
  `workflow_state.json`.
- Merging `normalize_handoff_interruption_state` and `finalize_handoff_artifacts_as_completed`. They stay separate.

## Verification

1. `just install` (workspaces are ephemeral, so dependencies may be stale).
2. `just check`.
3. Targeted run:
   `uv run pytest tests/test_axe_run_agent_helpers_handoff.py tests/test_agent_artifact_marker_mutation_audit.py -v`.
4. Confirm the plan-chain golden tests still pass: `uv run pytest tests/plan_chain_golden -v`.

## Success criteria

- A handoff SIGTERM that a provider reports as `exit code 1` (or any other code) leaves the planner's
  `workflow_state.json` at `status: "completed"` with `error` and `traceback` cleared, so ACE shows the planner as DONE
  rather than FAILED.
- `_is_sigterm_error` no longer exists; no code path branches on a provider CLI's exit code or stderr wording to decide
  whether a handoff happened.
- Genuine `bash` / `python` embedded post-step failures are still preserved in their markers.
- `just check` passes.
