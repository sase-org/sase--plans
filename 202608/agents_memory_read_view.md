---
tier: tale
title: View memory-web reads from the Agents tab
goal:
  Agents-tab memory hints open the equivalent non-auditing memory-show output for web,
  strand, and batch reads without blocking the TUI.
size: medium
proposed_by: bbugyi200.athena.0db
---

# Plan: View memory-web reads from the Agents tab

## Goal

Repair the Agents-tab `v` hint flow for audited `sase memory read` events that represent
a memory web, strand, or multi-selector batch. Selecting the hint for an event such as
`decisions:corpus-before-mechanism` must open a deferred Markdown report containing the
current output of the equivalent `sase memory show` invocation, without recording a new
read or blocking Textual's event loop. Preserve the existing raw-file behavior for
ordinary single-note reads whose audit event has a real `resolved_path`.

## Root cause

Schema-v2 batch read events intentionally leave `MemoryReadEvent.resolved_path` empty: a
web/strand selector can resolve to multiple strand files, so there is no truthful single
filesystem path. The Agents detail renderer predates that event shape and still
registers every MEMORY hint as `hint_number -> event.resolved_path`. Hint mode therefore
publishes an empty target for web and strand rows; submitting its number cannot give the
view-file pipeline anything to open.

The existing deferred tool-call and legacy-glossary report path already provides the
right UI architecture: register a deterministic future report path while rendering,
materialize selected reports off-thread only after submission, and then reuse the
pager/editor/clipboard routing. The fix should add modern memory reads to that
architecture rather than assigning a misleading strand source path or running a
subprocess on the keypress path.

## Implementation

1. Add a frontend-neutral deferred report builder for modern memory-read events under
   `src/sase/memory/`.
   - Define a `MemoryReadReportSpec` carrying the immutable `MemoryReadEvent`, optional
     family/clan role label, and deterministic report path.
   - Store reports in a bounded SASE state subdirectory, with atomic writes and pruning
     equivalent to the existing deferred report builders.
   - Reconstruct the original selector request from `event.selectors`, preserving
     `event.depth` and resolving from the event's recorded project/workspace context.
     Render through the same pure Markdown path used by `sase memory show`; expose a
     small public Markdown-rendering helper from `selector_render.py` if needed instead
     of redirecting process-global stdout or spawning the CLI.
   - Include the reproducible `sase memory show` command, recorded read metadata, and a
     current `## Output` section. Resolution failures (for example, a removed workspace
     or renamed strand) must still yield a useful report with the recorded selectors and
     a concise failure note rather than raising into the TUI.
   - This is a non-auditing view: do not call the `sase memory read` handler or append a
     memory-read event while materializing the report.

2. Register deferred memory reports in the Agents detail hint document.
   - In `_agent_memory_reads.py`, keep mapping a single-note event directly to its
     non-empty `resolved_path`. For a pathless schema-v2 event, allocate a deterministic
     report path, map the visible hint number to it, and record its
     `MemoryReadReportSpec` alongside the existing hint state.
   - Extend `HeaderHintState` and `AgentHintRender` with the memory-report mapping, and
     carry it through ordinary-agent, family, and clan hint renders.
   - Extend ACE hint-mode initialization, reset/cancellation, deferred first render, and
     refresh/enrichment republish paths so the report mapping always corresponds to the
     currently displayed hint numbers. Do not change `default_config.yml`: the existing
     app-level `view_files: "v"` binding is the intended entry point.

3. Materialize selected memory reports through the existing view-request pipeline.
   - Add `MemoryReadReportSpec` to the accepted deferred-report union and dispatch it to
     the new writer in `_write_selected_hint_reports`.
   - Snapshot only the selected report specs in `_prepare_view_input`, preserving hint
     order when memory reports are mixed with raw files, tool-call reports, legacy
     glossary reports, or commit views.
   - Keep report resolution and file writes inside the existing `asyncio.to_thread`
     materialization boundary. Preserve current pager, `@` editor, `%` clipboard,
     failure notification, and shutdown behavior.

4. Add focused regression coverage.
   - Add report-builder tests for deterministic paths, exact selector/depth command
     reconstruction, a strand such as `decisions:corpus-before-mechanism`, multi-target
     output, non-auditing behavior, graceful re-resolution failure, atomic writes, and
     bounded pruning.
   - Extend MEMORY renderer tests to prove that single-note events still map to their
     raw file while pathless strand/web/batch events map to deferred reports with the
     correct event and role attribution.
   - Extend view-file action tests to prove memory reports materialize off the
     event-loop thread and work with pager, editor, clipboard, mixed selections, and
     failure handling.
   - Extend active-hint refresh/enrichment tests so memory-report state survives every
     place that currently republishes tool-call and glossary report mappings.

## Acceptance criteria

- On the Agents tab, pressing `v` and choosing the hint for
  `decisions:corpus-before-mechanism` opens a report whose Output section matches the
  current Markdown output of `sase memory show decisions:corpus-before-mechanism` in
  that read's project context.
- Bare-web, strand, mixed-selector, and depth-limited reads are represented by their
  original selector batch; no pathless hint is published.
- Viewing a report does not append a memory-read audit event and performs no disk walk,
  selector resolution, subprocess, or report write on Textual's event loop.
- Existing single-note MEMORY hints and all existing `v` destinations retain their
  current behavior, ordering, editor/clipboard suffixes, and refresh safety.
- No feature flag or keymap migration is introduced for this repair.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run the focused memory-report, MEMORY renderer, hint materialization, and active-hint
   refresh test modules, including a regression built from a real pathless strand event
   shape.
3. Run `just check` and resolve every failure caused by this change. Use
   `tools/select_tests --explain` if the scoped selection is surprising; escalate to a
   monitored `just check-full` only if the touched-file broadening rules or scoped gate
   require it.
4. Manually compare a generated report's Output section with
   `sase memory show decisions:corpus-before-mechanism --format markdown`, and confirm
   the memory-read log count is unchanged by the view action.
