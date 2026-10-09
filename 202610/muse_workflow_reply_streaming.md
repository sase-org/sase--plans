---
tier: tale
title: Restore live Reply updates for workflow-backed Muse agents
goal:
  Show streamed assistant text in active Muse Reply cards before completion across
  workflow and session row shapes.
size: medium
proposed_by: bbugyi200.apollo.research.0r.linker.w0
create_time: 2026-10-09 18:49:31
status: wip
---

# Restore live Reply updates for workflow-backed Muse agents

## Outcome

When an active Muse agent writes assistant text to `live_reply.md`, its selected Reply
card must display and extend that text before the provider exits or the agent becomes
DONE. This must work for workflow-backed agent rows, workflow agent steps, and the
current agent turn shown inside a sequential session, as well as the existing RUNNING
row shape. Watcher delivery and the existing polling backstop must both work without
selection changes, pending tools, or full agent reloads.

This is a medium tale: one coding agent can make the small presentation fix and add the
loader-to-mounted-TUI regressions needed to prove it. There are no separate
implementation phases or dependencies on another repository.

## Evidence and root cause

Read the following artifact through `sase artifact read` for the original investigation
and its supporting observations:

`research:202610/muse_reply_card_follow_skips_workflow_agents/muse_reply_card_follow_skips_workflow_agents.md`

The planning investigation confirmed these facts in the current checkout:

- `src/sase/ace/tui/widgets/prompt_panel/_live_reply_follow.py:is_live_reply_agent`
  requires `agent.agent_type == AgentType.RUNNING` before checking in-flight state.
- `_loaders/_workflow_loaders.py:load_workflow_agents` constructs WORKFLOW rows with
  `appears_as_agent=True` for workflow-wrapped agents. Both the filesystem and snapshot
  workflow step loaders construct WORKFLOW agent steps.
- A read-only probe of the current code reports `is_agent_entry=True`,
  `agent_row_is_in_flight=True`, and `is_live_reply_agent=False` for both a running
  workflow-backed agent and a running workflow agent step. A plain RUNNING row passes,
  while bash steps and ordinary workflow containers correctly fail.
- The eligibility function controls both insertion of the live-reply region and
  selection of its follow source. The three rendering uses are the ordinary and
  legacy-followup branches in `_agent_display_render.py` and the session branch in
  `_agent_display_agent_session_render.py`. Without a region and source, watcher events
  and `maybe_probe_live_reply_drift` cannot update the card.
- `tests/ace/tui/test_live_reply_follow_mounted.py` exercises real Muse JSONL parsing
  and a mounted Reply card, but its `_agent` fixture constructs only a RUNNING row. It
  therefore misses the production representation.
- `_subprocess_muse.py` already routes `run.output.delta` to `append_stream_delta`;
  `_subprocess_artifacts.py` flushes each live-reply write. The report observes reply
  bytes on disk well before completion. The eligibility defect accounts for that delay
  without a transport change.

Muse can legitimately emit no assistant text while executing tools. The repaired card
should show the existing placeholder until text is actually written, then show the reply
during the remaining provider and host-finalization window.

## Implementation

### 1. Correct the shared presentation eligibility check

In `_live_reply_follow.py`, use the existing `Agent.is_agent_entry` property plus
`agent_row_is_in_flight(agent)` instead of the RUNNING enum restriction:

```python
def is_live_reply_agent(agent: Agent) -> bool:
    if not agent.is_agent_entry or not agent_row_is_in_flight(agent):
        return False
    return not (agent.is_workflow_child and agent.step_type in ("bash", "python"))
```

Keep the explicit non-agent step exclusion and remove the `AgentType` import if it
becomes unused. Reuse this one predicate at all existing call sites. Preserve the
current selection of a concrete session turn and its artifact directory; the session
container must follow the current turn's files and replace only that turn's region. Do
not broaden `Agent.is_agent_entry` or redefine active statuses.

This is Textual presentation eligibility, so it belongs in this repository and needs no
Rust wire change or `sase-core-revision.txt` update. Apply it for all providers. It
repairs intended behavior in a single completed change, with no deprecation or
compatibility migration, so no feature flag is warranted under `sase_flags.md`.

### 2. Cover eligibility and production loader representations

Extend `tests/ace/tui/widgets/test_live_reply_follow.py`, or add a focused sibling if
that keeps the tests manageable. Build a table that includes:

- Eligible: active RUNNING agent, active WORKFLOW with `appears_as_agent=True`, and
  active WORKFLOW child with `step_type="agent"`.
- Ineligible: ordinary workflow aggregate, bash/python steps, clan container, monitor,
  gate, named proc, terminal agent rows, and rows with a stop time even if their status
  still says RUNNING.
- Source selection: a sequential session with older completed turns and a current
  WORKFLOW agent turn resolves to the current turn; a session whose current turn is a
  monitor or gate does not attach an assistant-reply follower.

Use actual model fields for these shapes, not a stubbed `is_agent_entry` result. Retain
provider independence, for example by including a non-Muse eligible case.

