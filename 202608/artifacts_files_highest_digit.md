---
tier: tale
title: Number Artifacts sub-tabs by visual order so Files gets the highest digit
goal:
  Artifacts digit shortcuts follow the strip's left-to-right order, so the Files sub-tab
  always carries the highest digit instead of a fixed 4 that lands after higher-numbered
  document-provider tabs.
size: medium
proposed_by: bbugyi200.athena.z6
create_time: 2026-08-13 08:15:42
status: wip
---

# Number Artifacts Sub-Tabs By Visual Order So Files Always Gets The Highest Digit

## Problem

In the ACE Artifacts tab strip, the digit shortcut printed next to each sub-tab no
longer matches the left-to-right order of the panes once any document-provider tab is
configured. A live strip currently renders:

```
1 STITCHES │ 2 Patches │ 3 Beads │ 5 Plans │ 6 Researchs │ 4 Files
```

"Files" is rendered last but is labelled `4`, while the provider tabs that precede it
are labelled `5` and `6`.

The cause is that digits come from two independent sources in
`src/sase/ace/tui/artifact_tabs.py`:

- `FIXED_ARTIFACTS_DIGITS` hard-codes `stitches=1`, `patches=2`, `beads=3`, `files=4`,
  and `_fixed_descriptor()` stamps that digit onto the descriptor.
- `_provider_descriptors()` numbers provider tabs from `offset` starting at `5`
  (`digit_shortcut=str(offset) if offset <= 9 else None`).

But `resolve_artifacts_subtabs()` assembles the visual order as
`stitches, patches, beads, *providers, files` — providers are inserted _before_ the
fixed `files` descriptor, so the fixed digit `4` ends up after `5` and `6`.

## Goal

Digit shortcuts follow visual position in the resolved strip. Because the Files pane is
always rendered last, it always carries the highest assigned digit. With zero providers
the numbering is unchanged (`1 2 3 4`); with the two providers above it becomes:

```
1 STITCHES │ 2 Patches │ 3 Beads │ 4 Plans │ 5 Researchs │ 6 Files
```

This is a deliberate, user-requested reshuffle: Plans moves `5` -> `4`, Research moves
`6` -> `5`, Files moves `4` -> `6`.

## Design

Replace the two independent digit sources with a single numbering pass applied to the
fully assembled strip. All downstream surfaces (Textual bindings, command palette, panel
tab strip, `action_show_artifacts_digit`) already read
`ArtifactsTabDescriptor.digit_shortcut`, so they need no changes and inherit the fix.

### 1. `src/sase/ace/tui/artifact_tabs.py` — own the numbering in one place

Add a module-private digit alphabet and one exported helper:

```python
_ARTIFACTS_DIGIT_KEYS: tuple[str, ...] = tuple(str(digit) for digit in range(1, 10))


def assign_artifacts_digit_shortcuts(
    descriptors: Sequence[ArtifactsTabDescriptor],
) -> tuple[ArtifactsTabDescriptor, ...]:
    """Number Artifacts panes by visual position, Files highest."""
```

Semantics to implement and document in the docstring:

- `descriptors` arrive in visual (left-to-right) order, with the Files pane last.
- The Files descriptor (`id == "files"`) always receives a digit, and always the highest
  one assigned: its 1-based position clamped to the last available digit, i.e.
  `_ARTIFACTS_DIGIT_KEYS[min(len(descriptors), len(_ARTIFACTS_DIGIT_KEYS)) - 1]`.
- Every other descriptor receives its 1-based positional digit as long as that digit is
  strictly lower than the Files digit; any pane beyond that (only reachable with more
  than nine panes) receives `digit_shortcut=None`.
- If no descriptor has `id == "files"` (defensive; not reachable from
  `resolve_artifacts_subtabs()`), fall back to plain positional numbering with `None`
  past the ninth pane.
- `ArtifactsTabDescriptor` is `frozen=True, slots=True`, so build the result with
  `dataclasses.replace(descriptor, digit_shortcut=...)`.
- Import `Sequence` from `collections.abc` (the module already imports `Iterable` and
  `Mapping` from there).

Wire it in:

- `_fixed_descriptor()`: stop passing `digit_shortcut`; leave it `None` and let the
  numbering pass fill it in.
- `_provider_descriptors()`: stop passing `digit_shortcut`. Change the loop to
  `for index, kind in enumerate(sorted(by_kind, key=_natural_label_key)):` and index
  accents with `_PROVIDER_ACCENTS[index % len(_PROVIDER_ACCENTS)]`. This preserves
  today's accent assignment exactly, because the current expression is
  `_PROVIDER_ACCENTS[(offset - 5) % len(_PROVIDER_ACCENTS)]` with `offset` starting at
  `5` — do not let provider accent colors change.
- `resolve_artifacts_subtabs()`: wrap the assembled tuple, i.e.
  `descriptors = assign_artifacts_digit_shortcuts((...))`, before storing it in
  `_ARTIFACTS_TAB_CACHE` and returning it.
