---
tier: tale
title: Apply configured agent-tribe colors throughout the ACE TUI
goal: Every structured tribe identity in the ACE TUI uses its configured color consistently
  without changing semantic chrome or regressing render-cache performance.
---

- **AGENTS:**
  - [bbugyi200.athena.lg](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.lg/README.md)
  - [bbugyi200.athena.lg--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.lg.md#member-code)
- **COMMITS:**
  - [0bff029](https://github.com/sase-org/sase/commit/0bff029f30a56c04b8cf0488e68051355e25c49b) — feat(tui): color tribe identities consistently

# Apply configured agent-tribe colors throughout the ACE TUI

## Summary

Extend the existing `ace.tribes.<name>.color` presentation setting from tribe panel border titles to every structured
tribe identity rendered by the ACE TUI. The configured color must cover the tribe's existing icon and `@tribe` name
without recoloring surrounding selection markers, fold controls, counts, statuses, headings, or explanatory copy.

This is a tale because the work is one coupled presentation refactor: a single shared identity-color contract must be
introduced first, then all renderers and their cache keys must adopt it consistently. Splitting those callers across
independent phases would create avoidable overlap in the shared formatter, agent-row rendering, and visual goldens. No
Rust/core domain behavior or wire format changes are needed.

## Current behavior and evidence

- `src/sase/ace/tui/models/tribe_display.py` already loads and sanitizes each tribe's icon and `#RRGGBB` color once per
  merged-config token.
- Agent panel border titles resolve `tribe_display_for(key)` and correctly color their icon and `@tribe` name. Empty or
  invalid colors retain the gold `#FFD75F` fallback.
- The selected whole-tribe metadata document derives the configured icon for its `Name:` value, but
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py` paints the entire identity with its hard-coded
  `TRIBE_IDENTITY_COLOR`. This produces the reported mismatch: the `▲ @epic` panel title is lavender while
  `Name: ▲ @epic` remains gold.
- Other structured tribe identities bypass `tribe_display_for` as well: standalone/clan annotations in agent rows, the
  `Tribes:` field in clan metadata, wait/fork completion rows, the current-tribe and cleanup modals, cleanup
  filters/rows, and the wait modal's tribe annotation.
- The agent list is an especially sensitive path. Its rendered `Text` is cached by `AgentRenderCache`; adding color
  lookup without including the resolved presentation in `agent_render_key` can retain stale colors after a config token
  changes. Repeated config work inside each row render would also violate the established TUI hot-path constraints.

## Presentation contract

1. Treat the validated configured RGB value as the foreground color for a tribe identity wherever the renderer has a
   structured bare tribe or panel key. Preserve `#FFD75F` as the single fallback for empty, unknown, or invalid
   configuration.
2. Apply the identity color only to an existing configured tribe icon and the tribe-name token (`@tribe`, or the tribe
   token in a compact column). Selection/fold markers, section headings, kind/count badges, status glyphs, roster member
   names, and prose keep their current semantic styles.
3. Preserve the current text anatomy. Do not introduce tribe icons on surfaces that currently show only a name, and do
   not remove icons from panel/detail identities that already show them.
4. Do not scan or recolor arbitrary `@...` substrings in prompts, agent names, logs, persisted saved-group titles,
   notifications, or free-form confirmation text. Those tokens are not reliably tribes. Convert only call sites that
   already carry a semantic tribe value.
5. Keep all render paths pure and bounded: use the existing config-token cache, resolve presentation once per distinct
   identity/build context where rows are involved, and perform no filesystem access, parsing, or new asynchronous work
   during painting.

## Implementation

### 1. Centralize the tribe identity color/style contract

- In `src/sase/ace/tui/models/tribe_display.py`, expose a small public presentation API around the existing sanitized
  `_TribeDisplay` cache: centralize the gold fallback, resolve an effective identity color for both named tribes and the
  reserved default panel, and provide a safe way for Rich renderers to compose that foreground with their existing
  bold/dim emphasis.
- Keep icon sanitization, RGB validation, `None`-to-`default` mapping, and `initially_expanded` behavior unchanged.
- Reuse this contract from `src/sase/ace/tui/actions/agents/_display_panel_titles.py` so panel titles no longer own a
  second fallback-color definition. Preserve the existing rule that merged `All agents` titles ignore tribe presentation
  settings.
- Add focused model tests in `tests/ace/tui/models/test_tribe_display.py` for configured, unknown, explicit-empty, and
  reserved-default identity colors, plus bold/dim style composition. Retain the hostile-color and per-config-token
  memoization coverage.

### 2. Fix primary Agents-tab identity renderers

- In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`, render only the `Name:` identity
  (`snapshot.panel_key`, including its existing icon) with the resolved custom color. Keep the `TRIBE` title, foldable
  section headings, member/unit names, and other operational accents on their current palette.
- In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`, resolve each structured entry in
  `agent.clan_tribes` independently so a multi-tribe clan's `Tribes:` field can show different configured colors on the
  same line.
- In `src/sase/ace/tui/widgets/_agent_list_render_agent.py`, use each effective tribe's color for:
  - merged-panel standalone-agent `@tribe` annotations;
  - unsuppressed clan-tribe annotations; and
  - a clan's merged-panel tribe annotation when it is not already represented. Preserve split-panel
    suppression/deduplication and all agent/clan/family name colors.
- Thread a compact resolved tribe-color mapping or equivalent presentation fingerprint through
  `src/sase/ace/tui/widgets/_agent_list_build.py`, `src/sase/ace/tui/widgets/_agent_list_render_agent.py`, and
  `src/sase/ace/tui/widgets/_agent_list_render_cache.py`. Resolve it once per distinct list-build context, reuse it for
  patch renders, and include the relevant value in `agent_render_key` so a config change cannot hit a byte-nonidentical
  cached row. Do not call config loaders once per span or add a second list rebuild.
- Extend the focused renderer tests in `tests/ace/tui/widgets/test_agent_display_tribe.py`,
  `tests/ace/tui/widgets/test_agent_display_clan.py`, `tests/ace/tui/widgets/test_agent_render_cache.py`, and
  `tests/ace/tui/widgets/test_agent_render_cache_patching.py` to assert Rich spans, multi-tribe independent colors,
  fallback behavior, split-panel suppression, and cache invalidation/key separation when presentation colors differ.

### 3. Apply the contract to secondary structured TUI surfaces

- Update `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows.py` so tribe wait/fork completion rows color their
  tribe glyph/name with the configured identity color while the `tribe · <counts>` badge and aggregate-status glyph
  retain their existing roles.
- Update the structured tribe values in:
  - `src/sase/ace/tui/modals/agent_tribe_modal.py` (`Current: @tribe`);
  - `src/sase/ace/tui/modals/agent_cleanup_tribe_modal.py` (enabled, disabled, and marked selector rows);
  - `src/sase/ace/tui/modals/agent_cleanup_custom_modal.py` (active tribe filter and each candidate's effective tribe);
    and
  - `src/sase/ace/tui/modals/wait_modal.py` (the tribe portion of the compact role/tribe column).
- Compose state emphasis with the identity foreground rather than replacing it: disabled identities remain dim, active
  identities remain bold, selection markers retain their current color, and truncation/alignment remain unchanged.
- Add or extend focused tests in `tests/ace/tui/widgets/test_xprompt_arg_value_completion.py`,
  `tests/ace/tui/test_agent_cleanup_modal.py`, `tests/ace/tui/test_agent_tribe_modal.py`, and
  `tests/ace/tui/test_wait_modal.py` to verify the custom and fallback Rich spans without weakening navigation,
  selection, filtering, or insertion assertions.

### 4. Align documentation and visual coverage

- Update the `ace.tribes` descriptions in `src/sase/default_config.yml`, `src/sase/config/sase.schema.json`,
  `docs/configuration.md`, and `docs/ace.md` to distinguish TUI-wide identity color/icon behavior from the panel-only
  `initially_expanded` setting. State explicitly that colors affect structured tribe identities only and do not recolor
  semantic chrome.
- Strengthen `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py` so the configured-color scenario enters
  the `epic` tribe detail and verifies both the panel identity and `Name: ▲ @epic` use `#AF87FF`.
- Exercise the existing completion and agent-modal visual fixtures that contain tribe identities, add a narrowly
  targeted visual state only if existing fixtures cannot expose a configured non-fallback color, and update only the PNG
  goldens whose identity spans intentionally change. Inspect actual, expected, and diff artifacts before accepting
  snapshots.

## Validation

1. Before running checks in the implementation workspace, run `just install` as required for an ephemeral SASE checkout.
2. Run focused unit tests for the shared display model, panel titles, tribe and clan detail builders, agent row/cache
   behavior, completion rows, cleanup modals, tribe modal, and wait modal.
3. Run the focused ACE PNG snapshot tests for tribe panels, prompt target completion, and affected agent modals. Review
   generated diffs and accept only the identity-color changes.
4. Re-sweep `src/sase/ace/tui` for tribe renderers that still apply fixed gold, cyan, or magenta directly to a semantic
   tribe value. Fixed gold remains valid for structural `TRIBE`/section headings and as the centralized identity
   fallback.
5. Run `just test-visual` to catch secondary snapshot impact, then run the repository-required `just check`.

## Acceptance criteria

- With bundled defaults, every structured `epic`, `research`, `chop`, and `default` tribe identity rendered by the
  covered TUI surfaces uses its configured foreground; `pinned`, `review`, unknown tribes, and explicit empty colors use
  the gold fallback.
- The reported metadata case shows `Name: ▲ @epic` in the same lavender identity color as the panel title.
- Multi-tribe clan metadata and row annotations resolve each name independently.
- Disabled/selected/fold/status/count chrome retains its semantic styling, and plain text, ordering, alignment,
  suppression, and completion insertion values do not change.
- Agent-row render caches cannot serve a row created with an older resolved tribe color, and no config/file work is
  added per row span or per keypress.
- Focused unit tests, reviewed visual snapshots, `just test-visual`, and `just check` all pass.
