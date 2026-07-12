---
create_time: 2026-07-08 21:18:04
status: done
prompt: 202607/prompts/aggregate_output_variables_metadata.md
tier: tale
---
# Plan: Aggregate agent-child SASE variables in the agent metadata panel

## Problem

Agents set named SASE variables via the `/sase_va` (`sase_var`) skill / `sase var set KEY=VALUE`. These are stored
per-agent-run in `agent_meta.json` as an `output_variables: dict[str, str]` and surfaced in the `sase ace` TUI as the
**OUTPUT VARIABLES** section of the agent metadata panel.

When a **root/family agent entry** (an entry that groups an agent family like `foo--0`, `foo--code`, `foo--commit`) is
selected, the panel only shows the variables set by the **first** family member. Any variables set by the other child
agents are silently dropped, even though they are already loaded in memory. The root effectively renders its own
`output_variables` (it is normalized to the first member, `foo--0`) and never looks at its children.

**Goal:** when multiple agent children of a root set SASE variables, aggregate and display all of them in the OUTPUT
VARIABLES section, attributed to the child that produced each — and make it intuitive, reliable, and beautiful by
reusing the visual grammar the metadata panel already uses for other family-aggregated sections.

## Where the bug lives

- Renderer: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`
  - `build_header_text()` calls `_append_output_variables_section(header_text, agent.output_variables)`.
  - `_append_output_variables_section()` takes a single `dict[str, str]` and never consults `agent.followup_agents`.
- Data model: `src/sase/ace/tui/models/agent.py`
  - `Agent.output_variables: dict[str, str]` — the selected agent's own variables.
  - `Agent.followup_agents: list[Agent]` — the child agents attached to a root (populated at load time in
    `src/sase/ace/tui/models/_agent_status_apply.py`). **Each child already carries its own `output_variables`.**

No loader, storage, or scan change is required: every child's variables are already in memory. This is purely a
presentation/aggregation gap.

## Inspiration: how the panel already aggregates across children

Two established patterns, both living in the Python TUI layer (`src/sase/ace/tui/`):

1. **ARTIFACTS** (`src/sase/ace/tui/widgets/prompt_panel/_agent_artifacts.py`) — the simplest analog. It iterates
   `for child in agent.followup_agents`, merges each child's artifacts onto the parent, and de-dupes.

2. **SASE CONTEXT lanes** (MEMORY / SKILLS / WORKSPACES) — the richer, more beautiful analog and the model for this
   feature:
   - `src/sase/ace/tui/agent_context_members.py` builds the ordered contributor list `(agent, *agent.followup_agents)`,
     de-dupes by cache key + normalized artifacts dir, and assigns each contributor a **compact role label** (`plan`,
     `q`, `coder`, `commit`, `fb0`, `@name`, ...) via `_compact_role_label(agent)`.
   - `src/sase/ace/tui/widgets/prompt_panel/_agent_context_common.py` provides the shared visual grammar:
     - a **role column** gutter (`_format_role_column`, width `ROLE_COLUMN_WIDTH=7`, truncation `ROLE_LABEL_LIMIT=6`,
       style `COLOR_ROLE = "italic #AF87FF"`), shown only when a lane has attributed rows;
     - a header **detail suffix** `· N agents` (via `count_phrase`) shown only when more than one agent contributed.
   - `src/sase/ace/tui/widgets/prompt_panel/_agent_memory_reads.py` shows the exact convention: compute
     `distinct_agents`, add `· N agents` to the header when `> 1`, set `show_role_column`, and pass a per-row
     `role_label`.

This feature will reuse those primitives so OUTPUT VARIABLES visually matches the rest of the panel ("reads as one
system").

## Design

### Behavior by contributor count

Collect **contributors** = the ordered list of `(role_label, agent)` for `agent` and each
`child in agent.followup_agents` whose `output_variables` is non-empty, de-duplicated the same way
`build_context_members` de-dupes (by cache key + normalized artifacts dir) so a synthetic root that shares a child's
artifacts dir is not counted twice. Role labels come from the existing `_compact_role_label`.

- **0 contributors** → render nothing (unchanged).
- **1 contributor** → render **exactly as today**: `OUTPUT VARIABLES` header, no `· N agents`, no role column, a flat
  `key: value` list sorted by key, multiline values indented. This keeps the common single-agent case byte-for-byte
  identical (existing tests must pass unchanged) and avoids adding noise when a leaf child is selected.
- **≥2 contributors** →
  - Header: `OUTPUT VARIABLES` + dim ` · N agents` detail (mirrors the MEMORY/SKILLS/WORKSPACES lanes).
  - Rows **grouped by producer** in contributor order (root first, then followups); **keys sorted within each
    producer**. Grouping-by-producer matches how variables are consumed (`agents["build"].result_path` — variables are
    owned by a producing agent) and makes the repeated role labels form a clean contiguous left gutter.
  - Each row gets a **role-column gutter** (`_format_role_column(label)`, style `COLOR_ROLE`) before `key: value`.
  - **The reliability win:** the same key set by two different children yields two attributed rows instead of one child
    clobbering the other. No variable is ever lost.
  - Multiline values keep the current indented-block treatment; the `key:` line carries the role gutter and continuation
    lines are indented past the gutter so they stay aligned.

Keep the existing per-token colors: header `bold #D7AF5F underline`, key `bold #87D7FF`, value `#5FD75F`, and the
standard major-section divider before the section.

