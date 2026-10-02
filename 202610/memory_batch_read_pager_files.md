---
tier: tale
title: Open each memory file of a batch memory read separately in the pager
goal:
  Selecting a batch memory-read hint from the Agents tab `v` flow opens every requested
  memory note and strand file as its own pager section (and in $EDITOR / clipboard),
  instead of the inline `sase memory read` output report, with the report kept only as a
  failure fallback.
size: medium
proposed_by: bbugyi200.athena.0vh
create_time: 2026-10-02 13:49:30
status: wip
---

# Plan: Open each memory file of a batch memory read separately in the pager

## Problem

On the Agents tab, `v` hint mode numbers every row of the `SASE CONTEXT` / `MEMORY` lane
(and the same rows in clan/session context sections). Selecting a row behaves
differently depending on the recorded `MemoryReadEvent`:

- **Single-note read** (`event.resolved_path` is set): the hint maps straight to the raw
  memory file, so the pager opens it as an ordinary file section with link scanning,
  syntax, and the memory-history time band.
- **Batch read** (any `sase memory read` with several selectors, any web/strand
  selector, or a note with inline `![[...]]` notes; `resolved_path == ""` and
  `schema_version >= 2`): `register_memory_read_report_hint` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_memory_reads.py` maps the hint to a
  deterministic report path plus a deferred `MemoryReadReportSpec`. At submit time
  `_materialize_selected_view_files` in
  `src/sase/ace/tui/actions/hints/_view_processing.py` calls `write_memory_read_report`,
  which writes one Markdown file containing the reproduced command, recorded metadata,
  and the full `memory show --format markdown` output — every note and strand inline,
  separated by `---------- MEMORY FILE:` / `---------- MEMORY WEB:` rules. The pager
  then shows that single synthetic report.

So viewing a batch read such as `sase memory read sase_sizes.md glossary:Pomodoro` shows
the agent-facing blob instead of opening `sase_sizes.md` and the `pomodoro.md` strand as
two separate pager sections. The pager already accepts many paths
(`build_pager_document(files, ...)` → `document_from_paths` → one `path_section` per
file), and file-backed sections under a memory root are recognized by the memory-history
pager provider, so opening the real files also gives the user the memory time band for
each one.

## Goal

Selecting a batch memory-read hint opens **each memory file the read requested** as its
own file-backed pager section, in the order the read printed them. `@` opens those files
in `$EDITOR`, and `%` copies their paths. The generated report stays only as a fallback
when the read can no longer be resolved.

## Design decisions

1. **Which files.** Open only the _requested_ targets, meaning requested flat notes and
   requested strands. This matches the existing "requested vs. context" semantics of
   `memory_read_event_targets` and the lane's `N files` count. Link-expanded or
   mention-closure context (`included_targets`, `related` nodes, inline-embedded notes)
   is not opened as separate sections; it stays reachable through the opened file's own
   links. A bare web selector (for example `glossary`) requests every strand in the web,
   so it opens every strand file. The web descriptor is not opened because the batch
   output never prints it.
2. **Order.** Use the batch's top-level render order (`render_units`, falling back to
   notes then web sections), which is the order the agent saw. Inside a web section,
   keep node order. Dedupe paths while preserving order.
3. **Resolution policy.** Reuse the report's existing non-auditing re-resolution
   (`_resolve_report_view` in `src/sase/memory/memory_read_report.py`). It tries the
   recorded project's workspace first and falls back to the recorded `cwd`. One policy
   then decides both what the report would print and which files open. Re-resolution
   must never append a memory-read event.
4. **Fallback.** If re-resolution raises, or yields no requested file that exists on
   disk, fall back to today's behavior and write and open the report. The report already
   degrades to recorded metadata plus a "Could not re-resolve" note, so the user still
   learns why. If some resolved paths exist and others do not, open the existing ones
   and report the missing ones through the existing `File no longer exists: …` warning.
5. **No hint-registration change.** Hint numbering and registration stay as they are:
   one hint per read row, no I/O at render time (`tui_perf`). Expansion from one hint to
   N files happens only in the off-thread materialization step, which already runs under
   `asyncio.to_thread`.
6. **Boundary.** Memory selector resolution is Python-owned today
   (`sase.memory.selector`). The new path helper therefore lives next to the report
   builder in `sase.memory` as a frontend-agnostic function, with no `sase-core` change.
   No feature flag is needed: this is a requested fix, and the old branch survives only
   as the failure fallback.

## Implementation

### 1. `src/sase/memory/selector_render.py`: requested file paths in render order

Add a public helper next to the private `_render_items`:

```python
def memory_selector_batch_file_paths(batch: ResolvedMemorySelectorBatch) -> tuple[Path, ...]:
    """Return the requested note and strand files of *batch* in printed order."""
```

- Walk `_render_items(batch)`.
- For a `ResolvedMemoryNote` with `render_origin == "requested"`, yield
  `note.content.path.resolved_path`. This is the same value single-note events record as
  `resolved_path`.
- For a `MemoryWebReadSection`, yield `node.strand.path` for each node with
  `origin == "requested"`, in node order.
- Dedupe while preserving order, and add the helper to `__all__`.

Keeping this in `selector_render.py` avoids exporting the private `_render_items` across
modules (symvision private-misuse).

### 2. `src/sase/memory/memory_read_report.py`: event → memory file paths

Add:

```python
def memory_read_file_paths(event: MemoryReadEvent) -> tuple[str, ...]:
    """Return absolute paths of the memory files *event* requested, in read order."""
