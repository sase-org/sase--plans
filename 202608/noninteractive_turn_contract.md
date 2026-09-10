---
tier: epic
title: Stop agents ending a turn to wait for events that can never arrive
goal: "A SASE agent that ends its turn waiting for a background task, a monitor event,
  or a scheduled wake-up is prevented from doing so by a runtime-neutral single-turn
  contract carried on every provider invocation, detected by a shared classifier when it
  happens anyway, given bounded recovery passes to finish the work, and never recorded
  as `outcome: completed` when it did not.

  "
phases:
  - id: blockwait
    title: Bounded blocking wait for durable background tasks
    depends_on: []
    size: small
    description: 'blockwait: add a deadline to `sase task show --follow` and `sase task
      run --wait` with a distinct "still running" exit code, so an agent can block on a
      long command inside its command-timeout ceiling and resume blocking on the next
      tool call.

      '
  - id: classify
    title: Shared turn-progress classifier and its calibration corpus
    depends_on: []
    size: medium
    description: "classify: build a runtime-neutral classifier that scores one provider
      turn from its final text plus structural evidence read from `tool_calls.jsonl`,
      and calibrate it against the recorded historical run corpus with labelled true and
      false positives.

      "
  - id: contract
    title: Single-turn execution contract on every provider invocation
    depends_on:
      - blockwait
    size: medium
    description: "contract: promote the finalizer's single-turn contract text into a
      shared module, inject it at every provider's transport boundary, and retire the
      Antigravity-only print-mode directive in favour of it.

      "
  - id: recover
    title: Provider-neutral wait-recovery finalizer
    depends_on:
      - classify
      - contract
    size: medium
    description: "recover: run a bounded recovery finalizer after each provider turn
      that re-invokes the agent with evidence-specific escalation, records a result
      artifact, and fails the run rather than reporting an abandoned turn as completed.

      "
  - id: surface
    title: Make the stall visible in reports and docs
    depends_on:
      - recover
    size: small
    description:
      "surface: render the recovery verdict in the runner error report, publish the
      calibration report as a durable artifact, and document the single-turn contract
      for provider authors."
proposed_by: bbugyi200.athena.xu
create_time: 2026-09-09 20:00:15
status: wip
---

- **PROMPT:**
  [prompts/202608/noninteractive_turn_contract.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/noninteractive_turn_contract.md)

# Plan: Stop agents ending a turn to wait for events that can never arrive

## The failure

`sase-iy.land` (artifacts `ace-run/202608/11/20260811061808`, provider `claude`, model
`opus`) was the land agent for epic `sase-iy`. Its recorded final assistant text ends:

> `selection-health` is non-vacuous — 687 full-lane runs recorded, 0 current
> reproducible flakes. Waiting on `just check-full` (the full pytest lane plus the
> test-cost budget gate, which is the criterion that blocked both prior retire
> attempts).
>
> I'll stop polling and wait for the monitor to report completion.
>
> The full suite is still running. I'll wait for the monitor event rather than polling
> further.

`done.json` for that run records `"outcome": "completed"`. The epic's land verification
never finished, and SASE reported success.

The run's `tool_calls.jsonl` (108 records, 54 tool uses) shows the exact chain:

1. `Bash` with `run_in_background: true` running
   `just check-full > /tmp/check_full_iy.log 2>&1; echo "CHECKFULL_EXIT=$?" >> ...`.
2. A `Monitor` tool call.
3. Several foreground polls of the log tail.
4. A final `Bash` call whose command is literally `echo "waiting";` with description
   `No-op`, then end of turn.

Every tool use has a matching `ToolResult`, so nothing was left orphaned — the agent
finished its tools cleanly and then deliberately stopped, waiting for a notification.

## Root cause

Agent CLIs document background execution and event subscription as _asynchronous
notification_ affordances. Claude Code's `Bash(run_in_background)` re-invokes the agent
when the command exits; its `Monitor` tool streams each stdout line into the
conversation as a notification. Both contracts hold only where there is a later turn to
deliver into.

SASE invokes every provider as a one-shot, non-interactive subprocess:
`claude -p ... --session-id <fresh uuid>` built per attempt in
`src/sase/llm_provider/claude.py:255`, with the equivalent shape in `codex.py`,
`qwen.py`, `opencode.py`, `muse.py`, and `agy.py`. When the model emits its final
message, the process exits. There is no later turn, so the notification the model is
waiting for can never be delivered, and the background job dies with its parent.

The model is not misreading its tools; it is applying an interactive contract inside a
non-interactive harness that never told it the contract does not hold.

The bind that pushes it there is real. `just check-full` exceeds the 10-minute
command-timeout ceiling Claude Code allows for a single foreground `Bash` call, so the
agent cannot run it in one foreground tool call at all. Backgrounding is the only option
the model sees, and once it has backgrounded, waiting for the notification is the
documented idiom.

## Measured scope

Scanning every recorded agent run under `~/.sase/projects/*/artifacts/ace-run/` (6994
candidate directories; 1058 with a `done.json` recording `outcome: completed` and a
non-empty final response):

| signal                                                      | runs |
| ----------------------------------------------------------- | ---- |
| wait language in the final ~1200 characters                 | 25   |
| a `run_in_background: true` tool use anywhere in the run    | 40   |
| a `Monitor` / `ScheduleWakeup` tool use anywhere in the run | 39   |
| both wait language and one of those tool uses               | 8    |

Manual review of those 8 gives 6 true positives and 2 false positives:

**True positives** (all recorded `outcome: completed`):

- `ace-run/202607/31/20260731132625` — `ql--code`, claude
- `ace-run/202608/01/20260801075713` — `qx.f0--code`, claude
- `ace-run/202608/02/20260802094605` — `ru--code`, claude
- `ace-run/202608/10/20260810151037` — `sase-j0.w1--code`, claude
- `ace-run/202608/11/20260811060607` — `sase-iy.5`, **codex** ("I'll resume
  automatically once the `check-full` and `test-visual` background runs finish")
- `ace-run/202608/11/20260811061808` — `sase-iy.land`, claude

**False positives** (both genuinely finished and committed; the regex matched incidental
prose in a completion summary):

- `ace-run/202607/31/20260731070917` — `pw.f0--code`
- `ace-run/202607/31/20260731091735` — `qb--code`

Two conclusions follow, and they drive the design:

- **This is not Claude-specific.** `sase-iy.5` is a codex run that abandoned its turn
  the same way, in the same epic, on the same day. The fix must be runtime-neutral, per
  the "Uniform Agent Runtimes" convention.
- **Text alone is not a sufficient signal.** 25 text matches collapse to 8 once
  structural corroboration is required, and 2 of those 8 are still wrong. The classifier
  must require structural evidence and must be calibrated, not merely written.

## What already exists, and why it did not help

Three partial mechanisms are already in the tree. This epic completes them rather than
replacing them.

1. **The commit finalizer's single-turn contract.** The landed plan
   `finalizer_async_wait_deadlock.md` (2026-08-05) diagnosed this exact failure inside
   the commit finalizer and added `_SINGLE_TURN_CONTRACT`
   (`src/sase/llm_provider/commit_finalizer_prompting.py:106`) plus no-progress
   fingerprinting. It only engages when the working tree is dirty. `sase-iy.land` was a
   verification agent that changed no files, so `commit_finalizer_result.json` recorded
   `{"status": "clean", "reason": "no_changes", "passes": 0}` and no contract text was
   ever shown to the agent. That plan's own "Follow-up requiring separate approval"
   section names the remaining gap: nothing protects the agent's _original_ turn.

2. **The Antigravity print-mode guard.** `src/sase/llm_provider/agy.py` already carries
   the complete shape of the fix — a transport-level directive
   (`_AGY_PRINT_MODE_DIRECTIVE`, line 62), a no-progress classifier
   (`_classify_agy_no_progress`, line 189; `_looks_like_no_progress`, line 207), a
   bounded continuation budget, and a hard failure when the budget is exhausted (line
   541). It is wired into exactly one provider. Its existence also retires the earlier
   plan's premise that "SASE has no code-level standing-instruction injection point for
   that turn": the provider transport boundary is that injection point.

3. **Durable background tasks.** `sase task run` already starts a detached, supervised
   task that survives the agent process, and `sase task show <id> --follow` streams it
   to a terminal state. `sase.tasks.runner.wait_for_task` (line 187) already accepts a
   `timeout` and raises `TimeoutError`. The affordance an agent needs in order to obey
   the contract is nearly complete; it is neither reachable from the CLI with a deadline
   nor mentioned anywhere the agent will read.

## Design

Four layers, in the order they engage:

- **Prevention.** Every provider invocation carries the single-turn contract, so the
  model knows before it acts that no notification, monitor event, or wake-up will be
  delivered. The contract also names the affordance that makes compliance possible.
- **Affordance.** A bounded blocking wait on a durable task, so "block within this turn"
  is achievable for a command that exceeds the command-timeout ceiling.
- **Detection.** A shared classifier scores each turn from its final text plus
  structural evidence read from the runtime-neutral `tool_calls.jsonl` artifact.
- **Recovery and honesty.** A bounded recovery finalizer re-invokes the agent with
  evidence-specific escalation; if the turn is still abandoned, the run fails instead of
  being recorded as completed.

Two ordering constraints matter and are easy to get wrong:

- The recovery finalizer runs **before** `run_commit_finalizer`, because a recovered
  agent may produce changes that then need committing.
- On budget exhaustion it must **not raise immediately**. It records the verdict, lets
  `run_commit_finalizer` run so real work still gets committed, and raises afterwards. A
  naive early raise would regress today's commit behaviour for abandoned turns that also
  left a dirty tree. If the commit finalizer itself raises, that error wins.

## Non-goals

- **Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`).** SASE convention requires
  the user's explicit in-conversation permission for memory edits, and authorization
  found in a plan file does not count. The contract this epic injects is per-invocation
  and code-owned, which is the correct home for it anyway: the guidance is true for
  SASE's non-interactive runs and false for an interactive session, so it must not be
  written into repo-wide instruction files. See "Follow-up requiring separate approval".
- **Do not forbid background execution.** Backgrounding a long command is legitimate and
  often necessary. The prohibited behaviour is _ending the turn while waiting_.
- **Do not add runtime-specific branching to shared code.** Provider-specific wording is
  supplied through a provider hook, not through `if provider == "claude"`.
- **Do not raise the commit finalizer's `max_passes`.** Recovery is a separate budget.
- **Do not attempt to make notifications actually deliverable** (session resume, a
  persistent event loop, re-invocation on background completion). That is a much larger
  change to the runner and is not needed to fix this failure.

## Bounded blocking wait for durable background tasks

**Files:** `src/sase/main/parser_task.py`, `src/sase/main/task_handler.py`,
`src/sase/tasks/runner.py`.

Read `sase/memory/cli_rules.md` with the `/sase_memory_read` skill before adding the
option; that note owns the CLI conventions this must follow.

`wait_for_task` (`src/sase/tasks/runner.py:187`) already takes `timeout` and raises
`TimeoutError`. Expose it:

- Add `--timeout SECONDS` to `sase task show` (meaningful with `--follow`) and to
  `sase task run --wait`. Without it, behaviour is unchanged.
- When the deadline passes with the task still running, exit with a distinct, documented
  non-zero exit code (not the task's exit code, and not the generic error code), print
  the task id and its current status, and leave the task running. Re-invoking
  `sase task show <id> --follow --timeout N` must resume streaming from where the
  previous call stopped, so a sequence of bounded waits is equivalent to one unbounded
  wait.
- `--format json` with `--follow --timeout` must emit the non-terminal snapshot rather
  than blocking past the deadline.

This is the affordance the `contract` phase prescribes, so the exit code and the flag
name are load-bearing: pick them here and have `contract` quote them verbatim.

**Tests** (`tests/` alongside the existing task CLI tests): deadline expiry returns the
distinct exit code and leaves the task running; a task that finishes before the deadline
returns its own exit code; two sequential bounded follows see the full log exactly once,
with no duplicated or dropped lines; `--format json --follow --timeout` returns the
non-terminal snapshot.

## Shared turn-progress classifier and its calibration corpus

**New file:** `src/sase/llm_provider/turn_progress.py`. **Touches:**
`src/sase/llm_provider/agy.py`, `tests/llm_provider/test_agy_no_progress.py`.

One runtime-neutral classifier, no provider branching.

**Structural evidence.** `tool_calls.jsonl` is already written in the same normalized
schema by claude, codex, qwen, muse, and opencode (stream parsers) and by agy
(trajectory extraction) — see the module docstring of
`src/sase/llm_provider/_tool_calls.py`. Read it rather than inventing a new signal:

- `event: "ToolUse"` / `"ToolResult"` rows keyed by `tool_use_id`, so unresolved tool
  uses are computable exactly as `_orphaned_tool_uses` already does in
  `src/sase/llm_provider/_tool_call_finalize.py`.
- `tool_input_summary` for `Bash` already preserves `run_in_background` verbatim
  (`_summarize_tool_input` in `src/sase/llm_provider/_tool_call_common.py`), and
  preserves the redacted `command` string, which the recovery nudge needs to quote back.
- Tool names are recorded, so event-subscription tools (`Monitor`, `ScheduleWakeup`,
  `CronCreate`, `TaskCreate`, and their equivalents on other runtimes) are detectable
  without parsing text.

**Turn scoping.** A single agent run invokes the provider more than once (original turn,
then finalizer passes). Evidence must cover only the records appended by the turn being
classified. Snapshot the artifact's size or record count immediately before the provider
call and read the delta, mirroring the snapshot pattern
`prepare_agy_tool_call_extraction` already uses. A missing or unreadable artifact yields
empty evidence and must never raise into the invocation path.

**API.**

- `TurnEvidence` — `tool_use_count`, `unresolved_tool_use_count`,
  `background_commands: tuple[str, ...]`, `event_subscription_tools: tuple[str, ...]`,
  plus a flag recording whether structural evidence was available at all.
- `read_turn_evidence(artifacts_dir, *, snapshot) -> TurnEvidence`.
- `classify_turn(content, evidence) -> TurnVerdict` with
  `kind: "progress" | "abandoned_wait" | "no_progress"`, a stable machine `reason`, and
  the evidence that produced it.

**Combination rule.** Move agy's `_NO_PROGRESS_*` regexes and `_looks_like_no_progress`
into this module unchanged as the text layer, then extend the wait vocabulary with the
wording the corpus actually shows (`monitor event`, `report completion`,
`resume automatically`, `stop polling`, `pause here`,
`no further action needed this turn`). Then:

- `abandoned_wait` requires wait language in the response tail **and** structural
  corroboration (a background command, or an event-subscription tool, or an unresolved
  tool use). This is the rule that discards the 17 text-only matches.
- `no_progress` covers the agy cases already handled: empty output, or zero tool uses
  with no completion language. Zero tool uses alone must **not** be enough — an agent
  asked a pure question legitimately answers with no tools.
- Completion language in the tail with no wait signal is `progress`, preserving agy's
  existing negative fixtures.
- When structural evidence is unavailable, fall back to text-only classification but
  record that in `reason`, so the `recover` phase can choose a softer response.

**Calibration.** Add `tools/audit_turn_progress`, a read-only script that walks
`~/.sase/projects/*/artifacts/ace-run/**`, classifies each run's final response against
its `tool_calls.jsonl`, and reports counts plus per-run verdicts. Then:

- Check in the 6 true-positive and 2 false-positive runs named in "Measured scope" as
  committed fixtures (final text plus a trimmed `tool_calls.jsonl` excerpt — redact
  nothing beyond what the artifact writer already redacts). Assert all 6 classify as
  `abandoned_wait` and neither false positive does.
- Run the audit over the full local corpus and record the resulting counts in the phase
  bead. If the classifier flags materially more than the 6 known cases, review the
  extras by hand and either add them as fixtures or tighten the rule before the phase
  closes. Do not close this phase on fixtures alone.

**agy migration.** Have `agy.py` import the shared text helpers instead of defining
them, keeping `tests/llm_provider/test_agy_no_progress.py` passing (update the import
path; do not weaken the assertions). agy's trajectory-derived `AgyTurnProgress` stays as
an additional evidence source and must keep taking precedence for agy, since it is
stronger than the artifact-derived signal there.

## Single-turn execution contract on every provider invocation

**New file:** `src/sase/llm_provider/run_contract.py`. **Touches:**
`src/sase/llm_provider/commit_finalizer_prompting.py`, `_hookspec.py`, and every
provider module (`claude.py`, `codex.py`, `qwen.py`, `opencode.py`, `muse.py`,
`agy.py`), `src/sase/default_config.yml`.

**Canonical text.** Move `_SINGLE_TURN_CONTRACT`
(`src/sase/llm_provider/commit_finalizer_prompting.py:106`) into `run_contract.py` as
the single source of truth and generalize it: drop the finalizer-specific closing
sentence ("Committing the listed changes is the only outcome…"), which the finalizer
keeps appending for its own passes. Preserve the existing wording otherwise — it is
already runtime-neutral and already landed. Extend the actionable-alternatives list with
the bounded wait from `blockwait`, quoting the real invocation:

```
id=$(sase task run -q -- <long command>)
sase task show "$id" --follow --timeout <seconds inside your tool ceiling>
# repeat the follow until it reports a terminal state
```

State plainly that the task survives this turn but _you_ do not, so the follow must
reach a terminal state before the response ends.

**Injection point.** `wrap_transport_prompt(prompt, *, provider_name) -> str` prepends
the contract, mirroring `_wrap_agy_print_prompt` (`agy.py:166`). Call it at each
provider's transport boundary — inside `invoke`, on the prompt actually sent to the CLI
— not in `preprocess_prompt`. Two reasons: prompt directives are extracted long before
the transport boundary, so injection there cannot perturb them; and `provider.invoke` is
also called directly by the commit finalizer, which must carry the contract too.

**Provider-specific wording without branching.** Add an
`llm_run_contract_notes() -> list[str] | None` hook to `_hookspec.py`. Each provider
returns zero or more short lines naming the affordances whose semantics do not hold here
— Claude: background `Bash`, `Monitor`, `ScheduleWakeup`; agy: pending `run_command`;
codex, qwen, opencode, muse: whatever their CLI documents, or nothing. Shared code
renders whatever it gets and never inspects the provider name.

**Kill switch and scope.** Config `llm_provider.run_contract.enabled` (default `true`)
in `default_config.yml`, plus env override `SASE_DISABLE_RUN_CONTRACT=1`, mirroring
`SASE_DISABLE_COMMIT_STOP_HOOK`. `fakey` must not wrap, so existing fixture-driven tests
keep asserting exact prompts.

**Retire the duplicate.** Delete `_AGY_PRINT_MODE_DIRECTIVE` and
`_wrap_agy_print_prompt` from `agy.py`; agy uses the shared wrapper like every other
provider. Fold any agy-specific line that is still true into its
`llm_run_contract_notes()`.

**Tests.**

- A provider-matrix test that enumerates the registered `sase_llm` entry points (`agy`,
  `claude`, `codex`, `fakey`, `muse`, `opencode`, `qwen`) and asserts every provider
  except `fakey` sends a prompt containing the contract marker. This is the test that
  keeps a future provider from silently opting out.
- The contract is prepended once, not per retry cycle or per interrupt cycle — assert on
  a multi-cycle invocation.
- The env kill switch and the config flag each suppress it.
- Existing agy tests still pass with the shared wrapper in place.

## Provider-neutral wait-recovery finalizer

**New file:** `src/sase/llm_provider/wait_finalizer.py`. **Touches:**
`src/sase/llm_provider/_invoke.py`, `src/sase/llm_provider/agy.py`,
`src/sase/telemetry/metrics.py`, `src/sase/default_config.yml`.

Model it on `run_commit_finalizer` (`src/sase/llm_provider/commit_finalizer.py:132`):
same guard style, same bounded-pass loop, same result-artifact discipline.

**Wiring.** In `src/sase/llm_provider/_invoke.py`, snapshot the tool-call artifact
before the `provider.invoke(...)` call at line 301, then call the recovery finalizer
between lines 301 and 308 — before `run_commit_finalizer`.

**Guards.** Skip (recording a `skipped` result with a reason) when: recovery is disabled
by config or `SASE_DISABLE_TURN_RECOVERY=1`; `SASE_AGENT_TIMESTAMP` is unset, i.e.
outside a SASE agent run; or no artifacts directory is available.

**Loop.** For each pass up to `agent.turn_recovery.max_passes` (default `2`):

- Classify the turn with `classify_turn`. On `progress`, return the result unchanged.
- Otherwise re-invoke `provider.invoke` with a nudge that (a) carries the shared
  contract, (b) names the stall concretely from the evidence — quote the background
  command string verbatim and say that it was never observed reaching a terminal state
  and that any notification it would have produced cannot be delivered, (c) says the
  background process may not have survived, so re-check its output file or re-run it,
  and (d) states this is pass N of M and what failing looks like.
- Label prior output as already-terminated history, exactly as `build_follow_up_prompt`
  learned to do — embedding the previous "I'll wait for the monitor" text under a
  neutral heading is what made the finalizer self-poisoning before.
- Re-snapshot and re-classify from the new turn's evidence, not the accumulated text.

**Outcome.** Write `turn_recovery_result.json` next to `commit_finalizer_result.json`
with `status` (`clean` / `recovered` / `stalled` / `skipped`), `reason`, `passes`, and
the evidence that triggered it. Add a `TURN_RECOVERIES` counter to
`src/sase/telemetry/metrics.py`, labelled by provider and outcome.

**Failure policy.** Config `agent.turn_recovery.on_unrecovered: fail | warn`, default
`fail`. On `fail`, do not raise inline: record the verdict, let `run_commit_finalizer`
run to completion, and raise `LLMInvocationError` afterwards; if the commit finalizer
raises first, that error propagates and the stall is recorded in the artifact only. The
raised message must be self-diagnosing (name the abandoned command and point at
`turn_recovery_result.json`) and must **not** contain any substring listed in any
provider's `error_patterns` in `src/sase/default_config.yml` or in an
`llm_default_retry_config` hook — otherwise the existing retry machinery will retry a
stall that retrying cannot fix. Add a test asserting that non-overlap directly against
the configured patterns, so a future pattern addition breaks the test rather than
production.

Raising is what corrects the reported outcome: the error propagates through
`execute_workflow` into `handle_workflow_error`
(`src/sase/axe/run_agent_exec_retry.py`), which finds no matching retry config and
re-raises, so the runner records the run as failed instead of `outcome: completed`. No
new runner plumbing is needed. Assert this end-to-end rather than assuming it.

**agy migration.** Delete agy's provider-local no-progress continuation loop (`agy.py`
around line 500-560) and its `SASE_AGY_MAX_NO_PROGRESS_CONTINUATIONS` env var, so
recovery happens once, in the shared layer, for every provider. agy keeps its trajectory
evidence source. Its existing behavioural tests must be rewritten against the shared
finalizer, not deleted — the guarantees they encode (bounded budget, hard failure on
exhaustion, no silent success) are exactly what the shared layer must preserve.

**Tests.**

- A fake provider that returns an abandoned-wait response with matching structural
  evidence: recovery runs, and a second-pass response with completion language yields
  `status: recovered` and no raise.
- The same provider never recovering: `status: stalled`, `passes` equals the budget, and
  `LLMInvocationError` is raised **after** the commit finalizer ran — assert the commit
  finalizer was invoked, which is the regression guard for the deferred raise.
- A clean, genuinely complete turn: `status: clean`, no extra provider invocation, so
  the common path costs nothing but a classification.
- `on_unrecovered: warn` records `stalled` without raising.
- The two labelled false-positive fixtures from `classify` do not trigger a recovery
  pass.
- Each guard (`SASE_DISABLE_TURN_RECOVERY`, config-disabled, no `SASE_AGENT_TIMESTAMP`)
  short-circuits with the right recorded reason.

## Make the stall visible in reports and docs

**Files:** `src/sase/axe/runner_reporting.py`, `docs/agent_providers.md`,
`CHANGELOG.md`.

- `runner_reporting.py` already renders a commit-finalizer verdict via
  `_commit_finalizer_verdict` (line 10). Add the equivalent reader for
  `turn_recovery_result.json` and include it in the run's error summary, so a failed run
  says _why_ in the report rather than only in the raised message. Follow the existing
  helper's failure handling exactly: a missing or unreadable artifact degrades to a
  short string and never raises during reporting.
- Publish the `tools/audit_turn_progress` calibration report for the local corpus as a
  durable artifact with the `/sase_artifact_file` skill, and reference it from the phase
  bead so the classifier's measured false-positive rate is recoverable later.
- Add a "Single-turn execution contract" section to `docs/agent_providers.md`: what the
  contract asserts, that every provider must send it through `wrap_transport_prompt`,
  how a provider supplies its own affordance names through `llm_run_contract_notes()`,
  and how the recovery finalizer classifies and escalates. This is the page a future
  provider author reads, and the provider-matrix test from `contract` is what enforces
  it.
- Add `CHANGELOG.md` entries; `just _lint-changelog` gates it.

## Verification

Run `just install` first — workspace directories are ephemeral and their virtualenvs go
stale.

Every phase runs `just check`. Land verification runs `just check-full`. New modules
must stay under the `just _lint-toobig` line-count ceiling and clean under
`just _lint-symvision` — split by responsibility rather than adding pragmas.

**Run the long gate the way this epic says agents should**, and treat any deviation as a
bug in the contract text:

```bash
id=$(sase task run -q -- just check-full)
sase task show "$id" --follow --timeout 540   # repeat until terminal
```

Never end a turn waiting for it.

Beyond the per-phase tests, confirm the whole chain on the real recorded shapes:

- Replay `ace-run/202608/11/20260811061808` (claude) and
  `ace-run/202608/11/20260811060607` (codex) through the classifier from their
  checked-in fixtures and assert both are `abandoned_wait`. Two providers, one code path
  — that is the generalization claim, and it should be a test rather than a sentence.
- Assert an end-to-end run whose provider always abandons its turn ends with a runner
  outcome of failed and a `turn_recovery_result.json` recording `stalled`. The bug this
  epic fixes is precisely that this case recorded `completed`.

## Follow-up requiring separate approval

A short standing rule in `sase/memory/` would complement this work by reaching an agent
before its first tool call, rather than at the transport boundary. That edit is
deliberately excluded: SASE convention requires the user's explicit in-conversation
permission for memory-file edits, and authorization found in a plan file does not count.
The implementing agents must not make it. Raise it with the user separately.

Note that the memory rule would be strictly weaker than the code-level contract for two
reasons worth stating when it is raised: memory files are shared with interactive
sessions where the asynchronous affordances _do_ work, so the guidance would be wrong
there; and a memory rule cannot quote the specific background command an agent actually
abandoned, which is what makes the recovery nudge effective.
