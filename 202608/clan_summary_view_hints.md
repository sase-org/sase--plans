---
tier: epic
title: View hints for agent clan metadata panels
goal: 'Pressing `v` on a selected agent clan container annotates the clan metadata document in place — clan summary
  paths, member-attributed bodies, SASE CONTEXT entries, slow tool calls, and member commits all receive `[N]` hints
  that resolve to correct targets — instead of destroying the clan document and reporting "No files or commits found".

  '
phases:
  - id: plumbing
    title: Clan-aware hint render path and clan summary hints
    depends_on: []
    size: medium
    description: "plumbing: thread HeaderHintState, the cached clan section snapshot, and panel fold state through the
      clan branch of build_header_text into build_clan_detail_text; stop the hint render from appending an agent prompt
      tail or starting agent-header enrichment for synthetic clan rows; give clans a bounded hint-render cache key; let
      hint-preserving repaints and clan enrichment results reach clan containers; annotate the clan summary block with
      span-preserving file hints.

      "
  - id: sections
    title: Member-attributed clan body hints
    depends_on:
      - plumbing
    size: medium
    description: "sections: annotate the visible bodies of the clan ERRORS, REPLIES, PROMPTS, and variable sections with
      file hints, resolving each fragment against its own member's workspace and sharing one HintContentBudget across
      the whole document, so hints exist exactly where text is visible at the active fold level.

      "
  - id: context
    title: Structured SASE CONTEXT lane hints
    depends_on:
      - plumbing
    size: medium
    description: "context: register exact-path hints for individually rendered SASE CONTEXT entries by reading the typed
      objects already carried on ClanContextEntry.values, so PLAN, ARTIFACTS, and MEMORY entries hint their real
      absolute paths instead of guessing from label text.

      "
  - id: tools
    title: Clan slow tool call report hints
    depends_on:
      - plumbing
    size: small
    description: "tools: register tool-call report hints for the clan SLOW TOOL CALLS section the same way the per-agent
      section does, so a hinted clan tool call materializes a report file through the existing writer.

      "
  - id: commits
    title: Clan commits lane and commit view hints
    depends_on:
      - context
    size: medium
    description: "commits: aggregate member commits from already-loaded step output into a new COMMITS context lane and
      register each rendered commit as a commit-view hint so the clan document supports the commit viewer, raw diff
      opening, and SHA copying.

      "
  - id: resolve
    title: Worker-resolved clan hint path index
    depends_on:
      - context
      - sections
    size: medium
    description: "resolve: compute a token-to-absolute-path index off-thread during clan enrichment and consult it
      before workspace-relative fallback, so logical plan references and other paths printed by clan summary scripts
      resolve to the files they name.

      "
  - id: polish
    title: Documentation, footer, and end-to-end coverage
    depends_on:
      - sections
      - context
      - tools
      - commits
      - resolve
    size: small
    description:
      "polish: update the ace docs and help popup for clan view hints, audit hint numbering against the member jump
      gutter, add the clan keypath to the view-hints perf harness, and cover the whole `v` flow on a clan with an
      app-level test."
proposed_by: bbugyi200.athena.r3
create_time: 2026-08-01 08:34:53
status: wip
---

- **PROMPT:** [202608/prompts/clan_summary_view_hints.md](prompts/clan_summary_view_hints.md)

# Plan: View hints for agent clan metadata panels

## Problem

On the Agents tab, `v` (`view_files`) enters hint mode: the metadata panel re-renders with `[N]` markers in front of
file paths, commits, and slow tool calls, and the hint input bar accepts numbers (plus `@` to open in `$EDITOR` and `%`
to copy). Selecting a synthetic **agent clan** container row and pressing `v` today produces a warning toast, "No files
or commits found in agent details", and no hints — even though the clan metadata document is full of hint-worthy
content. For an epic clan the summary alone prints the plan path, the epic prompt path, the bead id, and the hosted bead
page URL.

