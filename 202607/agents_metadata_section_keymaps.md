---
tier: tale
title: Agents metadata section navigation
goal: 'Ctrl+J and Ctrl+K cycle through the selected agent''s rendered metadata sections
  in either direction, placing every target title at the top of the metadata viewport
  without slowing the TUI navigation path.

  '
create_time: 2026-07-16 17:44:30
status: wip
prompt: 202607/prompts/agents_metadata_section_keymaps.md
---

# Plan: Add Agents Metadata Section Navigation

## Context

The Agents tab renders the selected agent's metadata, prompt, and reply as one Rich document inside
`#agent-prompt-panel`, hosted by the `#agent-prompt-scroll` `VerticalScroll`. The document is not a tree of section
widgets: ordinary agent fields, optional enriched metadata (`SASE PLAN`, output variables, commits, deltas, artifacts,
context, and slow calls), workflow/step views, prompt text, and reply/chat content are assembled from `Text`, `Group`,
responsive plan, Markdown, and Syntax renderables. Consequently, logical newline counts are not reliable viewport
coordinates: Markdown and responsive sections reflow with the pane width, and the full header can be replaced after the
cheap first paint by asynchronous enrichment.

`Ctrl+K` is also already the configurable `jump_to_entry_forward` binding on every tab. The new reverse-section shortcut
must take over that key only on Agents, while preserving jump-stack behavior on Artifacts and AXE. Both new shortcuts
must remain app-level configurable keymaps, participate in help and command-palette metadata, and defer to focused
prompt-input widgets that already own `Ctrl+J`/`Ctrl+K` locally.

This is presentation-only Textual behavior. No Rust core or backend change is expected. The TUI performance constraints
apply: a section-navigation keypress must perform only cached in-memory lookup and scrolling, with no disk access,
subprocess, document rebuild, or slow callback on Textual's serial message pump.

## Product Behavior

- On the first section-navigation keypress for the current metadata document, `Ctrl+J` jumps to its first rendered
  section and `Ctrl+K` jumps to its last rendered section.
- Once a section is active, either key moves relative to that shared active section: `Ctrl+J` selects the next title and
  `Ctrl+K` selects the previous title.
- Navigation wraps in both directions. Advancing from the final title selects the first; reversing from the first
  selects the final title.
- Every jump is immediate and positions the selected section title as the first visible content line in the metadata
  viewport. This includes the final section, even when it has too little body content to fill the remainder of the
  viewport.
- The cursor resets when the selected agent/document changes (including switching into or out of a pinned attempt view).
  Cheap-to-enriched rerenders, live-reply refreshes, and width-dependent reflow of the same document preserve the active
  section by stable identity when that section still exists; if it disappears, the next keypress uses the normal
  first/last initial behavior.
- Manual scrolling does not silently reinterpret the section cursor: the next shortcut continues from the last section
  selected by these keys. A metadata document with no marked section is a safe no-op.
- The actions always target `#agent-prompt-scroll`, regardless of whether file or tools content is also visible in the
  right pane. They do not change the selected agent, panel mode, file/tool state, hint mode, or entry-jump history.
- Outside the Agents tab, `Ctrl+K` retains its current jump-forward meaning. While a prompt editor or prompt-history
  control has focus, its existing local `Ctrl+J`/`Ctrl+K` behavior continues to win.

## Implementation Outline

### 1. Give rendered metadata headings stable, non-visual identities

- Introduce a small prompt-panel section-marker contract that attaches Rich style metadata to the title span while
  preserving its current text, color, underline, spacing, copy output, and snapshots. Do not infer sections by matching
  title strings in prompt or reply content.
- Route the existing major headings through that contract, including the standard `append_section_heading()` paths and
  headings currently emitted directly by specialized renderers. Cover all actual top-level metadata-document sections:
  responsive `SASE PLAN`; `OUTPUT VARIABLES`; `COMMITS`; `Deltas`; `Artifacts`; `WORKFLOW VARIABLES`; `SASE CONTEXT`;
  `SLOW TOOL CALLS`; `ERROR`; xprompt/prompt/reply/chat; workflow details/inputs/steps; bash/Python/parallel source and
  output sections; and pinned-attempt error/prompt/reply sections. Keep ordinary field rows such as `Name`, `Wait`,
  `Model`, and `Xprompts`, plus nested `SASE CONTEXT` lanes, out of the cycle because they are not standalone titled
  sections.
