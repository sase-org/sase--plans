---
tier: tale
title: Crisp top-bar chip text and a stash label
goal:
  "The top-bar stash chip, and every other full-density chip, renders its value in its
  own crisp colors instead of a leaked dim gray, and the stash group reads `stash: ≡ N`."
size: small
proposed_by: bbugyi200.athena.0q9.f2.f0
create_time: 2026-09-23 18:22:28
status: wip
---

# Top bar: undim the chip text, and rename `prompts:` to `stash:`

## Goal

This follows `plan:202609/top_bar_icon_chips.md`, which landed as
`feat(ace): add provider-priority icon chip to top-bar indicator cluster`. The user
reviewed the live result and asked for two changes:

1. **Fix the stash count's text color.** In the user's screenshot the stash chip shows
   `≡ 2` in a washed-out mid-gray (sampled at about `#5f5f5f`) on the pink fill. It
   should be the crisp near-black the chip style asks for.
2. **Rename the group label** from `prompts:` to `stash:`.

## Root cause (verified; it is not the palette)

The stash chip body is `Text(" ≡ N ", style="bold #1a1a1a on #FF87D7")`, which has an
8.0:1 contrast ratio. That is why `tests/ace/tui/test_top_bar_palette.py` passes: it
checks the body on its own. The gray comes from how `TopBarGroup._composed_text()`
(`src/sase/ace/tui/widgets/top_bar_group.py`) adds the label at full density:

```python
text = Text(f"{self.GROUP_LABEL}: ", style="dim")   # base style of the WHOLE Text
text.append_text(self._body)
```

`Text(..., style="dim")` sets the `Text` object's **base** style, and Rich applies a
base style to every character, including text appended later. So every chip body at full
density renders as `bold dim #1a1a1a on #ff87d7`. A quick Rich render shows the output
segment `' ≡ 2 '` with style `bold dim #1a1a1a on #ff87d7`. Terminals and the SVG
exporter draw dim near-black as muddy gray.

This is a cluster-wide bug; the stash chip is just where it shows most:

- In `tests/ace/tui/visual/snapshots/png/top_bar_indicators_full_220x40.png` (full
  density), `CLAUDE off ∞` and `≡ 4` render gray.
- In `top_bar_indicators_compact_120x40.png` (compact density, no label), the same chips
  render crisp black.
- In the user's screenshot, `CODEX +2` samples at the same `#5f5f5f`. The inbox's
  actionable `bold` counts (`?9 #3 f2 ◆132`) are visibly muted compared with the
  un-dimmed `load:`/`model:` values in the status row beneath.

The status row gets this right. `agent_load_indicator.py` appends its label as a dim
**span** (`text.append(_LOAD_LABEL, style="dim")`), and the top-bar docs say the cluster
"speaks the same visual language" as that row.

## Design decisions

- **Fix the cause in the shared base, not the stash chip.** A stash-only `not dim`
  override would fix the reported symptom but leave every other chip gray and
  inconsistent with compact mode. Build the label as a span instead:

  ```python
  text = Text()
  text.append(f"{self.GROUP_LABEL}: ", style="dim")
  text.append_text(self._body)
  ```

  New invariant: **the label is the only thing a group dims. A group's body renders
  exactly the same at full and compact density.** This keeps the "`<type>:` recedes,
  value speaks" hierarchy the status row already uses.

- **Visible side effect, called out on purpose.** The other full-density chips get
  crisper too: `disabled:`, `priority:`, and `overrides:` pills; `procs:`/`monitors:`
  gear chips; the `updates:` badge; and the inbox's actionable counts. They end up
  looking exactly as they already do at compact density, which is how their builders
  were designed. Dims a body asks for itself are kept: `inbox: 0`
  (`Text("0", style="dim")`), the snoozed-only inbox badge, and the dim `+N` suffix. So
  quiet states stay quiet and only the accidental dimming goes away.
- **Stash chip colors stay the same.** Keep orchid pink `#FF87D7` with `bold #1a1a1a`
  text and the `≡` glyph. The user approved the pink and objected only to the count's
  text color. The chip now matches its own compact rendering and every sibling chip:
  near-black on a bright fill.
- **Label `stash`.** It is one lowercase noun like its neighbors (`procs`, `updates`,
  `disabled`, `inbox`), and it names the feature the click opens (the prompt stash
  picker). `stash: ≡ 2 · inbox: …` reads naturally. It is also 2 cells shorter, so full
  density fits slightly sooner.
- **Display-only rename.** Keep the widget id `stashed-prompts-indicator`, the class
  `StashedPromptsIndicator`, the tooltip text, and `CLICK_ACTION`. No config, keymap
  (`src/sase/default_config.yml`), or Rust-core surface is involved; this is
  presentation-only Textual code.

## Implementation

