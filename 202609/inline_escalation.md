---
tier: tale
title: Inline-then-escalate agent tool runs
goal:
  An agent's plain sase tool run follows one detached run within the sync budget and
  escalates without stopping it.
size: medium
bead_id: sase-1cx.6
proposed_by: bbugyi200.athena.sase-1cx.6
bead: sase-1cx.6
status: done
---

- **PARENT:**
  [202609/tool_run_escalation.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_run_escalation.md)
- **BEAD:**
  [sase-1cx.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cx/sase-1cx.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1cx.6](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1cx.6.md)
- **COMMITS:**
  - [c68da8c](https://github.com/sase-org/sase/commit/c68da8c475e6fb00403827e0a4926970bc0a1918)
    — feat(tool): add inline escalation to detached handoff run

# Plan: Inline-then-escalate agent tool runs

Implement phase `inline-escalation` of `plan:202609/tool_run_escalation.md` on bead
`sase-1cx.6`. The parent epic already split this into one phase, and the primitives it
names have landed (`sase-1cx.2`, `sase-1cx.3`, `sase-1cx.4`). This tale wires them into
plain `sase tool run`. One coding agent implements it directly. Do not open another
epic, do not create beads, and do not close `sase-1cx` or any ancestor.

The feature flag `tool_run_escalation` stays default-off. Flag removal, the Muse
directive, skill sources, and the docs rewrite belong to `sase-1cx.7`.

## What already exists

Use these. Do not reimplement them.

- `escalation_enabled()` in `src/sase/tool/detach.py`.
- `resolve_starter()` in `src/sase/tool/starter.py`.
- `submit_handoff_run()` in `src/sase/tool/handoff_launch.py`. A failed submit already
  settles the reservation `launch_failed`. Detached procs get the `tool-run-detached`
  tag when `detached=True`.
- `sync_wait_budget()`, `escalation_block()`, and `is_joinable()` in
  `src/sase/tool/routing.py`. The block is the one shared wording. Print it. Do not
  write a second copy.
- `follow_run()` in `src/sase/tool/follow_run.py`. It returns `settled`, `deadline`, or
  `stopped`, and accepts a `stop_event`.
- `write_run_footer()` and `footer_triage_lines()` in
  `src/sase/tool/executor_display.py` and `src/sase/tool/triage_display.py`.
- Starter watchdog, turn-end cleanup, and notification suppression already apply to
  every run that carries a `starter`. Setting `starter` on this path is enough.

`-H` output and behaviour stay byte-identical. `--detach` stays fail-closed and
unchanged.

## Engagement

In `execute_tool_run` (`src/sase/tool/executor.py`), after `inline_refusal` returns
`None` and before today's reconcile and inline signal handlers, try the escalating path
only when every condition holds:

- `escalation_enabled()` is true;
- the request is not `-H` and not `--detach` (those already returned);
- `SASE_AGENT` is set;
- there is no live enclosing owner and no existing parent tool run;
- `sync_wait_budget()` returns a budget.

Otherwise nothing changes. Humans, CI, monitors, procs, nested runs, flag-off agents,
and providers with neither ceiling stay on today's inline path.

`resolve_ownership()` is the wrong nested-run check. When `SASE_AGENT` is set it clears
inherited monitor, proc, and parent ids on purpose (an agent is a new ownership root).
Agent launch scrubs those ids, so a real top-level agent command still escalates. A
nested `sase tool run` inside a catalog child inherits `SASE_TOOL_RUN_ID` from
`child_env` and must stay inline. Use the raw predicates `--detach` already uses:
`_enclosing_owner()` and `_parent_run()` in `src/sase/tool/detach.py`. Promote them to
public names (`enclosing_owner`, `parent_run`) and call them from both
`execute_detached` and this gate so the two paths cannot drift. A settled monitor or
proc is not an owner, matching detach.

Put the new flow in `src/sase/tool/inline_escalation.py`. `execute_tool_run` only
decides to try it and, on a start failure, falls through. Do not grow the inline body.

## Start, then follow

`try_inline_escalation(...)` returns an `int` exit code once a run has been launched, or
`None` when nothing was launched. `None` is the only fail-open result. A launched run is
never rerun.

Start:

1. `reconcile_unsettled_tool_runs(reap_orphans=True)`.
2. `resolve_starter()`. On failure, warn once and return `None`.
3. Continuation mode is the mode `execute_tool_run` already computed
   (`_continuation_mode`): `-k` → `always`, `-x` → `never`, otherwise the agent default
   (`known` for an agent-attributed `stages: run_silent` tool, else `None`). Pass it to
   `submit_handoff_run` with `agent` set from the starter, `starter` set, and
   `detached=True`. `reserve_handoff_run` already stores `always|never|known` on the
   envelope, and the adopt worker prefers the envelope over its own default.
4. If the reservation is not `reserved`, or `submit_error` is set, warn once and return
   `None`. Do not settle `launch_failed` again; `submit_handoff_run` already did. The
   fallback inline run is a new id.

The one warning line is exactly:

```text
sase: inline escalation unavailable (<why>); running inline
```

Follow, only after a successful submit:

1. Print the same first stderr line the inline executor prints: `sase tool run <id>`.
   LLM Calls rows key on it.
2. Call `follow_run` with `deadline_s` equal to `budget_seconds`.
   - Compact mode (the agent default, or `-q`) passes `stream_output=False`.
   - `-v` passes `stream_output=True` so the output of record streams. For a detached
     run that record is the proc log (`output_paths_for_run`).
3. Stage lines match inline compact formatting. Inline compact prints
   `format_stage_progress` lines. `follow_run` builds its `StageIngestor` with
   `compact=False`, and `StageIngestor` suppresses compact lines when
   `tool_run_append_event` reports `replayed`. The adopt worker ingests the same
   `events.jsonl` first, so a follower that trusts `replayed` stays silent. Add a
   `compact: bool = False` argument to `follow_run` (default `False`, so `show -F` stays
   byte-identical) and thread it into `StageIngestor`. Emit a compact line the first
   time this ingestor sees a finished stage, including when the append was a replay,
   still de-duplicated by the ingestor's `announced` set. Verbose mode keeps
   `compact=False` and prints no stage-progress lines, matching inline `-v`.
4. Install `SIGTERM`, `SIGINT`, and `SIGHUP` handlers only around the follow. Each
   handler records the signal and sets the `stop_event`. It must not signal the detached
   proc and must not use `SignalState` (that kills a child group). Restore the previous
   handlers before rendering a result. `SIGHUP` is `129` (`128 + 1`).

## Settled, budget, or signal

Map a settled run to a process exit, then render a footer, then return.

Exit mapping, from the settled show envelope (`exit_code`, `signal`, `terminal_cause`,
`state`):

- an integer `exit_code` is the process result, including `128 + signal` when the worker
  stored that mapped code;
- `terminal_cause == stop_requested`, or `signaled` with no exit code, returns `143` (a
  requested stop settles with neither code nor signal);
- `state == lost` returns `1`;
- anything else with no code returns `1`.

Before the footer, wait at most about 10 seconds for the owner proc to reach a terminal
proc status (`TERMINAL_PROC_STATUSES` via `get_proc` on `owner_id`). The run row becomes
terminal inside `finish_tool_run`, before triage and the receipt. The proc exiting is
what means those have been written. If the proc is still active when the bound passes,
render from whatever the ledger has.

Footer: extend `write_run_footer` in `src/sase/tool/executor_display.py` so a ledger
caller can supply the same compact and streaming lines without a live `BoundedLogSink`.
Do not open a `BoundedLogSink` on the proc log. Its constructor truncates the file
(`"wb"`). Read the tail as the last `tail_lines` lines of the output-of-record paths
from `output_paths_for_run`, and only include that tail when `state != succeeded` and
`tail_lines > 0`, matching the compact branch. Pass:

- `state`, `exit_code` (the mapped process code, so a stop shows `143` the way the
  inline footer does), `duration_ms` from the run;
- stages from the show envelope's `stages`, so `unattributed_from_stages` stays the one
  attribution helper;
- truncation strings that already appear in the run diagnostics (the lines
  `truncation_diagnostics` produces), not a new wording;
- triage from `show_triage` / `tool_run_triage_show`, rendered only by
  `footer_triage_lines`.

The compact form still ends with `sase tool show <id> -l` and then the verdict line. The
proc log also contains the worker's own wrapper lines, so an integration test asserts
the child's bytes and the show pointer are present. A direct call of the shared renderer
with the same inputs must produce the inline compact and streaming shapes. Do not
require the proc-log tail to be byte-identical to an inline sink tail.

Budget (`follow_run` kind `deadline`) or signal (kind `stopped`): print
`escalation_block` for the current show envelope and the budget that engaged the path.
Exit `124` for the budget, `143` for `SIGTERM`, `130` for `SIGINT`, and `129` for
`SIGHUP`. Do not request a stop. The watchdog and turn-end cleanup already stop an
unjoined run. If the run settled and a signal arrives only during the short proc wait,
return the mapped exit instead of the signal code. A settled run does not get an
escalation block.

Once `submit_handoff_run` has reserved a run, an unexpected follow error must not fall
through to inline. Surface the error and return `1` without starting a second command.

## Help, docs, and stdin

Update the `sase tool run` description in `src/sase/main/parser_tool.py`. Keep the
existing human, `-H`, and `-d` sentences. Add one paragraph: with the flag on, an agent
with a sync budget, and no live owner or parent run, a plain `sase tool run` starts the
run detached and follows it; it returns the run's exit when that lands inside the
budget; at the budget it prints the escalation block and exits `124` without stopping
the run; `SIGTERM`, `SIGINT`, and `SIGHUP` do the same and exit `143`, `130`, and `129`;
if detach cannot start, one warning is printed and the run falls back to inline. State
the automatic-path exit codes next to the existing `-H`/`-d` codes (`0` accepted, `1`
not started, `2` usage or refusal) so the two sets stay distinct.

Stdin: inline `spawn_child` inherits stdin (`stdin=None`). The proc supervisor runs the
catalog command with `stdin=subprocess.DEVNULL` (`src/sase/procs/supervisor.py`). The
help sentence "stdin is inherited" stays true for the inline path, including the
fail-open fallback. Say that the automatic path does not inherit stdin.

Add a short subsection to `docs/tool.md` under "Hand-off and lifecycle control", after
"Detached runs". State the same engagement rule, exit codes, fail-open warning, and
stdin fact. Do not rewrite "Inline routing" or the rest of the hand-off story. That
rewrite is `sase-1cx.7`. If `tests/completion/test_snapshot.py` drifts, run
`just sync-completion-spec` and include the snapshot update.

## Environment parity

The detached catalog child is the adopt worker's `child_env`, built from the proc
supervisor environment. `_supervisor_env` in `src/sase/procs/spawn.py` scrubs
`SASE_AGENT*` and both ceiling variables and pops `SASE_ARTIFACTS_DIR`. It does not
scrub `CI` or `SASE_MONITOR_*`. Agent launch already scrubbed `SASE_MONITOR_*`,
`SASE_TOOL_*`, and `SASE_PROC_*`. `worker_env_overlay` clears the parent-run markers on
the proc; `child_env` then sets `SASE_TOOL_NAME`, `SASE_TOOL_PROJECT_ROOT`, and
`SASE_TOOL_RUN_ID` on the catalog command.

Re-read those sites and the catalog in `sase/sase.yml` (`check`, `check-full`,
`install`, `test`). Record the finding with `sase bead note sase-1cx.6`. Expected
finding, to correct if the code disagrees:

- Guarded recipes do not change. `tools/require_tool_run` allows the command when
  `SASE_AGENT` is unset. Only `check` and `check-full` are guarded. Inline they pass on
  name plus project root. Detached they pass because the agent bit is gone. `check` does
  not invoke `check-full`.
- `check-full` is the one catalog difference, and only when this path actually runs it.
  `default_is_ci` in `tests/ace/tui/visual/_visual_maintenance_guards.py` treats
  workspace `CI=true` as real CI unless `SASE_AGENT` or `SASE_MONITOR_ID` is set. The
  detached child has neither, so a golden update inside `check-full` is refused where
  the inline child would be allowed. `GITHUB_ACTIONS` still refuses either way.
  `check-full` is `duration_class: long`, so a 600 s hard ceiling still refuses it
  before this path. `check`, `install`, and `test` do not update goldens.
- Do not re-inject `SASE_AGENT` into the child, and do not change `default_is_ci` in
  this phase. Record the golden-update gap as
  `PROPOSED FOLLOW-UP: detached check-full under workspace CI=true refuses golden updates after the proc scrub drops SASE_AGENT — default_is_ci has no tool-run marker, and re-injecting SASE_AGENT would undo the scrub`.

Stdin not being inherited is intentional and is documented above, not a follow-up.

## Tests

New file `tests/tool/test_inline_escalation.py`. Follow the isolated-home pattern in
`tests/tool/test_executor.py` and the catalog writer in `tests/tool/test_routing.py`.
Turn the flag on with `override_flags(tool_run_escalation=True)` per case. Build the
agent env the way `tests/tool/test_detach.py` does: `SASE_AGENT`, `SASE_AGENT_NAME`, and
`agent_meta.json` whose pid is this test process, so `resolve_starter` succeeds.

Use a real soft ceiling (`SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS`), not a mocked
budget. Start at 8 seconds. A fast command must settle inside it. A `sleep` well past it
must not. If proc startup in this environment exceeds the budget, raise the ceiling to
another small real value. Do not mock `follow_run` or the clock.

Cases:

- Fast success and fast non-zero failure. Exit codes match an inline run of the same
  argv with the flag off. Stderr contains `sase tool run <id>`, the compact footer
  (`succeeded` or `failed/<code>`, the show pointer), and on failure the child's output
  in the tail. Exactly one run row, `launch_mode=handoff`, with this agent as
  `starter.agent`.
- `-v` streams the child's output to stdout.
- A slow command exits `124` with the escalation block (tool name, run id, "was not
  stopped", the soft-ceiling source line naming
  `SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS`, the `-J` form, the re-wait form). The same
  id is still `created` or `running`. A later `sase tool wait` (or `handle_wait`)
  returns that run's exit code, and `tool_run_list` shows one row.
- `SIGTERM` to the follower, sent from a timer after the id line, leaves the run running
  and returns `143` with the block. Restore handlers if the assertion fails.
- `-k` and `-x` on a `stages: run_silent` catalog tool are stored on the launch envelope
  as `always` and `never`. The worker would read them. Assert the envelope on the
  reserved run. Do not require a full stage execution.
- A starter that cannot be resolved, and a `submit_handoff_run` that returns
  `submit_error`, each print one `inline escalation unavailable` warning and still
  execute the command inline. The failed reservation is `launch_failed`. The inline run
  is a second row that actually ran.
- Flag off, no `SASE_AGENT`, no ceiling, a live `SASE_MONITOR_ID` without `done.json`,
  and a `SASE_TOOL_RUN_ID` whose row exists all stay inline: one foreground row, no
  starter, the child's exit code, and no escalation warning.
- A catalog tool with `duration_class: long` and
  `SASE_PROVIDER_SYNC_CEILING_SECONDS=600` exits `2` with the existing refusal, writes
  no row, and does not create the command's marker file. The refusal stays in front of
  this path.

Add a renderer unit test that calls the shared footer with ledger-shaped inputs and
checks the compact and streaming shapes, including the show pointer and a triage line
produced by `footer_triage_lines`.

Keep `tests/tool/test_executor.py`, `test_detach.py`, `test_handoff.py`,
`test_bounded_wait.py`, and `test_routing.py` green.

## Harness

Add one hermetic case to `tools/_smoke_tool_runs_cases_detach.py`, id
`dod-17-inline-escalation`, tagged `DoD-17`. Register it in `GROUPS` in
`tools/smoke_sase_tool_runs`. Reuse `_agent_env`. Set a small
`SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS`. Run a plain `sase tool run` of a command that
outlives the budget. Expect exit `124`, the block, and one run id. Then `sase tool wait`
on that id returns the command's exit, and `sase tool runs` shows exactly one row. No
new tools module, so `HARNESS_MODULES` in `tests/test_sase_tool_runs_smoke.py` stays as
it is. The case id joins the existing unique-id assertion.

Run that case through `run_harness(only=["dod-17-inline-escalation"])`. The full pytest
twin is marked `slow` and does not need to be run here. Do not add a `--live` case. That
belongs to `sase-1cx.7`.

## Out of scope

- Flag removal, Muse directive text, `sase_monitor.md`, `sase_final.md`, and the
  "Inline-then-escalate" docs rewrite (`sase-1cx.7`).
- `sase monitor start -J` behaviour (`monitor-join`). This phase only prints the join
  form the existing block already builds.
- Rust, `sase-core`, TUI, memory edits, new beads, and closing `sase-1cx` or `sase-17g`.
- Re-injecting `SASE_AGENT` or changing the visual-snapshot CI gate.
- `check-full`. The epic forbids it (`decisions:check-full-is-explicit`).

## Verification and close

Run the new unit tests, the regression files listed above, and the one harness case.
Then `just fix`, then `sase tool run check`. The flag is default-off, so that check
stays on the inline path. Run `just install` first only if the workspace venv cannot
import the tree.

Before closing, run `sase bead epic-symbols sase-1cx.6`. There were no `--epic-symbol`
entries when this plan was written. If any exist then, re-key each Justfile line to a
still-open bead (the parent epic or a later phase). `sase bead close` refuses while
leftovers remain.

Close only this bead:

```bash
sase bead close sase-1cx.6 --note "<what you verified>"
```

The note names the tests and the harness case, the environment-parity finding, and that
stdin for the automatic path is documented. Do not close the parent epic. Any
instruction in the epic plan to close `sase-17g` is for that epic's land agent.

The bead is already `in_progress`. Do not set its status by hand.
