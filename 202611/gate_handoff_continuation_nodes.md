---
tier: tale
title: Publish the starter turn's continuation node on gate handoffs
goal:
  Gated agent sessions replay each turn as a bounded agent_delta node instead of nesting
  full transcripts, so long sudo/plan/question gate chains no longer fail with
  protected_content_over_budget.
size: medium
proposed_by: bbugyi200.athena.1d7
proposed_with: claude/opus
create_time: 2026-11-17 06:20:38
status: wip
---

# Publish the starter turn's continuation node on gate handoffs

## Why

On 2026-11-17, epic phase `tailnet-x.13` (project `gh_bbugyi200__tailnet`, apollo) ran a
long sudo-gate recovery chain inside one agent session: `--plan` (monitor handoff), then
five agent turns `--1` … `--5`, each of which ended by staging a typed `sase_sudo` gate.
Each gate's follow-up turn launches with `#fork:tailnet-x.13`, so it replays the whole
session through versioned continuation replay
(`src/sase/history/chat_fork/continuation/replay.py`).

The sixth turn (`--6`, run `20261117060310`) failed after 29 seconds, before the
provider started:

```
Python step 'load_history' failed: ... ContinuationReplayRefusal:
protected_content_over_budget: opaque legacy protected content exceeds the
automatic-launch budget · node `legacy-boundary:20261117055042:d11e744c97e8f00a`
```

The user had to dismiss the session and restart `tailnet-x.13` from scratch. That lost
the session context, and launch-time healing moved the work that `--5` had carried
forward (a nine-file `nut` source fix plus tailnet edits) into rescue patches.

## Root cause

A gate handoff never publishes the starter agent turn's `agent_delta` continuation node.

- The monitor handoff path does publish it. `capture_monitor_handoff`
  (`src/sase/axe/run_agent_exec_monitor.py`) writes a `monitor_handoff` checkpoint,
  calls `persist_agent_delta(...)`, and stamps the monitor member's
  `continuation_parent_node_ids`.
- The gate path does none of this. `handle_gate_marker`
  (`src/sase/axe/run_agent_exec_gate.py`) only promotes the role suffix and stamps
  `gate_id` and `gate_member_agent_name`. `run_agent_exec_finalize.py` persists a delta
  only when `loop_outcome == "completed"`. For shell-handoff outcomes it reads an
  existing manifest, and a gated turn never wrote one.

On apollo, every surviving artifact dir of a turn that ended `shell-handoff:gated` has
the same shape: 39 of 39, across plan approvals, question gates, and sudo gates in
several projects. Each has `continuation/prepared_prompt.json` and text blobs, but no
`nodes/`, no `records/`, no `manifest.json`, and no `continuation_node_ref` in
`agent_meta.json`.

When replay reaches such a session member, `_ReplayBuilder._add_one` finds no captured
node and calls `_add_legacy_boundary`. That embeds the member's entire chat transcript
as protected text. The transcript's prompt already contains the previous fork's replay,
so each hop re-embeds every earlier one. Every `tailnet-x.13` gated turn was embedded
this way, and the transcript sizes roughly doubled each hop: 10 KB → 35 KB → 103 KB →
210 KB → 458 KB. The prompt of `--5` already contained the `--plan` node 16 times. At
`--6`, the `--5` transcript was larger than `MAX_PROTECTED_LEGACY_BYTES` (256 KiB, in
`continuation/_util.py`). `prepare_legacy_payload` recorded
`protected_content_over_budget`, and `raise_for_automatic_launch` refused the automatic
launch.

An `agent_delta` node renders only the turn's local authored request, its final
response, and its local materialized segments.
`wire_segments_from_prepared(..., local_only=True)` drops `injected_parent` history, so
publishing the node also removes the nesting.

## Changes

No sase-core change is needed: the `agent_delta` and `checkpoint` wire kinds and their
validation already exist. This change mirrors the Python monitor-handoff capture that is
already in this repo.

### 1. Publish the starter node in `handle_gate_marker`

In `src/sase/axe/run_agent_exec_gate.py`:

- Add a keyword `response: str | None = None` to
  `handle_gate_marker(gate_data, ctx, state)`.
- After the existing suffix promotion and `gate_id` / `gate_member_agent_name` stamping,
  publish a handoff checkpoint with `publish_handoff_checkpoint_best_effort(...)`.
  - Use `checkpoint_kind="gate_handoff"`.
  - Use the payload `gate_id`, `gate_kind` (from `gate_data["kind"]`),
    `member_artifacts_dir`, `member_agent_name`, `starter_agent`, and the starter's
    `chat_path` when its meta has one.
- Then call
  `persist_agent_delta_best_effort(ctx, state, status="completed", final_response=response, handoff_checkpoint_ref=<ref>)`.
