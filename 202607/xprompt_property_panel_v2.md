---
create_time: 2026-07-10 19:09:47
status: done
prompt: 202607/prompts/xprompt_property_panel_v2.md
tier: tale
---
# XPrompt Property Panel v2 — In-Place XPrompt Authoring

## Product context

The frontmatter ("xprompt property") panel that sits above the prompt-input stack (`FrontmatterPanel`,
`src/sase/ace/tui/widgets/frontmatter_panel.py`) already gives structured, schema-driven editing of a prompt's
frontmatter. But it falls short of the real goal: **creating and modifying xprompts — and ordinary user prompts, which
share the exact same shape — should be so fluid in the TUI that reaching for `$EDITOR` is never justified except by
habit.**

Today the panel (and the flows around it) miss that bar in five structural ways:

1. **Modal hops for structured properties.** Editing an `input` or `xprompts` item opens a full-screen modal
   (`InputItemModal` / `XPromptItemModal`) that hides the prompt body and every other property. Worse, _saving_ the
   modal closes the panel and dumps you back into the body (`_commit_to_body()` in
   `src/sase/ace/tui/widgets/_frontmatter_panel_editing.py`), while _cancelling_ keeps you in the panel — an asymmetry
   that makes multi-item editing feel like whack-a-mole.
2. **No way to edit an existing xprompt in the TUI.** The XPrompt Browser's "edit" action shells out to `$EDITOR`
   (`xprompt_browser_actions.py`); loading an xprompt into the bar only ever inserts its _rendered expansion_. There is
   no "open the raw body + frontmatter, edit it structurally, save it back" loop at all — the single biggest reason
   users still need an editor.
3. **Saving a new xprompt is a 3–5 modal chain** (`gx` → save-target → location → name → maybe overwrite-confirm → maybe
   git-commit), and overwriting an existing xprompt is _blind_ — it replaces the file with the current draft without
   ever showing what is being replaced.
4. **Silent data loss on round-trip.** `PromptFrontmatter` deliberately models a parity field set and **drops** any
   other key (`log_skill_use`, `output`, unknown keys) on parse. Fine for drafts; fatal for a load→edit→save loop over
   existing files.
5. **Surprising / fragile edges.** `enter` on the `input`/`xprompts` header silently _adds_ an item instead of editing;
   a raw-mode parse failure traps you in raw mode with the exception text jammed into the border subtitle; input types
   are free-text fields; there is no way to reorder inputs even though order defines positional-arg semantics; three
   hardcoded per-field `if/elif` chains can drift from the Rust-core schema; `skill`/`snippet` bool-vs-string is guessed
   from keyword sniffing (a provider literally named `"on"` parses as a bool).

Because a normal user prompt is just "shared frontmatter + `---`-separated pane bodies" — byte-compatible with an
xprompt `.md` file (`PromptStackState.join`, `save._build_markdown_xprompt`) — every improvement below applies to both
automatically. That equivalence is the backbone of this design: **one editing surface, three storage surfaces** (xprompt
`.md` files, config `xprompts:` entries, and unsaved drafts).

## Design principles

- **Never leave the panel to edit a property.** All property editing happens inline, in place, with the prompt body
  still visible. Zero modals inside the panel.
- **The draft knows where it lives.** A prompt stack can be _bound_ to an xprompt source (file or config entry); save is
  one keystroke, dirty state is always visible, and conflicts are detected — the same contract an editor buffer gives
  you.
- **Schema is the single source of truth.** Field kinds, input types, ordering, and validation all flow from the Rust
  core schema (`frontmatter_field_schema()` / `input_type_schema()` / `validate_frontmatter()`); the panel contains no
  per-field-name special cases.
- **Lossless or loud.** Round-tripping an existing xprompt through the TUI must preserve every key it does not
  understand, and must warn (never silently degrade) in the one case it cannot preserve (YAML comments inside
  frontmatter).
- **Beautiful means calm.** Aligned columns, a consistent color language, per-mode key hints in the border subtitle, and
  states that transition without layout jumps.

