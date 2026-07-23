---
tier: tale
title: 'Redesign the AXE lumberjack/chop config editor: reliable first render + beautiful,
  intuitive UI'
goal: Opening the AXE entry editor always renders a fully-populated, single-visual-system
  form (never the blank/broken state), and the edit + preview surfaces are intuitive,
  legible, and beautiful across wide and narrow terminals.
create_time: 2026-07-23 07:31:44
status: wip
---

- **PROMPT:** [202607/prompts/axe_editor_redesign.md](prompts/axe_editor_redesign.md)

# AXE lumberjack/chop config editor — reliability fix + visual/UX redesign

## 1. Context

`AxeEntryEditorModal` is the shared schema form used to **add and edit** AXE lumberjacks and chops from the `sase ace`
AXE tab (opened via `e edit config`, and as the final step of the add flow). It is launched from
`src/sase/ace/tui/actions/axe_config_actions.py::_push_axe_editor_session`.

Two problems make it unacceptable today:

1. **It does not work on open (reliability).** In the live app the modal renders almost entirely blank: no title, no
   status, an empty scope box, no property rows, empty field detail, and **both** the single-line input and the
   multiline YAML textarea visible at once. Only the seed value (e.g. a chop name like `sase_chop_pr_submitted_checks`)
   appears, because `compose()` seeds the editors directly. The user reported exactly this.

2. **It looks bad even when it does render (aesthetics).** The intended layout (captured in the committed PNG goldens)
   is cluttered "box soup": a bordered scope box whose file path wraps mid-word (`/home/bryan/.config/sase/sase.ym` +
   `l` on the next line), an empty `Advanced` header with nothing under it that reads as broken, mismatched inner
   borders, a single-line value input floating in a large void, a bright `[NORMAL]`/`[INSERT]` vim banner on a one-line
   field, and a footer hint line that overflows and truncates (`… esc:`).

This plan fixes the reliability defect **and** redesigns the presentation so the editor is intuitive, reliable, and
beautiful.

## 2. Root cause of the blank render (VERIFIED)

`src/sase/ace/tui/modals/axe_entry_editor_rendering.py::on_mount` renders synchronously during mount:

```python
def on_mount(self) -> None:
    self._set_narrow(self.app.size.width < _NARROW_BELOW)
    self._render_all(force_editor=True)   # runs before composed children are ready
    self._focus_editor()
```

`_render_all` begins with `if not self.is_mounted: return`. During `on_mount` the composed children are not yet fully
mounted, so this call **no-ops** and the modal is left in its raw composed state (empty `Static`s, both editors
visible). Nothing re-renders afterward on the live path, so the blank state persists.

The **sibling** Config Center editor already solved this correctly (`src/sase/ace/tui/modals/config_edit_modal.py:178`):

```python
def on_mount(self) -> None:
    self._sync_expanded_class()
    # Defer first render/focus until composed children are mounted.
    self.call_after_refresh(self._initialize)

def _initialize(self) -> None:
    self._render_all()
    self._focus_editor()
```

The AXE editor diverged from this proven pattern.

**Empirically confirmed** with a headless Textual harness that mounts the modal through the real lifecycle (`App` +
`push_screen`, no manual render):

- Live path → `#axe-editor-title` renders `''`, scope rail renders `''`, and both `#axe-editor-input` and
  `#axe-editor-textarea` have `display == True`.
- After a manual `_render_all(force_editor=True)` → title/scopes populate and the textarea is correctly hidden for a
  string field.
- After changing `on_mount` to defer via `call_after_refresh` → the real lifecycle (no manual render) produces the
  correct populated result.

**Why the tests miss it:** the visual-snapshot helper
`tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py::_push_visual_editor` calls
`modal._render_all(force_editor=True)` immediately after mounting. That manual call masks the bug — the goldens look
correct while the live app is broken. The test suite must exercise the real mount path so this class of bug cannot
silently return.

## 3. Constraints & non-goals

- **Rust core boundary:** This is presentation + TUI-lifecycle only. Do **not** touch `../sase-core`. The backend/domain
  model (`schema_object_form.py`, `config_transaction.py`, `axe_config_actions.py`, the AXE config backend) is correct
  and stays as-is. The mutation/plan/apply/reload contract, sparse-operation semantics, scope layering, and provenance
  data must be preserved exactly.
- **Keep the shared vim editor widgets.** `SingleLineVimTextArea` / `VimTextArea` are shared across many modals. Do not
  change their core vim behavior. You may use the documented host hook `_update_vim_mode_display(indicator="")` to route
  the mode indicator into calmer chrome (border-title/subtitle) for this modal only.