- Ensure alternate render paths carry the same marker identities: cheap header-only paint, full/enriched paint,
  file-hint paint, workflow rendering, attempt-pinned rendering, and `ResponsivePlanSection`'s logical and
  width-responsive Rich output. Dynamic visible labels should map to stable semantic IDs so a harmless label/count
  change does not lose the current anchor.

### 2. Collect width-aware title rows during the existing Rich render pass

- Add a lightweight section-tracking render wrapper/visual owned by `AgentPromptPanel`. As Rich emits the already-needed
  wrapped segments, record the ordered `(section identity, rendered y)` anchors from the marker metadata for that exact
  content generation and width. Preserve `AgentPromptPanel.content` as the original renderable so zoom seeding, copy,
  editing, and existing renderable-inspection tests do not start seeing an internal wrapper.
- Cache only the latest valid ordered anchor set (and the width/generation needed to reject stale results). Do not scan
  plain text, call `render_lines()` over the full document, or perform a second Rich/Markdown render when a shortcut is
  pressed; long prompts and replies must not turn section navigation into an O(document) keystroke path.
- Reconcile the active semantic section against each refreshed anchor set. Preserve it across same-document enrichment
  or reflow, reset it for a new document identity, and discard a stale active identity that no longer renders.
- If a key arrives in the narrow interval after content/layout invalidation but before the new anchors have painted,
  retain the requested direction and perform at most one thin synchronous after-refresh retry. The callback must only
  consume the newly published cache and scroll; it must not await work or start rendering on the message pump.

### 3. Add deterministic section cycling and exact viewport alignment

- Expose one prompt-panel/detail helper that resolves the initial or adjacent target with modular arithmetic, updates
  the shared active identity only after a valid target is available, and returns its rendered title row.
- Add Agents-only next/previous action methods in the navigation mixin. Query `AgentDetail`, `AgentPromptPanel`, and
  `#agent-prompt-scroll`, resolve the target from the cached section index, and use Textual's virtual-region scrolling
  with `top=True`, vertical-only, no animation, and the correct child offset so container borders/padding do not create
  an off-by-one row.
- Provide enough non-copyable trailing layout reserve for Textual's clamped `max_scroll_y` to align the last title at
  the viewport top. Keep that reserve out of the Rich document itself, and make ordinary `G`/bottom scrolling continue
  to mean the end of real metadata rather than an empty synthetic tail. Recompute the reserve from the live metadata
  viewport height so split, expanded/info, swapped, and resized layouts behave consistently.
- Keep the action path read-only and disk-free. It should neither trigger the detail debouncer nor refresh the selected
  agent; current live/enrichment workers remain responsible for publishing new content and anchors.

### 4. Wire configurable, context-gated keymaps without breaking `Ctrl+K`

- Add `next_agent_metadata_section` and `prev_agent_metadata_section` app keymap fields with defaults `ctrl+j` and
  `ctrl+k` in `src/sase/default_config.yml`, `AppKeymaps`, binding metadata, and the fallback `DEFAULT_BINDINGS` list.
  Place the bindings together and give them user-facing next/previous section labels.
- Allow the intentional built-in `ctrl+k` overlap with `jump_to_entry_forward`, following the existing contextual
  duplicate-binding pattern. Extend `AceApp.check_action()` so metadata-section actions are active only on Agents and
  `jump_to_entry_forward` is inactive there; on other tabs the inverse applies. Keep user overrides independently
  configurable and covered by the duplicate-key validation behavior.
- Register the two commands as Agents-only Navigation entries in the command catalog/availability layer. Update the
  Agents help section and keybinding footer/catalog labels to show `Ctrl+J / Ctrl+K` as next/previous metadata section,
  remove the misleading Agents jump-forward hint, and leave the Artifacts/AXE help descriptions for jump-stack
  navigation unchanged.