The cause is structural, not a missing regex. `build_header_text()` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py` returns early for `agent.is_clan_container`, and that
early return drops the `hint_state` argument entirely. Nothing downstream of it can register a hint.

Four further defects follow from the same gap, and the current toast hides all of them:

1. `_update_display_with_hints_impl()` in `_agent_display_hints.py` has no clan branch. After the clan document is built
   it falls through to the ordinary agent body path, appends an `AGENT PROMPT` heading, finds no prompt file, and
   appends `No prompt file found.` to a synthetic row that can never have one.
2. That same hint-mode call site passes no `clan_snapshot`, no `clan_fold_level`, and no `clan_section_fold_overrides`,
   unlike `_update_display_impl()` and `update_header_only()`. So the annotated document is rebuilt from a fresh
   in-memory aggregation at the default collapsed level: every disk-backed section (`REPLIES`, `SASE CONTEXT`,
   `SLOW TOOL CALLS`, `PROMPTS`) disappears and the user's session fold level is ignored. The panel visibly changes
   under the user, then flips back when `_remove_hint_input_bar()` triggers a refresh.
3. The hint path calls `_start_agent_detail_header_enrichment_from_context(agent)` at the end, which `update_display()`
   deliberately never does for clan rows — it cancels that worker and starts clan section enrichment instead. Pressing
   `v` therefore starts per-agent enrichment against a synthetic container.
4. `_render_agent_detail_with_hints()` in `src/sase/ace/tui/actions/agents/_display_detail_render.py` is explicitly
   gated with `not current_agent.is_clan_container`, so even if hints existed, any repaint during hint mode would
   discard them.

## Design principles

These constraints are load-bearing; every phase must respect them.

**The hint render stays pure.** `_render_agent_hint_document()` runs as a pump-free task on the event loop, not on a
thread. It may do the bounded, memoized path work the existing renderer already does (`resolve_agent_workspace_dir`,
`resolve_file_path`), but it must not perform plan-store, bead-store, or VCS lookups. `agent_associated_plan.py` states
this contract explicitly for plan resolution: those lookups belong in the deferred worker. The `resolve` phase exists
precisely so correct resolution happens off-thread and the renderer only performs lookups in a precomputed index.

**Hints annotate exactly what is visible.** Clan sections are fold-aware: level 1 shows headings and counts, level 2
shows bounded triage previews, level 3 shows full bodies and per-entry rows. A hint may only be registered for content
rendered at the active level. At the default level 1 that means the clan summary is the only hint source, which is also
the case the user reported. Because hint numbering therefore depends on fold level, the hint-render cache key must keep
including fold level and overrides (it already does).

**Hints are numbered in document order.** `build_clan_detail_text()` emits header fields, the member roster, the clan
summary, `ERRORS`, variables, `REPLIES`, `SASE CONTEXT`, `SLOW TOOL CALLS`, then `PROMPTS`. Registering each hint as its
section renders keeps `[1]`, `[2]`, `[3]` reading top to bottom. Do not post-process or re-sort hint numbers.

**Hint markers and member jump digits are different systems.** The clan roster prints a fixed 0–9 jump gutter that must
not change. Hint markers are the inline yellow `[N]` form used everywhere else in the TUI. Never renumber, never reuse
the gutter, and never register hints on roster rows.

**Presentation stays in Python.** This is TUI rendering and annotation work, so it stays in this repo per the Rust core
backend boundary. The one piece of shared domain behavior involved — plan reference resolution — already lives behind
the Rust binding (`sase.sdd.plan_refs.resolve_plan_reference`); call it, do not reimplement it.

**Existing hint vocabulary is reused, not extended.** `HeaderHintState` already carries `hint_counter`, `hint_mappings`,
`workspace_dir`, `tool_call_reports`, and `commit_views`, and `AgentHintRender` already reports them plus
`header_enrichment_pending`. The clan document populates the same structures so
`src/sase/ace/tui/actions/hints/_processing.py` needs no new hint kinds.

## Non-goals

- **Whole-tribe detail panels.** Selecting a focused tribe panel makes `_get_selected_agent()` return `None`, so `v` is
  inert there for reasons unrelated to clans. Tribe summary hints are a separate follow-up; the shared helpers this epic
  adds should be written so a tribe phase could reuse them, but no tribe behavior changes here.
- **Live VCS fetching for clan commits.** The `commits` phase uses only the in-memory `meta_commits` step output that
  `agent_commit_groups()` already parses. It does not add the linked-repo live fetch the per-agent commits panel does.
- **Changing what clan summary scripts emit.** `clan_summary` stays an opaque bounded Rich markup string produced by an
  arbitrary script. This epic reads it; it does not add a structured contract to it.

---

## 1. Clan-aware hint render path and clan summary hints

This phase makes `v` behave correctly on a clan container and delivers the reported feature: file hints in the clan
summary. It is the foundation every later phase builds on.

### Thread hint state into the clan document

- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`: in the `agent.is_clan_container` branch of
  `build_header_text()`, forward `hint_state` to `build_clan_detail_text()`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`: add a keyword-only
  `hint_state: HeaderHintState | None = None` parameter to `build_clan_detail_text()`. Existing callers
  (`_update_display_impl`, `update_header_only`, and the tests under `tests/ace/tui/widgets/`) keep passing nothing and
  must render byte-for-byte identically.

### Fix the hint-mode call site

In `_update_display_with_hints_impl()` (`_agent_display_hints.py`), add a clan branch **before** the existing family
branch that mirrors what `_update_display_impl()` already does for clans:

- call `prepare_clan_section_snapshot(self, agent)` and `set_clan_disk_sections_required(...)` with
  `clan_disk_sections_for_fold_state(fold_level, overrides)` so the snapshot the annotated render reads is the same one
  the plain render reads;
- pass `clan_snapshot=get_cached_clan_section_snapshot(self, agent)`, `clan_fold_level`, and
  `clan_section_fold_overrides` into `build_header_text()`;
- publish the document with `self.update(self._prepare_cached_hint_renderable(clan_text))` so
  `hint_document_is_current()` and the annotated-document cache work for clans;
- return an `AgentHintRender` immediately — no `AGENT XPROMPT`, no `AGENT PROMPT`, no `No prompt file found.`;
- do **not** call `_start_agent_detail_header_enrichment_from_context()`. Call
  `_start_clan_section_enrichment_from_context(agent)` instead, and cancel the per-agent bead, linked-delta, and
  detail-header workers exactly as `update_display()` does for clans.

Apply the same substitution on the cache-hit path in `update_display_with_hints()`: when a cached clan entry reports
work still pending, restart clan section enrichment rather than agent header enrichment.

### Report pending enrichment instead of an empty result

`AgentHintRender.header_enrichment_pending` already means "more hints may still arrive", and
`src/sase/ace/tui/actions/hints/_files.py` uses it to suppress the "No files or commits found in agent details" warning.
Reuse it: for a clan, set it when `snapshot.loading_sections` is non-empty or when the sections required by the active
fold level are not yet in `snapshot.disk.loaded_sections`. `_files.py` then needs no change, and the warning fires only
when an enriched clan genuinely has nothing to hint.

### Give clans a bounded cache key

`_agent_hint_render_cache_key()` currently derives `agent_state_digest` from `_digest_parts(agent)` and `source_digest`
from `_source_digest(agent)`. `Agent.runtime_children` has no `repr=False`, so `repr()` of a clan container recursively
expands every member — an unbounded cost paid on every `v` press and on every `hint_document_is_current()` check during
hint mode. Add a clan branch that builds the key from bounded inputs instead:

- ordered member identities and their display statuses,
- `agent.agent_clan`, `agent.agent_clan_generation`, and a hash of `agent.clan_summary`,
- the clan snapshot revision (below),
- the fold level and overrides that are already part of the key.

To make snapshot changes observable cheaply, add `revision: int = 0` to `ClanSectionSnapshot` in
`src/sase/ace/tui/models/_agent_clan_sections.py` and increment it in `cache_clan_disk_snapshot()`
(`_agent_clan_aggregation.py`) when a worker result is merged; `prepare_clan_section_snapshot()` and
`mark_clan_snapshot_loading()` carry the current value forward. This is what makes the annotated document rebuild once
enrichment lands.

### Let repaints keep clan hints

In `src/sase/ace/tui/actions/agents/_display_detail_render.py`, drop the `not current_agent.is_clan_container` condition
guarding `_should_render_agent_detail_with_hints()`. `on_clan_section_snapshot_loaded()` already schedules the debounced
detail update, so once the guard is gone an enrichment result repaints with hints preserved and republishes
`_hint_mappings`, `_hint_commit_views`, and `_hint_tool_call_reports`.

### Annotate the clan summary

`build_clan_detail_text()` renders `agent.clan_summary` through `Text.from_markup()`. When `hint_state` is present,
route that `Text` through a span-preserving annotator instead of appending it directly. Do not write a new one: the
family renderer already solves this exact problem in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family_render.py::_family_text_with_hints`, which inserts hint
markers into plain text and shifts every source span by the width of the markers inserted before it. Lift that logic
into a shared, panel-independent helper (a new module such as
`src/sase/ace/tui/widgets/prompt_panel/_container_hint_text.py` is appropriate) and have both the family renderer and
the clan renderer call it. Keep the family renderer's observable output unchanged.