Add a temporary on-disk workflow fixture with `agent_meta.json`, `workflow_state.json`,
and `prompt_step_main.json`, as applicable. Load the root and agent step through
production loader functions, with directory discovery restricted to that fixture. Cover
anonymous and named/VCS workflow metadata. Assert the loader returns WORKFLOW rows with
the relevant agent/child flags and that those rows are eligible. Existing patterns are
in `tests/ace/tui/widgets/test_agent_list_runtime_loader.py` and
`tests/ace/tui/_retry_agent_session_loader_fixture.py`.

Supply valid prompt files and explicit artifact paths for root and step views; workflow
child prompt lookup uses the step-specific filename. Keep all fixture reads and writes
isolated from the real SASE home and this agent's artifacts. At least the mounted
workflow regression below must consume a row returned by a production loader, not a
manually relabeled RUNNING row.

### 3. Prove that the mounted card updates before completion

Extend `tests/ace/tui/test_live_reply_follow_mounted.py` around its existing gated fake
Muse subprocess and real `stream_and_parse_muse_json_output` parser. Cover the existing
RUNNING case, loader-created workflow root and agent-step cases, and a session container
displaying an active WORKFLOW turn. Reuse fixtures and helpers so this remains one
comprehensible regression path.

For each relevant display shape:

1. Mount the selected agent with empty reply files. Assert that a live source is
   configured for the right concrete identity and artifact directory, and that the Reply
   card contains the corresponding replaceable region.
2. Emit the first batch of deltas while the fake provider is gated before its next batch
   and terminal record. Deliver the real artifact-event route and assert the rendered
   Reply card contains the partial text while the subprocess is still alive, with no
   terminal record or `done.json`.
3. Emit a second batch without a watcher event. Exercise the existing stat-only polling
   backstop and assert the text grows while the terminal gate remains closed. Use
   bounded condition waits that report the screen on failure.
4. Assert that the stream did not schedule a full Agents reload or dirty the roster, and
   that fragments concatenate without duplicate text or timestamp dividers. For a
   session, older reply blocks, block identity, and Context must remain intact while
   only the current turn grows.
5. Release the provider and render the terminal state. Assert the final reply is present
   exactly once and following stops for the completed concrete turn.

Keep the agent free of pending tool calls during the streaming assertions so a slow-tool
refresh cannot mask the bug. Always point `SASE_ARTIFACTS_DIR` at the test directory
before invoking the parser. Preserve subprocess cleanup in `finally`. Demonstrate the
workflow regression fails with the original enum gate and passes with the fix; the
failure must concern source setup or visible text, not a malformed fixture.

Keep existing cancellation, generation/selection revalidation, attempt pinning, and
hint-mode guards intact. Reuse any relevant existing tests; add focused checks where
needed to show that a newly eligible workflow source cannot publish after selection or
attempt context changes.

## Verification and completion criteria

Before implementation, read `tui.md`, `tui_perf.md`, `lint_and_test.md`, and
`tui_screenshot.md` through the audited memory reader. Use the workspace virtualenv and
established test tooling. Run focused regressions for live reply following, mounted Muse
streaming, session reply blocks, and Muse chunk rendering; existing provider stream
tests should continue passing without parser changes.

Preserve the existing 0.3-second throttle, off-thread source/snapshot work, navigation
and prompt-input gates, one-second selected-source poll, and local region replacement.
Do not add disk I/O to the render/message pump, a broad scan, or more frequent global
refreshes. Check navigation responsiveness during the mounted streaming case; use
`SASE_TUI_PERF=1` for a bounded local measurement if there is evidence of a regression
rather than running an unrelated full benchmark.

Run the targeted visual lane for the existing live Reply growth/split-view case:

```sh
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py -k live_reply
```

Inspect the PNG report and any golden changes. Appearance is expected to stay the same;
partial reports do not prove skipped captures current. Run formatting and
`sase tool run check` (the governed `just check` recipe) after changes. Follow the
monitor skill if verification needs a handoff; do not start `check-full` as part of this
plan. A successful implementation has a demonstrated pre-fix failure, passing
focused/mounted regressions, reviewed targeted visual results, and the required check
result.

For any real local smoke check, use a fresh TUI process importing the changed checkout:
an already-running editable-install TUI retains its imported code. Compare the selected
Muse card with its live-reply file after text starts, and record that the card updates
before completion. The deterministic gated test is the required proof and does not
require launching a paid provider agent.

## Scope boundaries

Keep this change focused on enabling the established follower for actual agent rows and
proving that behavior end to end. Tool-activity placeholders, rejection tracing,
snapshot-prefix/starvation redesign, terminal-only fallback text, provider upgrades, and
fixture recapture from a live Muse CLI are separate work. Do not write tool progress
into `live_reply.md`, change Muse transport or parser semantics, add a global refresh
workaround, or introduce new configuration or CLI options. No research-sidecar or memory
edits are required.
