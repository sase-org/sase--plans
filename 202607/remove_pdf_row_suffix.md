---
tier: tale
title: Remove the PDF finalization label from ACE agent rows
goal:
  ACE Agents-tab rows no longer render the transient "PDF <n>/<m> <path>" finalization suffix; live PDF progress stays
  visible only in the labeled Activity field of the prompt/detail header.
create_time: 2026-07-29 07:23:15
status: wip
---

- **PROMPT:** [202607/prompts/remove_pdf_row_suffix.md](prompts/remove_pdf_row_suffix.md)

# Remove the PDF finalization label from ACE agent rows

## Problem

Agent rows on the ACE Agents tab sometimes render a long, unlabeled suffix such as:

```
sase (DONE) x5 research.o.final    PDF 3/3 sase/repos/research/202607/scalable_skill_disclosure/scalable_skill_d
```

The `PDF 3/3 <path>` text is transient Markdown-to-PDF finalization progress. It is unlabeled (nothing says what "PDF"
means), it embeds a full workspace-relative source path, and it is right-aligned to the panel width, so it stretches the
row past the panel edge and truncates mid-path. It also crowds out the runtime/timestamp suffix that normally occupies
that slot.

## Root cause

The label is produced during successful-agent finalization and consumed by the agent-row renderer.

Producer:

- `src/sase/axe/run_agent_exec_finalize.py::_render_markdown_pdfs` installs a progress callback around
  `render_markdown_pdf_attachments` and calls `update_workflow_pdf_status()` on every progress event.
- `src/sase/axe/run_agent_exec_markers.py::update_workflow_pdf_status` writes a `pdf_status` dict into the run's
  `workflow_state.json` and derives a compact `activity` string from it via `_pdf_activity_from_status` (the
  `source_started` / `engine_started` branch is what yields `PDF <index>/<total> <source>`).
- `src/sase/axe/run_agent_exec_markers.py::clear_workflow_pdf_activity` pops `activity` once rendering finishes. It
  deliberately does not clear `pdf_status`.

Consumer (the row):

- `src/sase/ace/tui/widgets/_agent_list_render_layout.py::build_activity_suffix` renders `_agent_activity_label(agent)`,
  which returns `agent.activity` when set and otherwise re-derives the same kind of string from `agent.pdf_status`.
- `src/sase/ace/tui/widgets/_agent_list_render_agent.py` merges that into the row's right-hand suffix via
  `combine_suffixes(build_activity_suffix(agent), runtime_with_file_change)`.

Evidence confirming this is the exact mechanism in the screenshot: the run behind that row wrote three Markdown PDFs
between 07:13:13 and 07:13:16, and its third source is
`sase/repos/research/202607/scalable_skill_disclosure/scalable_skill_disclosure.md`, which matches the truncated
`PDF 3/3 ...scalable_skill_d` text exactly. The durable state left behind is
`{"stage": "completed", "total": 3, "generated": 3, "skipped": 0, "cap": 10, "active": false}` with no `activity` key.

A sweep of every `workflow_state.json` under `~/.sase/projects/*/artifacts/` on this host (1678 files, 1388 carrying
`pdf_status`) found every persisted `pdf_status` at `stage: "completed"` and zero files retaining `activity`. So nothing
durable depends on the row label; it exists only for the few seconds of PDF conversion.

Secondary defect worth removing along with it: because `clear_workflow_pdf_activity` never clears `pdf_status`, the
row's `pdf_status` fallback is dead in the happy path (terminal `stage: "completed"` maps to `None`) but would pin a
permanently stale `PDF n/m <path>` onto a finished row if a runner were killed mid-render.

## Decision

Delete the activity suffix from the agent row entirely - both the `agent.activity` path and the `agent.pdf_status`
fallback.

