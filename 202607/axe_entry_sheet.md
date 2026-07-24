---
tier: tale
title: Redesign the AXE chop/lumberjack editor as a single-page property sheet
goal: 'Editing an AXE lumberjack or chop from the AXE tab happens on one page that
  shows every property at once, where arrow keys always move between properties, values
  are edited in place, and no capability of the current two-pane editor is lost.

  '
create_time: 2026-07-24 17:15:14
status: wip
---

- **PROMPT:** [202607/prompts/axe_entry_sheet.md](prompts/axe_entry_sheet.md)

# Plan: AXE Entry Sheet — One Page, Every Property

## Context and root cause

`e` on the AXE tab opens `AxeEntryEditorModal` (`src/sase/ace/tui/modals/axe_entry_editor_modal.py` +
`axe_entry_editor_rendering.py` + `axe_entry_editor_types.py`, styled at `styles.tcss:5579-5795`). It is a master-detail
surface: a 42-column left rail holds the scope chips and a list of _included_ properties, and the right pane shows one
field's help plus one value editor. Everything else on screen is empty. Three structural problems follow from that
shape.

**1. Keyboard navigation is unreachable.** The modal binds `j`/`k`/`up`/`down` to `_move_field`, but it also calls
`_focus_editor()` on mount and after every move, and a focused `TextArea` swallows those keys. Verified against the
installed Textual with a focused `SingleLineVimTextArea`:

- INSERT mode — `up`/`down` are consumed as cursor motion; `j`/`k` are _typed into the value_ (a value of
  `sase_chop_tg_inbound` becomes `jksase_chop_tg_inbound`). No screen binding fires.
- NORMAL mode — the vim tower consumes all four as motions. No screen binding fires.
- Only with no focused widget do the modal's bindings run.

Since the editor is focused from mount onward, property navigation is keyboard-unreachable in the default state, and so
are the other single-character bindings (`space` toggle, `a` add property, `g`/`G`). The `ctrl`-tier bindings (`^S`,
`^O`, `^R`, `^T`, `^L`) do reach the modal in every mode.

**2. Most properties are hidden.** A chop has 11 editable properties and a lumberjack 3 (`axeChop` and
`axe.lumberjacks.*` are closed schemas — `additionalProperties: false`), but the rail only lists fields that are already
set. The rest are reachable only through a second modal (`PropertyPickerModal` via `a`). The form model already emits
operations for _touched_ fields only (`SchemaObjectForm.patch()`), so hiding untouched properties buys no write safety —
it only hides the schema.

**3. One shared editor forces defensive bookkeeping.** Because a single input/textarea pair is reloaded whenever the
active field changes, the modal carries `_ignore_editor_values`, `_loaded_editor_name`, and a `force_editor` render flag
purely to suppress echoes from its own writes.

Also misleading today: the scope chips render as `1 user` / `2 overlay:sase_athena.yml`, but `1`–`9` are bound to
enum-option selection, not scope selection.

This is presentation-only Textual work. Planning, YAML surgery, and writes stay behind the injected
`plan`/`apply`/`reload` callbacks (`AxeEditorSession`, `sase.axe.config_backend`); no Rust core or wire change is
required, and `AxeEntryEditorSeed` / `AxeEntryMutationRequest` / `AxeEntryEditorResult` keep their current shapes so
`src/sase/ace/tui/actions/axe_config_actions/` is untouched.

## Design principles

- **One page.** Every schema property is always on screen, in schema-friendly order, whether or not it has a value.
- **Navigation always works.** Property movement never competes with a text editor for a keystroke.
- **Edit where you look.** A value is edited in the row that displays it, not in a detached pane.
- **Calm layout.** The sheet does not reflow as the selection moves; the panel is only as large as its content needs.
- **Nothing lost.** Every capability of the current editor keeps a key and a visible affordance.

## Target UX

### The sheet

