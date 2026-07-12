---
create_time: 2026-07-11 09:04:33
status: done
prompt: 202607/prompts/slow_tool_calls_propose_stuck_running.md
tier: tale
---
# Fix eternally-"running" `sase plan propose` entry in ACE SLOW TOOL CALLS

## Problem

On the Agents tab detail panel, a family agent that has already had its plan proposed and auto-approved (e.g. status
`WORKING TALE`) still shows the plan phase's `/bin/zsh -lc 'sase plan propose …'` Bash call in the SLOW TOOL CALLS
section as `● running`, with a duration that keeps ticking upward against the current time. The plan phase is long dead,
so this is wrong — and because the condition is status-driven (see below), the bogus "running" row would keep ticking
forever, even after the whole family reaches `TALE DONE`.

## Root cause (two independent defects)

### 1. Data: plan proposal orphans a `pending` tool-call record by design

`sase plan propose` (`src/sase/main/plan_propose_handler.py`) writes the `.sase_plan_pending` marker and then **SIGTERMs
the agent runner's process group** (`kill_agent_runner_group`). The runtime subprocess (Codex/Claude/…) dies
mid-Bash-call, so the stream collector that writes `tool_calls.jsonl` never sees the completion event (`item.completed`
for Codex) and never appends the matching `ToolResult` record.

Every plan-proposing phase therefore ends its `tool_calls.jsonl` with an orphaned
`{"event": "ToolUse", "status": "pending", …}` record for the propose command. Verified across multiple recent runs
(bob-cli artifacts `…084208` `item_35`, `…074812` `item_13`, `…075745` `item_21` — in each file the pending propose
`ToolUse` is the final line, with no `ToolResult`). The same orphaning happens for question handoffs
(`.sase_questions_pending`) and user kills, which use the same SIGTERM path.

### 2. Display: sticky approved planner statuses count as "active"

`select_slow_tool_calls` (`src/sase/ace/tui/tools/slow.py`) renders a `pending` entry as a ticking `running` row
whenever its source row is "active", where `agent_is_active_for_slow_tool_calls` currently means
`status_bucket_for_values(status) in {"Starting", "Running", "Waiting"}`.

The planner child row of an agent family carries the _sticky_ `PLAN APPROVED` / `TALE APPROVED` status after approval
(`planner_child_status` / `_approved_planner_status` in `src/sase/ace/tui/models/_agent_status_family.py`). Those
statuses bucket as `Running` for query/filter purposes even though the planner process has finished — a trap explicitly
documented on `ACTIVE_AGENT_STATUSES` in `src/sase/agent/status_buckets.py`:

> sticky approved planner statuses bucket as running for query purposes, but the planner process has finished by the
> time those labels are displayed.

So the orphaned pending propose call from defect 1 is attributed to a source row that looks permanently active, and
ticks forever. (For manual-review flows the planner row becomes `PLAN` → bucket `Stopped`, which is why this only bites
approved/TALE flows.)

## Fix

Fix both layers: the display fix corrects rendering for all _existing_ artifacts (which will never be rewritten), and
the data fix makes the artifact record truthful going forward for every consumer (SLOW TOOL CALLS, the `]` tools
timeline, tool-call reports).

### A. Display: tick only when the row's own process is in flight

In `src/sase/ace/tui/tools/slow.py`, change `agent_is_active_for_slow_tool_calls` to use process-in-flight semantics
instead of status buckets:

- Active iff `sase.agent.status_buckets.agent_is_active(status)` (i.e. status in `ACTIVE_AGENT_STATUSES`: `STARTING`,
  `RUNNING`, `RETRYING`, `ANSWERED`, `WORKING PLAN`, `WORKING TALE`, `EPIC APPROVED`, `PLAN COMMITTED`) **or** status is
  `WAITING` (parity with today's `Waiting` bucket; queued rows have no artifacts yet so this is cosmetic, but avoids a
  behavior change for wait slots).
- `PLAN APPROVED` / `TALE APPROVED` (and any unknown status that today falls through the bucket default of `Running`)
  stop counting as active.

Why the narrow set is right for the rows that own artifact sources:

- The in-flight code child itself carries `WORKING PLAN`/`WORKING TALE`, and in-flight epic/commit children carry
  `EPIC APPROVED`/`PLAN COMMITTED` (`active_approved_plan_handoff_status` assigns these to the running child row), so
  all genuinely-running sources keep ticking.