- **Preserve all existing behavior**: scope cycling (`ctrl+t`), add property (`a`), inherit/reset (`ctrl+r`), bool/enum
  toggle (`space`) and number-pick (`1`–`9`), preview/save (`ctrl+s`/`enter`), save-only (`ctrl+o`), reload after
  conflict (`ctrl+l`), preview scrolling (`j/k`, `ctrl+d/u`, `g/G`), mouse click on scope rail and field rows. Keys stay
  the same; only presentation changes.
- **Both kinds and both modes:** must remain correct for lumberjack and chop, for new-entry (Add) and existing-entry
  (Edit), for generated-instance rows (warning banner), running vs stopped AXE, and the validation-error and
  diff-preview stages.
- No changes to memory files (`sase/memory/*.md`, `AGENTS.md`, provider shims).

## 4. Design vision

Replace the current "box soup" with **one coherent frame and one visual system**. The AXE identity is a warm
lumberjack/wood palette; keep that theme but execute it cleanly and with proper contrast. Everything the user needs to
answer — _what am I editing, in which config layer, what is its current value, what will change, and how do I save_ —
should be legible at a glance.

Principles:

- One frame: compact header → hairline divider → two-pane body (rail | detail) → compact footer. The preview stage
  reuses the same header/footer frame.
- No wrapping metadata. Paths are `~`-collapsed and middle-ellipsized to fit.
- No empty-looking sections. Every section that can take a property shows an inline `+ add property (a)` affordance
  instead of a bare header.
- The property list is **scannable**: each row shows name + a value preview + a compact provenance/edited badge, so
  users read values without selecting each field.
- The editor is **anchored**, with a clear `Value` label and a calm mode pill, not a floating field with a loud vim
  banner.
- Consistent single styles: one divider color, one selected-row treatment, one accent for the active/primary affordance,
  full-contrast body text, `dim` reserved for genuinely secondary metadata.

## 5. Detailed UI specification

Palette (reuse existing AXE tokens; keep them consistent):

- Accent / active / primary: `#F0C674` (amber).
- Identity / honey: `#E6B450` / `#D7A85B`.
- Section labels & dividers (single bronze): `#B87333`.
- Edited / success: `#87D787`. Inherit marker: `#B87333`.
- Warning: `#FFAF5F`. Error: `#FF8787`. Running state: `#87AF87`.
- Body text at full `$text`; use `$text-muted`/`dim` only for secondary text.

### 5.1 Header (`#axe-editor-title` + `#axe-editor-status`)

- Line 1: `Edit AXE chop` (bold amber) then a middot separator then the identity `hooks.main / hook.checks` (honey). For
  lumberjacks: `Edit AXE lumberjack · nightly.docs`. `Add` verb for new entries.
- Line 2 (status strip, secondary): a state chip — `● running` (green) or `○ stopped` (dim); append `· disabled` when
  the chop is disabled, and `· instance <name>` for a generated instance.
- Generated-instance warning renders on its own amber line prefixed `⚠`.
- A thin bronze hairline separates the header from the body.

### 5.2 Scope selector (replaces the wrapping bordered box)

Render the writable scopes as a compact, **non-wrapping** control in the rail:

- `Scope` bronze label, then one chip per writable scope on a single line: active chip highlighted (amber background /
  inverse), inactive chips dim, each prefixed with its `1`/`2` index. Trailing `· ctrl+t` hint.
- Next line: the active scope's file path, `$HOME`→`~` collapsed and **middle-ellipsized** to the rail width so it never
  wraps mid-word; append `· new` (amber) when the target file does not exist yet.
- Preserve the click-to-select behavior (`on_click` on `#axe-editor-scopes`); keep hit-testing aligned to the new
  single-line chip layout, or convert the rail to per-scope clickable widgets — either is fine as long as clicking a
  scope selects it and `ctrl+t` still cycles.

### 5.3 Property list (`#axe-editor-properties`)

- Section headers `BASICS` and `ADVANCED` (bronze, bold).
- Each included field is one row with aligned columns:
  - active marker `▸` (amber) else space;
  - `name` (bold when active) + required `*` (amber);
  - a right-aligned **value preview** (effective/draft value via `format_value`, truncated with ellipsis);
  - a compact badge: `edited` (green) when touched, `inherit` (bronze) when reset, else the source `·<provenance>`
    (dim). Keep it to one badge so rows stay clean.
- Under each section that has `addable_fields()`, render a subtle final row `+ add property  a` (bronze/dim). This
  removes the "empty Advanced" broken look and puts the add affordance where users expect it. The standalone
  `#axe-editor-add-guidance` line may be removed once this is in place.
