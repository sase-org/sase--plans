---
tier: epic
title: Sticky collapsible agent header panel on the Agents tab
goal: 'On the Agents tab, the selected node''s identity header (every field from the
  kind line through Timestamps, plus Fold where present) renders in its own always-visible
  panel above the scrolling metadata document whenever the metadata panel is shown.
  The panel is collapsed to two concise rows by default and expands to the full field
  list with a configurable `d` keymap. Agent clan nodes are excluded.

  '
phases:
- id: document
  title: Detachable identity header in prompt-panel documents
  depends_on: []
  size: medium
  description: 'document: prompt-panel builders can split the identity header out
    of the metadata document into an IdentityHeader (expanded and two-row compact
    forms) that travels with the document and is published to an optional sink; nothing
    attaches a sink yet, so rendering is unchanged.'
- id: keymap
  title: Inert toggle_agent_header keymap plumbing
  depends_on: []
  size: small
  description: 'keymap: add the `toggle_agent_header` app action on `d` end to end
    (config, dataclass, registry shared-key pairs, binding, availability, palette
    entry, action), gated so it stays unavailable until the panel exists.'
- id: panel
  title: Agent header panel widget, layout, and visual verification
  depends_on:
  - document
  - keymap
  size: medium
  description: 'panel: mount the bordered header panel above the metadata scroll,
    wire the prompt panel''s sink, sync visibility with the metadata panel, make `d`
    toggle it, then update help/docs/tests and refresh and inspect the PNG goldens.'
proposed_by: bbugyi200.athena.0pi
create_time: 2026-09-22 13:53:05
status: wip
bead_id: sase-16k
---

- **PROMPT:** [prompts/202609/sticky_agent_header_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sticky_agent_header_panel.md)
- **BEAD:** [sase-16k](https://github.com/sase-org/sase--beads/blob/main/pages/sase-16k/README.md)

# Plan: Sticky collapsible agent header panel on the Agents tab

## Context

Today the Agents-tab metadata panel (`#agent-prompt-panel` inside the
`#agent-prompt-scroll` `VerticalScroll`, composed by `AgentDetail` in
`src/sase/ace/tui/widgets/agent_detail.py`) renders one long document. Its first block
is the identity header built by `build_header_text()`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`): a kind line
(`AGENT SHELL` / `FAMILY` / `PROC SHELL`, via `append_kind_header`), then the fields
from `append_agent_metadata_fields()` (`_agent_display_header_metadata.py`): Name,
Owner, Weight, Capacity, Bead, agent page, Retry chain, Project/Patch/Step, Workspace,
Workflow, Machine/Remote/Feed error/Remote actions, Auto, Model (or the family `Shells:`
lanes), Xprompts, VCS, PID, BUG, Queue, Wait lanes, Retries/Fallback, Activity,
Timestamps. For family containers with fold enabled, a `Fold: N/M` line follows
(currently emitted after the runner QUEUE section). Tribe documents
(`append_tribe_header` in `_agent_display_tribe_header.py`) and top-level workflow
documents (`build_workflow_detail_renderable` in `_workflow_render.py`, rendered in a
worker thread) have their own equivalent header blocks. As soon as the user scrolls, all
of it leaves the viewport.

This epic moves that block into a separate sticky panel at the top of the right-hand
detail column. Everything here is presentation-only Textual/Rich code, so it stays in
this repo (no `sase-core` change).

## Design

### What the user sees

```
╭─ AGENT SHELL ─────────────────────────────────────────────────────╮
│  sase-16h.1 · MUSE(muse-spark-1.3-contributor) @ xhigh ← @med…    │
│  ⌘ #gh · ▣ #bd/work_phase_bead · ⏳ sase-16h.2 +1 · ⚡ PLAN        │
╰──────────────────────────────────────────────────────── ▾ d more ─╯
┌───────────────────────────────────────────────────────────────────┐
│  ▸ ◆ NEIGHBORS · 6          (metadata document, scrolls as today) │
```

- **Its own panel.** A `round` border in the node kind's accent color, dimmed to about
  50% strength so it never reads as a focus outline (full-strength gold is already the
  list's focus color). The border title (top left) is the kind label in the full accent
  (`AGENT SHELL` gold `#FFD700`, `FAMILY` `#00AFFF`, `PROC SHELL` `#5FD7FF`, `TRIBE` in
  that tribe's identity color, `WORKFLOW` `#AF87D7`; `GATE`, `MONITOR`, `STEP`, or
  `AGENT` for rows that have no kind line today, with colors taken from existing list
  styling constants). The kind line moves into the title, so it stops using a content
  row. The border subtitle (bottom right) says what the key will do: `▾ d more` when
  collapsed, `▴ d less` when expanded. The key comes from the live keymap
  (`key_display_name`) and is omitted when unbound.