### Why the role column instead of merge-with-suffix

Alternatives considered:

- _Last-writer-wins merge_ (like the `meta_*` workflow-variables path): loses data on key collisions across children —
  fails the "reliable" bar.
- _`#N` de-collision suffixing_ (like `aggregate_meta_fields`): keeps data but is cryptic and doesn't say _who_ set what
  — fails "intuitive".
- **Role-column attribution (chosen):** loses nothing, says exactly who produced each variable, and reuses the panel's
  existing family-aggregation look — intuitive, reliable, and beautiful.

Alternative layout noted but not chosen: per-producer sub-headers (`▸ coder` then indented vars). Rejected because the
role-column gutter is the convention already used by the sibling SASE CONTEXT lanes; reusing it maximizes visual
consistency and code reuse.

### Code organization

- **New file** `src/sase/ace/tui/widgets/prompt_panel/_agent_output_variables.py` with
  `append_agent_output_variables_section(text, agent)` housing both the aggregation and the rendering (mirrors the
  one-file-per-section convention of `_agent_artifacts.py`, `_agent_commits.py`, `_agent_deltas.py`, etc.).
  - Reuse `_compact_role_label` from `agent_context_members`.
  - Reuse the role-column primitives/constants from `_agent_context_common` (`ROLE_COLUMN_WIDTH`, `ROLE_LABEL_LIMIT`,
    `COLOR_ROLE`, and the role-column formatter). Note: `append_lane_row` is **not** reusable as-is because it always
    emits a timestamp and output variables are not timestamped events — add a small KV-specific row renderer that lays
    out `{role gutter}{key}: {value}` using the same width/label/color constants so the gutter matches the lanes
    exactly. Expose the currently-private `_format_role_column` (promote to a shared name or add a thin public wrapper)
    so both call sites share one implementation.
- **Shared divider:** `build_header_text` uses a private `_append_major_section_divider`. The new module needs it too;
  promote it into `src/sase/ace/tui/widgets/prompt_panel/_helpers.py` (e.g. `append_major_section_divider`) and update
  the header's existing callers. Low-risk mechanical refactor (the header already imports from `_helpers`).
- **Update** `_agent_display_header.py`: replace the
  `_append_output_variables_section(header_text, agent.output_variables)` call with
  `append_agent_output_variables_section(header_text, agent)` and delete the obsolete private helper.

### Scope of "children"

Aggregate over `agent.followup_agents` — the same child list ARTIFACTS and the SASE CONTEXT lanes use. A selected leaf
child has no `followup_agents`, so it continues to show only its own variables (the 1-contributor path). This matches
the user's ask ("multiple agent children set variables") and stays consistent with the rest of the panel.

## Performance

The aggregation iterates the small in-memory `followup_agents` list and their in-memory `output_variables` dicts — **no
disk I/O, no JSON parsing, no subprocess**. Per the TUI perf rules, it is safe to keep inline in `build_header_text`
(the detail-panel path is already debounced). It must **not** be routed through `build_detail_header_summary`; that
off-thread summary exists only for sections that read audit-log files (memory/skill/workspace), which output variables
do not.