```

- Use `_event_selectors(event)`. If it is empty, return `()`.
- Call `_resolve_report_view(event, selectors)` and return
  `tuple(str(p) for p in memory_selector_batch_file_paths(view))`.
- Catch the same exception family the report catches
  (`MemoryReadError, OSError, UnicodeError, RuntimeError, ValueError`) and return `()`.
- The helper is non-auditing and must never append a read event.
- Update the module docstring to say it also exposes the requested-file expansion, and
  add the function to `__all__`.

### 3. `src/sase/ace/tui/actions/hints/_view_processing.py`: expand on materialize

In `_materialize_selected_view_files`, handle a `MemoryReadReportSpec` before the
generic report branch:

- Call `memory_read_file_paths(spec.event)`, importing it at module level next to the
  existing `write_memory_read_report` import so tests can monkeypatch it on this module.
- If at least one returned path exists, append each existing path to `materialized`,
  append each non-existing one to `missing`, and `continue`. Do not write a report.
- Otherwise, fall through to the current `write_memory_read_report(spec)` path,
  unchanged, including the `failed` handling.
- Make the final `materialized` tuple order-preserving unique, for example with
  `tuple(dict.fromkeys(materialized))`. This keeps the invariant `parse_view_input`
  already enforces, now that one hint can expand into files that another selected hint
  also opens (for example, a single-note read and a batch read of the same note).

`_finish_view_request` needs no structural change. Pager, `@` editor, and `%` clipboard
destinations all consume `outcome.files`, so they all receive the expanded memory files
automatically. Check that the media-detection and empty-selection branches still behave,
and that the `Failed to build hint report` / `File no longer exists` notifications still
fire on the fallback and partial paths.

### 4. Tests

`tests/test_memory_read_report.py` (reuse `_seed_decisions_web`, `_note`, `_event`):

- A mixed batch (`decisions:corpus-before-mechanism`, `tui_perf.md`) returns the strand
  path then the note path, matching the printed order, as absolute existing paths.
- A bare web selector (`decisions`) returns every strand file and not the descriptor.
- A fixture whose closure adds a related strand (or an inline note) still returns only
  the requested files.
- An unresolvable selector (`decisions:missing`) returns `()`.
- Calling the helper appends no memory-read event (mirror
  `test_write_report_records_no_memory_read_event`).

ACE view-hint tests (`tests/ace/tui/actions/test_view_files_reports.py`, helpers in
`tests/ace/tui/actions/_view_files_helpers.py`):

- Keep the existing memory-report tests hermetic. `_memory_spec` uses project `sase`,
  which can resolve against a real enabled project on a dev machine. Monkeypatch
  `sase.ace.tui.actions.hints._view_processing.memory_read_file_paths` to return `()` in
  every test that expects the report, then rename or re-docstring those tests as
  fallback coverage. This covers the pager, off-thread, editor, clipboard, mixed-order,
  and failure tests.
- New tests that monkeypatch the resolver to return two real tmp files:
  - Pager: `_assert_pager_document_paths` sees both memory files in order, and
    `write_memory_read_report` is never called.
  - `@`: `_open_files_in_editor` receives both memory files.
  - `%`: `_copy_files_to_clipboard` receives both memory paths.
  - Mixed selection order: a batch hint between a plain file hint and a tool-call report
    hint expands in place, preserving the selection order around it.
  - Dedupe: a plain hint for `a.md` plus a batch hint resolving to `a.md` and `b.md`
    yields `[a.md, b.md]`.
  - Partial: one resolved path is missing. The existing file opens and a
    `File no longer exists:` warning fires.
  - The resolver runs off the event-loop thread (mirror the existing thread-id test).

No change is expected in `tests/ace/tui/widgets/test_agent_memory_reads.py` or
`tests/ace/tui/widgets/test_agent_display_clan_context_hints.py`, because hint
registration is unchanged. Run them to confirm.

### 5. Docs

- `docs/memory.md`: after the sentence ending "drives the `SASE CONTEXT` / `MEMORY` lane
  of the agent Main deck, including clan aggregates.", add a short paragraph.
  - A numbered `v` hint on a single-note read opens that note.
  - On a batch read, the hint opens each requested note or strand file as its own pager
    section, in the order the read printed them. `@` opens them in `$EDITOR` and `%`
    copies their paths.
  - Link/mention context is not opened.
  - When the read can no longer be resolved, the hint pages a generated report with the
    recorded metadata instead.
- `docs/ace.md` has no `MEMORY` lane bullet today; its `GLOSSARY` bullet only covers
  legacy reads. Add a one-sentence pointer where the `GLOSSARY` bullet says current
  `glossary:<keyword>` reads "surface in the `MEMORY` lane". It should say that the
  lane's hints open the requested memory files, with a link to the `docs/memory.md`
  paragraph. Do not touch the legacy-glossary report wording.

## Verification

- `sase tool run check`: the whole-repo lint gates, including mypy, ruff, and symvision
  for the two new public helpers, plus the diff-scoped tests.
- Targeted pytest during development: `tests/test_memory_read_report.py`,
  `tests/ace/tui/actions/test_view_files_reports.py`,
  `tests/ace/tui/widgets/test_agent_memory_reads.py`,
  `tests/ace/tui/test_agents_view_hint_survives_refresh.py`.
- Optional manual check in ACE: on the Agents tab, select an agent with a multi-selector
  memory read, press `v`, and enter its hint number. The pager should show one section
  per memory file (`N files` title) with the memory time band available on each.
