---
create_time: 2026-07-12 09:02:51
status: wip
prompt: 202607/prompts/prompt_stash_preview_pane.md
tier: tale
---
# Plan: Prompt Stash Preview Pane with XPrompt + Markdown Syntax Highlighting

## Product Context

The prompt stash picker (`StashedPromptsModal`, opened via `@` / `Ctrl+G p` / empty `Ctrl+S`) currently shows each
stashed prompt as a single truncated line — a 36-character first-line snippet next to age, project, and bundle chips. To
decide which prompt to restore (or which pinned row to overwrite), users must recognize a prompt from its first ~36
characters. That fails for prompts that start with a directive line, share a common prefix, or are multi-prompt bundles.

This plan adds a **preview pane on the right side of the stash picker** that renders the full text of the highlighted
entry with rich syntax highlighting:

- **Markdown elements** — headings, emphasis, lists, links, code fences — via the markdown lexer.
- **Native xprompt elements** — invocations (`#foo`, `#!foo`, `#foo(a, b)`, `#foo:arg`, `#foo+`), directives (`%wait`,
  `%m:opus`, `%auto`), alt fan-out branches (`%{a | b}`), and swarm/bundle segment separators (`---`) — layered on top
  using the _real_ xprompt parsers, so the preview highlights exactly what would expand at launch (tokens inside fenced
  code blocks and `%xprompts_enabled:false` zones stay literal).

Design goals, in priority order: **intuitive** (see the whole prompt before restoring, zero new concepts), **reliable**
(highlighting can never break the modal; graceful degradation on narrow terminals and oversized prompts), **beautiful**
(semantic colors consistent with the rest of ACE, markdown that reads like markdown, xprompt tokens that pop).

## Current State (facts)

- `src/sase/ace/tui/modals/stashed_prompts_modal.py` — `StashedPromptsModal`: fixed 96-col modal; `Container` → title
  `Label` + `OptionList` (height 18) + hints `Static`. No preview, and no `OptionHighlighted` handling. Bindings: `tab`
  pop-toggle, `space` pin, `a` all, `d` delete, digits `1`–`0` restore-by-index, plus `OptionListNavigationMixin` nav
  keys (`j/k/up/down/ctrl+n/ctrl+p/esc/q`).
- `src/sase/ace/tui/modals/prompt_stash_row.py` — single-line row labels with fixed column budgets (`_PREVIEW_WIDTH=36`,
  `_PROJECT_WIDTH=14`, `_AGE_WIDTH=9`, `_BUNDLE_WIDTH=10`) sized for the 96-col modal.
- `src/sase/ace/tui/modals/update_pinned_stash_modal.py` — `UpdatePinnedStashModal`, the "which pinned row do I
  overwrite?" picker; same layout, same limitation.
- Entries (`PromptStashEntryWire`) carry `text`, `frontmatter`, `project`, `created_at`, `pinned`, `source`. A "stash
  all" bundle is ONE entry whose `text` joins panes with `\n---\n` (`entry_prompt_segments` in
  `src/sase/ace/tui/prompt_stash_entries.py` splits it on restore).
- **Precedent**: `PromptHistoryModal` (`src/sase/ace/tui/modals/prompt_history_modal.py`) already uses a `Horizontal`
  split — list panel left, `Preview` pane right (`Label` + `VerticalScroll` + content `Static` + metadata `Static`),
  with a dynamic row-width budget (`prompt_preview_width_for_list_content`). Its preview is _plain unhighlighted text_.
- **No xprompt highlighter exists in the TUI.** Prompt bodies elsewhere render through
  `lazy_renderable(content, "markdown")` (`src/sase/ace/tui/util/lazy_syntax.py`: Rich `Syntax`, monokai theme,
  plain-text fallback above `MARKDOWN_SYNTAX_HIGHLIGHT_MAX_BYTES = 24_000`).