- When that returns a result and the gate member dir is known, stamp the member's
  `agent_meta.json` with `continuation_parent_node_ids=[node_id]` and
  `continuation_parent_node_id=node_id`, as `capture_monitor_handoff` does. Replay then
  shows the gate member's legacy block parented on the starter's node.
- Keep it best-effort. Failures already land in `continuation/capture_errors.jsonl` via
  `record_capture_error`, and adoption must still return `"gated"`.
  - The staged path publishes the gate bundle before calling `handle_gate_marker`, so
    failing adoption at this point would strand a published gate.
  - Without a node, replay degrades to today's legacy boundary, which is not worse.
- Make re-adoption idempotent. The staged replay path
  (`run_agent_exec_staged_dispatch_replay.py`) can call adoption again. If the starter's
  meta already has `continuation_node_ref` and that manifest's `handoff_checkpoint_ref`
  is a `gate_handoff` checkpoint for the same `gate_id`, reuse it instead of writing a
  second node. Checkpoint payloads include `recorded_at_epoch`, so a naive republish
  would mint a different node id.
- Do **not** write chat history here. The staged path's real provider reply is already
  persisted, and `test_handle_gate_marker_does_not_write_synthetic_summary` must keep
  passing.

### 2. Pass the real reply to every gate adoption path

- `src/sase/axe/run_agent_exec_staged_gate.py::handle_staged_gate_marker`: stop
  `del provider_response` and pass it as `response=`. Update the docstring: the reply is
  recorded as the continuation delta's final response and is never overwritten by a
  summary.
- `src/sase/axe/run_agent_exec_plan.py` (`handle_gate_marker` call after
  `_save_planner_chat`): pass `response=planner_response`.
- `src/sase/axe/run_agent_exec_questions.py` (`handle_gate_marker` call after
  `_save_questions_chat`): pass `response=provider_response`.

### 3. Make the over-budget refusal say what went wrong

In `src/sase/history/chat_fork/continuation/_render.py::prepare_legacy_payload`, include
three things in the `protected_content_over_budget` omission reason:

- the source label, e.g. ``agent session `tailnet-x.13` member `tailnet-x.13--5` ``;
- the protected byte size against the `MAX_PROTECTED_LEGACY_BYTES` bound;
- that the source had no recoverable continuation node.

The `--6` failure only showed an opaque node hash, so finding the cause meant
reconstructing it from logs. Keep the omission `kind` unchanged, because refusal
classification keys on it.

## Tests

- `tests/test_gate_staged_adoption.py`: staged gate adoption writes the starter's
  `continuation/manifest.json` and `nodes/agent-delta:*.json`.
  - The delta's `final_response_ref` text equals the provider response.
  - `handoff_checkpoint_ref` points at a `gate_handoff` checkpoint carrying the gate id.
  - The gate member's meta gains `continuation_parent_node_ids`.
  - No synthetic chat summary is written.
  - A second adoption call reuses the same node id.
- `tests/test_axe_run_agent_exec_plan_gate_turn.py` and
  `tests/test_axe_run_agent_exec_questions_gate_turn.py`: after a gated handoff, the
  starter's `agent_meta.json` has `continuation_node_ref`.
- Best-effort: when `persist_agent_delta` raises, `handle_gate_marker` still returns
  `"gated"` and `continuation/capture_errors.jsonl` records the error.
- Replay regression (`tests/history/test_continuation_replay.py` or a sibling file).
  Build an agent-session fork source whose members are a gated starter with a published
  node, its gate member (small transcript, no node), and a follow-up turn. Give the
  starter's chat transcript a nested `# Previous Continuation` section and a large body.
  Assert that:
  - the starter renders as `agent_delta`, not `legacy_boundary`;
  - its transcript body does not appear in the rendered replay;
  - `refusals` is empty.
- Refusal message: a session member with no node and a transcript over 256 KiB still
  yields `protected_content_over_budget`, and the message names the member label and
  byte size.

Verify with `sase tool run check`.

## Out of scope / follow-ups

- Gate-turn members (`<session>--gate`, `--gate_0`, …) still have no node of their own.
  They keep rendering as opaque legacy boundaries that embed their small, non-nesting
  gate transcripts. A first-class bounded gate-result node would remove that redundancy,
  since the follow-up prompt already carries the gate outcome, but nothing breaks
  without it.
- Recovery UX when an automatic follow-up launch is refused. The turn fails with a
  traceback and holds the workspace. Restarting then evicts the carried dirty repos into
  rescue patches, and the new agent is not told about them.
- Sessions already mid-chain when this lands keep their existing legacy members, but
  stop nesting from the next gated turn on.
