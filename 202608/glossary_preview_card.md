---
tier: tale
title: Redesign the ACE glossary preview as a purpose-built definition card
goal:
  Pressing `K` on a glossary term in the ACE prompt opens a compact, content-hugging
  definition card that renders the definition as prose and the entry's properties as a
  styled chip row plus an aligned label/value grid, cross-links the other glossary terms
  a definition mentions, and hands off to the real `sase.yml` definition line.
size: medium
proposed_by: bbugyi200.athena.wn
create_time: 2026-08-09 13:43:49
status: wip
---

# Plan: Redesign the ACE glossary preview as a purpose-built definition card

## Problem

`K` on a glossary term currently builds a synthetic Markdown document and pushes it
through the generic file-preview modal
(`src/sase/ace/tui/widgets/_prompt_glossary.py:399` `_glossary_preview_markdown` ->
`src/sase/ace/tui/modals/preview_panel_modal.py` `PreviewPanelModal`). A glossary entry
is a small structured record, not a file, so routing it through the file reader produces
a panel with these concrete defects:

1. **Half the panel is empty.** `PreviewPanelModal > Container` is fixed at
   `width: 96%; height: 85%; max-width: 150; max-height: 42`
   (`src/sase/ace/tui/styles.tcss:4638`). A three-line definition gets a frame sized for
   a source file, so roughly two thirds of the card is dead space and the definition
   paragraph is stretched to ~140 columns -- far past a readable prose measure.
2. **The term is rendered twice.** The modal title bar prints the term, and the
   synthetic body opens with `# {term}`, which Textual's `Markdown` renders as a large
   centered heading. The eye reads the same words twice before reaching content.
3. **Properties are shouted prose.** `ALIASES:`, `PROJECT:`, and `SOURCE:` are plain
   Markdown paragraphs in the body flow. They are indistinguishable from the definition
   text, have no alignment, no grouping, and no visual weight difference.
4. **The source path is printed twice** -- once in the title bar, once in the `SOURCE:`
   paragraph.
5. **The `reference` slot lies.** The payload passes `span.matched_text` as `reference`,
   which the title bar renders in the position other preview kinds use for a reference
   path, followed by `->  {path}`. The screenshot reads
   `xprompt memory  -> /.../sase.yml`, which looks like a path resolution but is
   actually "the phrase you put your cursor on".
6. **`R source` shows a file that does not exist.** The "source" view of a glossary
   preview is the synthetic Markdown blob the code just built, not the `sase.yml` that
   actually defines the term. `/` search searches that fiction, and `y contents` copies
   it.
7. **`o` opens `sase.yml` at line 1.** `PreviewPanelModal.action_open_in_editor` calls
   `build_editor_args`, which takes no line/column, even though the entry carries an
   exact `definition_range` and the sibling `Ctrl+]` jump already uses it.
8. **The wrong alias set is shown.** The preview renders `configured_aliases`, while
   `sase memory init` renders `display_aliases`
   (`src/sase/main/init_memory/glossary.py:167`). The Rust core derives
   `display_aliases` by dropping aliases that are a plural of the term or of another
   alias (`crates/sase_core/src/glossary.rs` `derive_display_aliases`), precisely
   because those are noise for a human reader. The preview shows the noise; the
   generated glossary memory does not.

## Design

Replace the synthetic-Markdown-through-the-file-reader path with a dedicated
`GlossaryPreviewModal`. This follows the grammar this codebase already uses for
structured records rather than files: `WordDefinitionModal`
(`src/sase/ace/tui/modals/word_definition_modal.py`), `SpellcheckPanelModal`, and
`CommitViewModal` are each purpose-built modals with a shared title/scroll/footer shape.
The generic `PreviewPanelModal` keeps serving xprompts, skills, workflows, and files
unchanged.

### Layout

```
+-- accent border, width auto (max 84), height auto (max 80%) ---------------+
|  G  GLOSSARY   Agent Hood                                            sase  |
|  matched "hood"                                                            |
+----------------------------------------------------------------------------+
|                                                                            |
|   An agent hood is a group of agents that are all named with the same      |
|   `<name>.` prefix. For example, agents named `foo.bar`, `foo.baz`, and    |
|   `foo.bar.1` are all apart of the same `foo` agent hood.                  |
|                                                                            |
|   ALSO KNOWN AS   hood   agent neighborhood                                |
|   SEE ALSO        1 Agent Lane   2 Agent Clan                              |
|                                                                            |
|   ------------------------------------------------------------------      |
|   Project   sase                                                           |
|   Source    sase/sase.yml:20:5                                             |
|   Matches   3 forms (term, 2 aliases)                                      |
+----------------------------------------------------------------------------+
   j/k scroll | 1-9 see also | y definition | Y path | o editor | Z viewer | esc close
```

