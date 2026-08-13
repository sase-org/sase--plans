---
tier: tale
title: Frame the snippet pane instead of filling it
goal:
  The pinned snippet prompt pane renders its contents with exactly the same syntax
  highlighting as every other prompt pane, and announces itself through solid-colored
  borders that survive 256-color terminals.
size: medium
proposed_by: bbugyi200.athena.zq
create_time: 2026-08-13 13:43:24
status: wip
---

# Plan: Frame the snippet pane instead of filling it

## Problem

Opening a snippet target pane (`⇥ <trigger>`) paints the pane with a translucent fill,
and in a real terminal that fill can come back as a saturated salmon slab. Every xprompt
/ markdown highlight color inside the pane is chosen for contrast against a near-black
background, so on that fill the pane's contents become close to unreadable — directives,
invocations, artifact refs, and body text all collapse into one another.

The user's requirement: **snippet pane contents must be highlighted identically to any
other prompt pane.** The pane's identity has to move off the fill and onto the borders.

### Root cause (diagnosed, not guessed)

`src/sase/ace/tui/styles.tcss:3498-3505`:

```tcss
PromptInputBar .prompt-pane.snippet-target.active {
    border-left: thick $primary;
    background: $primary 12%;      /* <-- the bug */
}

PromptInputBar .prompt-pane.snippet-target.active:focus {
    background: $primary 12%;      /* <-- the bug */
}
```

ACE pins the `flexoki` theme (`src/sase/xprompt/highlight_theme.py:16`, applied at
`src/sase/ace/tui/actions/_state_init_runtime.py:65`), where `$primary = #205EA6` and
`$surface = #1C1B1A`. Textual flattens `$primary 12%` over `$surface` to **`#1C232A`** —
confirmed by sampling the committed golden
`tests/ace/tui/visual/snapshots/png/prompt_stack_snippet_new_120x40.png`, whose snippet
pane is `rgb(28, 35, 42)`.

In the reported screenshot the same pane is `rgb(241, 155, 116)`. The chain that turns
one into the other:

1. **The terminal is running Rich in 256-color mode, not truecolor.** In that screenshot
   flexoki's `$background #100F0F` renders as `#121212` (xterm-256 index 233),
   `$surface #1C1B1A` renders as `#1C1C1C` (index 234), and body text lands on the
   232-255 grey ramp. The bar's own `border: solid $accent` (`#9B76C8`) renders as
   `#8787D1`, one unit off xterm index 104's `#8787D7`. Those are downgrades, not
   truecolor.
2. **`#1C232A` downgrades to xterm-256 index 16.** Verified:
   `rich.color.Color.parse("#1c232a").downgrade(ColorSystem.EIGHT_BIT).number == 16`.
   Any low-alpha tint of a theme color over a near-black background lands in the 16-21
   range — these are the darkest corner of the 6×6×6 cube, so a near-black blend has
   nowhere else to go.
3. **Indices 16-21 are the slots base16/base24-style terminal palettes overwrite.**
   `base16-shell` and friends remap 16 to `base09` (an orange accent) and 17-21 to
   further scheme colors. The observed `#F19B74` is that remapped index 16.

The same fault applies to the _parked_ snippet pane: `border-left: thick $primary 30%`
flattens to `#14263C`, which downgrades to index **17** — also in the remapped range.
The agent pane's `border-left: thick $accent 30%` flattens to `#392D46` → index 53,
which is a real cube slot, so it survives, but it is still the same unsafe idiom.

**The rule this establishes:** in the prompt bar, never express state as a low-alpha
blend of a theme color over the dark background. Use solid theme colors, and encode
emphasis with _weight_ instead of _opacity_.

## Design

### The snippet pane is framed, not filled

The pinned snippet pane is always the bottom item of the stack
(`_prompt_stack_state.py:275-284` appends it and never reorders it), so it already owns
four edges without reserving a single extra terminal row:

```
──── ▍⇥ repic · ~/…/sase.yml new ────────────   <- top edge: the snippet separator (Static)
▊ <snippet body — normal $surface fill>          <- left edge: the pane's accent bar
                                                    right edge: the bar's own frame
[Enter] save ⇥ repic  [Esc] normal  …            <- bottom edge: the bar's frame + subtitle
```

Recolor those four edges and the snippet draft gets a complete enclosure while its fill
stays byte-for-byte the fill of an agent pane. That is the whole design; everything
below is how each edge behaves.

### Two encodings, no alpha

- **Hue = what kind of pane this is.** `$accent` for an agent prompt pane, `$primary`
  for the pinned snippet draft. Unchanged from today.