## Target UX

### The panel, redesigned

Visual layout keeps the current bones (bordered `frontmatter ⟨✓⟩` panel above `#prompt-stack`) with these changes:

```
╭─ frontmatter  ⟨✓⟩ ────────────────────────────────────────────────╮
│ ▸ description  Refactor the auth module across services           │
│   tags         refactor, backend                                  │
│   input        ▾ 2 items                                          │
│     • service   word   (required)                                 │
│     • dry_run   bool   = False      skip writes                   │
│   xprompts     ▾ 1 item                                           │
│     • _rules    "Follow the team review checklist"                │
│   skill        false                                              │
╰──────────── a property · o item · e edit · d del · u undo · R raw ╯
```

- **Item cells align in columns** across items (name / type / default / description), computed per render — today each
  item line is free-form.
- **Rows view keys** (normal mode): `j/k` move (fields _and_ items), `h/l` fold/unfold, `enter`/`e` edit, `a` add
  property, `o`/`A` add item under the selected structured field, `d` delete, `u` undo, `J/K` move the selected item
  down/up (input order is positional-arg order, so reordering is a first-class need; mirrors the `gJ/gK` pane-reorder
  muscle memory), `R` raw mode, `esc`/`q` done. `enter` on a structured _header_ now toggles fold — adding is only ever
  `o`/`A`.

**Inline cell editing replaces both item modals.** `e`/`enter` on an item row expands it into an in-panel cell strip;
`tab`/`shift+tab` move between cells; `enter` commits the row; `esc esc` cancels. The active cell renders with a soft
reverse highlight; the border subtitle switches to cell-mode hints.

- `input` item cells: **name → type → default → description**. The _type_ cell is an enum cycler over the core
  `input_type_schema()` catalog (`space`/`h`/`l` cycle; typing a unique prefix like `wo` or `int` jumps directly); the
  per-type rule hint ("A whole number.") renders under the row while the cell is active. The _default_ cell shows
  `(required)` when blank and live-coerces via `InputArg.validate_and_convert`, surfacing coercion errors inline in red
  under the row — same position the panel already uses for diagnostics.
- `xprompts` (local helper) item cells: **name → description → inputs → content**. Name enforces the `_` prefix as you
  type (auto-prefix like `normalize_local_xprompt_name` does today). The _inputs_ cell edits the item's own args with
  the same cell strip pattern (nested one level, entered with `enter`, never a modal). The _content_ cell is multiline:
  activating it swaps in a bounded in-panel `VimTextArea` (the same slot pattern raw mode uses), so the prompt body
  stays visible below while editing helper content.
- `skill` / `snippet` stop guessing bool-vs-payload: the value edits as **two cells — state and payload**. State is a
  cycler (`true` / `false` / `providers…` for skill; `true` / `false` / `trigger…` for snippet); the payload cell only
  activates for the list/string states. Keyword sniffing (`_parse_bool_word`) dies.

**Adding is inline too.**

- `a` (add property) no longer opens `AddPropertyModal`: the panel's bottom line becomes an inline chip picker of the
  unset schema fields (`name · description · tags · input …`), filtered as you type, first-letter accelerators
  preserved, `enter`/letter selects, `esc` dismisses. Scalars drop straight into inline edit; structured fields drop
  into a ghost item row.
- `o`/`A` (add item) inserts a **ghost row** directly under the selected structured field, already in cell-edit mode at
  the name cell. Committing keeps you in the panel with the new row selected; `esc esc` removes the ghost.

**Commit symmetry.** Committing any edit (cell strip, inline scalar, ghost row) re-renders and _stays in the panel_ with
the edited row selected. Only `esc`/`q` leaves.

**Undo.** The panel keeps a bounded snapshot stack of the working model; `u` reverts the last mutation (delete, edit,
add, reorder, raw apply). Deletes stop being scary.