- **Aligned with the body.** Horizontal padding `0 2` plus `scrollbar-gutter: stable`
  match the metadata panel, so text columns and right edges line up.
- **Collapsed by default: always exactly two rows**, each a single non-wrapping line
  that ends in `…` when truncated (`no_wrap=True`, `overflow="ellipsis"`). The fixed
  height keeps j/k from shifting the body up and down. Chips are joined by a dim `·` and
  reuse the existing field styles:
  - Row 1, who and how: name (kind name style; dim `unassigned`), then the model value
    (`model_value_text()` from `sase.llm_provider.model_label`, including effort and
    alias provenance) or, for families, a `N shells` summary. Then the auto-approve chip
    (`⚡ PLAN` / `⚡ TALE` / `⚡ EPIC`, same styles as `Auto:`) and, for remote rows, a
    machine chip (`⇄ <alias>`).
  - Row 2, what and state, each only when present: xprompt chips (glyph plus `#name`,
    using the `Xprompts:` glyphs and colors, no args, at most 3 then a dim `+N`); a
    queue chip (`Queue #p/n`); a wait chip (`⏳` plus the first wait lane's value,
    truncated, then `+N` when there are more lanes); a retry chip (`↻ n/max`, plus
    `fallback` when active); a feed-error chip for remote rows; activity; and, on
    fold-aware documents (family, tribe), a fold chip (the `FOLD_CHARS` glyph and `N/M`
    in the `FOLD_STYLES` color) so `z` presses show feedback without expanding.
  - When row 2 would be empty it falls back to a quiet dim context line: project ·
    `#workspace` · the first timestamp tag and time (for example `RUN 13:08:46`). The
    panel keeps its two-row shape.
  - Tribe: row 1 is the label (tribe identity style) and the status with count chip; row
    2 is composition · runtime · fold chip. Workflow: row 1 is the workflow name and
    model; row 2 is status and activity, falling back as above. Proc shell: row 1 is the
    name and project; row 2 is a shortened cwd and activity, falling back as above.
- **Expanded (`d`):** exactly the lines the inline header renders today, minus the kind
  line (now the title), with the same styles, width-responsive page/wait/shell lanes,
  and file hints. Family containers also get their `Fold:` line; tribes get Name,
  Status, Composition, Runtime, and Fold. Height is `auto` with `max-height: 50%` of the
  detail column; past that the panel scrolls with the mouse wheel and is not focusable.
- **Visibility:** shown only when the metadata panel is visible (native scroll or the
  `,/` search overlay) and the current document has an identity header. Hidden for clan
  documents, "No agent selected", and the `SECONDARY_ONLY` (files or LLM Calls only)
  layout, where the metadata panel takes back the space.
- **State:** the collapsed/expanded state is per session (defaults to collapsed on
  start) and holds across j/k, tribe focus, and layout changes. Toggling does not
  rebuild the document, because both forms are prebuilt. The body keeps its scroll
  position, a bottom-pinned body stays pinned, and the header's own scroll resets to the
  top when the metadata document identity changes.
