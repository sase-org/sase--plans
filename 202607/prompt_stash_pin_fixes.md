---
create_time: 2026-07-11 17:20:54
status: done
prompt: 202607/prompts/prompt_stash_pin_fixes.md
tier: tale
---
# Fix Prompt-Stash Panel Pin & Selection Bugs

## Problem

The unified prompt-stash panel (`@` / `Ctrl+G p`) has a set of related bugs around the `<space>` pin keymap and pop/keep
marking, the flagship one being the user-reported symptom: **`<space>` does not pin multi-prompt stashes**.

## Investigation Summary / Root Causes

### Bug 1 (primary): `<space>` silently refuses to pin bundle (multi-prompt) rows

`StashedPromptsModal.action_toggle_pin` (`src/sase/ace/tui/modals/stashed_prompts_modal.py:191`) early-returns when
`_is_selectable(entry)` is false, and `_is_selectable` returns false for every bundle row (any entry whose stored text
splits into >1 prompt segment).

This guard is **vestigial**:

- Commit `599d71caa` ("bundle multi-pane prompt stash rows") introduced `_is_selectable` to keep _bulk restore
  selection_ ("space = restore+keep" at the time) limited to single-prompt rows.
- Commit `832656055` ("add persistent prompt stash pins") repurposed `<space>` from "mark restore+keep" to "toggle
  persistent pin" but inherited the guard unchanged.
- Pinning is id-generic in the Rust store (`set_prompt_stash_pinned` just flips a flag on matching ids — verified in
  `sase-core` `crates/sase_core/src/prompt_stash/store.rs`); nothing downstream requires pinned rows to be
  single-prompt. The keep-restore path, the `gS` update-pinned path, and the badge pinned count all handle bundles fine.

Aggravating factors that make this worse than a missing feature:

- The no-op is **silent** — no toast, no glyph change — so it reads as flaky ("doesn't work all of the time").
- **Pinned bundles already exist** through a side door: `gS` (update pinned stash) rewrites a pinned row's text from the
  current draft; a multi-pane draft turns a pinned single row into a pinned bundle — which `<space>` then _cannot
  unpin_.
- **False-positive bundles**: bundle detection (`entry_is_bundle`) splits the stored text on top-level `---` lines
  (outside code fences). A _single_ stashed prompt containing a markdown horizontal rule is classified as a bundle, so
  `<space>` silently no-ops on what the user believes is a single prompt. (The `---` split itself is the canonical
  multi-prompt format shared with the prompt bar and is not in scope to change; fixing Bug 1 makes the misclassification
  harmless for pinning.)

The behavior is codified by `test_space_is_inert_for_bundle_rows` in
`tests/ace/tui/modals/test_stashed_prompts_modal.py`, which must flip.

### Bug 2: confirm pops pinned rows marked via `tab`/`a`, contradicting the digit/enter path

The panel has two restore paths with **contradictory pin semantics**:

- Single-row restore (digit keycaps, or `enter` with nothing marked) goes through `_single_restore_result`, which
  respects pins: pinned → `keep_ids` (load into the bar _without_ popping the stash). Codified by
  `test_digit_restores_pinned_row_to_keep`.
- Marked restore (`tab` / `a` then `enter`) goes through `action_confirm`, which routes **all** marked ids to `pop_ids`
  regardless of pin state — the code even notes "keep_ids only comes from the pin-aware single-row restore path".

Consequence: `a` + `enter` ("restore everything") silently **destroys every pinned template**, and `tab`-marking a
pinned row pops it while digit-restoring the same row keeps it. Root cause: pins were bolted on after the pop/keep
transport existed and `action_confirm` was never taught about `_pinned`.

**Proposed resolution (decision point for review):** on confirm, route marked ids that are pinned into `keep_ids`
(restore + stay stashed) and the rest into `pop_ids`. Pinned rows then survive every restore path; `d` (explicit delete)
remains the way to remove them, and unpin-then-pop remains available. This reverses the tested behavior in
`test_pin_is_orthogonal_to_pop_selection` (pin stays orthogonal to _marking_, but confirm resolution becomes pin-aware)
— flagged here so it can be vetoed at plan review if the "explicit tab always pops" semantics are preferred.

### Bug 3: `tab`/`a` cannot mark bundle rows even though `enter`/digits restore+pop them