### Design decisions and rationale

- **Hug the content.** `height: auto` with `max-height: 80%`, following
  `SpellcheckPanelModal > Container` (`src/sase/ace/tui/styles.tcss:4778`), which is the
  established content-hugging modal in this app. A short definition gets a short card; a
  long one scrolls. This single change removes the dead space in the screenshot.
- **Cap the prose measure.** `width: auto` with `max-width: 84`. Definitions are
  paragraphs, and paragraphs stop being readable past roughly 80 columns. The current
  150-column frame is sized for source code.
- **Say the term once.** The canonical term lives in the title bar only. The body starts
  with the definition. Delete the synthetic `# {term}` heading.
- **Keep the definition as Markdown.** Glossary definitions are authored with inline
  code (`` `<clan>` ``, `` `%tribe:<name>` ``, `` `sase agent tribe` ``) and that
  formatting carries real meaning. Render the definition body -- and only the definition
  body -- through Textual's `Markdown` widget, exactly as today, so inline code keeps
  its surface. Everything that is _not_ prose stops being Markdown.
- **Properties become structure, not sentences.** Aliases render as chips; project,
  source, and match count render as an aligned two-column `Table.grid` with muted
  small-caps labels in a fixed-width left column, mirroring the metadata grid
  `CommitViewModal._build_title` already uses (`Table.grid(expand=True, ...)`). Labels
  become quiet and scannable instead of shouting `ALIASES:` mid-paragraph.
- **Chips reuse the pill grammar.** Alias chips use the two-tone-over-accent style
  documented in `src/sase/ace/tui/widgets/_override_pill.py` (bold dark foreground over
  an accent background) so the card looks like the rest of the app rather than a new
  visual dialect.
- **Show `display_aliases`, not `configured_aliases`.** This matches what
  `sase memory init` writes into `AGENTS.md` and drops derived-plural noise. Nothing is
  lost: the derived forms are accounted for in the `Matches` property row, which reports
  the `effective_aliases` count -- so a user who wonders why `agent clans` highlighted
  in the prompt has an answer on the card.
- **Disclose the matched phrase honestly.** When `span.matched_text` differs from the
  canonical term case-insensitively, the title's second line reads `matched "hood"`.
  When the user's cursor was on the canonical term itself, that line is omitted. The
  `reference ->  path` construction that made matched text look like a path is gone.
- **Cross-link the glossary to itself.** Scan the definition text with the already-warm
  compiled matcher (`scan_glossary_spans(catalog.compiled, entry.definition)`) to find
  the other entries this definition mentions. Give those spans the same bold + underline
  - theme-accent treatment they get in the prompt, and list them under `SEE ALSO`
    numbered `1`-`9`. Pressing a digit opens that entry in place; `<backspace>` walks
    back through the history. This turns a dead-end popup into a browsable glossary, and
    it costs nothing: `catalog.entries` and the compiled matcher are already in memory,
    so following a cross-reference is an in-memory swap with no I/O and no worker.
    Numbered selection matches `SpellcheckPanelModal`, which already binds `1`-`9` to
    act immediately.
- **Make the handoffs real.** `o` opens `sase.yml` at the definition's line and column
  via `build_jump_editor_argv` (`src/sase/ace/tui/widgets/_prompt_jump_target.py:99`),
  the same helper `Ctrl+]` uses -- not `build_editor_args`, which drops the position.
  `y` copies the definition text. `Y` copies the source path. `Z` opens the artifact
  viewer. The fake `R source` toggle and the `/` search over a synthetic document are
  dropped, because there is no longer a synthetic document to search.
- **Theme-derive the accent.** Compute the card's accent with
  `derive_argument_color(app_theme.primary, foreground=..., background=...)` from
  `sase.xprompt.highlight_theme`, the same call `_prompt_glossary.py:352` uses for the
  in-prompt glossary underline. The phrase you highlighted in the prompt and the card
  that opens then share one color, in both light and dark themes.

### Degraded states

Every one of these must render without an empty label, a stray separator, or a dangling
key hint:

- No `display_aliases` -> omit the `ALSO KNOWN AS` row entirely.
- No cross-referenced terms -> omit the `SEE ALSO` row and the `1-9` footer hint.
- No `definition_range` -> `Source` shows the bare path; `o` opens the file at its top.
- No resolvable source path -> omit the `Source` row and keep the existing
  warning-notify behavior for `Y`, `o`, and `Z`.
- Definition longer than the card -> the body scrolls; the property grid stays inside
  the scroll region so it is reachable, not pinned.
