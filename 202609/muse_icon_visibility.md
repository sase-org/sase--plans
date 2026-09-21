---
tier: tale
title: Make the Muse provider icon easy to spot at a glance
goal:
  The Muse/Meta provider badge and provider-colored text are as easy to spot at a glance
  as every other provider's, on every surface that shows them.
size: small
proposed_by: bbugyi200.apollo.0s.f0.f0.w2.w0
create_time: 2026-09-20 22:26:30
status: wip
---

# Plan: Make the Muse provider icon easy to spot at a glance

## Problem

The Muse Code (Meta) provider is marked by the `♾️` emoji on every surface that shows a
provider badge. That badge is hard to see, for two reasons:

1. **The glyph itself.** `♾` (U+267E) is shown as text by default and only turns into
   an emoji because of the trailing variation selector (U+FE0F). Many terminal and font
   setups ignore that selector. They draw it as a thin, one-color text glyph in the
   current foreground color, which is often a dim row color. Where it does draw as an
   emoji, most emoji fonts show a thin gray or blue-gray outline loop. It has almost no
   filled area, so it disappears next to the solid, colorful badges of the other
   providers (🎭 🤖 🐼 🐙 🪐 🧪).
2. **The brand blue.** Muse's primary color `#0064E0` is a dark, saturated blue. Against
   the TUI's dark background its contrast is roughly 3.3:1, and it is darker still when
   rendered `dim`. So the `MUSE` name, the `(`/`)` delimiters, and the group-header rule
   are also the hardest provider text to read. Every other provider primary (`#FF5F00`,
   `#10A37F`, `#00C8D7`, `#D75FFF`, `#FFD75F`, `#6E5DE7`) is noticeably brighter.

## Approach

### 1. Replace the emoji badge (the icon)

In `src/sase/integrations/provider_badges.py`, change the `muse` and `meta` entries of
`_PROVIDER_EMOJI_BADGES` from `♾️` to **`🦋`** (U+1F98B BUTTERFLY).

Why this one:

- It has `Emoji_Presentation=Yes`. It is always drawn as a full-color, 2-cell-wide
  emoji, with no variation selector that a terminal can ignore. That also removes the
  cell-width ambiguity VS16 sequences cause in Textual/Rich layout.
- It is a large, filled, saturated blue shape. It keeps Meta's blue identity but reads
  clearly on dark and light backgrounds.
- Its outline looks nothing like any other provider badge, and the symbol isn't already
  used anywhere in `src/` (checked with `git grep`). Muse, a source of inspiration, fits
  the butterfly, a traditional symbol of Psyche and of transformation.

This one table drives every badge surface, so no call site needs to change:

- the agent-list row prefix (`ace/tui/widgets/_agent_list_render_agent_prefix.py`)
- the provider-usage header indicator (`ace/tui/widgets/_provider_usage_indicator.py`)
- the integrations agent-list entry wire field `provider_badge`
  (`integrations/_agent_list_entry_builder.py`), which external frontends consume

### 2. Lift the Muse / Meta text palette (the color)

Brighten the blue family so provider-colored text reads as well as the other providers'
text, while staying recognizably Meta blue:

- `src/sase/llm_provider/muse.py` `llm_cli_status_color()`: `#0064E0` → `#3D9BFF`. This
  primary is fed through `provider_cli_status_color_map()` and overrides `name_style` in
  `provider_styles._with_primary`. It also colors the provider column of
  `sase agents list` (`agents/cli_list.py`).
- `src/sase/llm_provider/_registry_catalog.py` `_PROVIDER_FAMILY_COLORS["meta"]`:
  `#0064E0` → `#3D9BFF`, so the `meta` alias matches.
- `src/sase/ace/tui/provider_styles.py` `_PROVIDER_FALLBACK_STYLES` entries `"muse"` and
  `"meta"` (keep the two identical):
  - `name_style`: `bold #3D9BFF`
  - `delimiter_style`: `#1877F2` (Facebook blue, a mid shade that is still readable, in
    place of the near-invisible `#0064E0`)
  - `model_style`: `#8CC4FF`
  - `secondary_style`: `#1877F2`
  - `dim_style`: `dim #8CC4FF`

  This mirrors the other built-in palettes: a bright primary name, a darker shade for
  delimiters and the header rule, a lifted model tint, and a dim tint from the model
  hue. Before settling, check each new foreground on the default ACE dark background:
  `name_style` and `model_style` should be ≥ 4.5:1 and `delimiter_style` /
  `secondary_style` ≥ 3:1. If a value misses, nudge it lighter within the same hue.

### 3. Update tests and docs

- `tests/ace/tui/test_usage_header.py`: the `"♾️" in rendered` assertion becomes `"🦋"`.
- `tests/llm_provider/test_muse_provider_core.py`: the
  `llm_cli_status_color() == "#0064E0"` assertion becomes `"#3D9BFF"`.
- Add a focused assertion like the other providers' badge tests (see
  `tests/llm_provider/test_grok_provider_core.py`, which asserts
  `provider_emoji_badge("grok")`). In `tests/llm_provider/test_muse_provider_core.py`,
  assert `provider_emoji_badge("muse") == "🦋"` and
  `provider_emoji_badge("meta") == "🦋"`. Also assert the badge contains no U+FE0F. That
  guards against slipping back to a text-default glyph that needs a variation selector.
- `docs/ace.md` provider badge table (around line 2443): replace `♾️` with `🦋` for
  "Muse Code (Meta)".
- `docs/blog/posts/structured-agentic-software-engineering.md` (around line 271): leave
  the published post's wording alone. It describes the UI at publication time. Only
  update it if the implementer finds the repo treats blog posts as living docs (for
  example, other posts were updated for later UI changes).
- Leave `demos/out/sase_ace_multi_model_fanout.gif` and
  `docs/images/blog/sase_ace_multi_model_fanout_still.png` alone. They are recorded
  media, not goldens.

### 4. Visual snapshots

Read the `tui` reference memory (`sase memory read tui.md -r "..."`) before touching TUI
styling. Then run the TUI visual snapshot suite (`tests/ace/tui/visual/`, including the
model-completion PNG snapshots whose fixtures include Muse rows). Regenerate only the
goldens that change because of the Muse color or badge, using the documented update
workflow, and check the regenerated images show the new badge and brighter blue.

## Out of scope

- No Rust core change. `sase-core` has no provider badge or Muse color (checked), so
  this is presentation-only and stays in this repo.
- Other providers' badges and palettes.
- `src/sase/default_config.yml`: provider colors and badges are not user config keys
  there, so nothing to update.

## Verification

- `just check` passes (per the `lint_and_test` reference memory; read it before
  finishing).
- Manually confirm in a live ACE session (or a screenshot via the TUI screenshot
  tooling) that a Muse agent row shows 🦋 at the same width as the other badges, with no
  column misalignment, and that `MUSE(model)` suffixes and the Muse model-picker group
  header are clearly readable.