- Delete `FIXED_ARTIFACTS_DIGITS` (now dead, and keeping it invites drift): remove the
  dict, its `__all__` entry in `artifact_tabs.py`, and its import plus `__all__` entry
  in `src/sase/ace/tui/widgets/artifacts/types.py`. Nothing else imports it — verify
  with `rg FIXED_ARTIFACTS_DIGITS` before and after.
- Add `assign_artifacts_digit_shortcuts` to `artifact_tabs.__all__`. Do **not** add it
  to the `widgets/artifacts` re-export surface; no widget needs it, and symvision flags
  unused re-exports.

### 2. `src/sase/ace/testing/_startup.py` — keep the fast test strip honest

`_fast_artifacts_subtabs()` hand-builds a deterministic strip and hard-codes
`digit_shortcut="5"` on its `ref:plan` descriptor. Drop that argument and return
`artifact_tabs.assign_artifacts_digit_shortcuts((...))` so the fake strip is numbered by
exactly the production rule (Plans -> `4`, Files -> `5`). This is what makes every
AcePage-driven test and PNG snapshot exercise the new numbering.

### 3. `src/sase/ace/tui/widgets/tab_quickstart.py` — stop hard-coding the jump row

`_build_card()` inserts two artifacts-only rows with hard-coded content:

```python
(("1", "2", "3", "4"), "Jump: Stitches · Patches · Beads · Files.")
(( <cycle keys> ), "Cycle Artifacts: Stitches · Patches · Beads · Files.")
```

The jump row becomes factually wrong the moment Files stops being `4`, and both rows
already omit provider tabs. Build them from the resolved strip instead:

- Inside the `if tab == "artifacts":` block, do a function-local
  `from ..artifact_tabs import resolve_artifacts_subtabs`. It must be function-local:
  `sase.ace.testing._startup` installs the fast strip with
  `patch.object(artifact_tabs, "resolve_artifacts_subtabs", ...)`, which a module-level
  `from ... import` would bypass.
- Jump row keys: each descriptor's `digit_shortcut`, skipping `None`. Jump row text:
  `"Jump: " + " · ".join(labels) + "."` over those same descriptors.
- Cycle row text: `"Cycle Artifacts: " + " · ".join(labels) + "."` over _all_
  descriptors (cycling with `[` / `]` visits panes that lost their digit too).
- If the jump row would have no keys at all (defensive), skip inserting it and keep the
  remaining row insert indices consistent.
- Leave the existing "Files: previous / next version." row untouched.

Row width is handled by `_keycap_width()` and the existing wrapping in
`_append_key_row()`, so a six-pane strip needs no layout work.

### 4. `docs/ace.md` — describe the rule, not a frozen digit

Rewrite the paragraph at `docs/ace.md:97`, which currently claims:

> Within Artifacts, the fixed tab keys are **1 Stitches · 2 Patches · 3 Beads · 4
> Files**. Configured document-provider tabs such as Plans and Research appear between
> Beads and Files; use `[` / `]` to cycle through the complete runtime strip. The fixed
> number keys keep their meaning even when provider tabs are present.

Replace it with the positional rule: number keys follow the visible left-to-right order
of the strip; Stitches, Patches, and Beads are always `1`, `2`, `3`; configured
document-provider tabs such as Plans and Research take the next digits; and Files, which
always renders last, always carries the highest digit — `4` with no provider tabs, `6`
with two. Note that digits stop at `9`, and that Files keeps its digit even then. Keep
the surrounding sentences about `[` / `]` cycling, Artifacts-only scope, and `p`.

Lines 366 and 457 of the same file describe the fixed panes without digits — leave them
alone. Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims
(`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`); none of them document this
numbering, and editing them requires separate explicit user permission.

### 5. Deliberately unchanged

`src/sase/ace/tui/bindings.py:103-106` keeps its static
`Binding("4", "show_artifacts_digit(4)", "Show Files")` fallback. `DEFAULT_BINDINGS` is
only the class-level `AceApp.BINDINGS` seed; `actions/_state_init_late.py`
unconditionally replaces `self._bindings` with `build_app_bindings(...)`, which derives
digits from the live descriptors. The static block therefore describes the zero-provider
strip, which is still exactly `1 2 3 4`. Add a one-line comment above those four
bindings noting that the provider-aware digits are rebuilt at runtime, and leave the
assertion at `tests/test_keymaps_app_bindings.py:220` (`{"1", "2", "3", "4", "0"}`) as
is.

Sub-tab selection persists by pane id through `normalize_artifacts_subtab()`, never by
digit, so no persistence or migration work is needed.

## Test Plan

### New unit coverage for the numbering rule

Add `tests/ace/tui/test_artifact_tab_digits.py` covering
`assign_artifacts_digit_shortcuts` directly (build descriptors inline; no AcePage
needed, so these stay fast):

1. Fixed-only strip (`stitches, patches, beads, files`) -> `1 2 3 4`.
2. Two providers -> `1 2 3 4 5 6` with `files == "6"`; provider tabs get `4` and `5`.
3. General invariants for 0-5 providers: assigned digits are the strictly increasing
   prefix `1..n`, contain no duplicates, and the Files digit equals the maximum assigned
   digit.