- Very long alias or see-also lists -> chips wrap onto continuation lines under the
  label column.

### Performance

The modal does no disk I/O and starts no workers. The catalog is already warm before `K`
can resolve a term (`_glossary_match_under_cursor` returns the cold sentinel and
notifies otherwise), `catalog.entries` is in memory, and the only computation on open is
one Rust `scan` over a single short definition string. This satisfies the "render paths
never stat/glob" and "never block the event loop" rules in the `tui_perf` memory note.

## Implementation

### 1. Extract the shared source-file actions

Both the file preview and the new card need "copy path / open in editor / open in
viewer" with identical semantics. Extract them once rather than forking them.

- Add `src/sase/ace/tui/modals/_source_file_actions.py` with a `SourceFileActionsMixin`
  providing `action_copy_path`, `action_open_in_editor`, and `action_open_in_viewer`,
  driven by two overridable hooks: `_source_action_path()` and
  `_source_action_position()` returning `(line, col)` defaulting to `(None, None)`.
- Implement the editor action with `build_jump_editor_argv` so a position is honored
  when one is supplied and the behavior is byte-identical to today when it is not
  (`build_jump_editor_argv` falls back to `[editor, path]` for non-vim editors and for
  `line is None`).
- Have `PreviewPanelModal` use the mixin, returning `self._payload.source_path` and no
  position. Its observable behavior must not change.

### 2. Build the render layer

Add `src/sase/ace/tui/modals/glossary_preview_render.py` holding pure, widget-free
functions so the visual grammar is unit-testable without mounting an app:

- `glossary_card_accent(theme) -> str` -- theme-derived accent shared with the prompt
  underline.
- `build_glossary_title(entry, *, matched_text, project_name, accent) -> RenderableType`
  -- icon + `GLOSSARY` + term on the first line with the project name right-aligned via
  `Table.grid`, and the optional `matched "..."` line beneath.
- `build_alias_chips(display_aliases, *, accent) -> Text | None` -- returns `None` when
  there are no aliases so the caller can omit the row.
- `build_see_also_chips(references, *, accent) -> Text | None` -- numbered chips.
- `build_property_grid(entry, *, project_name, source_display, effective_count) -> RenderableType`
  -- the aligned label/value grid, omitting absent rows.
- `glossary_source_display(catalog, entry) -> str | None` and
  `glossary_source_path(catalog, entry)` and `glossary_definition_position(entry)` --
  moved verbatim from `_prompt_glossary.py` (they are already correct and already tested
  through the jump path); re-export from `_prompt_glossary.py` if that keeps the jump
  code unchanged.
- `glossary_cross_references(catalog, entry) -> tuple[GlossaryEntry, ...]` -- scan the
  definition with `scan_glossary_spans`, resolve `entry_index` through
  `catalog.entries`, drop self-references, de-duplicate while preserving first-mention
  order, and cap at nine so every result has a digit key.

### 3. Build the modal

Add `src/sase/ace/tui/modals/glossary_preview_modal.py` with
`GlossaryPreviewModal(CopyModeForwardingMixin, SourceFileActionsMixin, ModalScreen[None])`:

- Constructor takes the `EditorGlossaryCatalog`, the `GlossaryEntry`, and the matched
  text.
- `compose` yields `#glossary-preview-title`, a `VerticalScroll`
  `#glossary-preview-scroll` containing the definition `Markdown`
  (`#glossary-preview-definition`) followed by a `Static` `#glossary-preview-meta` for
  the chip rows and property grid, and `#glossary-preview-footer`.
- Bindings: `escape`/`q` close, `j`/`k`/`ctrl+d`/`ctrl+u`/`g`/`G` scroll (same
  vocabulary as the other reader modals), `1`-`9` follow a cross-reference, `backspace`
  go back, `y` copy definition, `Y`/`shift+y` copy path, `o` editor, `Z`/ `shift+z`
  viewer.
- Keep an internal `list[GlossaryEntry]` history for `backspace`. Following a reference
  clears the matched-text line (the user navigated deliberately; nothing was matched)
  and resets scroll to the top.
- The footer is built from live state so hints for absent affordances never appear: no
  `1-9 see also` without cross-references, no `Y`/`o`/`Z` hints without a source path.
- `y` copies the definition text through `schedule_copy_delivery` with a
  `"glossary definition"` label.

### 4. Style the modal

Add a `GlossaryPreviewModal` block to `src/sase/ace/tui/styles.tcss` next to the other
modal blocks:

- `> Container`:
  `width: auto; max-width: 84; height: auto; max-height: 80%; border: thick $primary; background: $surface; padding: 1 2;`