- **Weight = whether it has focus.** The left accent bar switches border _type_, not
  opacity: `thick` (a full `█` block) when active, `vkey` (a `▏` hairline) when parked.
  Both are fully opaque theme colors, so both are palette-safe. The perceived ink is
  close to today's (a 30%-alpha full block and a 100% quarter-width block cover
  comparable area), so this is not a loudness change — it is the same look expressed in
  a way the terminal cannot mangle.

Only `thick`, `tall`/`panel`, and `vkey` use a single glyph for every row of a left edge
(`textual._border.BORDER_CHARS`); `wide`, `inner`, and friends vary by row and would
render a broken bar. Use `thick` and `vkey`.

### The bar frame answers "what does Enter do?"

`<enter>` on the snippet pane **saves the snippet**; `<enter>` anywhere else launches
agents. That is a bar-level mode, so the bar-level frame should say so — exactly the
precedent xprompt targeting already sets
(`PromptInputBar.xprompt-target { border: double $secondary; }`).

This yields a coherent grammar for the bar frame:

| Frame               | Meaning                                                     |
| :------------------ | :---------------------------------------------------------- |
| `solid $accent`     | ordinary prompt; `<enter>` launches                         |
| `double $secondary` | bound to an xprompt target                                  |
| `double $primary`   | **new** — snippet pane focused; `<enter>` saves the snippet |
| `double $warning`   | unsaved changes that would overwrite something on disk      |

`double` = "you are editing something that gets written to a file." Hue names the
destination. That reads at a glance and adds no new visual vocabulary.

The frame follows **focus**, not mere existence: park the snippet with `gk` and the
frame reverts to `$accent` because `<enter>` means "launch" again. The snippet pane
stays identified by its own left bar and separator title.

### Dirty state

Reuse the three states the separator already computes in `_snippet_separator_info`
(`_prompt_input_bar_stack_rendering.py:337-351`), so the frame and the separator's
marker can never disagree:

- `new` (destination does not exist) → frame `$primary`; the green `new` chip in the
  separator already distinguishes it. Typing into a brand-new snippet is not a hazard,
  so it must not raise a warning frame.
- `clean` → frame `$primary`.
- `dirty` (loaded from disk **and** modified) → frame `$warning`. This is the one case
  where saving overwrites existing content, and it matches both the separator's `●`
  marker and the existing `.xprompt-target.dirty` treatment.

Two frame colors, not three. The separator's marker carries the third state.

The separator (the pane's own top edge) shows dirty **regardless of focus** — it is a
property of the pane. The frame shows dirty **only while focused** — it is a property of
the bar's current keybinding mode. When both apply, all four edges are `$warning` and
the escalation reads as one frame.

## Implementation

### 1. `src/sase/ace/tui/styles.tcss` — pane accent bars

Replace the `.prompt-pane` alpha borders with the weight ladder (block starting at line
3470). The fill declarations on `.active` / `.active:focus` / `.inactive` stay exactly
as they are.

```tcss
PromptInputBar .prompt-pane {
    border: none;
    border-left: vkey $accent;
    padding: 0 0 0 1;
    margin: 0;
}

PromptInputBar .prompt-pane.active {
    border-left: thick $accent;
    background: $surface;
}

PromptInputBar .prompt-pane.inactive {
    border-left: vkey $accent;
    background: $background;
    color: $text-muted;
}
```

Refresh the block comment above it to state the hue/weight encoding and the "no alpha
over a dark background" rule with the index 16-21 reason.

### 2. `src/sase/ace/tui/styles.tcss` — snippet pane, no fill

Replace the whole `.snippet-target` block (lines 3491-3511):

```tcss
PromptInputBar .prompt-pane.snippet-target.active,
PromptInputBar .prompt-pane.snippet-target.active:focus {
    border-left: thick $primary;
}

PromptInputBar .prompt-pane.snippet-target.inactive {
    border-left: vkey $primary;
}
```

- **No `background` declaration at all.** The fill falls through to
  `.prompt-pane.active` / `.prompt-pane.inactive`, which is the requirement.
- Both selectors carry three classes, so they outrank the two-class generic rules
  without depending on source order.
- The old `.snippet-target.inactive` fill was already a verbatim duplicate of
  `.prompt-pane.inactive`; deleting it changes nothing.

### 3. `src/sase/ace/tui/styles.tcss` — bar frame

Add immediately after the `PromptInputBar.xprompt-target*` rules (after line 3214), so
the two-class snippet selectors win the source-order tie-break against the equally
specific `.xprompt-target.dirty` / `.stale` / `.readonly` rules:

```tcss
PromptInputBar.snippet-mode.snippet-safe {
    border: double $primary;
}

PromptInputBar.snippet-mode.snippet-dirty {
    border: double $warning;
}
```

`snippet-mode` is always paired with exactly one of `snippet-safe` / `snippet-dirty` so
every snippet frame rule is two classes. Do **not** reuse the bare `dirty` class — that
one belongs to xprompt targeting.

Border type does not change the bar's height or padding (every Textual border edge is
one cell), and `.xprompt-target` already proves `double` renders the border title and
subtitle correctly.