4. Overflow: ten or more panes -> Files still gets a digit and it is `"9"`; panes that
   would need a tenth digit get `None`; no digit is assigned twice.
5. Provider accents are unchanged by the refactor: assert
   `_provider_descriptors`-produced accents for a known kind order still match
   `_PROVIDER_ACCENTS` positionally (or assert the first provider keeps `#AF87FF` via
   `ARTIFACTS_ACCENTS["ref:plan"]`).

### Update digit-coupled existing tests

- `tests/ace/tui/test_artifacts_scaffold.py` hard-codes the fast-strip digits in several
  places and must be reworked to derive keys from `view.descriptors` rather than
  literals. Add a local helper such as `_digit_for(view, subtab) -> str` and use it at:
  - the `await page.press("3")` / `await page.press("4")` Files hop around line 157;
  - `test_ctrl_space_dispatches_repeat_agent_from_every_subtab` (line ~173), which zips
    `("1", "2", "3", "4")` against `("stitches", "patches", "beads", "files")`;
  - `test_number_keys_jump_artifacts_without_entering_from_other_tabs` (line ~186),
    including the `provider_five` lookup that assumes the provider owns digit `5` — look
    the provider up by `is_provider` and press its own `digit_shortcut`;
  - the digit-map assertion near line 551, which asserts
    `{"stitches": "1", "patches": "2", "beads": "3", "files": "4"}`. Replace it with
    order-based assertions: the three leading fixed panes are `1`, `2`, `3`;
    `descriptors[-1].id == "files"`; and the Files digit is the highest assigned digit.
  - The `("1", "2", "3", "4", "5", "6", "asterisk")` loops near lines 223 and 232 assert
    that digits do _nothing_ outside the Artifacts tab. They stay as literals.
- `tests/ace/tui/widgets/test_tab_quickstart.py`:
  `test_artifacts_quickstart_advertises_every_subtab` asserts the exact strings
  `"Jump: Stitches · Patches · Beads · Files."` and
  `"Cycle Artifacts: Stitches · Patches · Beads · Files."`. `render_content()` is called
  without AcePage, so it would otherwise hit real provider discovery and be
  host-dependent — monkeypatch `sase.ace.tui.artifact_tabs.resolve_artifacts_subtabs` in
  these tests and assert both shapes: a fixed-only strip (keys `1 2 3 4`, unchanged
  text) and a one-provider strip (keys `1 2 3 4 5`, text
  `"Jump: Stitches · Patches · Beads · Plans · Files."`). Check
  `test_artifacts_quickstart_uses_configured_subtab_keys` in the same file and give it
  the same deterministic strip if it renders the artifacts card.
- `tests/test_keymaps_app_bindings.py` derives its expectations from
  `resolve_artifacts_subtabs()` and should need no edits — confirm it still passes.

### PNG snapshots

Every Artifacts-tab snapshot renders the strip, so the digit glyphs shift (Plans `5` ->
`4`, Files `4` -> `5` under the fast test strip), and the two changespecs-onboarding
snapshots also gain the Plans entry in the quickstart card. All digits are single
characters, so no width or layout change is expected.

1. Run `just test-visual` and expect failures across `artifacts_*`,
   `changespecs_onboarding*`, and any other Artifacts-tab goldens.
2. Inspect at least one actual/expected/diff triple under `.pytest_cache/sase-visual/`
   and confirm the only change is the digit run (and the quickstart card row) — if any
   snapshot shows a color, width, or layout change, stop and re-check the accent
   indexing in `_provider_descriptors()`.
3. Re-run with `--sase-update-visual-snapshots` to accept, then run `just test-visual`
   again clean.

### Gates

- `just install` first (ephemeral workspace).
- `just check` after the source and test edits.
- `just test-visual` (see above) for the goldens.
- `just check-full` before landing, launched through `/sase_monitor`
  (`sase monitor start --command 'just check-full' …` with a `--next` action) — never
  inline.

### Manual verification

Run `sase ace`, open Artifacts in a project that has both the `plan` and `research`
document providers configured, and confirm the strip reads
`1 STITCHES │ 2 Patches │ 3 Beads │ 4 Plans │ 5 Researchs │ 6 Files`. Press `6` and land
on Files; press `4` and land on Plans. Then check the command palette shows
`Show Artifacts: Files` bound to `6`.

## Risks

- **Muscle memory churn.** Anyone with a configured provider tab has learned `4` =
  Files; it becomes `6`. This is the explicitly requested behavior, and it is what makes
  the printed number match the pane position.
- **Provider count above nine.** Only reachable with ten or more panes. The rule keeps
  Files numbered (`9`) and drops digits from the trailing provider tabs, which remain
  reachable via `[` / `]` and the command palette. Covered by unit test 4.
- **Accidental accent shift.** The provider accent index is currently entangled with the
  digit offset. Test 5 and the snapshot inspection step both guard against changing
  provider colors while removing the digit from that loop.
