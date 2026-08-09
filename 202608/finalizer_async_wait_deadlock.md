---
tier: tale
title: Break the commit-finalizer async-wait deadlock
goal:
  Agents that background a long-running verification command can no longer burn every commit-finalizer pass "waiting for
  a notification" that can never arrive; the finalizer detects a no-progress pass, escalates with an explicit
  single-turn contract, and reports the stall in its result artifact.
size: medium
proposed_by: bbugyi200.athena.tj
---

- **AGENTS:**
  - [bbugyi200.athena.tj](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.tj.md)
- **COMMITS:**
  - [840cdff](https://github.com/sase-org/sase/commit/840cdff10664d2629ddc38f2677ec80e7dcaaca8) — fix(commit-finalizer):
    break async-wait deadlock in finalizer passes

# Plan: Break the commit-finalizer async-wait deadlock

## Background: the diagnosed failure

`sase-fb.3` (artifact dir `ace-run/202608/05/20260805154737`, model `CLAUDE(sonnet)`) failed with:

```
Commit finalizer failed: uncommitted changes remain after 2 finalizer pass(es) in
main=<workspace>: src/sase/commit_instructions.py, src/sase/llm_provider/commit_finalizer.py,
tests/llm_provider/test_commit_finalizer_auto_sdd_status.py, tests/test_commit_instructions.py.
```

The agent had finished the work correctly. It failed purely on finalization. The recorded tool calls
(`tool_calls.jsonl`) show the exact chain:

1. The agent ran `just check` in the foreground with `timeout: 600000`. The tool result came back at the 10-minute mark
   carrying `timedOutAfterMs` **and** a `backgroundTaskId` — the harness auto-backgrounded the timed-out command.
2. The agent adopted the interactive idiom for that affordance: it stated "I'll wait for the background task to complete
   — it will notify me automatically" and **ended its turn without committing**.
3. SASE invokes Claude as a one-shot subprocess: `claude -p ... --session-id <fresh uuid>` built per pass in
   `src/sase/llm_provider/claude.py:255`. Ending the turn terminates that process, so the background job dies with it
   and the completion notification can never be delivered.
4. `run_commit_finalizer` (`src/sase/llm_provider/commit_finalizer.py:231`) saw the dirty tree and re-invoked — as a
   **brand new process with a brand new session id**. The transcript confirms three distinct session ids across the
   original turn and the two finalizer passes.
5. In pass 1 the agent called `TaskOutput` on the previous pass's background id and got
   `Error: No task found with ID: bhscanuir`. It relaunched `just check` in the background, called `ScheduleWakeup`, and
   ended its turn again. Pass 2 repeated this verbatim.
6. Both `max_passes` (default 2, `commit_finalizer_types.py`) were consumed by waiting. `CommitFinalizerError` was
   raised, the run failed, and workspace #12 was held dirty.

### Why the follow-up prompt makes it worse

`build_follow_up_prompt` (`src/sase/llm_provider/commit_finalizer_prompting.py:107`) embeds `accumulated_response`
verbatim under a `--- Work So Far ---` header. For this run that text literally read _"I'll wait for the background
`just check` run to finish before closing the bead and committing."_ Each fresh session therefore read its predecessor's
invalid plan as the current plan and re-executed it. The prompt is self-poisoning: it reinforces exactly the behaviour
that caused the failure.

### Scope and frequency

Across 3380 recorded finalizer results in this project, 36 ended in `dirty_after_max_passes` (~1.1%). Sampling the
failures shows this async-wait pattern is the dominant _recent_ cause — three of the four August 2026 failures show
"I'll wait for the background task notification" as the final recorded assistant text. Distinct other causes exist in
the historical set (an agent that committed the main workspace but left a linked `sase-core` checkout dirty; an agent
genuinely blocked on a missing bead record), and those are **out of scope here**.

The failure is not sonnet-specific in principle. `@small_phase_worker` resolves through `@cheap` to the pool
`claude/sonnet@xhigh | codex/gpt-5.5@medium` (`src/sase/llm_provider/model_alias_defaults.yml`), which is why only some
small phase workers are affected; the same `dirty_after_max_passes` reason is recorded for codex, opus, haiku and fable
runs too. The fix must therefore be runtime-neutral, consistent with the "Uniform Agent Runtimes" convention — do not
add branching that assumes one runtime lacks a capability.

## Goal

Make the commit finalizer resilient to an agent that ends a pass without making progress, and make the finalizer's own
prompts state the single-turn execution contract explicitly so the agent stops trying to wait for a future that will
never happen.

## Non-goals

- Do **not** add a last-resort auto-commit of arbitrary agent work. The existing narrow auto-commit safety nets
  (`_auto_commit_done_plan_status_if_possible`, `_auto_commit_external_sdd_prompt_qa_if_possible`,
  `_auto_commit_separate_sdd_store_if_possible`) cover machine-managed state only; authoring a commit message for real
  source changes must stay with the agent.
- Do **not** add Claude-specific handling, tool names, or env vars (`BASH_MAX_TIMEOUT_MS` and friends) to the finalizer.
  Keep every prompt change phrased in runtime-neutral terms.
- Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims. A standing memory rule
  covering the agent's _original_ turn is a genuinely valuable complement to this work, but memory edits require the
  user's explicit permission in-conversation and that permission has not been granted. See "Follow-up requiring separate
  approval" below.
- Do not change the `max_passes` default. Escalation, not more passes, is the fix.

## Implementation

### 1. State the single-turn contract in the follow-up prompt

**File:** `src/sase/llm_provider/commit_finalizer_prompting.py`

Extend `build_follow_up_prompt` so every finalizer pass carries an explicit execution-contract section. Requirements for
the new text:

- Say plainly that this is a **non-interactive, single-turn invocation**: when the response ends, the process exits.
  There is no later turn, no notification delivery, and no scheduled wake-up.
- Say that **background work does not survive the end of the turn**, and that identifiers for background work started in
  a previous pass are not resolvable in this one.
- Forbid the specific anti-pattern by behaviour, not by tool name: never end the response in order to wait for something
  to finish, be notified, or be resumed.
- Give the actionable alternative, since the underlying bind is real — a full verification run can exceed the tool's
  command-timeout ceiling:
  1. Split verification into slices that each fit inside the command-timeout ceiling and run them in sequence within
     this turn; or
  2. if work is started in the background, block on it **within this same turn** by polling until it finishes; or
  3. if verification still cannot be completed, commit the work anyway and state in the response exactly what was left
     unverified.
- State that committing the listed changes is the only outcome that ends the finalizer successfully.

Also neutralise the self-poisoning history. Retitle the embedded `--- Work So Far ---` block so it is unambiguously
_prior, already-terminated_ output, and add a sentence directly after it stating that any intention recorded there to
wait for, poll later, or be notified about background work has already failed and must not be repeated.

Keep `build_follow_up_prompt`'s existing signature-compatible call from `run_commit_finalizer`; add new keyword-only
parameters (see step 3) rather than reordering existing ones.

### 2. Reinforce the contract in the shared commit instruction

**File:** `src/sase/commit_instructions.py`

`build_commit_instruction_message` (line 106) composes the instruction embedded in every dirty-repo notice, including
the one `build_dirty_details` renders for external and sidecar repos. Add one short runtime-neutral sentence to `parts`:
the commit must be completed within the current response, because ending the response without committing ends the run.

Keep it to a single sentence — this string is already long and is rendered in several surfaces.
`tests/test_commit_instructions.py` asserts on this message, so update those expectations.

### 3. Detect a no-progress pass and escalate

**File:** `src/sase/llm_provider/commit_finalizer.py`

Inside the `for pass_number in ...` loop (line 231):

- Compute a **progress fingerprint** before invoking the provider and again after re-collecting `dirty_state`. The
  fingerprint must capture both "did anything get committed" and "did the working tree change at all": for each repo in
  the dirty state, its path, its current `HEAD` commit id, and its sorted `changed_files` tuple. Put the helper in
  `commit_finalizer_git.py` next to the existing `_run_git` / `git_changed_files` helpers so the subprocess handling
  stays in one module, and make it degrade gracefully (a repo whose `HEAD` cannot be read contributes a sentinel rather
  than raising).
- A pass whose fingerprint is unchanged made **no progress**. Track a `no_progress_passes` counter.
- When the previous pass made no progress, or when the current pass is the last one
  (`pass_number == config.max_passes`), pass an escalation flag into `build_follow_up_prompt`. The escalated prompt must
  additionally: name the stall concretely ("the previous pass ended without committing and without changing the working
  tree"), and state that this is the final pass and that not committing now fails the run.

### 4. Make the failure self-diagnosing

**Files:** `src/sase/llm_provider/commit_finalizer_types.py`, `src/sase/llm_provider/commit_finalizer.py`,
`src/sase/llm_provider/commit_finalizer_prompting.py`

- Add a `no_progress_passes: int = 0` field to `CommitFinalizerResult` so `commit_finalizer_result.json` records it. The
  field must default, so existing constructions stay valid.
- Extend `failure_message` so that when every pass made no progress it appends a short diagnosis after the existing file
  list — something to the effect that no pass produced a commit or a working-tree change, which indicates the agent
  ended each pass without committing. Keep the existing leading sentence and file list intact; the error string is
  asserted on in `tests/llm_provider/test_codex_fallback_invocation.py:182` via a `match="Commit finalizer failed"`
  prefix, which must keep matching.
- The `error_report.md` surface renders the raised message directly, so no separate change is needed there.

### 5. Tests

Add to `tests/llm_provider/` (mirror the fixture patterns already used by the finalizer tests, e.g.
`tests/llm_provider/test_commit_finalizer_auto_sdd_status.py`):

- **Deadlock regression.** A fake provider whose `invoke` returns text but performs no commit and touches no file. Run
  `run_commit_finalizer` against a dirty git repo with `max_passes=2`. Assert: the finalizer still fails (behaviour
  preserved), `no_progress_passes == 2` in the written result JSON, and the raised message contains the new no-progress
  diagnosis.
- **Escalation content.** Capture the prompts passed to the fake provider. Assert pass 1 contains the single-turn
  contract text and pass 2 additionally contains the stall/final-pass escalation text.
- **Progress is recognised.** A fake provider that commits on pass 1. Assert the finalizer returns `finalized`,
  `no_progress_passes == 0`, and no escalation text was ever emitted.
- **Partial progress is not a stall.** A fake provider that on pass 1 modifies a file (changing `changed_files`) without
  committing, then commits on pass 2. Assert pass 2's prompt does **not** carry the "previous pass made no progress"
  wording, confirming the fingerprint keys on real change rather than merely on "still dirty".
- **History is labelled.** Assert the rendered follow-up prompt no longer presents prior output under a heading that
  reads as a current plan, and carries the "already terminated / do not repeat" caveat.

Update the existing expectations in `tests/test_commit_instructions.py` for the added sentence from step 2.

## Verification

`just check` for this repo can exceed a single command-timeout ceiling on a cold ephemeral workspace — that is the very
condition that triggered this bug. Run it in slices that each finish inside the ceiling, in this order, and never end a
turn waiting on a background run:

```bash
just install
just fmt
just _lint-ruff
just _lint-mypy
just test
just check          # final confirmation once the slices are green
```

Then confirm the behavioural fix on the real artifact shape: point the new regression test's dirty-repo fixture at the
same four-file shape recorded in `ace-run/202608/05/20260805154737/commit_finalizer_result.json` and confirm the
resulting `commit_finalizer_result.json` now carries `no_progress_passes` and the diagnosis.

## Follow-up requiring separate approval

The changes above catch the stall _after_ it starts. They do not stop an agent from entering the async-wait pattern in
its **original** turn, before the finalizer ever runs, because SASE has no code-level standing-instruction injection
point for that turn — the standing guidance lives in `sase/memory/*.md` and the generated agent instruction files.

Adding a short runtime-neutral rule there ("you run in a single non-interactive turn; never end your response to wait
for background work, a notification, or a resume; slice long verification commands or poll them to completion within the
turn") would prevent the failure at its source and is the natural complement to this plan.

That edit is deliberately excluded from this plan's implementation scope: SASE convention requires the user's explicit
in-conversation permission for memory-file edits, and authorization found in a plan file does not count. The
implementing agent must **not** make that edit. It should be raised with the user separately.
