---
tier: tale
title: Render shell-aware metadata for agent families
goal:
  Make the agent-family metadata panel describe every SASE shell accurately and present
  monitor commands or reasons responsively without changing agent-shell model metadata.
size: medium
proposed_by: bbugyi200.athena.0aj
create_time: 2026-08-22 11:33:24
status: wip
---

# Plan: Render shell-aware metadata for agent families

## Outcome

When the selected row is an agent family, its metadata header will show a `Shells:`
field whose ordered lanes represent both agent shells and proc shells. Agent-shell lanes
will retain their provider/model, effort, and alias presentation. Monitor lanes will
never invent or display a model: they will show the monitored command when it fits on
one rendered line and otherwise show the monitor reason in a readable wrapped block.
Selecting one concrete agent shell will continue to show the existing single-line
`Model:` field unchanged.

## Interaction and visual contract

- Use the selected row's identity as the semantic switch. A family container always
  renders `Shells:`, even during a partial or compatibility projection; a concrete agent
  shell continues to render `Model:`. Do not infer the label from the number of
  currently loaded members.
- Keep the established family ordering, suffix gutter, 12-lane cap, provider/model
  colors, reasoning-effort suffixes, and model-alias chips for agent shells. Rename the
  overflow copy from “members” to “shells” while retaining the `FAMILY MEMBERS`
  navigation pointer.
- Represent lanes as typed shell presentation data instead of a tuple that assumes every
  value is a model. An agent-shell lane owns a rendered model value (falling back to the
  existing `default` treatment when its model metadata is absent). A monitor lane owns
  its command and reason and must never call the model renderer, even if a stale monitor
  record happens to contain model/provider fields.
- Render a monitor lane with the existing monitor glyph/accent so its proc-shell nature
  is visible at a glance. If the trimmed command is a single physical line and its Rich
  cell width fits the actual value column after the `Shells:` label, widest suffix
  gutter, separator, and glyph, show the command verbatim with a command/code accent.
  Treat multiline commands as long regardless of total character count.
- If the command does not fit, render a compact `why` marker on the monitor lane and put
  the normalized reason on an indented `↳` continuation block. Wrap at whitespace using
  Rich cell widths, keep continuation lines aligned after the glyph, hard-split only an
  individually overlong token, and use the shared reason palette. Give the reason block
  the remaining panel measure rather than constraining it to the narrow model value
  column. This should produce a hierarchy like:

  ```text
  Shells: --plan · CLAUDE(opus)
          --mon  · ⚙ why
            ↳ Full-suite verification before
              landing
  ```

- Handle incomplete historical metadata without crashing or falling back to a model: if
  a long/multiline command has no nonblank reason, wrap the command itself in the same
  continuation treatment as a last-resort diagnostic. Empty command and reason values
  receive a quiet, explicit unavailable placeholder.
- Preserve metadata ordering: the family `Shells:` block remains where its `Model:`
  block currently sits (after `Auto:` and before `Xprompts:`), while retry, wait, and
  timestamp fields remain unchanged.

## Implementation

1. Replace the model-centric family section in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_model_section.py` with a shell-aware
   section (and shell-oriented module/type/constant names). Build its lanes from the
   existing concrete family-shell projection so monitors stay in durable family order,
   but branch lane construction on `Agent.is_monitor` before any model formatting. Keep
   lane construction pure and in-memory for the TUI hot path.
2. Give the responsive shell section a deterministic logical-text representation for
   header inspection plus an actual-width Rich rendering path. Centralize display-cell
   measurement and monitor command/reason selection so the exact-fit boundary, Unicode
   cell widths, indentation, wrapping, lane limit, and hidden-shell tail cannot drift
   between logical and rendered forms. Do not add file reads, subprocesses, timers, or
   asynchronous work to rendering.
3. Update the agent metadata/header plumbing and responsive renderable union in
   `_agent_display_header_metadata.py`, `_agent_display_header.py`, and
   `_agent_display_header_renderable.py` to register the shell section under a
   shell-oriented responsive ID. Keep the existing `append_model_field` path solely for
   selected concrete agent shells and other non-family agent entries.
4. Replace and extend the focused widget tests (renaming the model-section test surface
   where appropriate) to cover mixed agent/monitor families, stable ordering and gutter
   alignment, agent model effort/alias styling, the 12-shell cap, family containers with
   a partial projection, and unchanged `Model:` behavior for selected agent shells. Add
   adversarial monitor cases proving that stale model metadata never leaks, an exactly
   fitting command stays on one line, a command one cell too wide or containing a
   newline switches to the reason, long prose wraps with the intended hanging
   indentation and no line overflow, and missing reason metadata degrades usefully.
5. Add or update a focused ACE PNG snapshot in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py` that keeps the
   family metadata panel visible with agent and monitor shells together at a
   representative narrow width. Use a deliberately long command and natural-language
   reason so the committed golden reviews the `Shells:` label, monitor styling, `why`
   treatment, and multi-line reason rhythm. Keep unit render tests as the exact proof
   for both short-command and fallback branches; update other family-panel goldens only
   where the label change intentionally affects them.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run the focused nonvisual widget tests for the shell section, metadata-header
   integration, and monitor prompt panel.
3. Regenerate only the intentionally affected PNG goldens with the repository's visual
   snapshot update flag, inspect the actual images and diff artifacts, then rerun those
   visual node IDs without the update flag to prove exact equality.
4. Run `just check`. If scoped selection escalates or reports unusual coverage, follow
   repository guidance and run `just check-full` through `/sase_monitor`.
