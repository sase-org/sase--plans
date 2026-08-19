---
tier: tale
title: Show a glossary read's own output when its ACE hint is selected
goal:
  Selecting the numbered hint on a SASE CONTEXT GLOSSARY row pages a generated report of
  that `sase glossary read`'s output -- command line, recorded audit metadata, and the
  resolved term closure -- instead of the project's `sase/sase.yml` config file.
size: medium
proposed_by: bbugyi200.athena.07l
create_time: 2026-08-19 09:14:29
status: wip
---

# Show `sase glossary read` output when a GLOSSARY hint is selected

## Problem

On the ACE Agents tab, the `SASE CONTEXT / GLOSSARY` lane lists an agent's audited
`sase glossary read` invocations, and each row carries a numbered `[N]` view hint.
Selecting that hint pages the project's `sase/sase.yml` config file instead of the
output the agent actually got back from the command.

## Root cause

`sase glossary read` records the project's glossary **config path** as the audit event's
`source_path` (`src/sase/glossary/cli_read.py:39`, `source_path=resolved.config_path`),
and both ACE hint-registration sites map the GLOSSARY row's hint straight to that path:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_glossary_reads.py:78-84` — the per-agent
  and family detail lane registers `hint_state.hint_mappings[n] = event.source_path`
  whenever `source_path` is set.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_context.py:79-80` —
  `clan_context_entry_hint_target` returns `value.event.source_path` for `GLOSSARY` rows
  on clan container panels.

So `v` + the hint number resolves to `sase/sase.yml` and the pager shows the whole
config file. The behavior is even documented as intended today in `docs/ace.md` (~line
3901): "A numbered file hint targets the term's recorded `source_path` in
`sase/sase.yml`."

## Goal

Selecting a GLOSSARY hint shows the output of that read: the command line that was run,
the recorded audit metadata, and the resolved closure — every requested term plus its
related terms with definitions — rendered as Markdown and paged the same way every other
view hint is paged.

## Approach

Mirror the existing **deferred report** pipeline that slow tool-call hints already use
(`src/sase/ace/tui/tools/report.py`,
`src/sase/ace/tui/widgets/prompt_panel/_tool_call_report_hints.py`,
`src/sase/ace/tui/actions/hints/_processing.py:50-68`): at hint-render time register a
deterministic report path plus an in-memory spec (no disk I/O on the render hot path),
then materialize the file off the event loop only when the user actually selects that
hint, and hand the materialized path to the normal pager/editor/copy routing.

The report body is produced by re-resolving the recorded terms through the same resolver
the CLI uses, so the content is the command's own output rather than a paraphrase.

### Step 1 — Expose the Markdown closure renderer

In `src/sase/glossary/render.py`, promote `_glossary_closure_markdown` to a public
`glossary_closure_markdown(closure, *, project_name) -> str`, have
`render_glossary_closure`'s `markdown` branch call it, and add it to `__all__`. No
behavior change — this is what `sase glossary read -f markdown` already prints, now
reusable in-process.

### Step 2 — New module `src/sase/glossary/read_report.py`

Frontend-agnostic on purpose: it sits beside `src/sase/glossary/read_log.py` (which owns
the events it renders) rather than under `ace/`, so a future non-TUI surface can reuse
it. Contents:

- `GlossaryReadReportSpec` — frozen dataclass: `event: GlossaryReadEvent`,
  `agent_label: str | None`, `report_path: str`.
- `glossary_read_report_path(event) -> str` — deterministic and I/O-free:
  `sase_subdir("glossary_read_reports") / f"{slug}-{hhmmss}-{digest}.md"`, where `slug`
  is a filename-safe form of the first recorded term, `hhmmss` comes from
  `sase.core.time.format_local(event.timestamp, ...)`, and `digest` is
  `sha256(event.id)[:8]`. Same shape as `tool_call_report_path`
  (`src/sase/ace/tui/tools/report.py:42-48`); stable across repeated selections of the
  same read.
- `build_glossary_read_report(spec) -> str` — the content builder:
  - Title plus the reproduced command line,
    `sase glossary read "<term>" ... [-d <depth>] -r "<reason>"`.
  - A recorded-metadata block: read time (local), agent name and, when `agent_label` is
    set, the family role label; project display name; requested terms; recorded
    related-term count; depth limit; definition bytes; and the recorded `source_path`
    (so the config file remains one copy-paste away).
  - The output itself: resolve the project with
    `resolve_glossary_cli_project(event.project)`
    (`src/sase/glossary/cli_common.py:39`), resolve the closure with
    `resolve_glossary_closure(resolved.catalog, resolved.compiled, event.terms, depth=event.depth_limit)`,
    and render it with `glossary_closure_markdown`. The header uses
    `resolved.project_name` — the configured display name, never a ProjectSpec key.
  - Never raises: `GlossaryCliError`, `GlossaryLookupError`, and `OSError` are caught
    and degrade to the metadata block plus a short note naming the failure and listing
    the recorded terms, so a renamed or deleted term still yields a useful report.
  - When the freshly resolved related-term count differs from
    `len(event.related_terms)`, append a one-line drift note. Definitions are
    re-rendered from the project's current glossary, and this makes that visible instead
    of silent.
- `write_glossary_read_report(spec) -> str | None` —
  `ensure_sase_directory( "glossary_read_reports")`, atomic `mkstemp` + `os.replace`
  write, prune to the newest 50 reports, return the path; `None` only on `OSError`.
  Directly mirrors `write_tool_call_report` (`src/sase/ace/tui/tools/report.py:52-90`).

### Step 3 — Register the hint against the report instead of the config path

- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`: add
  `glossary_reports: dict[str, GlossaryReadReportSpec] = field(default_factory=dict)` to
  both `HeaderHintState` and `AgentHintRender`. Defaulted, so no existing construction
  site (including the many in tests) has to change.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_glossary_reads.py`: build the spec from
  the event and `item.agent_label`, set `hint_mappings[n] = spec.report_path` and
  `glossary_reports[spec.report_path] = spec`. Change the registration condition from
  `event.source_path is not None` to a non-empty `event.terms`: the hint no longer needs
  a config path, and a read with terms always has renderable output.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_context.py`: drop the
  `GLOSSARY` branch from `_typed_context_value_path` so `clan_context_entry_hint_target`
  returns `None` for that lane.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_sections.py:280-311`: add a
  `glossary_spec` branch beside the existing `commit_spec` branch that pulls the first
  `GlossaryReadDisplayEvent` out of `entry.values` on a `GLOSSARY` lane and registers it
  the same way. Clan aggregation emits one row per term for a single read
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_clan_disk_aggregation.py:142-151`), so
  those rows share one report path; `parse_view_input` (`src/sase/ace/hints.py:186-192`)
  already dedups by path, so selecting several of them opens one report.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py`: pass
  `glossary_reports=header_hint_state.glossary_reports` through each
  `AgentHintRender(...)` return (the two early
  `AgentHintRender(file_hints={}, tool_call_reports={})` returns keep the default).

Registration stays pure in-memory dataclass construction — no stat, glob, or catalog
load on the render path.

### Step 4 — App-side plumbing and write dispatch

- `src/sase/ace/tui/actions/hints/_types.py`: declare
  `_hint_glossary_reports: dict[str, GlossaryReadReportSpec]`.
- `src/sase/ace/tui/actions/_state_init_navigation.py`: initialize it to `{}`.
- `src/sase/ace/tui/actions/hints/_files.py`: reset it in `_action_view_files_impl` and
  `_view_agent_files_impl`, and publish `hint_render.glossary_reports` in
  `_render_agent_hint_document` alongside the existing two dicts.
- `src/sase/ace/tui/actions/agents/_display_detail_render.py:255-262`: publish it in
  `_render_agent_detail_with_hints` so a repaint during hint mode keeps the mapping
  alive.
- `src/sase/ace/tui/actions/hints/_processing.py`: merge both report dicts when building
  `_PreparedViewRequest.report_items`, widen its element type, rename the private
  `_write_selected_tool_call_reports` to `_write_selected_hint_reports`, and dispatch
  per spec type (`GlossaryReadReportSpec` → `write_glossary_read_report`, otherwise
  `write_tool_call_report`). The write still happens inside the existing
  `asyncio.to_thread(...)` in `_finish_view_request`, keeping catalog loading off the
  event loop. Generalize the failure toast from
  `Failed to build tool-call report: {path}` to `Failed to build hint report: {path}`.

`@` (open in editor) and `%` (copy path) therefore act on the generated report, exactly
as they already do for slow tool-call report hints; the recorded `sase/sase.yml` path
stays visible inside the report's metadata block.

### Step 5 — Tests

- New `tests/glossary/test_read_report.py` (or the repo's matching existing location for
  `sase/glossary` tests, e.g. `tests/test_glossary_read_report.py`):
  - deterministic, I/O-free `glossary_read_report_path`, stable across calls;
  - report contains the reproduced command line, the recorded reason/agent/time, and the
    definitions of both a requested and a related term;
  - the project **display name** is used in the header, not a ProjectSpec key;
  - unresolvable project, unknown term, and a term deleted since the read each degrade
    to a metadata-only report with a note rather than raising;
  - the drift note appears when the current related-term count differs from the recorded
    one;
  - pruning keeps the newest 50 reports;
  - the write is atomic (no partial file left behind) and rewrites idempotently.
- `tests/ace/tui/widgets/test_agent_glossary_reads.py`: update
  `test_hint_state_maps_each_visible_event_and_aligns_reason` to assert report paths and
  populated `glossary_reports`; replace `test_event_without_source_path_gets_no_hint`
  with a terms-based gate — no terms, no hint; `source_path=None` with terms still gets
  one.
- `tests/ace/tui/widgets/test_agent_display_clan_context_hints.py`: GLOSSARY rows no
  longer resolve to `sase/sase.yml`; they register a report spec through the clan
  section path, and several term rows from one read share one path.
- `tests/ace/tui/actions/test_view_files_reports.py` (plus the shared
  `_view_files_helpers.py` harness): a glossary hint materializes its report and pages
  it; materialization runs off the event-loop thread; a mixed glossary + tool-call +
  plain-file selection preserves order; update the failure toast assertion at line 132.
- Confirm the ACE PNG snapshot suite is unaffected — hint numbering and row text are
  unchanged — and only refresh goldens if `just test-visual` proves otherwise.

### Step 6 — Docs

- `docs/ace.md`, the `SASE CONTEXT / GLOSSARY` bullet (~lines 3896-3904): replace the
  `source_path` sentence with the new behavior — the hint pages a generated report of
  that read's output (command line, recorded metadata, resolved closure), `@` opens the
  report and `%` copies its path, and the report names the `sase/sase.yml` source.
- `docs/memory.md` glossary section: add one sentence noting the ACE lane's hint shows
  the read's output, if it reads naturally there; otherwise leave it to `docs/ace.md`.

## Rejected alternatives

1. **Snapshot the rendered output at read time** (write the output next to the JSONL
   event and store a new `output_path` field). It needs a `schema_version` bump, and
   `_event_from_mapping` (`src/sase/glossary/read_log.py:271-273`) drops every row whose
   `schema_version` is not the current one — so bumping it would empty the GLOSSARY lane
   for every already-logged read. It also adds a file write to every
   `sase glossary read`, and older events would still need the re-render fallback.
   Rejected; the drift note in Step 2 covers the one advantage it had.
2. **An in-TUI modal like `CommitViewModal`.** More surface (modal, bindings, its own
   copy/edit semantics, its own multi-select handling) for the same content the pager
   path already delivers, including `@` and `%`.
3. **Rich/ANSI output in the report.** The pager runs `bat --color=always … | less -R`
   (`src/sase/ace/tui/actions/hints/_files.py:266-273`), which would surface raw escape
   sequences. Markdown is a genuine `sase glossary read -f markdown` output format and
   highlights properly in the pager.

## Notes and non-goals

- **No Rust core change.** Glossary closure resolution and rendering already live in
  shared Python (`src/sase/glossary/`) and back the CLI; the report builder joins them
  there rather than under `ace/`, so the TUI and any other frontend render identical
  content. The Rust matcher/validator behind `sase.core.glossary_facade` is untouched.
- **Fidelity.** Definitions are re-rendered from the project's current glossary, not
  captured at read time. The report states the recorded read time and flags related-term
  drift, which is the honest and retroactively-working trade (it fixes reads already in
  the log).
- Hint numbering, row text, and lane ordering are unchanged; only the hint's target
  changes.

## Verification

```bash
just install
just check
```

Plus targeted runs while iterating:

```bash
pytest tests/ace/tui/widgets/test_agent_glossary_reads.py \
       tests/ace/tui/widgets/test_agent_display_clan_context_hints.py \
       tests/ace/tui/actions/test_view_files_reports.py
```

Manual check in ACE: select an agent with a recorded `sase glossary read`, press `v`,
choose the GLOSSARY row's number, and confirm the pager shows that read's terms and
definitions; repeat on a clan container panel; confirm `@` opens the report and `%`
copies its path.
