---
create_time: 2026-07-09 12:46:34
status: done
prompt: 202607/prompts/ctrl_g_ctrl_c_prompt_chord_fix.md
tier: tale
---
# Fix: `<ctrl+g><ctrl+c>` does nothing in the prompt input widget

## Problem

In the `sase ace` TUI prompt input widget, the `<ctrl+g><ctrl+c>` chord (labeled `^G ^C`, "cancel all panes") does
nothing. The `^G` prefix panel opens and lists `^C cancel all panes`, but pressing `ctrl+c` afterward silently clears
the prefix without cancelling the prompt stack.

Every other `^G` continuation works: `^G g` (editor), `^G <enter>` (submit), `^G -`/`^G =` (structural),
`^G j`/`k`/`J`/`K` (pane nav), `^G s`/`S`/`x`/`X`/`p` (stash/xprompt). Only the `ctrl+c` continuation is dead.

## Root Cause (verified)

The `^G` prefix is a two-key chord. After the prefix is pending, the _second_ key is resolved in
`src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`, in two near-identical handlers —
`_handle_insert_g_prefix_key` and `_handle_normal_g_prefix_key` — with this line:

```python
key = event.key if event.key == "enter" else event.character or event.key
```

The resolved `key` is then passed to `PromptInputBar.dispatch_g_prefix_key(key, ..., via_ctrl_g=True)`, which matches it
against the declarative `_PROMPT_G_PREFIX_BINDINGS` table in `_prompt_input_bar_g_prefix_actions.py`. That table keys
the cancel action as the literal string `"ctrl+c"`:

```python
_PromptGPrefixBinding("ctrl+c", "action_cancel_all", ..., ctrl_g_only=True)
```

The `event.character` normalization exists on purpose: named punctuation keys report `event.key == "minus"` /
`"equals_sign"`, but the binding table uses the literal chars `"-"` / `"="`, so the code prefers `event.character`
(`"-"`, `"="`) to match. This works for every printable key.

It breaks for `ctrl+c`. In a **real terminal**, Textual delivers `ctrl+c` as `Key(key="ctrl+c", character="\x03")` — the
ETX control byte `0x03`. Because `"\x03"` is truthy, `event.character or event.key` evaluates to `"\x03"`, not
`"ctrl+c"`. `dispatch_g_prefix_key("\x03", ...)` finds no matching binding, returns `False`, and `action_cancel_all` is
never called. The handler still returns `True` (the key is consumed), so the prefix is cleared and nothing visible
happens — exactly the reported symptom.

`enter` avoids this only because it is explicitly special-cased (its character `"\r"` is likewise non-printable and
would otherwise misresolve). `ctrl+c` has no such special-case, so it falls through to the broken path. This is a latent
bug for **any** `ctrl+<x>` continuation, not just `ctrl+c` — `ctrl+c` is simply the only control chord currently in the
table.

### Confirmations

- **Textual 8.2.7 behavior** (verified in the workspace venv): real-terminal characters are `ctrl+c → "\x03"`,
  `ctrl+g → "\x07"`, `enter → "\r"` (all non-printable), while `minus → "-"`, `equals_sign → "="`, letters → themselves
  (all printable single chars).
- **The same chord already works elsewhere the correct way.** `frontmatter_panel.py`'s `_handle_ctrl_g_prefix`
  implements `Ctrl+G Ctrl+C` by matching `event.key == "ctrl+c"` directly (not `event.character`) and calls
  `action_cancel_all`. This proves `ctrl+c` _is_ delivered as a key event and that `event.key` is the correct
  discriminator; the prompt text area is the outlier.

### Why existing tests did not catch it

- The unit test `test_dispatch_g_prefix_key_routes_each_continuation` calls
  `bar.dispatch_g_prefix_key("ctrl+c", via_ctrl_g=True)` directly with the already-correct string, bypassing the buggy
  key-resolution step entirely.
- The end-to-end tests (`pilot.press("ctrl+g", "=")`, etc.) only press printable continuations. Critically, Textual's
  test `Pilot` produces `character=None` for `ctrl+c` (verified), so a naive `pilot.press("ctrl+g", "ctrl+c")` test
  would resolve `None or "ctrl+c" → "ctrl+c"` and **pass even with the bug present**. The test harness does not
  reproduce the real terminal's `"\x03"`, which is why the gap went unnoticed. Any regression test must inject the
  real-terminal character.

## Scope / Boundary

This is Textual key-dispatch (keybinding) logic — presentation-only per the repo's Rust core boundary rule ("keybindings
… and Python glue can stay in this repo"). No `sase-core` / Rust change is required. No config, wire, or binding-schema
change is required.

## Recommended Fix (Option A): make the chord key-resolution robust

Fix the second-key resolution so control chords resolve to their canonical `event.key` name while printable keys still
normalize through `event.character`. Prefer `event.character` **only when it is a single printable character**;
otherwise fall back to `event.key`:

```python
character = event.character
if character and len(character) == 1 and character.isprintable():
    key = character
else:
    key = event.key
```