### 4. `src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py`

Add to `PromptInputBarStackRenderingMixin`:

```python
def _snippet_frame_state(self) -> str | None:
    """Return ``"safe"``/``"dirty"`` while the snippet pane holds focus.

    ``None`` off the snippet pane: the bar frame answers "what does
    ``<enter>`` do right now", so it tracks focus, not mere existence.
    """
    if self._mode != "prompt":
        return None
    item = self._stack.selected_item
    if not item.is_snippet_pane or item.snippet_target is None:
        return None
    return "dirty" if self._snippet_separator_info(item).state == "dirty" else "safe"

def _refresh_snippet_frame_classes(self) -> None:
    """Sync the bar-level snippet frame classes with the active pane."""
    state = self._snippet_frame_state()
    self.set_class(state is not None, "snippet-mode")
    self.set_class(state == "safe", "snippet-safe")
    self.set_class(state == "dirty", "snippet-dirty")
```

`_mode` is already declared in the mixin's `TYPE_CHECKING` block; add it there if mypy
disagrees.

Call `_refresh_snippet_frame_classes()` from the **top** of `refresh_cursor_readouts()`
(line 484).

> **Trap:** `refresh_cursor_readouts()` early-returns on `len(self._stack) <= 1` (line
> 493). The frame refresh must run _before_ that return, or the classes go stale the
> moment the stack shrinks back to a single pane.

`refresh_cursor_readouts()` is the right single hook: it already fires on text change,
selection change, resize, focus change, and post-rebuild
(`_prompt_input_bar_stack_lifecycle.py:106,112,249`,
`_prompt_input_bar_stack_rendering.py:482`, `_prompt_input_bar_snippet_pane.py:114`,
`prompt_input_bar.py:642`), and it already recomputes `_snippet_separator_info` for the
separator.

### 5. Separator dirty color

In `refresh_cursor_readouts()`, where the snippet separator's info is refreshed (line
502-503), also sync a state class:

```python
if item.is_snippet_pane:
    info = self._snippet_separator_info(item)
    separator.set_snippet_info(info)
    separator.set_class(info.state == "dirty", "snippet-dirty")
```

Set the same class up front in `_build_pane_widgets()` (line 300-303) so a freshly
mounted dirty snippet paints correctly on the first frame rather than one refresh later.

Add to `styles.tcss` after the existing `.prompt-stack-separator.snippet` rule (line
3531):

```tcss
PromptInputBar .prompt-stack-separator.snippet.snippet-dirty {
    color: $warning;
}
```

## Tests

### New: `tests/ace/tui/widgets/test_prompt_stack_snippet_pane_frame.py`

Mount a real bar through the existing `CaptureApp` harness
(`tests/ace/tui/widgets/prompt_stack_submit_cancel_test_support.py`) and reuse the
`_save_target` / `_name_result` / `_open_snippet` helpers already written in
`test_prompt_stack_snippet_pane_lifecycle.py` (import or copy them; do not re-derive the
`SnippetSaveTarget` shape).

1. **Identical fill (the headline requirement).** With the snippet pane focused, the
   flattened background of the snippet `PromptTextArea` equals that of a focused agent
   pane. Compare `text_area.background_colors[1]`, which is the opaque rendered fill —
   `styles.background` still carries unflattened alpha and would pass a broken
   implementation.
2. **Identical parked fill.** Park the snippet (focus an agent pane); the snippet pane's
   flattened background equals an inactive agent pane's.
3. **Frame follows focus.** Opening the snippet adds `snippet-mode` + `snippet-safe`;
   focusing an agent pane drops all three classes; refocusing the snippet restores them;
   closing the snippet drops them.
4. **Dirty escalation.** Open on an existing body, edit it → `snippet-dirty` replaces
   `snippet-safe`; a _new_ (non-existent destination) snippet stays `snippet-safe` no
   matter how much is typed.