Same vestigial `_is_selectable` guard as Bug 1, applied to `action_toggle_pop` and `action_toggle_all`. It is internally
inconsistent today: pressing `enter` on a highlighted bundle (or its digit keycap) restores and pops it
(`test_enter_with_no_toggle_restores_highlighted_bundle`, `test_digit_restores_bundle_row_to_pop`), yet `tab` cannot
mark the same row, and `a` excludes it from "restore all". The transport (`_apply_stash_restore` +
`entries_to_restore_items`) expands bundles uniformly for both paths, so the restriction protects nothing. Fix: delete
`_is_selectable` entirely; all rows become markable.

### Verified non-bugs (no action)

- **Store concurrency**: the Rust store takes an exclusive file lock around each full read-modify-write with atomic
  temp-file rename; pin-vs-pop lost updates cannot happen, in-process or cross-instance.
- **Pin persistence pipeline**: `PinToggled` → app handler → async task → `set_prompt_stash_pinned`, serialized by the
  app-level write lock; message ordering guarantees the pin persists even when `space` is immediately followed by
  `enter`/digit/escape.
- **Key routing**: Textual 8.2.7's `OptionList` binds no `space` key, so the screen-level binding is never shadowed.

## Implementation Plan

All changes are presentation-layer (TUI modal selection state) plus tests/docs — per the Rust core boundary rule, no
`sase-core` changes are needed (the pin flag operation there is already id-generic).

### 1. `src/sase/ace/tui/modals/stashed_prompts_modal.py`

- Remove `_is_selectable` and its three call sites so `space` (pin), `tab` (pop mark), and `a` (mark all) work on every
  row, bundles included.
- Make `action_confirm` pin-aware: marked ids split into `keep_ids` (pinned) and `pop_ids` (unpinned), using the modal's
  optimistic `_pinned` set. Fold the duplicated `if not pop_ids and not delete_ids` blocks while there.
- Update the module docstring ("space toggles a single-prompt row's pin" → any row; document pin-aware confirm) and the
  in-panel hint text if wording needs it (`tab ✓ restore+pop` still describes unpinned rows; keep or adjust to
  `tab ✓ restore` — implementer's call, hints must stay accurate).

### 2. `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash.py`

- No behavior change required (`_apply_stash_restore` already honors `keep_ids`; the pin persist handler is id-generic).
  Update the stale comment claiming "The panel does not currently mark keep ids".
- Doc-only fixes: `stash_all_panes` docstring in `src/sase/ace/tui/widgets/_prompt_input_bar_stash_actions.py` still
  claims "each non-empty pane becomes its own stash entry" — stale since bundling; reword to the single-bundle-row
  reality.
- Optional cleanup: delete the dead `_apply_prompt_stash_count` shim (no production callers; it clobbers the pinned
  count to 0 if ever used) and its test-host stubs.

### 3. Tests (`tests/ace/tui/modals/test_stashed_prompts_modal.py`, `tests/ace/tui/actions/`)

- Flip `test_space_is_inert_for_bundle_rows` → `space` pins a bundle row: glyph state, `PinToggled` posted, and
  (app-layer) persisted via `set_prompt_stash_pinned`.
- Add: unpinning a bundle row; digit/enter restore of a _pinned bundle_ returns `keep_ids` and expands to all segments
  without popping the store.
- Flip `test_toggle_all_selects_only_single_prompt_rows` → `a` includes bundle rows; add `tab` marks a bundle row and
  confirm returns it in `pop_ids`.
- Update `test_pin_is_orthogonal_to_pop_selection` per Bug 2's resolution: pinned + `tab` + `enter` → `keep_ids`; add
  mixed case (`a` over pinned + unpinned rows → pinned in `keep_ids`, unpinned in `pop_ids`, store pops only the
  unpinned).
- App-layer confirm tests in `tests/ace/tui/actions/test_prompt_stash_restore_confirm.py` extended for the keep-path
  store behavior (entry still on disk after restore).
- Existing PNG goldens are unaffected (no rendering change); the visual stash-modal test sets marker state directly.

### 4. Docs / help sync

- The help modal only lists "Restore stashed prompt" for `@` (no per-key panel docs), so no help-modal change; the
  in-panel hint line is the user-facing documentation and is covered in step 1.

## Verification

- `just check` (lint + mypy + full test suite, including the visual snapshot suite).
- Manual TUI pass: stash a multi-pane draft (`gs`), open the panel, verify `space` pins the bundle row (glyph + badge
  pinned count + persisted across reopen), `tab`/`a` mark it, restore of a pinned row keeps it stashed, and `d` still
  deletes pinned rows.