Behavior across all cases (real terminal and Pilot):

- `ctrl+c` → character `"\x03"` (or `None` in Pilot), non-printable/empty → `key = "ctrl+c"` → matches binding →
  `action_cancel_all` fires. **Fixed.**
- `minus`/`equals_sign` → `"-"`/`"="` printable → unchanged (`"-"`/`"="`).
- letters (`j`, `x`, `S`, …) → printable → unchanged.
- `enter` → `"\r"` non-printable → `key = "enter"`; the explicit `enter` special-case becomes redundant and is removed,
  collapsing two branches into one helper.
- `ctrl+g` (the `^G ^G` editor path) → still detected via the existing `event.key == "ctrl+g"` check.

Implementation notes:

- Extract the resolution into a single module-level helper (e.g. `_resolve_g_prefix_second_key(event) -> str`) and call
  it from **both** `_handle_insert_g_prefix_key` and `_handle_normal_g_prefix_key`, which currently duplicate the buggy
  line. This removes the drift risk between the insert- and normal-mode paths.
- Leave the subsequent `if key == "g" or event.key == "ctrl+g":` editor branch unchanged.
- No change to the binding table, hint labels, the `^C` display in `_display_g_prefix_key`, the footer subtitles
  (`[^C] cancel`), `frontmatter_panel.py`, or any docstring — the intended `<ctrl+g><ctrl+c>` binding is preserved
  exactly as documented across the codebase.

This directly answers the user's primary request ("diagnose the root cause and fix it") and keeps the existing,
conventional `^C` cancel chord consistent with the rest of the prompt UI (footer, frontmatter panel).

## Alternative Fix (Option B): rebind to `<ctrl+g>c`

The user offered this fallback ("if this is because `<ctrl+c>` is special, change the trigger to `<ctrl+g>c`"). It would
change the single binding-table entry key from `"ctrl+c"` to `"c"`. `"c"` is a printable letter, so it resolves
correctly through the existing character path.

Not recommended, because:

- It leaves the underlying normalization bug in place; the next `ctrl+<x>` continuation added would hit the same dead
  chord.
- It has a wider blast radius than it appears: `_display_g_prefix_key`'s `"ctrl+c" → "^C"` special-case, the
  `_g_prefix_label_cancel_all` / `_g_prefix_available_cancel_all` docstrings, the `frontmatter_panel.py` `Ctrl+G Ctrl+C`
  handler, footer subtitles showing `[^C] cancel`, and the unit-test expectation `("ctrl+c", "cancel all panes")` would
  all need updating to stay coherent.
- It diverges the prompt-stack cancel chord from the `^C` convention used everywhere else in the prompt UI.

`ctrl+c` is not "special" in an unfixable sense — it is delivered as a normal key event (proven by the working
frontmatter-panel handler). Option A is the cleaner fix. Fall back to Option B only if the user prefers the bare-`c`
chord.

## Testing

1. **Red-green regression test** (the important one). Add an end-to-end test in
   `tests/ace/tui/widgets/test_prompt_g_prefix_hints.py` mirroring
   `test_normal_ctrl_g_continuation_dispatches_in_normal_mode`, but for the cancel chord. Because
   `pilot.press("ctrl+c")` does **not** reproduce the real-terminal `"\x03"` character, the test must drive the prefix
   with `ctrl+g` and then deliver a **synthetic** `Key("ctrl+c", "\x03")` to the active text area (e.g. post the message
   / invoke the key handler with that event), asserting that `action_cancel_all` fires and the hint panel closes.
   Confirm this test **fails before** the fix and **passes after** — proving it reproduces the real bug rather than the
   Pilot-sanitized one. Cover both the insert-mode and normal-mode handlers, since they are separate code paths.

2. **Focused helper unit test** (optional but cheap): assert `_resolve_g_prefix_second_key` maps
   `Key("ctrl+c", "\x03") → "ctrl+c"`, `Key("enter", "\r") → "enter"`, `Key("minus", "-") → "-"`, and
   `Key("j", "j") → "j"`.

3. Keep the existing `test_dispatch_g_prefix_key_routes_each_continuation` unit test green.

4. Run `just check` (install first in an ephemeral workspace via `just install`), plus a targeted run of the prompt
   g-prefix test module.

## Files Touched

- `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` — add the shared `_resolve_g_prefix_second_key` helper;
  use it in `_handle_insert_g_prefix_key` and `_handle_normal_g_prefix_key`; drop the redundant `enter` special-case.
- `tests/ace/tui/widgets/test_prompt_g_prefix_hints.py` — add the synthetic-`\x03` regression test(s) and the optional
  helper unit test.

## Acceptance Criteria

- Pressing `<ctrl+g><ctrl+c>` in the prompt input widget cancels all prompt panes in both insert and normal vim modes,
  in a real terminal.
- The `^G` hint panel still lists and then closes correctly for the cancel continuation.
- All other `^G` continuations remain unchanged.
- New regression test fails on the pre-fix code and passes on the fixed code; `just check` passes.