5. **Precedence over xprompt targeting.** With an `XPromptBinding` set _and_ the snippet
   pane focused, the resolved `bar.styles.border_top` color is `$primary` (or `$warning`
   when dirty) — not `$secondary`/`$warning` from the xprompt rules. This is the
   source-order tie-break; it must be pinned.

### New: `tests/ace/tui/widgets/test_prompt_bar_palette_safety.py`

The regression guard that encodes the root cause. Across the four pane states (agent
active/inactive, snippet active/parked) and the four bar frame states (plain,
xprompt-target, snippet-safe, snippet-dirty), collect every resolved
`styles.background`, `styles.border_left`, and `styles.border_top` color and assert:

1. **No alpha.** Every border color is fully opaque (`color.a == 1.0`).
2. **No remapped palette slot.** Every flattened color (`widget.background_colors[1]`
   for fills, the opaque color itself for borders) satisfies:

```python
from rich.color import Color as RichColor
from rich.color import ColorSystem

index = RichColor.parse(color.hex).downgrade(ColorSystem.EIGHT_BIT).number
assert index is None or not (16 <= index <= 21), (
    f"{label} resolves to {color.hex}, which downgrades to xterm-256 index "
    f"{index}; slots 16-21 are remapped by base16-style terminal palettes."
)
```

Include a module docstring explaining _why_ 16-21 is the forbidden range, with the
concrete example (`$primary 12%` over `$surface` → `#1C232A` → index 16 → salmon).

### Visual snapshots

Add one scenario to `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py`:
`prompt_stack_snippet_parked_120x40` — open the snippet pane via the existing
`_open_snippet_pane` helper, then focus the agent pane, so the golden locks "frame
reverts to `$accent`, snippet keeps its `$primary` hairline and separator."

Regenerate affected goldens with `just test-visual -- --sase-update-visual-snapshots`
and **review every diff** in `.pytest_cache/sase-visual/` before committing. Expected to
change:

- `prompt_stack_snippet_new_120x40`, `prompt_stack_snippet_dirty_120x40` (fill removed,
  frame recolored)
- every multi-pane golden, because the inactive agent accent bar goes from a 30%-alpha
  `█` to a solid `▏`: `prompt_stack_two_panes`, `prompt_stack_active_upper`,
  `prompt_stack_compact_inactive_80x30`, `prompt_stack_completion_panel`,
  `prompt_stack_g_prefix_hints`, `prompt_stack_targeted_clean`,
  `prompt_stack_targeted_dirty`, `prompt_stack_targeted_readonly`,
  `prompt_codeblock_highlight_stack_dark`, `prompt_codeblock_highlight_stack_light`,
  `prompt_cursor_readout_stack`, `prompt_todo_stack`, `prompt_xprompt_highlight_stack`

Enumerate by running the suite, not from this list — treat the list as the expected set
and investigate anything outside it. Any golden that changes _fill_ rather than
accent-bar weight is a bug in the change.

## Scope decision

Step 1 also converts the **agent** pane's inactive `border-left: thick $accent 30%` to
the solid weight ladder. That is deliberate: leaving one alpha idiom next to the new
solid one puts two different "inactive pane" languages in the same widget, and
`$accent 30%` is the same provably lossy construct (it only escapes the remapped range
by luck of where it lands in the cube). It is a one-line change in the same block.

The cost is golden churn: ~13 extra PNGs instead of ~2. If that churn is unwanted,
dropping the agent-pane half of step 1 leaves the rest of the plan intact and correct —
the snippet pane fix does not depend on it. Ask before dropping it rather than silently
narrowing.

## Out of scope

`styles.tcss` has 17 `background: <color> <n>%` declarations. Several of the low-alpha
ones over dark surfaces will have the same index 16-21 fault elsewhere in the TUI (agent
list, modals, AXE). This plan fixes the prompt bar only. **File a task bead** for a
repo-wide audit of low-alpha backgrounds using `/sase_new_task`; do not expand this
change to cover them.

Terminal capability detection, forcing truecolor, and any change to the pinned flexoki
theme are all out of scope. The fix must work correctly in a 256-color terminal, not
require a better one.

## Verification

```bash
just install
just test-visual -- --sase-update-visual-snapshots   # then review every diff
just check
```

Hand `just check-full` to `/sase_monitor` before landing — this touches shared
prompt-bar CSS that many suites render through.

Manual confirmation in a real terminal (the failure mode is invisible in truecolor CI):
open ACE, press the snippet-target key, and confirm the pane's background is
indistinguishable from the agent pane above it, that highlighting inside it matches, and
that the frame turns `$primary` on focus and `$accent` when parked.