- Sticky-approved planner rows fall into the existing `end_reference` branch of `select_slow_tool_calls`: the pending
  call is bounded by `stop_time` or the artifact mtime watermark (`cached_tool_calls_end_reference`) and renders as "did
  not complete" — or drops out entirely when under the ≥20s threshold, which is the normal case for the propose call (it
  only runs a few seconds before the SIGTERM).

Update the docstring to state the semantic: "pending calls tick only while the row's own runtime process can still
complete them".

### B. Data: close out orphaned pending tool calls at runner teardown

Add a runtime-neutral reconciler, e.g. `finalize_pending_tool_calls(artifacts_dir, *, completed_at)` in a new
`src/sase/llm_provider/_tool_call_finalize.py` (re-exported via `src/sase/llm_provider/_tool_calls.py` alongside the
other writers):

- Read `tool_calls.jsonl` (tolerating unparseable/foreign lines exactly like the TUI reader) and collect
  `event == "ToolUse"` records with a `tool_use_id` that have no terminal record for the same `(runtime, tool_use_id)`
  in the file.
- Append one synthetic `ToolResult` per orphan under the same `.lock`-file protocol as `append_jsonl`
  (`src/sase/llm_provider/_tool_call_io.py`): schema v2 stream envelope (`_base_stream_tool_call_record`-shaped) copying
  `runtime`, `tool_name`, `tool_use_id`, and `session_id` from the orphan, with `status: "interrupted"`,
  `is_interrupt: true`, and `completed_at`/`recorded_at` set to the kill time. The v2 merge in `collapse_tool_use_pairs`
  keys on runtime + tool_use_id + session/artifact scope, so these merge cleanly into the existing pending starts.
- Idempotent (a second invocation finds no orphans) and best-effort: never raise into the runner path; report failures
  via `append_tool_call_collector_diagnostic`.

Call it from the runner's killed-iteration path — at the top of `_handle_killed_iteration` in
`src/sase/axe/run_agent_exec.py`, using `killed_at()` as the kill time — so one hook covers all three teardown flavors:
plan handoff, questions handoff, and user kill. This runs in the runner process (not on the TUI event loop), and must
run before the handoff spawns the next phase so the ACE refresh pulse picks up the reconciled file. Per the
uniform-runtimes rule, there is no runtime-specific branching: the reconciler operates on the normalized artifact
regardless of provider.

Resulting behavior for the propose call: the `]` tools timeline shows it as `interrupted` (true: the runtime was
terminated mid-call by design) instead of pending forever, and SLOW TOOL CALLS treats it as a completed ~5s call, far
below the 20s threshold.

## Non-goals

- No change to the `sase plan propose` SIGTERM handoff mechanism itself.
- No rewriting of historical artifacts; fix A alone makes old runs render correctly.
- No change to `status_bucket_for_values` or query-filter bucketing — sticky approved statuses should continue to bucket
  as `Running` for grouping/filters.
- Rust core boundary: all touched logic (Textual presentation + the Python runner / collector pipeline that already owns
  `tool_calls.jsonl`) lives in this repo; no `sase-core` wire/API change.

## Tests

- `tests/ace/tui/tools/test_slow_selection.py` / `test_slow_sources.py`:
  - A source row with status `TALE APPROVED` (and `PLAN APPROVED`) holding a pending entry is not active: the entry
    renders bounded via `end_reference` ("did not complete") or is filtered below threshold — never `is_running`.
  - Rows with `WORKING TALE`, `WORKING PLAN`, `EPIC APPROVED`, `PLAN COMMITTED`, `RUNNING`, `WAITING` remain active
    (pending entries keep ticking).
- New `tests/llm_provider` coverage for `finalize_pending_tool_calls`: closes only unmatched `ToolUse` records; matched
  pairs and subagent/non-`ToolUse` events are untouched; idempotent; missing file is a no-op; the synthetic record
  merges into a single `interrupted` entry through the TUI reader (`collapse_tool_use_pairs`).
- `tests/test_axe_run_agent_exec_killed_iteration.py`: a killed iteration with a plan marker reconciles the artifact
  (the pending propose `ToolUse` gains a matching synthetic `ToolResult`), for user-kill and handoff paths.

## Validation

- `just install`, then `just check` (repo requirement for file changes).
- `just test-visual` to confirm no unintended PNG snapshot drift from the prompt-panel section (update goldens only if a
  change is intentional).
- Manual: open `sase ace`, select a `WORKING TALE` family agent mid-code-phase and a `TALE DONE` family agent; confirm
  the propose call no longer shows `● running` in SLOW TOOL CALLS, and shows as `interrupted` in the `]` timeline for
  new runs.