- `#glossary-preview-scroll`:
  `height: auto; max-height: 100%; scrollbar-gutter: stable;`
- `#glossary-preview-definition` / `#glossary-preview-meta`:
  `width: 100%; height: auto;`
- `#glossary-preview-footer`: the established centered, muted, `border-top` footer used
  by `WordDefinitionModal` and `SpellcheckPanelModal`.

Verify the container actually hugs its content: an `auto`-height container wrapping a
`VerticalScroll` needs the scroll region to be `height: auto` with a bounded
`max-height`, not `height: 1fr`, or the card will stretch again.

### 5. Switch the call site

In `src/sase/ace/tui/widgets/_prompt_glossary.py`:

- `_preview_glossary_under_cursor` pushes
  `GlossaryPreviewModal(catalog, entry, matched_text=span.matched_text)` instead of
  building a `PreviewPayload`.
- Delete `_glossary_preview_markdown`.
- Keep `_jump_to_glossary_definition_under_cursor` working; if the source helpers move
  to the render module, import them there so `Ctrl+]` behavior is untouched.

## Testing

- **New unit tests** for `glossary_preview_render.py` covering: alias chips use
  `display_aliases`; an entry whose only configured alias is a derived plural yields no
  alias row; the property grid omits `Source` when no path resolves; the matched line is
  omitted when the matched text equals the term case-insensitively;
  `glossary_cross_references` drops self-references, de-duplicates, preserves
  first-mention order, and caps at nine.
- **Update** `tests/ace/tui/widgets/test_prompt_glossary_navigation.py`: the `K` tests
  currently assert on `PreviewPanelModal._payload` and on the literal strings
  `"# Agent Clan"`, `"ALIASES: clan, agent clans"`, `"PROJECT: sase"`, and `"SOURCE:"`.
  Replace those with assertions against `GlossaryPreviewModal` state -- the entry, the
  matched text, and the rendered title/meta plain text. The cold-catalog and `Ctrl+]`
  tests must keep passing unchanged; that is the regression guard on the parts of this
  flow the redesign does not touch.
- **New behavior tests** for the modal: pressing `1` follows a cross-reference and swaps
  the displayed entry; `backspace` returns to the previous entry; `backspace` on the
  first entry is a no-op rather than a dismiss; digits with no corresponding
  cross-reference do nothing; `Y`/`o`/`Z` on an entry with no source path notify instead
  of raising.
- **New PNG snapshots** in `tests/ace/tui/visual/`, following
  `test_ace_png_snapshots_preview_panel.py` and the existing dark/light pair
  `prompt_glossary_highlight_{dark,light}_120x40.png`: one dark and one light snapshot
  of the card with aliases, cross-references, and a full property grid, plus one
  snapshot of the minimal entry (no aliases, no cross-references) so the degraded layout
  is pinned too. Generate with `--sase-update-visual-snapshots` and review the rendered
  output before accepting -- these snapshots are the actual acceptance criterion for
  "beautiful", so look at them.
- **Existing suites** that must stay green: `tests/ace/tui/widgets/`,
  `tests/ace/tui/modals/`, and the preview-panel visual snapshots (the shared-mixin
  extraction must not shift them by a pixel).

## Documentation

- `docs/ace.md`: the "Glossary terms" section currently states that `K` "opens the
  Markdown preview panel with the canonical term, definition, configured aliases,
  project, and source path". Rewrite it to describe the definition card, the alias
  chips, the numbered cross-reference navigation and `backspace`, the property grid, and
  `o` opening the definition's line. Check the prompt NORMAL-mode key table in the same
  file for any preview-panel key claims that no longer hold for glossary terms.
- `src/sase/ace/tui/modals/help_modal/binding_common.py:38` already reads
  `K / Ctrl+] on glossary term -> Preview / jump to definition`, which stays accurate.
  Per `src/sase/ace/CLAUDE.md` the help popup must stay in sync -- confirm no other help
  entry describes the glossary preview's keys.
- No `sase/memory/` file describes this panel, so no memory update is required. Do not
  edit memory files.

## Out of scope

- The nvim/LSP glossary hover (it lives in the `sase-nvim` plugin and renders its own
  markup).
- Any change to glossary matching, validation, or alias derivation in the Rust core.
- A full glossary browser listing every term in a project. The `SEE ALSO` navigation is
  deliberately limited to terms an open definition actually mentions; a browse-all entry
  point is a separate piece of work.

## Verification

Run `just install` first (workspaces are ephemeral), then `just check`. Because this
change touches `styles.tcss` and adds visual snapshots, also run `just test-visual` and
finish with `just check-full` before landing.