- Selected row: single consistent style — a warm amber left-accent bar plus a subtle background tint (do not stack
  multiple different highlight styles).

### 5.4 Detail & editor (`#axe-editor-value-pane`)

- Field header: `name` (bold amber) + a type chip (e.g. `string`, subtle background) + description on a readable (not
  overly dim) line.
- A tidy definition list with an aligned label column, only showing rows that apply: `Effective <value>`,
  `Layer(<scope>) <value>` (target-layer value when set), `Inherits <value>` (when inheriting). For a reset/inherit
  field, show the "Inherit/reset removes this target-layer field." note in bronze.
- Hairline divider, then a `Value` label and the editor:
  - `string`/`int`/`number` → single-line vim input. Give it border-title `Value` and route the vim mode into a calm
    border-subtitle (`NORMAL`/`INSERT`) via the `_update_vim_mode_display` host hook — no loud banner. Show a subtle
    placeholder when empty.
  - `text`/`string_list`/`yaml` → multiline area with one consistent border and a small `yaml`/`list` type hint; keep
    `show_line_numbers=False`.
  - `bool`/`enum` → option chips (`1 true  2 false`, or enum values), the active value highlighted; `space` toggles,
    digits pick. Keep current behavior.
- Validation line reserves one row (no layout jump): `✗ <error>` (soft red), the deferred-YAML hint, or a status message
  (dim). Preserve the existing precedence in `_validation_text`.

### 5.5 Footer / hints (`#axe-editor-hints`)

- Single line, grouped, prioritized, and **never truncated** (ellipsize gracefully on narrow widths). Edit stage,
  essential keys only, e.g. `↑↓ field · a add · space toggle · ^R inherit · ^T scope · ⏎ preview · esc`.
- Preview stage: `↑↓ scroll · ^D/^U page · ⏎ save & restart · ^O save only · esc back` (drop `^O save only` when AXE is
  not running, matching current logic). The exhaustive key list stays in the help modal per the ace footer convention.

### 5.6 Preview stage

- Reuse the same header + footer frame so the preview does not feel like a different screen. Keep the existing
  transaction-preview renderer (`render_transaction_preview` / `coerce_transaction_preview`) and the
  `#axe-editor-preview-scroll` container and scrolling actions unchanged.

### 5.7 Narrow layout (`.-narrow`, width `< _NARROW_BELOW = 90`)

- Keep the vertical stack (rail on top, detail below) but apply the same clean header/scope/list/editor styles. Verify
  no mid-word wrapping and no truncated footer at the 70×36 size used by the constrained-width snapshot.

## 6. Reliability fix (do this first; it is the "doesn't work" fix)

In `axe_entry_editor_rendering.py`:

```python
def on_mount(self) -> None:
    self._set_narrow(self.app.size.width < _NARROW_BELOW)
    # Defer first render/focus until composed children are mounted (see
    # ConfigEditModal). Rendering synchronously here no-ops on the live path
    # because _render_all bails while is_mounted is still False.
    self.call_after_refresh(self._initialize_editor)

def _initialize_editor(self) -> None:
    if not self.is_mounted:
        return
    self._render_all(force_editor=True)
    self._focus_editor()
```

Keep the `if not self.is_mounted: return` guard in `_render_all` (it is correct defensively). The `is_mounted` guard in
`_initialize_editor` prevents a fast open/close from rendering into a torn-down screen. Confirm no other code path
depends on the synchronous first render.

## 7. Implementation steps (file by file)

1. **`src/sase/ace/tui/modals/axe_entry_editor_rendering.py`**
   - Apply the §6 deferred-render fix.
   - Rework `compose()` and the `_render_*` helpers for the §5 design: header, non-wrapping scope control, scannable
     property rows with value previews + inline `+ add property` affordance, anchored editor with `Value` label + calm
     vim mode, reserved validation line, grouped non-truncating footer.
   - Add a `~`-collapse + middle-ellipsize path helper for the scope line.
   - Keep `_sync_visibility` semantics (only one editor visible per field kind; body vs preview-scroll toggling).
2. **`src/sase/ace/tui/modals/axe_entry_editor_modal.py`**
   - Minor: ensure any moved rendering hooks still line up; keep `on_click`, `on_text_area_changed`, submit handling,
     and all actions intact. If the scope rail becomes per-scope widgets, update `on_click` accordingly.
   - If routing the vim mode into a subtitle, add a tiny `SingleLineVimTextArea` subclass (or host hook) scoped to this
     modal for the input; do not modify the shared widget's defaults.