```
╔ Edit AXE chop · telegram / tg_inbound ═══════════════════════════════════════════════════════╗
║ ● running · 2 pending edits                       Scope  ▎1 user▎  2 overlay:sase_athena.yml ║
║ ! Editing base chop 'tg_inbound' affects every generated instance.   ~/.config/sase/sase.yml ║
╟──────────────────────────────────────────────────────────────────────────────────────────────╢
║ BASICS                                                                                       ║
║   script *      sase_chop_tg_inbound                                                  ·user  ║
║ ▌description    Process inbound Telegram messages▏                                   edited  ║
║   enabled       true                                                                 ·local  ║
║   run_every     5m                                                                    ·user  ║
║   timeout       90s                                                                ·inherit  ║
║ ADVANCED                                                                                     ║
║   env           SASE_AXE_MODE: strict                                              ·overlay  ║
║   inhibit_if    —                                                                      unset ║
║   trigger       —                                                                      unset ║
║   once_per      —                                                                      unset ║
║   for_each      2 targets                                                          ·overlay  ║
║   vars          mentor: security                                                   ·overlay  ║
╟──────────────────────────────────────────────────────────────────────────────────────────────╢
║ description  string                                                                  INSERT  ║
║ Human-facing summary shown in AXE listings and notifications.                                ║
║ Effective "Process inbound messages"  Layer(user) "Process inbound messages"  Inherits —      ║
╟──────────────────────────────────────────────────────────────────────────────────────────────╢
║ ⏎ commit · esc browse · ^R inherit · 1-2 scope · ^S preview & save                            ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝
```

Regions, top to bottom: **header** (title, run status + pending-edit count, numbered scope chips, target path, and the
generated-instance warning when present), **sheet** (group headers plus one row per property), **detail dock** (fixed
height so the sheet never shifts: name + editor kind + required marker + vim mode, wrapped description, and the
Effective / Layer / Inherits values), **status line** (validation error, parse-deferred note, or transaction status),
and a mode-aware **hint bar**.

The modal container sizes to its content (`height: auto` with a percentage cap; the sheet is a `VerticalScroll` with
`height: auto` and a capped `max-height`, verified to size-to-content and then scroll). A three-property lumberjack
renders as a compact dialog instead of today's near-empty full-screen box; a chop that outgrows a short terminal scrolls
with the active row kept in view.

### Row anatomy and state

`selection bar │ name (+ ``*`` when required) │ value │ badge`, with name and badge in fixed columns so values align.
Row states, all distinguishable at a glance:

| State                        | Value cell                       | Badge                                  |
| ---------------------------- | -------------------------------- | -------------------------------------- |
| Set in the target layer      | value                            | `·user` / `·overlay:<name>` / `·local` |
| Inherited from another layer | inherited value, dim             | `·<source>`                            |
| Unset everywhere             | `—`, dim                         | `unset`                                |
| Edited in this session       | draft value                      | `edited` (green)                       |
| Marked inherit/reset         | value that will be restored, dim | `inherit` (copper)                     |
| Draft fails validation       | raw draft text, red              | `edited` + message in the status line  |

Multi-line values (`env`, `for_each`, `vars`, `inhibit_if`, `trigger`, `once_per`, and any `text` field) render a
single-line summary in browse mode and expand in place when edited.

### Modes and keys

Two modes inside the edit stage. **Browse** is the default (no focused widget, so every binding fires); **cell** means
the active row's editor is focused. An add flow (`seed.new_entry`) opens directly in cell mode on the first property so
typing still starts immediately; editing an existing entry opens in browse mode.

Browse mode:

| Key             | Action                                                            |
| --------------- | ----------------------------------------------------------------- |
| `j` `k` `↑` `↓` | Previous / next property                                          |
| `g` `G`         | First / last property                                             |
| `⏎` `i`         | Edit the active property in place (a bool toggles instead)        |
| `space`         | Toggle bool / cycle enum forward                                  |
| `h` `l` `←` `→` | Cycle enum backward / forward                                     |
| `^R`            | Mark inherit/reset, or restore the property to its original value |
| `1`…`9`         | Select the numbered writable scope                                |
| `^T`            | Cycle scope                                                       |
| `^S`            | Preview & save                                                    |
| `^L`            | Reload after a stale-write conflict                               |
| `esc`           | Close the panel                                                   |

Cell mode (the `ctrl`-tier keys above keep working, since they reach the modal through a focused editor):

| Key               | Action                                                        |
| ----------------- | ------------------------------------------------------------- |
| `⏎`               | Single-line: commit and return to browse. Multi-line: newline |
| `esc`             | INSERT → NORMAL, then NORMAL → commit and return to browse    |
| `tab` `shift+tab` | Commit and edit the next / previous property                  |
| Vim keys          | The full `VimTextArea` layer, unchanged                       |

Escape therefore escalates predictably: INSERT → NORMAL → browse → closed. Edits stay live-as-you-typed (as today);
leaving cell mode never discards a draft, and `^R` is the way back to a property's original value.

