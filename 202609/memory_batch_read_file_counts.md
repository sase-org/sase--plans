---
tier: tale
title: Count every file a batched memory read delivered
goal:
  A single multi-selector `sase memory read` is credited with every file it requested,
  so the ACE MEMORY lane shows e.g. `1 read · 3 files`, and the clan MEMORY lane, `sase
  memory log` summaries, and memory panel read counts agree.
size: small
proposed_by: bbugyi200.athena.0x
create_time: 2026-09-13 07:04:50
status: wip
---

# Plan: Count every file a batched `sase memory read` delivered

## Problem

When an agent reads several memories in one command (for example
`sase memory read sase_flags.md tui_perf.md lint_and_test.md -r "..."`), the ACE agent
detail panel's MEMORY lane header shows `1 read · 1 file`. It should show
`1 read · 3 files`. The row itself already lists all three selectors.

## Root cause

A multi-selector read logs ONE `MemoryReadEvent` (built by
`build_memory_read_batch_event` in `src/sase/memory/_read_log_events.py`, called from
`build_memory_read_event_for_view` in `src/sase/memory/cli_read.py`). For a batch,
`canonical_path` is only the first resolved target (kept for pre-web consumers), while
the full list of requested files is in `resolved_targets` (notes by canonical path,
requested strands as `web:slug`; a bare web selector marks every strand requested).
Strands pulled in only by link expansion go to `included_targets`.

Every "distinct files" computation keys on `event.canonical_path`, so a batch collapses
to its first target:

1. `append_agent_memory_reads_section` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_memory_reads.py` —
   `distinct_paths = len({item.event.canonical_path for item in events})`. This is the
   bug in the screenshot.
2. `_agent_clan_disk_aggregation.py` (clan SASE CONTEXT MEMORY lane) — adds one
   accumulator entry keyed by `memory_event.canonical_path`, so the clan view lists only
   the first file of a batch and the SASE CONTEXT count undercounts.
3. `summarize_memory_reads_by_path` / `summarize_memory_reads_by_agent` in
   `_read_log_events.py` group by / count `canonical_path`. These feed `sase memory log`
   (paths panel, agents panel "Paths" column, JSON summary) and the memory panel
   catalog's per-note read counts (`_read_summaries_for` in
   `src/sase/ace/tui/memory_panel_catalog.py`), so the 2nd..Nth notes of a batch never
   get credit for being read.
4. `_build_memory_log_summary_payload` in `src/sase/memory/cli_log.py` computes
   `total_memory_paths` from `canonical_path`.

Note: `filter_memory_read_events` already treats a batch as reading every target
(`path_filter in event.resolved_targets`), so the summaries are currently inconsistent
with the filter.

## Design decision: what counts as a "file"

Count the event's **requested** targets (`resolved_targets`), not the link-expanded
`included_targets`. This matches the GLOSSARY lane precedent, whose header counts
distinct `terms` and ignores `related_terms`, and matches the selectors the MEMORY row
displays.

Preserve existing single-target semantics exactly: only fan out when an event has MORE
THAN ONE resolved target. Single-note events and single-strand events keep using
`canonical_path` (for those, it equals the lone resolved target). This matters because
existing tests (e.g. `test_memory_read_aggregation_groups_by_path_and_agent` in
`tests/test_memory_read_log.py`) build variants via
`dataclasses.replace(base, canonical_path="bar.md")` while leaving
`resolved_targets=("foo.md",)`; preferring `resolved_targets` unconditionally would
silently break them. v1 events (no `resolved_targets`) also fall back to
`canonical_path`.

## Implementation

### 1. Shared helper (`src/sase/memory/_read_log_events.py`)

Add a public helper and export it from `_read_log_events.__all__` and from
`src/sase/memory/read_log.py` (import + `__all__`), matching how the other helpers are
re-exported:

```python
def memory_read_event_targets(event: MemoryReadEvent) -> tuple[str, ...]:
    """Return the distinct memory files one read event requested, in read order.

    A multi-target batch covers every ``resolved_targets`` entry; any other event
    (single note, single strand, v1 row) is identified by ``canonical_path``.
    Link-expanded ``included_targets`` are context, not requested reads.
    """
    if len(event.resolved_targets) > 1:
        return tuple(dict.fromkeys(t for t in event.resolved_targets if t))
    return (event.canonical_path,) if event.canonical_path else ()