**Raw mode, untrapped.** Raw YAML mode stays (it is the escape hatch that makes everything else honest) with two fixes:
core diagnostics render as a line-anchored strip _below_ the editor (line number + message, count in the chip) instead
of only a count; and on an unparseable buffer, `esc esc` keeps you editing while a new explicit discard binding
(`ctrl+c`, hinted in the subtitle) always exits without applying. Diagnostics runs stay debounced off the keystroke
path.

**Scrolling.** The rows view gains a viewport (selection-follows-scroll) so a long input list no longer clips at the
panel's max height.

### The authoring loop: bind → edit → write

New concept: an optional **binding** on `PromptStackState` — the xprompt source this draft was loaded from:
`{kind: file|config, path, entry_name?, format, loaded_fingerprint (mtime+content hash)}`.

- **Bar title shows the binding + dirty state**: `Prompt · review · ~/.xprompts/review.md ●` (dot = unsaved changes,
  cleared on write). Unbound drafts look exactly like today.
- **`gw` / `Ctrl+G w` — write.** Bound: serialize (frontmatter + `---`-joined panes) and write back through the existing
  writers (`save_markdown_xprompt` / `save_config_xprompt`), atomically (tmp + rename), off the event loop. If the
  source changed on disk since load (fingerprint mismatch), a conflict prompt offers _overwrite / reload from disk /
  save-as_ — never a silent clobber. Unbound: `gw` falls through to save-as. The existing post-save git-commit offer is
  preserved.
- **`gx` — save-as, one screen.** The SaveTarget → Location → Name modal chain collapses into a single unified save
  modal: name field (prefilled from frontmatter `name` or the binding) + location list with resolved-path preview + a
  live collision indicator that flips the action to "overwrite #review" with a preview of the existing definition as you
  type. One `enter` from open to saved in the common case; a successful save-as binds the stack to the new target.
  Overwrite is no longer blind.
- **Edit an existing xprompt — the missing loop:**
  - **XPrompt Browser** (Admin Center tab + `#@` selector browser): `enter` on an editable `.md`/config-backed simple
    xprompt now loads its **raw** body + frontmatter into the prompt bar (via the existing
    `load_stack_from_xprompt_markdown` machinery, which already lifts frontmatter and splits `---` segments) and binds
    it. The panel auto-shows. `$EDITOR` remains available, demoted to `E` (and stays the _only_ edit path for `.yml`
    workflow rows — step-graph editing is out of scope). Read-only sources (plugins, built-ins) load unbound with a
    toast ("read-only source — gw will save-as").
  - **`gd` / `Ctrl+G d` — edit definition under cursor.** With the cursor on a `#name` reference in any pane, `gd`
    (goto-definition mnemonic) opens that xprompt for editing, bound, after the usual dirty-draft stash guard.
    Config-backed simple xprompts reconstruct markdown from their model; workflows fall back to the browser preview.
- **Round-trip safety (prerequisite for all of the above):** `PromptFrontmatter` learns **extras passthrough** —
  non-parity keys (`log_skill_use`, `output`, unknown) survive parse → serialize verbatim (order-stable, appended after
  parity fields). The panel renders them as dim read-only rows (`log_skill_use  false  (raw-only)`) editable through raw
  mode. If the loaded frontmatter contains YAML comments (the one thing canonical re-serialization cannot keep), loading
  warns once and the raw editor initializes from the _original_ text so the user can see exactly what is at stake. Body
  text is preserved byte-for-byte throughout.

### Normal user prompts

Nothing extra to learn: an unbound draft _is_ a normal prompt. All panel improvements apply as-is; `gX`
(convert-pane-to-local-helper) is rewired to drop into the panel's ghost xprompt-item row (content prefilled from the
pane, name cell active) instead of its own name modal — on commit the pane body is replaced with the invocation skeleton
exactly as today. Promoting a good draft to a reusable xprompt is just `gw`.

## Technical design (phases)

### Phase 1 — Schema-driven editing core + lossless model (pure Python, no visible UI change)

The foundation everything else sits on.