The preview stage is unchanged in behavior: `↑`/`↓`, `^D`/`^U`, `g`/`G` scroll; `⏎` saves (and restarts AXE when it is
running); `^O` saves only; `esc` returns to the sheet.

Mouse: clicking a row selects it, clicking its value cell edits it, and clicking a scope chip selects that scope.

### Retirements

- **`a` / `PropertyPickerModal`** — every property is already on the sheet, and the dock shows the description and
  allowed values the picker used to carry. `PropertyPickerModal` itself stays; it keeps four other callers in
  `axe_add_modals.py` and `add_property_modal.py`.
- **`1`–`9` as enum selection** — replaced by scope selection, which is what the numbered chips have always implied.
  Enums keep `space` and `h`/`l`/`←`/`→` (neither AXE schema currently declares an enum).
- **`_ignore_editor_values`, `_loaded_editor_name`, `force_editor`** — a per-edit editor makes them unnecessary.
- **The inclusion API in `SchemaObjectForm`** (`include`, `addable_fields`, `initially_included`, the `included` flag) —
  now that the only consumer shows every field, prune it and update `tests/ace/tui/test_schema_object_form.py`.
  Sparse-patch behavior (`touched` → operations) is unchanged.

## Implementation

1. **`src/sase/ace/tui/modals/axe_entry_sheet.py` (new, no Textual imports).** Pure view model over `SchemaObjectForm`:
   ordered row descriptors (group, name, required, single-line value summary, badge text/style, row state), the
   name/badge column widths for a given width, detail-dock lines, status-line text, and the hint string per mode/stage.
   This is where the rendering logic becomes unit-testable without a pilot.
2. **`axe_entry_editor_types.py`.** Keep basics/advanced ordering, but replace the `allowed` allowlist filter with an
   explicit hidden-key set (`name` for chops, `chops` for lumberjacks) so a future schema property appears on the sheet
   automatically (defaulting into ADVANCED) instead of silently vanishing. Drop `initially_included`.
3. **`axe_entry_editor_rendering.py`.** Replace the two-pane compose with header / sheet / dock / status / hints. Each
   property is a `Horizontal` row of `Static` cells. Entering cell mode mounts one `_AxeValueInput` (height 1,
   borderless, in the value column) or `_AxeValueTextArea` (fixed height, expanding the row) into the active row and
   focuses it; leaving cell mode removes it. That is one mount per edit session, not per keystroke — a height-1
   borderless `SingleLineVimTextArea` inside a row has been verified to lay out, render, accept typing, and honor the
   INSERT→NORMAL escape. Both editor subclasses route `_update_vim_mode_display` to the dock instead of a border title,
   and intercept `tab`/`shift+tab` in `_on_key` (screen bindings cannot see those keys past a focused editor). Keep
   rendering synchronous, allocation-light, and free of file or schema I/O.
4. **`axe_entry_editor_modal.py`.** Add the `browse`/`cell` mode to the existing `edit`/`preview` stage, rewire
   `BINDINGS` to the tables above, implement scope digits, `enter`/`i` to edit, `esc` escalation, `tab`/`shift+tab`
   movement, and scroll-active-row-into-view. Keep the `ConfigTransactionControllerMixin` contract
   (`_build_transaction_mutation`, `_plan_transaction`, `_apply_transaction`, `_accept_transaction_reload`,
   `_transaction_scope_changed`, `_dismiss_transaction_result`) and the running/save-only restart semantics exactly as
   they are.
5. **`styles.tcss`.** Replace the `AxeEntryEditorModal` block with the new regions: auto-height container, capped
   auto-height sheet, fixed-height dock/status/hints, selected-row accent, and badge/value color classes. Keep the
   existing amber/copper palette. The `-narrow` variant (below 90 columns) shrinks the name column, drops the badge
   column, and keeps a single-column dock rather than switching layouts.
6. **`src/sase/config/sase.schema.json`.** Add `description` strings for the AXE chop properties that lack them
   (`description`, `run_every`, `timeout`, `env`, `inhibit_if`, `trigger`, `once_per`), sourced from the chop property
   table in `docs/axe.md`. A one-page sheet shows help for the selected row, and six of eleven chop rows currently have
   none. Additive metadata only — no validation change; Config Center picks it up for free.
7. **`docs/ace.md`.** Update the AXE add/edit prose and add a key table for the panel, covering browse mode, cell mode,
   and the preview stage. Modal bindings are hardcoded, so `src/sase/default_config.yml` needs no change; check
   `help_modal/axe_bindings.py` only for stale wording about the edit flow.