```

### 2. Summaries (`src/sase/memory/_read_log_events.py`)

- `summarize_memory_reads_by_path`: append each event to the group of every target in
  `memory_read_event_targets(event)` (instead of only `canonical_path`). An event with
  no targets is skipped. `read_count`, `distinct_agent_count`, and the latest-read
  fields then naturally credit every file of a batch.
- `summarize_memory_reads_by_agent`: `distinct_path_count` becomes the size of the union
  of `memory_read_event_targets` over the agent's events. Leave `last_path` as the
  latest event's `canonical_path` (it is a single-string display field).

### 3. `sase memory log` (`src/sase/memory/cli_log.py`)

- `_build_memory_log_summary_payload`: `total_memory_paths` = size of the union of
  `memory_read_event_targets` across `event_tuple`.
- When a `path_filter` is set, restrict the summary rows (JSON `summary` list and the
  `_paths_panel` table) to the row whose `canonical_path == path_filter`. Without this,
  fanning out would make `sase memory log --path foo.md` also list the other files of
  any matching batch. Keep `total_memory_paths` consistent with the rows that are shown
  (i.e. compute it from the restricted summary rows when a path filter is set). Do not
  change which events the filter matches.

### 4. MEMORY lane header (`_agent_memory_reads.py`)

In `append_agent_memory_reads_section`, replace the `distinct_paths` computation with:

```python
distinct_paths = len(
    {target for item in events for target in memory_read_event_targets(item.event)}
)
```

Import `memory_read_event_targets` from `sase.memory.read_log`. Nothing else in the
section changes (rows, hints, frontmatter marker, overflow footer stay as-is).

### 5. Clan SASE CONTEXT MEMORY lane (`_agent_clan_disk_aggregation.py`)

Mirror the GLOSSARY lane, which already fans out per term: loop
`for target in memory_read_event_targets(memory_event):` and call `_add_context` with
`key=target`, `label=target`, passing the same `memory_display` value. Single-target
events produce exactly the one entry they produce today, so
`test_agent_clan_aggregation.py`'s `by_label["MEMORY"].entries[0].count == 2` still
holds. Hint registration in `_agent_display_clan_sections.py` needs no change (a batch
event has an empty `resolved_path`, so each fanned-out entry registers the event's
deferred read report — same as GLOSSARY entries do).

The memory panel catalog (`memory_panel_catalog.py`) needs no code change; it picks up
the corrected `summarize_memory_reads_by_path` automatically.

## Tests

Add focused tests (keep existing ones passing unchanged):

1. `tests/test_memory_read_log.py`
   - `memory_read_event_targets`: single-note event → `(canonical_path,)`; multi-target
     batch → all resolved targets in order, de-duplicated; batch with `included_targets`
     → included targets NOT counted; v1-style event with empty `resolved_targets` →
     `(canonical_path,)`; empty `canonical_path` and no targets → `()`.
   - `summarize_memory_reads_by_path` on a batch event over `a.md`, `b.md`, `web:strand`
     plus a single-note `a.md` read → three rows; `a.md` has `read_count == 2`, the
     others `1`.
   - `summarize_memory_reads_by_agent`: the same agent's `distinct_path_count == 3`.
2. `tests/ace/tui/widgets/test_agent_memory_reads.py`
   - A pathless batch event (`resolved_path=""`, `kind="strand"` or `"note"`,
     `selectors`/`resolved_targets` of three notes) renders
     `▸ MEMORY · 1 read · 3 files\n` (the screenshot scenario).
   - A batch plus a separate single read of one of its files → `2 reads · 3 files`
     (distinct across events).
   - A batch whose `included_targets` has extra strands still counts only the requested
     targets.
3. `tests/ace/tui/widgets/test_agent_clan_aggregation.py` — a member summary with one
   batch memory read over two files yields two MEMORY entries labeled with those files.
4. `tests/main/test_memory_log.py` — a JSON summary with a seeded batch event reports
   one row per target and the matching `total_memory_paths`; with
   `--path <second target>`, the summary contains only that path's row and
   `total_memory_paths == 1`. The existing `memory_read_event` factory in
   `tests/main/memory_handler_helpers.py` builds `schema_version=1` single-note events
   with no `resolved_targets`; for the batch event use `dataclasses.replace(...)` (or
   extend the factory with optional keywords) to set
   `schema_version=READ_LOG_SCHEMA_VERSION`, `resolved_path=""`, `kind`, `selectors`,
   and `resolved_targets`, so the JSONL round-trip keeps the targets.

## Verification

Run `just check` (install first with `just install` if the workspace venv is stale). No
PNG visual snapshot is expected to change; if `just check` selects visual tests that
fail, inspect before updating goldens.

## Out of scope

- Changing the event schema or how `sase memory read` writes batch events.
- Changing the MEMORY row display, hint targets, or the read report contents.
- Counting link-expanded `included_targets` as reads.