Resolve summary paths against a **clan hint workspace**: the clan container itself has no `workspace_dir` and no
`effective_workspace_num` (`_agent_tree.py` builds it with only `project_file` from the anchor member), so resolve using
the first clan member row that has a workspace, via the existing
`resolve_agent_workspace_dir(effective_workspace_num, project_file, workspace_dir)`. Store the result on
`hint_state.workspace_dir`. Correct resolution of logical plan references is deliberately deferred to the `resolve`
phase; this phase establishes the fallback.

Use the capped appender (`append_bounded_text_with_file_hints` / `bound_hint_content` from `_hint_caps.py`) with one
`HintContentBudget` created per hint render, so a pathological 32 KiB summary cannot blow the scan budget. The budget
object is created here and threaded through the clan renderer for later phases to share.

### Tests

Extend `tests/ace/tui/widgets/` with a clan hint test module modeled on `test_agent_display_family_hints.py`, using the
existing clan fixtures in `tests/ace/tui/widgets/_agent_display_clan_helpers.py` and `FakePromptPanel`:

- a clan whose summary contains file paths produces ordered `file_hints` and `[N]` markers in the rendered plain text;
- the annotated clan document contains no `AGENT PROMPT` heading and no `No prompt file found.`;
- rendering with hints at a given fold level and snapshot produces the same section structure as rendering without hints
  at that same level (i.e. `v` does not collapse or drop sections);
