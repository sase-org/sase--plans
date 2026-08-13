---
tier: tale
title: Drop the tribe description "not set" hint from the ACE metadata panel
goal:
  A selected tribe panel with no configured description renders no description row at
  all, instead of a permanent "not set · add ace.tribes.<name>.description" nag.
size: small
proposed_by: bbugyi200.athena.z2
create_time: 2026-08-13 07:31:54
status: wip
---

# Drop the "description not set" hint from the selected tribe panel header

## Problem

When a tribe panel is selected in the ACE Agents tab, the `TRIBE` metadata header always
ends with an unlabeled description row. When the tribe has no configured
`ace.tribes.<name>.description`, that row renders a nag instead:

```
not set · add ace.tribes.monitor-smoke.description
```

This fires for every ad-hoc tribe an xprompt assigns with `%tribe:` — for example the
`monitor-smoke` tribe in the reported screenshot — even though such tribes are never
required to have an `ace.tribes` entry at all. The result is a permanent, unactionable
warning occupying two of the metadata panel's most valuable rows.

The user's ask: when a tribe has no description set, show nothing there.

## Root cause

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_header.py`

`append_tribe_header()` ends by calling `_append_description()` (line 69).
`_append_description()` unconditionally appends a blank separator line, then branches:

- `snapshot.description` non-empty → wrap it at `PROMPT_PANEL_LINE_CELL_LIMIT` (80
  cells) and emit it in `_DESCRIPTION_STYLE`.
- otherwise → emit the `not set · add <config key>` hint, in either a one-line or a
  two-line form depending on whether `"not set · add " + config_key` fits in 80 cells.

There is no third path. The empty-description case has no "render nothing" branch.

## Decision

**Remove the hint entirely.** When `snapshot.description` is empty,
`_append_description` renders nothing at all — no hint, and no separator blank line
either.

Rationale for full removal rather than a narrower "only hint for tribes that appear in
`ace.tribes` but lack a description" variant:

1. A configured tribe missing a description is _already_ surfaced twice, and far more
   loudly than a grey line in a metadata pane: `sase doctor -C config.tribes`
   (`src/sase/doctor/checks_config_tribes.py`) reports it as a `WARN` naming the exact
   key, and the ACE Config Center refuses to write _any_ config change while the problem
   exists (documented at `docs/configuration.md:834-837`). The panel hint adds no
   information those two surfaces do not already carry.
2. The narrower variant would need a new "is this tribe configured at all?" predicate in
   `src/sase/ace/tui/models/tribe_display.py`. Today `tribe_display_for()` returns
   `DEFAULT_TRIBE_DISPLAY` for an unconfigured tribe, which is indistinguishable from a
   configured tribe whose `description` is blank. That is real new API surface, new
   caching behavior, and new tests — to keep a nag whose only remaining audience is a
   state the Config Center already hard-blocks.
3. The reported case (`monitor-smoke`) is an unconfigured ad-hoc tribe, and the user's
   phrasing — "when an agent tribe panel doesn't have a description set" — is general,
   not scoped to unconfigured tribes.

If the user would rather keep the hint for configured-but-blank tribes, say so at the
approval gate; option 2 above is the change that would be made instead.

## Expected rendering after the change

With a description (unchanged):

```
Runtime: 1h
Fold: 1/4
<blank>
Epic phase-worker clans from sase bead work, one member per phase of an
approved plan.
<blank>
──────────────────────────────────────────────────
```

Without a description (new):

```
Runtime: 1h
Fold: 1/4
<blank>
──────────────────────────────────────────────────
```

Dropping the separator blank line along with the hint matters. Every following section
(`append_attention`, `append_member_roster`, …) opens through
`append_major_section_divider()`
(`src/sase/ace/tui/widgets/prompt_panel/_helpers.py:104`), which itself emits `\n` +
rule + `\n`. Keeping the separator while dropping the hint would leave two blank rows
before the divider — a visible ragged gap that no other section boundary has. With both
dropped, the spacing matches every other section boundary in the document.

In `cheap=True` mode (`build_tribe_detail_text` returns right after the header), the
document simply ends at `Fold: N/4\n`. That is correct: no trailing blank row.

## Implementation

### 1. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_header.py`

Rewrite `_append_description()` to return early when there is no description:

```python
def _append_description(text: Text, snapshot: AgentTribeSummarySnapshot) -> None:
    if not snapshot.description:
        return
    text.append("\n")
    for line in wrap_text_by_cells(snapshot.description, PROMPT_PANEL_LINE_CELL_LIMIT):
        text.append(f"{line}\n", style=_DESCRIPTION_STYLE)
```

Then remove everything the hint branch owned, so Symvision's unused-symbol gate stays
green (see `sase memory read symvision.md` if a lint failure needs interpreting):

- Delete the module constants `_DESCRIPTION_MISSING_STYLE` (line 27) and
  `_DESCRIPTION_CONFIG_KEY_STYLE` (line 28). Keep `_DESCRIPTION_STYLE`.
- Delete the now-unused import `from rich.cells import cell_len` (line 5).
- Delete `tribe_config_key` from the `...models.tribe_display` import (line 15), keeping
  `tribe_identity_style`.

Do **not** touch `tribe_config_key` itself in
`src/sase/ace/tui/models/tribe_display.py`. It is still used there (lines 115, 130), is
exported in `__all__`, and is covered by
`tests/ace/tui/models/test_tribe_display.py:110`.

