---
tier: tale
title: Make the Grok provider badge easy to see (🛰️ → 🚀)
goal: The Grok/xAI provider badge renders as a bold, full-width 🚀 everywhere provider
  badges appear, with docs and tests updated.
size: small
proposed_by: bbugyi200.athena.0ou
status: done
---

# Plan: Make the Grok provider badge easy to see (🛰️ → 🚀)

## Problem

The Grok (xAI) provider badge is `🛰️` (U+1F6F0 SATELLITE + U+FE0F). It is hard to see at
a glance for two reasons:

1. **Rendering:** U+1F6F0 is _not_ a default-emoji-presentation code point
   (`Emoji_Presentation=No`), so it only becomes a color emoji because of the trailing
   VS16 selector. Many terminals / fonts (and Textual's width calculation) treat it as a
   narrow, 1-cell text glyph, so it renders small, monochrome, or clipped next to the
   2-cell badges of the other providers (🎭 🤖 🐼 🐙 🦋 🪐).
2. **Visual weight:** even when it renders in color, the satellite is a thin, dark-grey
   diagonal shape with little contrast on a dark TUI background.

## Decision: use 🚀 (U+1F680 ROCKET)

- **Highly visible:** bright red/white, bold silhouette, high contrast on dark themes.
- **Reliable rendering:** U+1F680 has `Emoji_Presentation=Yes` — a single code point, no
  variation selector, consistently 2 cells wide everywhere, matching the other badges.
- **Intuitive:** xAI/Grok is strongly associated with Musk/SpaceX rockets; it also keeps
  the "space" theme of the old satellite so existing users map it immediately.
- **Distinct from other provider badges:** no other provider uses it; the closest space
  glyph (🪐 agy) is a different shape and color.

Alternatives considered and rejected:

- `⚡` — already the auto-approve row icon (`⚡`, `⚡T`, `⚡E`) in the same agent-row
  prefix; would be ambiguous.
- `🧠` — fits the word "grok" (to understand deeply) but has no xAI brand association.
- `🌌` / `🌠` — dark, low-contrast, small detail; same problem as today.
- `✖️` / `❌` (the X logo) — reads as error/reject; ✖️ has the same VS16 problem.

Known overlap (acceptable): `🚀` is also used as an _action_ icon in notification/gate
surfaces (LaunchApproval notification icon, tale "Launch coder agent" plan-gate option,
task/flag triage launch options). Those appear in notification modals and gate option
lists, not in the agent-row provider-badge slot or the provider usage indicator, so
there is no same-slot ambiguity. Do not change those action icons.

## Changes

1. `src/sase/integrations/provider_badges.py`: change both the `"grok"` and `"xai"`
   entries in `_PROVIDER_EMOJI_BADGES` from `"🛰️"` to `"🚀"`. This is the single source
   of truth; `sase.ace.tui.provider_styles.provider_emoji_badge`, the agent-list prefix
   renderer, the agent-list entry builder, and the provider usage indicator all read it.
2. `docs/ace.md` (provider badge table, ~line 2437): change the Grok Build (xAI) row
   badge to `🚀`.
3. Update test expectations that hard-code the old glyph (replace `🛰️` with `🚀`):
   - `tests/llm_provider/test_grok_provider_core.py` (both `grok` and `xai` asserts)
   - `tests/ace/tui/widgets/test_agent_list_provider_emoji_badges.py`
   - `tests/test_agents_tab_graph_isolation.py`
   - `tests/test_provider_usage_indicator_widget.py`
   - `tests/test_provider_usage_indicator_presentation.py`
   - `tests/test_provider_usage_indicator_presentation_style.py`
   - `tests/test_provider_usage_indicator_presentation_layout.py`

   Because `🛰️` is two code points and `🚀` is one, check any assertion that depends on
   string length, cell width, or column alignment (e.g. usage-indicator layout/overflow
   tests and visual snapshots) and update expected values rather than loosening them.

4. Re-grep the whole repo (`grep -rn "🛰" --exclude-dir=.git .`) to confirm no remaining
   references, including visual snapshot baselines under `tests/`. If any TUI visual
   snapshot baselines include the Grok badge, regenerate them per the `tui` reference
   memory (read it with `/sase_memory_read` first).
5. Check the linked repos that might mirror the provider badge map (`sase-telegram`,
   `sase-nvim`, `sase-github`) and the Rust core (`sase-core`) for `🛰` — open them only
   via the `/sase_repo` skill. If any contain the Grok badge, update them to `🚀` too;
   if none do, note that in the final summary.

No config (`default_config.yml`) or keymap changes are needed: the badge is not
user-configurable.

## Verification

- Read the `lint_and_test` reference memory and run the prescribed checks
  (`just check`).
- Run the directly affected tests, e.g.
  `pytest tests/llm_provider/test_grok_provider_core.py tests/ace/tui/widgets/test_agent_list_provider_emoji_badges.py tests/test_agents_tab_graph_isolation.py tests/test_provider_usage_indicator_*.py`.
- Optionally take a live TUI screenshot (per the `tui` memory) of an agents tab
  containing a Grok agent to confirm the 🚀 badge renders full-width and aligned with
  the others.