## Rust core boundary

No `sase-core` change. This is presentation-only aggregation of already-loaded in-memory `Agent.output_variables`,
exactly like the existing family aggregation for artifacts / memory reads / skill uses / opened workspaces, all of which
live in `src/sase/ace/tui/`. `agent_context_members.py` explicitly documents that this attribution "does not move any
audit-log reading into the Rust core." Variable storage and mutation stay in Python core
(`src/sase/core/agent_output_variables.py`, untouched); the read-only Rust scan already surfaces `output_variables` and
needs no wire/API change. The new aggregation helper is a pure function that could be promoted to core with minimal
effort if a second frontend ever needs identical behavior — noted as a deliberate, consistency-driven decision, not an
oversight.

## Testing

### Unit tests — `tests/ace/tui/widgets/test_agent_display_output_variables.py`

Existing tests (single-agent) **must keep passing unchanged**, proving zero regression for the 1-contributor case:
`absent_when_empty`, `orders_before_artifacts_and_workflow_variables`, `filesystem_and_wire_render_identically`.

Add tests (use `make_agent(followup_agents=[...])`; set each child's `role_suffix` / `agent_family_role` / `agent_name`
so `_compact_role_label` yields a deterministic label — verify expected labels against the `_compact_role_label` logic):

1. **Two children, distinct keys** → all variables shown; header contains `· 2 agents`; role column present with each
   child's compact label.
2. **Same key set by two children (the core regression test)** → both rows present, attributed to different labels;
   assert the value that used to be dropped is now shown.
3. **Root has no own vars, ≥2 children do** → aggregated; contributor count equals the number of children with vars.
4. **Root + one child both set vars** → 2 contributors → role column shown.
5. **Single contributor** (root only, and separately one-child-only) → **no** `· N agents`, **no** role column; output
   identical to the legacy flat rendering (guards against noise regressions).
6. **Multiline value under multi-contributor** → continuation lines aligned past the role gutter.
7. **Ordering** → root's variables precede followups' (contributor order); keys sorted within each producer.

### Visual snapshot — the "beautiful" acceptance artifact

- Add one PNG snapshot (e.g. `agents_output_variables_multi_agent_120x40`) showing a selected family root with ≥2
  children setting variables, exercising the role column + `· N agents`. Model it on the existing agents-metadata
  snapshots (`tests/ace/tui/visual/test_ace_png_snapshots_agents.py`, e.g. `test_custom_role_labels_png_snapshot`).
  Generate the golden with `just test-visual --sase-update-visual-snapshots` and eyeball it for alignment/beauty.
- Existing single-agent metadata snapshots must remain byte-identical (proof of no regression).

## Validation

- `just install` first (ephemeral workspace), then `just lint` (ruff + mypy) and targeted
  `pytest tests/ace/tui/widgets/test_agent_display_output_variables.py` plus the new visual snapshot.
- Finish with `just check`. Be aware of known, environment-specific pre-existing failures unrelated to this change
  (llm_provider `default_effort`, `_lint-pyvision` unused-defs, and full-suite SIGTERM sandbox kills); confirm the
  change itself via targeted pytest subsets + the static gates.
- Do **not** edit any `memory/*.md`, `AGENTS.md`, or generated provider shims — not required and not permitted without
  explicit user approval.

## Deliverables

1. New `_agent_output_variables.py` section module (aggregation + attributed rendering, single-contributor fast path).
2. `_agent_display_header.py` updated to call it; obsolete inline helper removed.
3. Shared `append_major_section_divider` (and a shared role-column formatter) promoted for reuse.
4. Unit tests covering multi-child aggregation, key-collision reliability, ordering, and single-contributor
   no-regression.
5. One new multi-agent PNG snapshot; existing snapshots unchanged.

## Acceptance criteria

- Selecting a root whose children set variables shows **all** children's variables, each attributed to its producer.
- Same-named variables from different children both appear (nothing clobbered).
- Single-contributor rendering is unchanged (no role column, no `· N agents`).
- OUTPUT VARIABLES visually matches the SASE CONTEXT lanes' role-column + `· N agents` grammar.
- Lint, targeted tests, and visual snapshots pass; no Rust core or memory-file changes.