3. **`src/sase/ace/tui/styles.tcss`** (lines ~5580–5728, the `AxeEntryEditorModal` block)
   - Replace the mismatched inner borders with the single-system styles: one divider color, one selected-row style,
     header hairline, scope control, property rows, editor framing, footer. Update the `.-narrow` overrides to match.
     Remove now-unused rules (e.g. the old bordered `#axe-editor-scopes` box, `#axe-editor-options` full border) and add
     rules for any new ids.
4. **AXE help guide:** if any modal keybinding label changes, update the AXE help content and its golden so the `?` help
   stays in sync (see `src/sase/ace/CLAUDE.md` → Help Popup Maintenance; golden `help_guide_axe_120x40.png`). Keys
   themselves should not change, so this is likely a no-op — verify.

## 8. Testing & validation

1. **New regression test (real mount path) — the core guard.** Add to `tests/ace/tui/test_axe_entry_editor_modal.py` a
   test that mounts the modal through the real lifecycle (an `App` that `push_screen`s the modal, then
   `await pilot.pause()`), **without** calling `_render_all` manually, and asserts:
   - `#axe-editor-title` content is non-empty and contains the identity label;
   - the scope rail is populated;
   - for a `string` field, `#axe-editor-input` is visible and `#axe-editor-textarea` is hidden. This test must FAIL
     against the current `on_mount` and PASS after the §6 fix.
2. **De-mask the visual helper.** Change `_push_visual_editor` in
   `tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py` to rely on the real mount render (remove the manual
   `modal._render_all(force_editor=True)`, or keep it only where a test intentionally drives a later state such as
   switching `_active_name`). The goldens must reflect what the live app shows.
3. **Regenerate PNG goldens** for the redesigned visuals: `just test-visual --sase-update-visual-snapshots` then re-run
   `just test-visual` to confirm exact-match. Affected goldens include: `axe_chop_editor_basics_120x40`,
   `axe_chop_editor_advanced_120x40`, `axe_generated_instance_warning_120x40`, `axe_editor_validation_failure_120x40`,
   `axe_editor_diff_preview_120x40`, `axe_editor_constrained_width_70x36` (and the help guide only if it changed).
   Inspect the new PNGs to confirm the design goals (no mid-word wrapping, no empty-looking Advanced, anchored editor,
   non-truncated footer, good contrast). The Add-flow modal goldens (`axe_add_chooser`, `axe_script_picker`,
   `axe_new_lumberjack_identity`) should be unaffected — leave them unless the design intentionally touches them.
4. **Update existing assertions** in `test_axe_entry_editor_modal.py` / `test_axe_config_actions.py` that depend on old
   rendered strings or widget structure; keep behavioral tests (scope cycle, add property, inherit, toggle,
   preview/apply, reload) green.
5. **Full gate:** run `just install` first (ephemeral workspace), then `just check` (ruff + mypy) and `just test`.
   Everything must pass.

## 9. Acceptance criteria

- Opening the editor from the AXE tab (both `e edit config` on an existing lumberjack/chop and the final step of the add
  flow) shows a **fully populated** form immediately — title, status, scope rail, property rows, field detail, and
  exactly one editor for the active field. The blank/broken state is gone.
- The new real-mount regression test passes and would fail without the fix.
- No mid-word path wrapping; no empty-looking `Advanced` section; a single consistent visual system (one divider color,
  one selected-row style); a footer that fits without truncation; calm vim mode presentation; readable contrast
  throughout, on wide (120×40) and narrow (70×36) terminals.
- All AXE editor PNG goldens are regenerated and pass exact-match; the visual helper no longer manually forces the first
  render.
- `just check` and `just test` pass. No changes to `../sase-core`, the schema/ transaction backend, the shared vim
  widgets' defaults, or memory files.

## 10. Risks & mitigations

- **Golden churn:** the redesign changes many PNGs. Mitigation: regenerate only the AXE editor goldens, and visually
  inspect each before committing.
- **Click hit-testing on the new scope control:** if the scope rail layout changes, `on_click`'s y-offset math must be
  updated (or replaced with per-scope widgets). Add/keep a click-selects-scope assertion.
- **Vim mode routing:** overriding `_update_vim_mode_display` must not break Escape's two-stage INSERT→NORMAL→cancel
  behavior; keep the subclass scoped to this modal and covered by an interaction test.
- **Deferred callback after dismiss:** guard `_initialize_editor` on `is_mounted` so a fast open/close cannot render
  into a torn-down screen.