- Replace the three per-field-name `if/elif` chains in `_frontmatter_panel_editing.py` (`_editable_text`,
  `_apply_inline_value`, `_clear_field`) and the hardcoded structured-field name checks (`frontmatter_panel.py`,
  `_frontmatter_panel_rendering.py`) with a **field-kind dispatch table** derived from `frontmatter_field_schema()`. The
  empty-state field list and add-picker both derive from the same schema — kill the duplicated literals.
- **Extras passthrough** in `PromptFrontmatter` (`src/sase/xprompt/prompt_frontmatter.py`): retain unrecognized /
  non-parity mappings through parse/serialize; expose them read-only to the panel. Round-trip property tests over the
  real built-in xprompt corpus (`src/sase/xprompts/*.md`, `src/sase/default_xprompts/*.md`).
- **Explicit value-state model** for `BOOL_OR_LIST` / `BOOL_OR_SCALAR` fields (state + payload) replacing bool-word
  sniffing.
- **Panel undo stack** (bounded model snapshots) with `u` wired in rows mode.
- Audit whether the core schema needs richer editing metadata (per-field examples, bool-state labels). If yes, extend
  the schema catalog in the sase-core linked repo (opened via `sase workspace open`) plus the binding — per the core
  boundary rule, anything the LSP or another frontend would need lives there; pure presentation stays here.

### Phase 2 — Inline editing: retire the modals

- **Cell-edit mode** in `FrontmatterPanel`: per-item cell strips, `tab`/`shift+tab` navigation, enum-cycler cells (input
  type; skill/snippet state), live coercion errors, in-panel multiline content slot for local-helper content, nested
  inputs editing for local helpers.
- **Ghost rows** for `o`/`A`; inline chip picker for `a`; commit symmetry (never `_commit_to_body()` on item save);
  `enter`-on-header = fold toggle; `J/K` item reordering; rows-view viewport/scrolling.
- **Raw mode**: line-anchored diagnostics strip, debounced validation, `ctrl+c` discard exit.
- Retire `InputItemModal`, `XPromptItemModal`, `AddPropertyModal`; migrate their validation logic (type catalog,
  `_`-prefix rule, compact-spec parsing where still useful) into the cell editors; migrate/replace their tests.
- Rewire `gX` conversion onto the ghost-row flow (retiring `LocalXPromptNameModal`).
- New/updated PNG goldens: cell-edit active, ghost row, chip picker, reorder, raw diagnostics, long-list scrolling.

### Phase 3 — Binding + one-keystroke save

- `PromptStackState.binding` (+ fingerprint) with clean serialization boundaries; bar-title dirty indicator
  (`_refresh_title`).
- `gw` write action in the g-prefix table (`_prompt_input_bar_g_prefix_actions.py`): atomic off-thread write via the
  existing `write_target_sync` dispatch, fingerprint conflict detection with overwrite/reload/save-as resolution,
  post-save git-commit offer reuse.
- **Unified save-as modal** replacing the `XPromptSaveTargetModal` → `XPromptLocationModal` → `XPromptNameModal` chain:
  single screen (name + grouped location list + live path/collision preview + existing-definition preview on overwrite).
  Save-as binds on success. Snippet creation keeps its current entry point inside the unified modal.
- Unify the duplicated config-insert implementations (`xprompt_browser_actions.py`'s local import vs
  `sase.xprompt.config_yaml`) onto the `sase.xprompt` writer.
- Stash integration: `gs`/stash keeps working; persisting the binding through the Rust-backed prompt stash requires a
  small wire extension in sase-core (optional metadata on stash entries). Do it here if cheap; otherwise restore unbound
  with a toast and file the follow-up — losing the binding must at least be _visible_.

### Phase 4 — Edit-existing entry points

- Browser `enter` = load-raw-and-bind (both Admin Center XPrompts tab and the `#@` selector browser); `E` = external
  editor; `.yml` rows keep `$EDITOR`-only with an explanatory hint. Read-only sources load unbound.