- **The canonical xprompt token grammar lives in Python** in `src/sase/xprompt/`:
  - `_parsing_references.py` — `XPROMPT_REFERENCE_PATTERN`, `iter_xprompt_references()` (markers `#`/`#!`, names incl.
    `ns/name`, hitl suffixes `!!`/`??`, paren/colon/shorthand/plus args).
  - `_directive_types.py` — `_DIRECTIVE_PATTERN`, `_KNOWN_DIRECTIVES` + aliases.
  - `_directive_alt.py` — `_ALT_DIRECTIVE_RE` for `%{a | b}` / `%alt(...)`.
  - `segment_separators.py` — `^---\s*$` separator regex.
  - `_fenced_blocks.py` — `fenced_block_ranges()` literal zones; `_disabled_regions.py` —
    `%xprompts_enabled:false … :true` zones.
- The pandoc PDF grammar (`src/sase/attachments/sase.xml`) already established the semantic mapping: separators →
  comment, directives → keyword, invocations → function, inline code → literal.
- `src/sase/history/prompt_metadata.py` — `summarize_prompt_for_preview()` returns
  `PromptPreviewSummary(vcs_tag, xprompts, directives)`; `_prompt_history_preview.py` has the aligned
  `append_metadata_row()` helper. Both are directly reusable.
- Perf machinery to reuse: `DetailPanelDebouncer` (`src/sase/ace/tui/util/debounce.py`, 150 ms, last-callback-wins).

**Rust core boundary check**: syntax highlighting is presentation-only (Textual/Rich rendering), and all token grammars
it consumes already live in this repo's Python xprompt modules. No `sase-core` changes are needed or appropriate.

## Design

### 1. Layout — split-pane stash picker

Follow the `PromptHistoryModal` pattern inside `StashedPromptsModal`:

```
┌─ Stashed Prompts ────────────────────────────────────────────────────────────┐
│ ┌─ list panel ──────────────────────────┐ ┌─ Preview ────────────────────┐   │
│ │ ✓ 📌 1  2h ago  [sase]  fix the …     │ │ %m:opus %wait:planner        │   │
│ │      2  5h ago  [sase]  # Review…     │ │                              │   │
│ │      3  1d ago  [chez]  3 prompts     │ │ # Review Checklist           │   │
│ │                                       │ │ Run #review(scope=diff) and  │   │
│ │                                       │ │ then %{ship | hold}…         │   │
│ └───────────────────────────────────────┘ │ --- Metadata ---             │   │
│ hints: tab pop · space pin · d delete ·   │ Project: sase · Pinned: yes  │   │
│        ctrl+d/u scroll preview            └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

- Container widens from fixed `width: 96` to responsive `width: 96%; max-width: 156` (unchanged `max-height: 90%`).
  Inside, a `Horizontal(#stashed-prompts-panels)` holds a left `Vertical` (title stays above; list + hints) and a right
  `Vertical` preview pane (`Label("Preview")` + `VerticalScroll` wrapping a body `Static` and a metadata `Static`).
  Split roughly 50/50 with a minimum list width that keeps chips + digit gutter readable.
- **Row width budget becomes dynamic**: `prompt_stash_row.py`'s fixed `_PREVIEW_WIDTH=36` becomes a parameter derived
  from the actual list content width (mirror the history modal's `prompt_preview_width_for_list_content` approach). With
  a full-text preview on the right, the in-row snippet can shrink without losing information.
- **Graceful narrow-terminal degradation**: when the available modal width can't fit both panels (below ~110 cols),
  toggle a CSS class on resize that hides the preview pane and restores today's exact single-panel 96-col layout.
  Nothing regresses on small terminals.
- **Preview scrolling**: `ctrl+d` / `ctrl+u` scroll the preview half a page (both keys are unbound in this modal today;
  vim-idiomatic and can't collide with list nav). Selection changes reset scroll to top. Update the modal's hints line
  accordingly (per ace help-doc conventions).
- `UpdatePinnedStashModal` gets the same preview pane by reusing the shared pane component — when overwriting a pinned
  stash row, seeing what you're about to destroy is exactly when a preview matters most.

### 2. Highlighting engine — markdown base + xprompt overlay

New module `src/sase/ace/tui/util/xprompt_syntax.py`:

- `highlight_prompt_text(text: str) -> rich.text.Text`:
  1. **Base layer**: `Syntax(..., "markdown", theme=monokai).highlight(text)` produces a styled `Text` (same lexer/theme
     the rest of ACE uses, so fences, headings, emphasis, and links look identical to other prompt renderings).
  2. **Literal-zone map**: compute protected ranges via `fenced_block_ranges()` and the
     `%xprompts_enabled:false … :true` disabled-region regex. No xprompt styling inside them — matching real expansion
     semantics is what makes the highlighting _trustworthy_.
  3. **Overlay xprompt spans** with `Text.stylize(style, start, end)` using the canonical grammar:
     - invocation references from `iter_xprompt_references()` (marker + name styled strong, arg payload styled lighter;
       `#gh:`/`#git:` refs are ordinary references and need no special case),
     - directive tokens from the directive pattern (known names + aliases; name strong, args lighter),
     - alt branches `%{ a | b }` (delimiters `%{`, `|`, `}` styled as structure),
     - segment separators `^---$` outside fences (styled as a distinct muted accent — these are bundle/swarm boundaries
       and should read as structure, not prose).
- **Style table** `XPROMPT_TOKEN_STYLES` as a module-level dict of named token → Rich style, the same shape as
  `QUERY_TOKEN_STYLES` (`src/sase/ace/query/highlighting.py`). Defaults follow existing ACE color language: invocations
  green (matches the history preview's "Workflows" metadata color), directives yellow (matches "Directives"), alt-branch
  delimiters magenta, separators dim bold. Exact shades tuned during implementation against the PNG snapshots.
- **Reliability rails**:
  - Size cap: above `MARKDOWN_SYNTAX_HIGHLIGHT_MAX_BYTES` (reused from `lazy_syntax.py`), skip highlighting and return
    plain `Text` — full content still visible.
  - Any exception inside highlighting degrades to plain `Text` (never propagate into the modal).
  - Pure function of the input string → trivially unit-testable and cacheable.

### 3. Preview pane content

New module `src/sase/ace/tui/modals/_prompt_stash_preview.py` (mirrors `_prompt_history_preview.py`), providing a small
reusable pane component + builders shared by both stash modals:

- **Frontmatter section** (only when `entry.frontmatter` is non-empty): rendered dim/yaml-highlighted above the body
  with a subtle rule, so users see the full restore payload. (Implementer verifies the stored frontmatter string format
  from `_captured_stash_frontmatter` before choosing the lexer.)
- **Body**: `highlight_prompt_text(entry.text)` — full raw text including bundle `---` separators (what you see is
  exactly what restores; separators are highlighted, not rewritten).
- **Metadata footer**: reuse `append_metadata_row()` + `summarize_prompt_for_preview()`: Project, Workflows, Directives,
  Prompts (bundle segment count when > 1), Created (relative age), Pinned, Source. Same visual dialect as the history
  preview.
- Placeholder text ("No prompt selected") when nothing is highlighted.

### 4. Event flow & performance (per tui_perf rules)

- Add `on_option_list_option_highlighted` to the modal → schedule the preview update through a `DetailPanelDebouncer`
  (150 ms): the list highlight paints immediately; a held `j`/`k` produces exactly one preview paint. Initial paint on
  mount for the first row.
- **Cache** highlighted renderables per entry id in the modal instance (entries are immutable for the modal's lifetime;
  pin/pop/delete toggles only change row labels). Re-highlights after `_refresh_rows()` rebuilds hit the cache and are
  effectively free; the debouncer additionally swallows programmatic `OptionHighlighted` echoes from rebuilds (watch
  perf-rule 8: if echoes still cause visible churn, add a synchronous guard flag around rebuilds).
- No disk I/O, no subprocess, no thread needed — entries are already in memory and highlighting a few-KB prompt is
  sub-millisecond; the cap handles pathological sizes.

## Implementation Steps

1. **`src/sase/ace/tui/util/xprompt_syntax.py`** — `XPROMPT_TOKEN_STYLES` + `highlight_prompt_text()` (markdown base,
   literal-zone map, xprompt overlays, size cap, exception fallback).
2. **`src/sase/ace/tui/modals/_prompt_stash_preview.py`** — preview pane component (frontmatter section, highlighted
   body, metadata footer, placeholder) shared by both stash modals.
3. **`stashed_prompts_modal.py`** — split-pane `compose()`, highlight handler + debouncer + cache, `ctrl+d`/`ctrl+u`
   preview scroll actions, narrow-terminal collapse on resize, hints-line update.
4. **`prompt_stash_row.py`** — make the in-row snippet width a computed budget from list content width instead of the
   fixed 36 (callers pass the budget; default preserves current behavior for the narrow-collapsed layout).
5. **`update_pinned_stash_modal.py`** — adopt the same split layout via the shared pane component.
6. **`styles.tcss`** — panel split CSS for both modals (responsive container width, list/preview panel sizing, preview
   label/body/metadata/frontmatter styles, `-narrow` collapse class).
7. **Docs/help**: modal hint lines (done in 3/5). No new global keybindings → no `default_config.yml` or help-modal
   changes; footer conventions untouched (modal-internal keys).

## Testing

- **Unit — highlighter** (new test module mirroring `xprompt_syntax.py`): span/style assertions for each token class —
  `#foo`, `#!foo`, `#ns/name`, `#foo(a, k=v)`, `#foo:arg`, `` #foo:`quoted` ``, `#foo+`, hitl suffixes, `%wait:x` /
  `%w`, `%m(...)`, unknown `%notadirective` (unstyled), `%{a | b}`, `^---$` separators; **negative cases**: tokens
  inside fenced code, inside `%xprompts_enabled:false` zones, `# Heading` not an invocation, mid-word `#`/`%`; size-cap
  and exception fallbacks return plain text.
- **Unit — preview builders**: metadata rows (project placeholder, bundle count, pinned), frontmatter section presence,
  placeholder state.
- **Modal behavior** (extend `tests/ace/tui/modals/test_stashed_prompts_modal.py` and the update-pinned twin): preview
  follows j/k highlight (after debounce), scroll keys move the preview and reset on selection change, pin/pop/delete
  toggles keep preview stable, bundle entries render full multi-segment text, narrow-terminal collapse hides the pane
  and restores current row widths.
- **PNG visual snapshots** (extend `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py`):
  - stash modal with a markdown+xprompt-rich entry selected (headings, fence, `#invocation(args)`, `%directives`,
    `%{alt | branches}`) — the "beautiful" gate;
  - a multi-prompt bundle entry with highlighted `---` separators + "Prompts: N" metadata;
  - narrow-terminal collapsed layout (guards the degradation path);
  - update-pinned modal with preview.
- Full repo gates: `just check`.

## Risks & Mitigations

- **Overlay drift vs. real parser behavior** → mitigated by consuming the canonical parsing modules
  (`iter_xprompt_references`, directive/alt/separator/fence regexes) rather than re-writing regexes; unit tests pin the
  shared behavior.
- **Monokai base styles fighting overlay colors** (e.g. markdown lexer already colors `# Heading`) → overlay spans are
  applied _after_ and take precedence; snapshot tests catch clashes.
- **OptionHighlighted echoes on row rebuilds** (perf rule 8) → debouncer + idempotent cached update; add a synchronous
  guard flag if snapshots/behavior tests show churn.
- **Modal width regressions on small terminals** → explicit collapse class + dedicated snapshot.

## Out of Scope (natural follow-ups)

- Reusing `highlight_prompt_text()` in the prompt-history preview pane, `PreviewPanelModal` payloads, and agent prompt
  display (`_agent_display_content.py`) — the module is designed for drop-in reuse there, but each has its own snapshot
  surface and should be its own change.
- A `sase` lexer entry for `.sase` files in file-panel lexer tables.
- Any change to stash persistence, restore semantics, or the Rust prompt-stash store.