- summary styling survives annotation — a styled span in the summary still covers the same visible characters after
  markers are inserted;
- `header_enrichment_pending` is `True` while clan sections are loading and `False` once the snapshot is enriched.

Also add an app-level test alongside `tests/ace/tui/actions/test_view_files_agent_hints.py` asserting that pressing `v`
on a selected clan container mounts the hint bar and does not emit the "No files or commits found" warning when the
summary has paths.

---

## 2. Member-attributed clan body hints

`append_errors_section`, `append_text_section` (used for both `REPLIES` and `PROMPTS`), and `append_variables_section`
in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_sections.py` render member-attributed text bodies. This
phase annotates them.

### What to annotate at which level

Follow the visibility rule strictly:

- **Level 1 (collapsed)**: heading and count only. No hints.
- **Level 2 (expanded)**: one bounded triage line per entry via `_append_triage_line`. Annotate the body text of each
  visible triage line, and nothing for entries suppressed by `_TRIAGE_ENTRY_LIMIT` or reported by the `+N more` tail.
- **Level 3 (fully expanded)**: full bodies via `_append_full_body`, grouped per member. Annotate every visible line.

For `ERRORS`, annotate the message body; the traceback rendered by `_append_traceback` is already highlighted per line
through `Syntax.highlight()`, so annotate it with the same span-preserving helper the summary uses, or leave the
traceback un-annotated if preserving its highlighting proves fragile — state which you chose in the phase notes rather
than silently dropping it.

For variable sections, annotate the rendered value text only (the previews at level 2, the value lines at level 3),
never the variable names.

### Per-member workspace resolution

Each entry carries `member_identity`. `clan_section_member_rows(agent)` returns the matching `Agent` rows, so build one
identity-to-row map per render and resolve each member's workspace lazily, exactly as `_family_member_hint_workspace()`
does: only call `resolve_agent_workspace_dir()` when the fragment actually contains a path match, and reuse the resolved
value across all fragments for that member. A member with no resolvable workspace falls back to the clan hint workspace
from the `plumbing` phase.

### Shared budget

All of these fragments consume the single `HintContentBudget` threaded through `build_clan_detail_text()`. A clan can
hold dozens of full reply bodies, so the truncation notice (`HINT_TRUNCATION_MESSAGE`) must be able to appear inside a
clan document just as it does inside a family document.

### Tests

- Level 2 annotates only the triage lines actually rendered, and the count of hints matches the number of visible
  entries with paths.
- Level 3 annotates full bodies, and a path appearing in two different members' bodies resolves to two different
  absolute paths under those members' respective workspaces.
- Total hints stay bounded and the truncation notice appears when member bodies exceed the shared budget.

---

## 3. Structured SASE CONTEXT lane hints

`ClanContextEntry` already carries `values: tuple[object, ...]` — the typed objects the disk aggregation accumulated for
that entry (`AssociatedPlanSummary`, `ArtifactFilePath`, `DeltaEntry`, `LinkedDeltaGroup` entries,
`MemoryReadDisplayEvent`, `SkillUseDisplayEvent`, `OpenedWorkspaceDisplayEvent`, `BeadSummary`). Those objects hold real
resolved paths, so context hints need no regex and no guessing.

### Where hints attach

`append_context_section()` renders lanes at three levels. Only level 3 renders one addressable bullet per entry; level 2
renders a comma-joined digest of up to four labels per lane. Register hints only for the level-3 per-entry bullets,
placing the `[N]` marker immediately after the `•` glyph and before the label.

### Mapping values to hint targets

Add a small pure resolver — in `_agent_display_clan_sections.py` or a focused sibling — that maps one `ClanContextEntry`
to an optional absolute path:

- `PLAN` entries: `AssociatedPlanSummary.actual_path`.
- `BEAD` entries: `BeadSummary.actual_plan_path` when set, matching how the per-agent `BEAD` lane hints in
  `_agent_context.py::append_bead_lane`; skip the entry when it is unset.
- `ARTIFACTS` entries: `ArtifactFilePath.actual_path` for artifact files; for `DeltaEntry` values, join the delta path
  onto its owning workspace directory, mirroring `_agent_deltas.py`. In-memory plan-path seeds contributed by
  `minimal_context_lanes()` keep their existing string value.
- `MEMORY` entries: the memory event's canonical path.
- `SKILLS` and `WORKSPACES` entries: no file hint (a skill name is not a file, and a workspace directory is not
  something the pager or `$EDITOR` should be handed). Leave them unmarked.

When an entry holds several values that resolve to different paths, register the first and leave the rest reachable
through the per-member agent rows — do not silently expand one visible row into several hint numbers, which would break
the "one visible marker, one hint" contract.

### Tests

- A level-3 `PLAN` lane entry hints the plan's `actual_path`, not its display path.
- Artifact and delta entries hint absolute paths under the correct workspace, including a linked-repo delta group.
- Skills and workspaces entries render without markers.
- Level 2 renders lane digests with no markers at all.

---

## 4. Clan slow tool call report hints

The per-agent `SLOW TOOL CALLS` section registers a hint per eligible call in
`src/sase/ace/tui/widgets/prompt_panel/_agent_slow_tools.py::_register_tool_call_report_hint`: it allocates a hint
number, computes `tool_call_report_path(entry)`, maps that path into `hint_state.hint_mappings`, and records a
`SlowToolCallReportSpec` in `hint_state.tool_call_reports` so
`src/sase/ace/tui/actions/hints/_processing.py::_write_selected_tool_call_reports` can materialize the report file
off-thread when the user selects it.

The clan section (`_agent_display_clan_sections.py::append_slow_tool_calls_section`, rendering `ClanSlowToolEntry` rows
through `_append_slow_tool_line`) does none of this. Give it the same treatment:

- register a hint only for entries whose `call.entry.status` is `success` or `failure`, matching
  `_is_report_hint_eligible`;
- pass the clan member's label as the report's `agent_name` so a materialized report names the member, not the synthetic
  clan;
- register only for rows actually rendered — the top `_TRIAGE_SLOW_TOOL_LIMIT` rows at level 2, all grouped rows at
  level 3, nothing at level 1;
- place the marker on the row so the existing column layout stays readable. The per-agent section reserves a fixed-width
  marker cell (`_hint_marker_width` / `_append_hint_marker_cell`) so rows stay aligned when hint numbers reach two or
  three digits; the clan row is a simple bullet line, so inserting `[N] ` after the bullet glyph is sufficient — but
  verify against a clan with more than nine hints that the result still reads cleanly.

Reuse the existing `SlowToolCallReportSpec` and `tool_call_report_path` helpers rather than adding a clan-specific
report format.

### Tests

Model on `tests/ace/tui/widgets/test_agent_slow_tools_hints.py`: a clan with slow calls across two members produces one
hint per eligible visible row, each mapped to its report path with a spec whose `agent_name` is the member label;
running and interrupted calls get no hint; level 1 produces none.

---

## 5. Clan commits lane and commit view hints

Clan documents currently show no commits at all. Member commits are excluded from the clan `WORKFLOW VARIABLES` section
on purpose — `_COMMIT_META_KEYS` in `_agent_clan_sections.py` filters `meta_commits`, `meta_new_commit`,
`meta_commit_message`, and `meta_commit_cwd` out of the variables aggregation — which is exactly the space this phase
fills.

`agent_commit_groups(agent)` in `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py` parses commits from in-memory
`step_output` only and performs no disk I/O, so clan commit aggregation is cheap and available before disk enrichment
completes.

### Aggregate

Add `COMMITS` to `CLAN_CONTEXT_LANE_ORDER` in `_agent_clan_sections.py`, positioned immediately before `ARTIFACTS` (the
per-agent `ARTIFACTS` lane prints its `Commits:` block first, so this keeps the reading order familiar). Populate it
from member rows in both `aggregate_clan_context_lanes()` (`_agent_clan_disk_aggregation.py`) and
`minimal_context_lanes()` (`_agent_display_clan_sections.py`), keyed by commit SHA so the same commit reported by two
members appears once with both member labels. Entry `label` should read `<short-sha> <subject>`; entry `values` should
hold the `CommitViewSpec`.

`_agent_commits.py` keeps its commit line and group types private, and Symvision lints private misuse across modules.
Export a small public accessor from that module — for example `agent_commit_view_specs(agent)` returning ordered
`(CommitViewSpec, repo_name, repo_kind)` records — and consume that from the aggregation rather than reaching into
`_AgentCommitGroup` or `_CommitLine`.

### Render and hint

Render `COMMITS` through the same fold-aware `append_context_section()` path as every other lane, and register a commit
hint for each level-3 entry: allocate a number, write `hint_state.commit_views[n] = spec`, and print the marker. This is
the whole integration — `_processing.py` already routes `commit_views` selections to `CommitViewModal`, prepends raw
diff paths for `@`, and copies SHAs for `%`.

### Tests

- Two members reporting the same SHA produce one entry attributed to both.
- A level-3 commits lane registers `commit_views` (not `file_hints`) with the member's `cwd` and diff path intact.
- Selecting a clan commit hint opens the commit view modal; selecting it with `@` resolves the raw diff path — assert
  through the same seams `tests/ace/tui/actions/test_view_files_commits.py` uses.
- Commits appear before disk enrichment completes, via the minimal in-memory lanes.

---

## 6. Worker-resolved clan hint path index

After the `plumbing` phase, clan summary paths resolve workspace-relative. That is wrong for the most valuable targets
an epic clan summary prints. `src/sase/scripts/sase_clan_summary_epic.py` renders the plan document with
`render_plan_document(...)`, whose `Path:` row prints `PlanDisplay.display_path` — the caller-supplied reference,
typically a logical `plans:<relative>` form. `FILE_PATH_RE` matches the `<relative>` remainder and `resolve_file_path()`
then joins it onto a member workspace, producing a path that does not exist. The `Prompt:` row has the same shape.

Correct resolution requires plan-store lookups, which the render path may not perform. Resolve off-thread instead.

### Build the index in the worker

`build_clan_disk_snapshot()` in `_agent_clan_aggregation.py` already runs on a worker thread and already receives the
clan container `agent`. Have it produce a `hint_paths: Mapping[str, str]` index — matched token to absolute path — and
carry it on `ClanDiskSnapshot`. Populate it from two sources:

1. **Known context paths.** Index the display and actual paths of everything the context aggregation already resolved
   (plan summaries, artifact files, deltas, memory reads). Index both the full string and its trailing path components,
   so a token that is a path-component suffix of a known target resolves to that target.
2. **Logical plan references in the clan summary.** Scan `agent.clan_summary` for `<kind>:<relative>` references, parse
   them with the pure `sase.sdd.plan_refs.parse_plan_reference` binding, and resolve them with
   `resolve_plan_reference(value, workspace_dir=..., workspace_num=...)` using the clan hint workspace. Resolution is
   best-effort: any failure leaves the token out of the index rather than raising, exactly as the clan summary script
   itself degrades.

Because `build_agent_group_disk_snapshot()` is shared with tribe enrichment, compute the index in the clan-specific
`build_clan_disk_snapshot()` wrapper and let the shared primitive default it to empty.

### Consume it in the renderer

Add a resolution hook to the hint appender used by clan content: before falling back to
`resolve_file_path(path, workspace_dir)`, look the matched token up in the index (exact match first, then path-component
suffix match). Keep the fallback so behavior degrades to today's semantics when the index has no answer and while
enrichment is still pending. Because the index lives on the snapshot and the snapshot revision is part of the hint cache
key from the `plumbing` phase, the annotated document rebuilds with better targets once enrichment lands.

### Tests

- A summary containing `plans:<relative>` hints the resolved plan file, not a workspace-relative join.
- A summary path that matches a known artifact's trailing components hints that artifact's absolute path.
- An unresolvable token still gets a hint, resolved through the workspace fallback (no silently dropped markers).
- The renderer performs no plan-store lookup: assert the resolver seam is untouched during a hint render.

---

## 7. Documentation, footer, and end-to-end coverage

### Docs and help

- `docs/ace.md`: in **Clan and Family Detail Panels**, document that `v` annotates the clan document, which sources
  produce hints (summary, member bodies, context entries, slow tool calls, commits), and that hint availability follows
  the active fold level — level 1 hints the summary only. Update the `v` rows in the two keymap tables if their wording
  no longer covers clans.
- The `?` help popup must stay in sync with actual functionality per the ace subcommand guidelines. Update the `v`
  entry, honoring the 32-character keybinding description limit and the 57-character box width.
- Footer: per the footer convention, a keymap appears only when it is conditional. `v` is a global Agents-tab binding
  and is not listed today, so the default is to add nothing. If the clan branch of
  `src/sase/ace/tui/widgets/_keybinding_bindings.py` (which returns early for clan containers) is changed at all,
  confirm the footer still shows the same set for clans.

### Numbering audit

Verify against a real clan with more than ten hints that inline `[N]` markers are never confused with the roster's fixed
0–9 member jump gutter: the gutter digits must stay unchanged in hint mode, and roster rows must carry no markers. If
the gutter and markers read ambiguously at level 3, adjust marker placement — not the gutter.

### Perf

`tests/perf/tui_trace/view_hints.py` drives the traced `v` keypath and `tests/perf/check_view_hints_regression.py` gates
it. Add a `clan_container_press` step exercising a multi-member clan with a large summary. The committed
`tests/perf/baselines/view_hints_baseline.json` is a frozen pre-optimization capture, so gate the new step with an
absolute `max_ms` only and no `max_baseline_ratio`; do not rewrite the baseline. Confirm `just view-hints-perf-check`
passes and that the clan press stays off the unbounded-`repr` path the `plumbing` phase removed.

### End-to-end coverage

Add one app-level test that drives the full loop on a clan container: select the clan, press `v`, submit a hint number,
and assert the resulting action (pager, editor, clipboard, or commit modal, depending on the hint kind chosen). Cover
the enrichment race explicitly — submit a hint number while clan enrichment is still in flight and assert the submission
waits on `_agent_hint_render_ready` and then resolves against the published mappings, the same guarantee
`_submit_agent_view_when_ready()` gives regular agents.

Consider a PNG snapshot for a clan panel in hint mode alongside the existing `agents_clan_panel_epic_*` goldens in
`tests/ace/tui/visual/snapshots/png/`; add it only if the annotated document renders deterministically, and accept new
goldens with `--sase-update-visual-snapshots`.

---

## Verification for every phase

Run `just install` first — workspace directories are ephemeral and dependencies may have changed — then `just check`
before finishing. Phases that touch clan rendering must also run the visual suite (`just test-visual`) because clan
panel goldens exist; unchanged rendering must leave them passing.