1. `src/sase/ace/tui/widgets/top_bar_group.py`
   - In `TopBarGroup._composed_text()`, build the full-density text as an unstyled
     `Text()`, append the label as a `dim` span, then `append_text(self._body)`. Do not
     change the compact and hidden branches.
   - Update the module and class docstrings to state the invariant: only the label is
     dim, and the body keeps its own styles at both densities.

2. `src/sase/ace/tui/widgets/stashed_prompts_indicator.py`
   - Set `GROUP_LABEL = "stash"`.
   - Update the class docstring from `prompts: ≡ N` to `stash: ≡ N`.

3. Tests
   - `tests/ace/tui/widgets/test_top_bar_group.py`: add a regression test built on the
     existing `_ProbeGroup` that renders `_composed_text()` through a truecolor Rich
     `Console`. Assert:
     - the label segment is dim;
     - no body segment is dim when the body did not ask for dim;
     - the body segments at full density equal the compact-density segments;
     - `_composed_text().style` is empty, so there is no base style;
     - a body that _does_ ask for dim (for example `Text("0", style="dim")`) still
       renders dim at full density.
   - `tests/ace/tui/test_top_bar_palette.py`: add a full-density guard. Wrap each
     `_neighbors()` chip body and `StashedPromptsIndicator._build_content(4)` in a
     labeled probe `TopBarGroup`. Render the composed text and assert that no segment
     carrying a background fill is dim, and that each fill's foreground still meets the
     existing `_MIN_TEXT_CONTRAST`. This closes the gap that let the body-only contrast
     test pass while the rendered chip failed.
   - `tests/ace/tui/test_top_bar_indicators.py`
     - In `test_busy_cluster_renders_all_labels_wide`, replace `"prompts:"` with
       `"stash:"`. Also assert `"stash:  ≡ 4 "` is in the cluster text and `"prompts:"`
       is not.
     - Add a mounted assertion: after `_drive_busy` at 220x40, walk the stash widget's
       rendered strip (`render_line(0)`) and require every segment containing `≡` or the
       count to be non-dim with foreground `#1a1a1a`.
   - `tests/ace/tui/test_top_bar_order.py`: in the comment listing group order, change
     `prompts` to `stash`.
   - `tests/test_stashed_prompts_indicator.py` asserts only body plain text and
     tooltips, so it should not need changes. Rerun it anyway.

4. Docs (`docs/ace.md`)
   - "Top-Bar Indicators" section: in the order list, change `prompts` to `stash`.
     Change "`≡` for prompts" to "`≡` for the stash" and "prompts opens the prompt stash
     picker" to "stash opens the prompt stash picker". Add one sentence stating that
     only the label is dim and that a group's value looks the same in full and compact
     modes.
   - Prompt-stash section: change "A small `prompts: ≡ N` pink-chip top-bar group" to "A
     small `stash: ≡ N` pink-chip top-bar group".
   - Grep `docs/` for any other `prompts:` top-bar mention and update it. At plan time
     there were none outside `docs/ace.md`.

5. Goldens
   - The fix changes the top-bar row of every golden where a chip or a non-zero inbox
     shows at full density, so run the **full** `just fix-tui-screenshots` through
     `/sase_monitor`. It took about 18 minutes last time.
   - Inspect the retained report and every changed golden before finalizing. Expected:
     - Only the top-bar row changes: chip text goes from gray to black, inbox counts get
       brighter, and `prompts:` becomes `stash:`.
     - `top_bar_indicators_full_220x40` and `stashed_prompts_indicator_badge_120x40` are
       updated.
     - `top_bar_indicators_compact_120x40` and other compact-density goldens are
       unchanged, because compact never had the label and never leaked dim. If a compact
       golden changes, check whether the 2-cell-shorter label flipped its density before
       accepting it.
     - Goldens that show only `inbox: 0` are unchanged.
     - No creations or removals.

## Verification

- Focused pytest: `tests/ace/tui/widgets/test_top_bar_group.py`,
  `tests/ace/tui/test_top_bar_palette.py`, `tests/ace/tui/test_top_bar_indicators.py`,
  `tests/ace/tui/test_top_bar_order.py`, `tests/test_stashed_prompts_indicator.py`, and
  `tests/ace/tui/visual/test_ace_png_snapshots_top_bar_indicators.py`.
- Full golden refresh plus report inspection (step 5).
- Live check: stash a prompt or two, then capture `sase screenshot` at a width that
  gives full density. Pixel-sample the stash chip: the `≡`/count glyph pixels should be
  near `#1a1a1a` on the pink fill, not about `#5f5f5f`. The `disabled:` pill subject
  should be black too.
- Run `just check` per the lint-and-test memory.

## Out of scope

- Changing the stash hue, glyph, or tooltip, or the chip style of any other group.
- Renaming the widget id or class, or the prompt-stash feature itself.