## Regression coverage

**Unit (`tests/ace/tui/test_axe_entry_sheet.py`, new).** Row ordering and grouping for both kinds including a property
absent from the basics/advanced lists; the six row states and their badges; value summaries for scalars, maps, and
lists; column widths at wide and narrow widths; dock content for a field with and without a description, with and
without a target-layer value; hint text per mode and stage.

**Behavior (`tests/ace/tui/test_axe_entry_editor_modal.py`, updated).** Keep the existing scope, sparse-draft,
preview-back, restart, and save-only coverage, then add:

- `up`/`down`/`j`/`k` move the active property from the mount-time state — the regression that motivated this work — and
  again after committing an edit.
- `⏎` enters cell mode and mounts exactly one editor; `esc` `esc` returns to browse with the draft intact; the value is
  not corrupted by navigation keys.
- Every schema property is present without any add step; the modal no longer pushes `PropertyPickerModal`.
- Digits select scopes and match the chip labels; `^T` still cycles.
- `space` toggles a bool from browse mode; `^R` marks inherit and restores.
- `tab` / `shift+tab` commit and move while a cell editor is focused.
- `^S`, `^R`, and `^T` still act while a cell editor holds focus.
- Invalid drafts surface in the status line and block preview, as today.

**Visual (`tests/ace/tui/visual/test_ace_png_snapshots_axe_editor.py`).** Refresh the affected goldens
(`axe_chop_editor_basics`, `axe_chop_editor_advanced`, `axe_generated_instance_warning`,
`axe_editor_validation_failure`, `axe_editor_diff_preview`, `axe_editor_constrained_width`) and add scenarios for browse
mode with unset rows, an active single-line cell editor, an expanded multi-line YAML cell, and the compact lumberjack
sheet that proves the panel shrinks to its content. Accept only these intentional golden changes.

## Verification

1. `just install`
2. `just test tests/ace/tui/test_axe_entry_sheet.py tests/ace/tui/test_axe_entry_editor_modal.py tests/ace/tui/test_schema_object_form.py tests/ace/tui/test_axe_config_actions.py`
3. `just test-visual` — inspect `.pytest_cache/sase-visual/` artifacts before accepting any snapshot update
4. `just check`
5. Drive the real panel once (`sase ace`, AXE tab, `e` on a chop and on a lumberjack): arrow keys move on open, an edit
   round-trips through preview to a written file, and the generated-instance warning still appears.

## Risks and safeguards

- **Mount/unmount focus races.** Entering cell mode mounts a widget and immediately focuses it. Follow the codebase's
  established pattern (`call_after_refresh` before focusing, as `_initialize_editor` already does) and assert the
  mounted-editor count in tests so a leaked editor from a previous row cannot steal keys.
- **Escape escalation regressions.** Three `esc` meanings (leave INSERT, leave cell, close panel) are the highest-risk
  interaction. Cover each hop explicitly, including NORMAL-mode `esc` on a multi-line editor, and keep the hint bar
  naming the next step in every mode.
- **Key routing surprises.** Printable keys and arrows never reach the modal past a focused editor; `ctrl`-tier keys do;
  `tab` moves focus rather than firing a binding. The design assigns keys accordingly, and the focused-editor binding
  tests lock that contract in.
- **Layout stability.** The dock is fixed-height so selection movement cannot reflow the sheet; only multi-line cell
  editing changes row height, and the active row is scrolled back into view afterward.
- **Responsiveness.** Rendering stays synchronous string/`Text` building over at most a dozen rows with no I/O on the
  keystroke path, per the TUI performance rules; planning and applying keep running in the existing thread workers.
- **Retired symbols.** Pruning the `SchemaObjectForm` inclusion API and the editor's picker path can surface Symvision
  unused-symbol findings; consult `sase/memory/symvision.md` if `just check` reports them rather than adding pragmas.

## Out of scope

- Config Center's single-field editor (`config_edit_modal.py`) and the shared transaction/preview machinery
  (`config_transaction*.py`) keep their current behavior.
- The add flows that run _before_ the panel (`AxeAddChooserModal`, `AxeScriptPickerModal`, `AxeNewEntryIdentityModal`)
  are unchanged.
- No change to AXE config planning, YAML surgery, chezmoi handling, restart semantics, or the post-write commit offer.
- Editing a chop's `for_each`-generated instances individually; the panel keeps editing the base chop and warning.