Nothing else in the repo references the removed styles or the hint text — verify with
`rg 'not set · add|_DESCRIPTION_MISSING_STYLE|_DESCRIPTION_CONFIG_KEY_STYLE'` before and
after. (`src/sase/ace/tui/modals/models_panel_rendering.py` defines its own unrelated
`_DESCRIPTION_MISSING_STYLE`; leave it alone.)

### 2. `tests/ace/tui/widgets/test_agent_display_tribe.py`

Three tests assert the hint and must be replaced, not merely deleted — the new behavior
needs equivalent coverage:

- `test_tribe_missing_description_hint_names_the_config_key` (line 128) → rename to
  something like `test_tribe_omits_the_description_row_when_none_is_configured`. Keep
  the same monkeypatched config (`{"ace": {"tribes": {"epic": {"icon": "▲"}}}}`, its own
  `current_config_token`, `cache_clear()`), then assert:
  - `"not set" not in detail.plain`
  - `"ace.tribes." not in detail.plain`
  - the header runs straight from the fold line into the next section with exactly one
    blank row — e.g. `"Fold: 1/4\n\n" + "─" * 50 in detail.plain`, or assert on
    `splitlines()` that the line after `Fold: ` is empty and the one after that is the
    divider. Read the actual divider width from `_helpers` rather than hard-coding it if
    that is how neighboring tests do it
    (`test_tribe_description_renders_below_the_header_fields` uses `"─" * 50`).
- `test_tribe_description_row_renders_for_unconfigured_tribes` (line 160) → invert to
  assert an ad-hoc tribe with no `ace.tribes` entry renders no hint. This is the exact
  regression from the screenshot; keep it and rename it accordingly (e.g.
  `test_tribe_ad_hoc_panel_renders_no_description_hint`).
- `test_tribe_missing_description_hint_maps_none_panel_to_default` (line 180) → the
  `None → "default"` mapping no longer has a rendering surface, so delete this test. The
  mapping itself remains covered by
  `tests/ace/tui/models/test_tribe_display.py:110-112`. Do not leave a test whose only
  assertion is that some string is absent from a document that no longer has any
  description machinery for `None` panels.

Leave the positive-path tests untouched and confirm they still pass unchanged:
`test_tribe_description_renders_below_the_header_fields`,
`test_tribe_long_description_wraps_with_a_hanging_indent`,
`test_tribe_description_renders_in_cheap_mode`,
`test_tribe_description_with_markup_characters_renders_literally`, and the
`pulse.startswith(...)` block in
`test_tribe_levels_have_distinct_glance_triage_inspect_and_forensics_jobs` (line 256) —
that snapshot uses `@epic`, which has a bundled description, so it must not change.

### 3. `docs/ace.md` (lines 1359-1366)

Rewrite the paragraph. It currently promises the row "always" ends the header and
documents the hint. New text should say the header ends with an unlabeled description
row **only when** the tribe has a configured `description`, keeping the existing wording
about the blank-line separation, the fixed 80-cell measure, and the absence of a hanging
indent; and state that a tribe with no configured description — including ad-hoc tribes
with no `ace.tribes` entry — renders no row at all. Point readers at
`sase doctor -C config.tribes` for finding configured tribes that are missing one.

### 4. `docs/configuration.md` (line 817)

The `description` row of the `ace.tribes` table says the value is "Shown as an unlabeled
row beneath the header fields … when that tribe's Agents-tab panel is selected." That
stays true for a set description, so the edit is optional — but if the wording is
adjusted, keep the `_required_` default and lines 834-837 (the doctor / Config Center
hard-block paragraph) intact. Those describe the enforcement that this change is
deliberately leaning on.

## Verification

1. `just install` first — workspace virtualenvs go stale.
2. `just check` (whole-repo lint gates + diff-scoped tests). Expect the lint gates to
   catch any leftover unused import or constant.
3. Targeted:
   `pytest tests/ace/tui/widgets/test_agent_display_tribe.py tests/ace/tui/models/test_tribe_display.py -q`.
4. `just test-visual` — the PNG snapshot suite. The tribe-panel goldens
   (`tests/ace/tui/visual/snapshots/png/agents_tribe_panel_*.png`,
   `agents_panel_fold_selection_120x40.png`, `agents_collapsed_panel_120x40.png`) all
   select `@default`, `@epic`, or `@chop`, every one of which carries a bundled
   description in `src/sase/default_config.yml:137-156`, so **no golden is expected to
   change**. If one does, inspect `.pytest_cache/sase-visual/` and confirm the only diff
   is the removed hint rows before accepting anything with
   `--sase-update-visual-snapshots`. A golden diff anywhere else means something
   unintended moved — investigate rather than accept.
5. `just check-full` before landing, launched through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …` with a `--next` action). Never
   run it inline.
6. Manual confirmation, if convenient: open `sase ace`, select a tribe panel for an
   ad-hoc tribe with no `ace.tribes` entry, and confirm the `TRIBE` header ends at
   `Fold:` with no `not set` row; then select `@epic` and confirm its description still
   renders.

## Out of scope

- The `config.tribes` doctor check and its `WARN` severity — unchanged, and load-bearing
  for this decision.
- The Config Center's refusal to write while a configured tribe lacks a description —
  unchanged.
- `ace.tribes.<name>.description` remaining `_required_` for configured tribes —
  unchanged. This plan changes only where the _absence_ of a description is displayed.