- Verify widget-local priority explicitly so multiline prompt input still inserts a newline with `Ctrl+J`, single-line
  prompt input still opens history with `Ctrl+K`, and modal-local `Ctrl+K` bindings continue to work.

### 5. Document the navigation contract

- Update the Agents-tab navigation table and metadata-panel documentation in `docs/ace.md` with the initial-direction,
  shared cursor, exact-top alignment, and wraparound behavior. State that only rendered titled sections participate and
  that the key pair operates on the metadata pane even when a secondary file/tools pane is visible.
- Keep unrelated PR/AXE jump-stack docs intact. If keymap inventory or generated help assertions expose another
  user-facing registry, update it from the same configured action names rather than hardcoding key strings.

## Tests

1. Add prompt-panel unit tests for section marking and layout indexing.
   - Assert the real regular-agent, workflow, step, failed, hint, associated-plan, and pinned-attempt render paths
     expose their section IDs in visual order without marking ordinary metadata rows or user-authored matching text.
   - Render representative content at wide and narrow widths and prove anchors follow Rich's wrapped output, including
     the responsive `SASE PLAN` renderable and Markdown before a later heading.
   - Prove anchor collection piggybacks on the normal render generation and a navigation call does not rerender content,
     read artifacts, or invoke async enrichment.

2. Add an Agents-tab pilot test around a scrollable, multi-section real agent document.
   - From a fresh document, verify first `Ctrl+J` selects the first title; repeated presses visit each title and wrap
     final-to-first.
   - Reset the document and verify first `Ctrl+K` selects the final title; repeated presses move backward and wrap
     first-to-final.
   - After every press, assert the target title occupies the top content row, including the final short section. Cover
     both the ordinary split layout and metadata-only/expanded layout, plus a width change that forces reflow.
   - Verify changing agents or pinned-attempt document resets initial behavior, while a same-agent enriched/live
     rerender preserves the active section and uses its new row.
   - Verify zero-section/empty states no-op without changing selection or scroll state.

3. Extend keymap and dispatch coverage.
   - Assert YAML defaults, `AppKeymaps`, `_BINDING_META`, fallback bindings, and command metadata remain in sync and
     expose `ctrl+j`/`ctrl+k` under the new action names.
   - Assert the duplicate `ctrl+k` candidates dispatch previous-section only on Agents and jump-forward only on
     Artifacts/AXE, including custom key overrides.
   - Update Agents help/catalog tests and retain focused prompt-widget/modal tests proving their higher-priority
     `Ctrl+J`/`Ctrl+K` semantics are unchanged.

4. Run focused and full verification.
   - Run the new prompt-section/widget/navigation tests together with existing keymap, help, prompt-input, jump-stack,
     metadata rendering, AgentDetail, and TUI j/k reliability suites.
   - Run `just test-visual` and inspect any PNG differences; marker metadata should be visually inert, so golden changes
     should be unnecessary unless a deliberate help/footer string is covered.
   - Run `just install` before verification as required for ephemeral workspaces, then run `just check` before handing
     off the implementation.

## Risks and Edge Cases

- A second full render or full-strip scan on every shortcut would regress navigation latency for large transcripts.
  Render-pass anchor collection plus a generation/width cache is a requirement, not an optional optimization.
- Logical line offsets drift whenever Markdown, Syntax, tables, or plan fields wrap. Only offsets observed from the
  actual Rich render width may drive scrolling.
- Asynchronous header enrichment can insert multiple sections ahead of the active one. Tracking a semantic identity
  instead of a numeric index prevents the next keypress from skipping or repeating the wrong section.
- Textual clamps scrolling at the real content end. The last-section acceptance test must catch implementations that
  merely request `top=True` but leave the title lower in the viewport because no trailing alignment extent exists.
- Marker metadata must remain attached through styled `Text` headings, responsive custom renderables, Groups, and hint
  overlays without leaking into copied/plain content or changing appearance.
- The intentional `Ctrl+K` duplicate depends on tab gating. Tests must lock down binding order and `check_action()` so a
  future catalog/keymap refactor cannot make both actions fire or steal jump-forward outside Agents.