- **Hint mode:** if the identity block received any file-hint markers (for example an
  `FBACK` timestamp path), the panel renders expanded while that hint document is shown,
  so every visible `[N]` stays selectable. The next plain document restores the user's
  choice.
- **Everything else keeps the full inline document:** the `Z` zoom modal (its own
  `AgentPromptPanel` never detaches), the `V` pager (separate builder), and tests or
  callers that invoke builders without detaching. The `,/` metadata search covers the
  scrolling body only, since header fields stay on screen.

### Architecture: the identity header travels with its document

Most render paths build `header_text` via `build_header_text()` and keep appending
sections to it before calling `self.update(Group(header_text, ...))`. The workflow
document is built in a worker thread. So instead of a separate widget recomputing the
header, which would duplicate inputs and risk drift, the **builder** splits the header
out and **attaches it to the document it came from**:

- New frozen dataclass `IdentityHeader` (new module
  `src/sase/ace/tui/widgets/prompt_panel/_identity_header.py`) with fields `kind_label`,
  `accent`, `expanded` (`AgentHeader`, i.e. `Text | AgentHeaderRenderable` with any
  page/wait/shell responsive lanes), `compact` (two-line `Text`), and `has_hints`, plus
  `inline_renderable()` (kind line and expanded block, used for zoom seeding).
- `AgentHeaderRenderable` (`_agent_display_header_renderable.py`) gains an optional
  `identity_header` slot/kwarg. In detached mode the builders always return the body as
  an `AgentHeaderRenderable` carrying the `IdentityHeader`, even with zero responsive
  sections. Its `plain`/`spans`/digest cover only the body.
- `AgentPromptPanel.update()` (`prompt_panel/__init__.py`) calls
  `find_identity_header(content)` (content itself, or the first carrier among a
  top-level `Group`'s renderables) and passes the result, or `None`, to the attached
  sink **before** the unchanged-digest early return. This runs on the UI thread and in
  the same call as the body update, so header and body can never come from different
  documents.

## Phase `document`: detachable identity header (prompt-panel package only)

Touch only `src/sase/ace/tui/widgets/prompt_panel/` and its tests. With no sink
attached, every existing render stays byte-identical: the existing tests and PNG goldens
must pass unchanged.

1. **Model and helpers** in `_identity_header.py`: `IdentityHeader`,
   `IdentityHeaderSink = Callable[[IdentityHeader | None], None]`,
   `identity_kind_for_agent(agent) -> (label, accent)` covering family, proc shell,
   agent entry, gate, monitor, workflow step (bash/python/parallel), top-level workflow
   (`AgentType.WORKFLOW`, not a child, not `appears_as_agent`, so the cheap and full
   paths agree), and the `AGENT` fallback. Also `find_identity_header(content)` and
   `strip_leading_document_chrome(text) -> (text, removed_chars)`. That last helper
   removes leading `\n`s and at most one leading light major divider (`"─" * 50`, the
   `append_major_section_divider` rule) with its trailing blank lines. It keeps heavy
   `━` roster/queue rules, which belong to their section headings.
2. **Compact builders** in a sibling `_identity_header_compact.py`:
   `build_agent_compact_lines(...)`, `build_tribe_compact_lines(...)`,
   `build_workflow_compact_lines(...)`, which produce the two rows described in Design.
   Inputs are only in-memory agent fields, the `DetailHeaderSummary`, the metadata
   result (reuse `_AgentMetadataFields.wait_section` lanes and `.shell_section` lanes
   instead of rebuilding them), and the effective fold level and scale. No disk I/O
   (`sase/memory/tui_perf.md` rules 1, 8, and 11). Keep new code in new modules so
   `_agent_display_header.py` and `_agent_display_header_metadata.py` stay well under
   the `toobig` limits.
3. **`build_header_text(..., detach_identity: bool = False)`.** When true:
   - Skip `append_kind_header`. Append the metadata fields into a separate identity
     `Text` with its own responsive-range map (page/wait/shell). Emit the family `Fold:`
     line into the identity block right after the metadata fields, and not after the
     runner QUEUE section. When not detached, keep today's order exactly.
   - Render the `Xprompts:` block from `summary.xprompts_used` whenever the cached
     summary has it, even on the `cheap=True` immediate path. It is in-memory data, and
     this stops the expanded header from growing about 150 ms after every j/k. Leave the
     non-detached cheap behavior unchanged.
   - Set `has_hints` when `hint_state.hint_counter` advanced during the identity build.
     Hint numbering order is unchanged because the identity is still built first.
   - Strip leading chrome from the body, shift the body's responsive ranges (bead, plan,
     slow tools) by the removed length, and return an `AgentHeaderRenderable` carrying
     the `IdentityHeader`. Factor the existing "ranges and sections into
     `AgentHeaderRenderable`" tail into a helper shared by the identity and body.
   - Clan containers keep returning early with no carrier.