Keep the existing `Activity: <label>` field in the prompt/detail header
(`src/sase/ace/tui/widgets/prompt_panel/_workflow_render.py` and
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`). That surface already renders the same string
with a self-describing `Activity:` label, is not width-constrained the way a row is, and only reads `agent.activity`, so
it self-clears when finalization ends. Live PDF progress therefore stays discoverable; it just stops deforming the
Agents list.

Then drop the `pdf_status` plumbing that becomes write-only in Python once the row stops reading it. The field stays in
the on-disk `workflow_state.json` contract and in the sase-core agent-scan wire
(`crates/sase_core/src/agent_scan/wire.rs`), so no change to the Rust core repo is required and no data migration is
involved - the Python side simply stops loading a field it no longer renders.

Non-goals: do not change axe's producer side, do not stop writing `pdf_status` or `activity` to `workflow_state.json`,
and do not touch the sase-core wire.

## Implementation steps

1. `src/sase/ace/tui/widgets/_agent_list_render_layout.py`
   - Delete `build_activity_suffix`, `_agent_activity_label`, and the now-unused `_ACTIVITY_STYLE` constant.
   - Delete `combine_suffixes` as well. Its only caller is the row renderer changed in step 2, and leaving an unused
     public function would fail the `symvision` lint stage of `just check`.
   - Update the module docstring's list of helpers if it still names the removed functions.

2. `src/sase/ace/tui/widgets/_agent_list_render_agent.py`
   - Drop the `build_activity_suffix` and `combine_suffixes` imports.
   - Replace the `suffix = combine_suffixes(build_activity_suffix(agent), runtime_with_file_change)` call in
     `format_agent_option` with `suffix = runtime_with_file_change`.

3. `src/sase/ace/tui/widgets/_agent_list_render_cache.py`
   - Remove `agent.activity` and the `tuple(sorted(agent.pdf_status.items())) if agent.pdf_status else None` entry from
     `agent_render_key`. Neither value can change row output any more, so keeping them would only cause spurious cache
     misses (and the `pdf_status` entry would not compile once step 4 removes the field).

4. Remove the now-unread `pdf_status` plumbing from the TUI model and loaders, keeping `activity` untouched everywhere:
   - `src/sase/ace/tui/models/_agent_state.py`: drop the `pdf_status` field from `Agent` and adjust the comment above
     `activity` so it still explains that field. Verify whether the `Any` import is still needed.
   - `src/sase/ace/tui/models/workflow.py`: drop `WorkflowEntry.pdf_status`.
   - `src/sase/ace/tui/models/_loaders/_workflow_loaders.py`: drop the `pdf_status` read in `load_workflow_states` and
     the `pdf_status=` arguments to `WorkflowEntry(...)` and `Agent(...)`.
   - `src/sase/ace/tui/models/_loaders/_workflow_snapshot_loaders.py`: drop the `pdf_status=wf_state.pdf_status` and
     `pdf_status=entry.pdf_status` arguments.

5. Tests
   - `tests/test_ace_tui_widgets.py`: delete `test_agent_row_displays_active_pdf_status_suffix`,
     `test_agent_row_honors_explicit_pdf_activity_suffix`, `test_agent_row_omits_completed_pdf_status_suffix`, and
     `test_agent_row_omits_inactive_skipped_pdf_status_suffix`. Remove the `pdf_status` parameter from the local
     `_make_agent` helper (keep `activity`, which other tests may use). Add one regression test that builds an agent
     with a live `activity="PDF 3/3 docs/notes.md"` and asserts `"PDF" not in suffix.plain` for
     `format_agent_option(...)`, so the label cannot silently return to the row.
   - `tests/test_agent_loader_workflow_states.py`: drop the `pdf_status` fixtures and the `entries[0].pdf_status` /
     `agents[0].pdf_status` assertions; keep the `activity` assertions in the same tests.
   - `tests/test_agent_model_bundle.py`: drop the `pdf_status={"path": ..., "done": True}` argument from the
     `Agent(...)` construction and any assertion that reads it back.
   - Confirm the header/detail-panel rendering of `activity` still has coverage; if the four deleted row tests were the
     only place `activity` reached a renderer, add a small test asserting the prompt-panel header still emits
     `Activity: <label>`.

6. Docs - correct the three places that currently claim the label appears on the Agents row:
   - `docs/ace.md` (the "During successful-agent finalization..." paragraph): say the label renders in the prompt/detail
     header only.
   - `docs/axe.md` (the "As PDFs are prepared, axe updates..." sentence): same correction.
   - `docs/agent_images.md` (the "While PDFs are being prepared..." paragraph): same correction, and drop the mention of
     ACE loading `pdf_status`, since the TUI no longer reads it.

## Verification

- `just install` first (ephemeral workspaces can have drifted deps), then `just check`. This must pass, including the
  `symvision` stage, which is the specific reason `combine_suffixes` is deleted rather than left behind.
- Targeted runs while iterating:
  `just test tests/test_ace_tui_widgets.py tests/test_agent_loader_workflow_states.py tests/test_agent_model_bundle.py`
  (or the equivalent `pytest` invocation).
- `just test-visual` if any PNG snapshot of the Agents tab shifts; only accept changes with
  `--sase-update-visual-snapshots` when the diff is exactly the removed suffix.
- Manual sanity check (optional): open `sase ace`, select a recently finished agent that produced Markdown PDFs, and
  confirm the row shows only the timestamp/runtime suffix while the detail panel still shows an `Activity:` line while a
  conversion is actually in flight.

## Risks and notes

- Low risk and fully reversible: the change is presentation-only on the Python/Textual side. No producer, on-disk
  format, or Rust wire change.
- `workflow_state.json` keeps both `pdf_status` and the transient `activity` key, so older and newer readers stay
  compatible and no artifacts need rewriting.
- `src/sase/integrations/_agent_list_entry_builder.py` reads `activity` from the workflow state for the integrations
  agent-list entry. That is a separate surface and must keep working - do not remove the `activity` field anywhere.
- If a reviewer prefers a smaller diff, steps 1-3 alone satisfy the user-visible request; step 4 is the cleanup that
  keeps `pdf_status` from becoming a write-only field. Steps 5 and 6 are required either way, scoped to whichever of
  steps 1-4 land.