- `gd` definition-under-cursor: resolve the `#name` token via the existing reference parser, load + bind, dirty-draft
  guard via the stash flow.
- Config-backed xprompt reconstruction (XPrompt model → markdown) for editing, writing back through
  `insert_xprompt_into_config` (minimal-edit, comment-preserving at file level).
- Comment-detection warning on load; extras rows visible in the panel.
- Optional (schema-gated, free after Phase 1): admit `log_skill_use` into the core parity schema so it becomes a normal
  editable bool row instead of an extras row.

### Phase 5 — Polish, help, docs

- Column alignment + color-language pass over rows/cells (keys `#87D7FF`, types green, required yellow, defaults dim,
  reverse-highlight cells); schema-generated empty state with dimmed examples; smooth height transitions
  (`reserved_height` interplay with the new viewport).
- Keymap surfaces, per the ace conventions: `?` help modal (57-char box discipline), g-prefix hint entries for
  `gw`/`gd`, panel border-subtitle hints per mode, footer conditional keymaps only where availability varies. Any
  configurable keymap additions land in `src/sase/default_config.yml`.
- Docs: xprompts authoring guide section for the TUI loop; CHANGELOG.
- Full visual-suite sweep (`just test-visual`) and a `SASE_TUI_PERF` sanity run.

## Out of scope

- **`.yml` workflow step editing** in the panel — the steps schema is a different beast; workflows keep the
  browser-preview + `$EDITOR` path (explicitly hinted in the UI).
- **YAML comment preservation inside frontmatter** — detected and warned, not preserved (body and, for config targets,
  the rest of the file remain untouched).
- Prompt history store changes (it stores joined text verbatim; unaffected).
- The submit-time Input Collection modal (Phase 5 of the original sase-4r epic) — unchanged.

## Risks & mitigations

- **Keymap collisions.** `gw`/`gd` must be audited against the vim-motion fall-through set in `dispatch_g_prefix_key`
  (which currently passes non-table keys to vim's own `g` commands) and panel-local keys (`o`, `u`, `J/K` are free
  today). The plan's keys are proposals; final assignments happen after that audit, and any configurable ones go through
  `default_config.yml`.
- **Event-loop discipline** (per the TUI perf rules): all file reads/writes and save-row loading off-thread
  (`asyncio.to_thread` / tracked tasks); raw-mode diagnostics debounced; cell editing reuses mounted widgets instead of
  rebuilding; no new full-rebuild paths; re-capture selection state after every await.
- **Round-trip regressions.** Extras passthrough + corpus round-trip tests land _before_ any load-existing entry point
  ships (Phase 1 gates Phase 4). Atomic writes + fingerprint conflict checks gate `gw`.
- **Modal retirement breaks existing tests/muscle memory.** Tests migrate with the flows in Phase 2; the raw-mode escape
  hatch guarantees no capability regression while people adjust.
- **Textual focus juggling** for in-panel editors (the codebase already handles the un-hidden-widget focus quirk with
  `call_after_refresh`; the cell editor reuses that pattern).

## Testing strategy

- **Unit:** dispatch-table coverage for every schema field kind; extras passthrough + idempotent serialization over the
  built-in xprompt corpus; value-state transitions for skill/snippet; undo stack; binding fingerprint/conflict logic;
  unified-modal target resolution (name collisions, read-only sources, chezmoi remapping).
- **Integration (Textual pilot):** cell-edit lifecycle (edit/commit/cancel/ghost), chip-picker add, reorder, raw-mode
  diagnostics + discard exit, `gw` bound/unbound/conflict paths, browser `enter` load-and-bind, `gd` resolution, `gX`
  ghost-row conversion, stash/restore behavior with a binding.
- **Visual (PNG goldens):** refreshed `frontmatter_panel_*` set plus cell-edit, ghost row, chip picker, raw diagnostics,
  extras rows, bound-dirty bar title, unified save modal, browser hint changes.
- **Perf:** no new event-loop work on keystroke paths; existing j/k benches stay green.