4. **Tribe and workflow.** `build_tribe_detail_text(..., detach_identity=False)`: the
   TRIBE kind line becomes the title; Name, Status, Composition, Runtime, and Fold
   become the identity; the description opens the body. In the `cheap` detached case, if
   the body would be empty, render a single dim `⋯ loading…` line rather than a blank
   panel. `build_workflow_detail_renderable(..., detach_identity=False)`:
   `WORKFLOW DETAILS` becomes the `WORKFLOW` title, the fields through PID become the
   identity, and the ERROR / WORKFLOW VARIABLES / INPUTS / slow-tools / WORKFLOW STEPS
   sections form the body. Both are safe for the workflow worker thread because they
   only build objects.
5. **Attempt-pinned view** (`_render_attempt_pinned` in `_agent_display_attempts.py`):
   when detached, build the identity cheaply from the cached summary with the same
   identity builder, wrap the banner as the carrier, and leave the rest of the body
   unchanged.
6. **Panel plumbing** in `AgentPromptPanel` and its mixins:
   `attach_identity_header_sink(sink | None)`, a `detaches_identity_header` property
   (true only when a sink is attached), and a stored last-published identity with
   `inline_document_renderable()`, which returns
   `Group(identity.inline_renderable(), content)` when one is published, else `content`.
   Pass `detach_identity=self.detaches_identity_header` at every builder call site:
   `_agent_display.py` (header-only and tribe), `_agent_display_render.py` (main and
   clan paths), `_agent_display_hint_render.py`, `_workflow_display.py` (sync and
   worker), and the attempt-pinned path. Audit every `self.update(` in the package; each
   agent, tribe, or workflow document must carry an identity in detached mode, so no
   intermediate paint hides the header.
7. **Tests** (new, for example `tests/ace/tui/widgets/test_identity_header.py` and
   `test_identity_header_compact.py`):
   - For each node kind (agent shell, family with fold, proc shell, gate, monitor,
     workflow step, top-level workflow, tribe cheap and full, attempt-pinned, clan): the
     detached body excludes every identity line and its leading divider; the expanded
     identity has exactly the fields that the non-detached document shows in that region
     (same styles); the compact form is two lines with the expected chips and fallback;
     `has_hints` is correct in hint mode.
   - The Xprompts block renders on the cheap path when cached.
   - Responsive lanes still render at several widths in both the identity and the body.
   - A mounted `AgentPromptPanel` with a fake sink receives the identity on each update,
     `None` for clan and empty documents, and is called before the digest early return.
   - Non-detached output is unchanged for representative documents.
   - If `symvision` flags a public symbol that only the `panel` phase consumes, add an
     `--epic-symbol <epic_bead_id>(<symbol>)` entry to the `Justfile` Symvision
     invocation, per `sase/memory/symvision.md`.

## Phase `keymap`: inert `toggle_agent_header` plumbing

Add the app action `toggle_agent_header` ("Toggle Agent Header"), default `d`, following
the `toggle_attempt_view` pattern. It must do nothing visible until the `panel` phase
lands, so no beta feature flag is needed:

- `src/sase/default_config.yml` (`keymaps.app`, near `toggle_attempt_view`): add
  `toggle_agent_header: "d"` with a comment that it is Agents-only and shares `d` with
  `show_diff` (Artifacts), `toggle_axe_description` (Services), and
  `stitches_toggle_sdd` (Artifacts Stitches).
- `keymaps/app_keymaps.py` field, `keymaps/metadata.py` entry, and the `bindings.py`
  `Binding("d", "toggle_agent_header", "Toggle Agent Header", show=False)`.
- `keymaps/registry.py` `_CONTEXTUAL_APP_DUPLICATES`: add the pairs with `show_diff`,
  `toggle_axe_description`, and `stitches_toggle_sdd` (tab-disjoint), each with a short
  comment.
- `_app_action_availability.py` `check_app_action`: available only when
  `current_tab == "agents"`, the prompt input does not own keys, and
  `AgentDetail.header_toggle_available()` exists and returns true. Look the method up
  with `getattr` so it is unavailable until `panel` adds it. Unavailable means Textual
  tries the other `d` bindings, which are all tab-gated off on Agents, so the key is a
  no-op.
- Action `action_toggle_agent_header` on `AgentPanelDetailMixin`
  (`actions/agents/_panel_detail.py`): on the Agents tab, call
  `agent_detail.toggle_header_expanded()` when present.
- Command palette: register it in `commands/_app_metadata_actions.py` and gate it in
  `commands/_availability_agents.py` with the same availability, so it is hidden until
  usable.
- `docs/configuration.md`: document the key and add the shared-`d` rows to the
  shared-key allowlist table. Leave the help modal and `docs/ace.md` to `panel`, so
  nothing user-facing advertises the key early.
- Tests: extend `tests/test_keymaps_defaults.py`, `tests/test_keymaps_app_bindings.py`,
  `tests/test_keymaps_validation.py` (a user override of `d` must not conflict with the
  tab-disjoint actions), `tests/test_command_availability_agents_actions.py`, and
  `tests/test_command_palette_wiring.py`. Add a tab-scoping check like
  `tests/ace/tui/test_artifacts_description_modes.py`, confirming that `d` still
  triggers `show_diff` / `toggle_axe_description` / `stitches_toggle_sdd` on their own
  tabs.

## Phase `panel`: widget, layout, toggle, and visual verification

1. **Widget** `AgentHeaderPanel(VerticalScroll)` (new
   `src/sase/ace/tui/widgets/agent_header_panel.py`, `can_focus = False`), which
   composes one `Static(id="agent-header-content")`. API:
   - `show_identity(header | None)` renders `expanded` if
     `(expanded or header.has_hints)`, else `compact`.
   - It sets the border title, the subtitle hint, and the dimmed kind-accent border
     (`styles.border = ("round", Color.parse(accent).with_alpha(0.5))` or a background
     blend; confirm visually). It updates the border only when the kind changes.
   - It skips unchanged paints with `renderable_content_digest` over the shown content,
     title, subtitle, and accent, which keeps the 5 s slow-tool ticks and auto-refresh
     cheap.
   - Also `has_identity`, `is_expanded`, and `toggle_expanded()`.
2. **`AgentDetail`**:
   - Compose the panel (`id="agent-header-panel"`, initially `hidden`) inside
     `#agent-detail-layout` above `#agent-prompt-scroll`.
   - On mount, attach the prompt panel's sink to `_on_identity_header(header)`, which
     shows the header and then syncs visibility.
   - Add an idempotent `_sync_header_visibility()`: hidden unless
     `is_metadata_visible()` and the panel has an identity. Call it at the end of every
     method that changes metadata-scroll visibility or publishes content. That means
     `_show_active_metadata_scroll()` and `_hide_metadata_scrolls()` in
     `_agent_detail_panels.py` (which also cover the search overlay's
     `_apply_detail_layout_classes()` calls), plus `show_empty()`,
     `show_tribe_summary()`, `update_display()`, and `update_display_immediate()`. The
     last four publish before they adjust scroll classes, so a publish-time sync alone
     could leave stale visibility.
   - Add `header_toggle_available()` (panel visible) and `toggle_header_expanded()`. The
     latter flips the state, repaints from the stored `IdentityHeader`, and asks a
     bottom-pinned prompt panel to reapply its pin after refresh.
   - Reset the header's scroll to the top on `AgentMetadataIdentityChanged`.
3. **CSS** (`src/sase/ace/tui/styles.tcss`, near the `#agent-prompt-scroll` rules):
   - `#agent-header-panel { height: auto; max-height: 50%; border: round $secondary; padding: 0 2; scrollbar-gutter: stable; border-title-align: left; border-subtitle-align: right; }`,
     a `.hidden { display: none; }` rule, and
     `#agent-header-content { width: 100%; height: auto; }`.
   - Change `#agent-prompt-scroll.expanded` from `height: 100%` to `height: 1fr`, so an
     expanded metadata panel shares the column with the auto-height header instead of
     overflowing. The fr layouts (`layout-priority`, `layout-equal`) already split the
     remaining space.
4. **Zoom seeding**: `_zoom_seed_from_detail` and `_zoom_seed_for_tribe` in
   `actions/agents/_panel_detail.py` seed `metadata_renderable` from
   `prompt_panel.inline_document_renderable()`, so the zoom's first frame still has the
   header.
5. **Help and docs**:
   - Add `(d(a.toggle_agent_header), "Expand / collapse agent header")` to
     `modals/help_modal/agents_bindings.py`, next to the attempt-view row.
   - In `docs/ace.md`, add the `d` row to "Keybindings: Agents Tab". Add a "Header
     panel" subsection to "Agents Tab Metadata Panel" covering placement, collapsed and
     expanded contents, visibility rules, clan exclusion, hint-mode behavior, and search
     scope. Adjust the existing `FAMILY` / `AGENT SHELL` bullets, whose kind lines now
     appear as the panel title.
   - Update the note that the `Ctrl+J`/`Ctrl+K` top waypoint "reveals any ordinary
     header fields".
6. **Tests**:
   - Update Agents-tab pilot and visual-harness tests that read header fields (`Name:`,
     `Model:`, `Timestamps`, `AGENT SHELL`, `Fold:`, and so on) from
     `#agent-prompt-panel`. At least `test_prompt_panel_section_navigation_actions.py`,
     `test_prompt_panel_bottom_pin.py`, `test_agent_tribe_summary.py`, and the
     `tests/ace/tui/visual/test_ace_png_snapshots_agents*.py` wait conditions need this.
     Read the header panel, or use a small helper that joins header and body text.
   - Add pilot tests for:
     - default collapsed at two rows with the right title and hint;
     - `d` expanding to the full field list and back;
     - state persisting across j/k and tribe focus;
     - the body no longer containing identity lines;
     - hidden on clan selection and in `SECONDARY_ONLY`, visible with a split secondary
       panel and during `,/` search;
     - a hint document forcing expansion;
     - a bottom-pinned body staying pinned across a toggle;
     - the zoom seed including the header;
     - the key doing nothing while the prompt input bar is open.
7. **Visual verification** (read `sase/memory/tui_screenshot.md` first):
   - Capture live screenshots: `sase screenshot -o /tmp/agents_header.png`, a `--keep`
     session driven with `d`, j/k, a family row, a tribe panel, and a clan row. Inspect
     each PNG for alignment, colors, and truncation, and polish until it is clean.
   - Then run the full `just fix-tui-screenshots` through `/sase_monitor`. Most
     Agents-tab goldens change. Inspect every creation, removal, and update group per
     the golden-maintenance rules before finalizing.

## Verification

Each phase runs `just check` (via `sase tool run check`) before finishing. `document`
and `keymap` must leave every PNG golden unchanged. `panel` owns the golden refresh and
the live screenshot review.
