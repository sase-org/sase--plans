---
tier: tale
title: Per-tribe Agents-tab panel icons and initial-expansion config
goal: 'ACE users can configure, per agent tribe, an icon rendered before the tribe
  name in the Agents-tab panel header and whether that panel starts expanded (default
  true). Explicit user expand/collapse actions are persisted as durable intent that
  always outranks the configured initial state, and the default config ships tasteful
  icons for the default, epic, research, chop, pinned, and review tribes with chop
  collapsed by default.

  '
create_time: 2026-07-19 18:43:04
status: done
prompt: 202607/prompts/tribe_panel_display_config.md
---

# Plan: Per-tribe Agents-tab panel icons and initial-expansion config

## Product context

The Agents tab of `sase ace` splits agents into tribe panels whose headers show `@<tribe>` (or `(no tribe)` for the
untagged panel, which will eventually be renamed `@default`). Today every panel looks the same at a glance and every
panel starts expanded unless the user previously collapsed it. This feature adds two per-tribe knobs:

1. **`icon`** — a short glyph rendered immediately before the tribe name in the panel header, making tribes
   distinguishable at a glance.
2. **`initially_expanded`** — whether the panel is expanded the first time it appears (default `true`). Once the user
   toggles a panel, their choice is remembered and always wins over this configured initial state.

The bundled default config ships icons for Bryan's high-traffic tribes and collapses `@chop` initially.

## Design overview

### Config shape

New `ace.tribes` section: a map keyed by bare tribe name (no `@`; tribe grammar is `^[A-Za-z0-9_.-]+$`, see
`TRIBE_NAME_RE` in `src/sase/core/agent_tribe.py:20`). Each entry is a closed object with two optional fields:

```yaml
ace:
  # Per-tribe Agents-tab panel display. Keys are bare tribe names (no "@").
  # The special "default" entry styles the untagged panel and will carry over
  # unchanged when that panel is renamed to @default.
  #   icon: short glyph shown before the tribe name in the panel header
  #         ("" disables an inherited default icon)
  #   initially_expanded: whether the panel starts expanded the first time it
  #         appears (default true); explicit user folds always win afterwards
  tribes:
    default:
      icon: "🏠"
    epic:
      icon: "🗻"
    research:
      icon: "🔬"
    chop:
      icon: "🪓"
      initially_expanded: false
    pinned:
      icon: "📌"
    review:
      icon: "🔍"
```

Rationale for the defaults:

- All six glyphs are single-codepoint emoji with **no VS16 variant selector** (the repo's `✏️` precedent shows VS16
  glyphs invite width trouble), and each has a distinct silhouette/color: 🏠 home for the default bucket, 🗻 a mountain
  for epic-scale work, 🔬 for research, 🪓 for chop (an axe — a nod to the axe subsystem), plus the built-in system
  tribes `pinned` (📌) and `review` (🔍) from `src/sase/ace/agent_tribes.py:32-33` so the whole set feels finished. If
  review flags pinned/review as scope creep they are trivially droppable.
- Deep-merge semantics mean users can add their own tribes or override a single field (e.g. `icon: ""` to remove a
  shipped icon) without restating the rest.
- Note the TUI ignores project-local `sase/sase.yml` (`set_include_local_config(False)`,
  `src/sase/config/core.py:91-98`), so docs must direct users to `~/.config/sase/sase.yml`.

### Schema

Add `tribes` to the `ace` object in `src/sase/config/sase.schema.json` (the section is `additionalProperties: false`, so
this is mandatory — `test_default_config_matches_public_schema` enforces lockstep). Shape: object with `propertyNames`
pattern `^[A-Za-z0-9_.-]+$`; each value a closed object `{icon: string (maxLength ~16), initially_expanded: boolean}`,
no required fields, with descriptions and defaults. Mirror the style of existing user-keyed maps (`telegram.commands`,
`llm_provider.model_aliases.custom`). The Config Center field model is schema-driven (Python hands the schema JSON to
the `config_field_model` Rust binding, `src/sase/config/inventory.py:247-251`), so **no sase-core changes are
required**; this feature is presentation-plus-config and stays entirely in this repo per the Rust-core boundary litmus
test.

### Tribe display resolution (new module)

Add a small model module (e.g. `src/sase/ace/tui/models/tribe_display.py`):

- Frozen dataclass `TribeDisplay(icon: str, initially_expanded: bool)` with a no-icon/expanded fallback for unconfigured
  tribes.
- `tribe_display_for(panel_key: PanelKey) -> TribeDisplay` reads `load_merged_config()["ace"]["tribes"]`, mapping panel
  key `None` (the untagged panel) to the `default` entry — this makes the config correct today and forward-compatible
  with the `@default` rename.
- Icon sanitization is defensive, never raising in a render path: strip whitespace, treat values containing control
  characters/newlines as absent, and cap the icon at a few terminal cells using Rich `cell_len` so a misconfigured icon
  cannot wreck panel layout.
- **Perf (tui_perf rules):** memoize resolution per `current_config_token()` exactly like model-alias resolution — no
  disk I/O, YAML parsing, or re-validation on render paths. Tests that mutate config must call `clear_config_cache()` as
  usual.

### Icon rendering in the panel header

The header is built in one place: `agent_panel_border_title`
(`src/sase/ace/tui/actions/agents/_display_panel_titles.py:121-169`). Insert the icon (plus a single space) immediately
before the tribe label, for both the `@{key}` branch and the `(no tribe)` branch (via the `default` entry); the merged
"All agents" header gets no icon. Resulting order: `[hint] ❖ ▸ 🪓 @chop · 3 [R1 W2]`.

- Thread the resolved icon through `_agent_panel_title` (`_display_panel_collection.py:89-129`); call sites in
  `_display_panel_widgets.py` need no structural change.
- Width negotiation already uses `Text.cell_len` (`widgets/agent_list.py:272-284`), so double-width emoji are safe in
  the border title. Panel border titles are rebuilt each sync and are **not** in `AgentRenderCache`, so no cache-key
  changes are needed. Do not add icons to in-panel banners (`_agent_list_render_banner.py` pads with `len()`, not cell
  width — out of scope).
- For cohesion, also prefix the icon in the tribe summary label `_panel_label`
  (`src/sase/ace/tui/models/agent_tribe_summary.py:145-146`) so the detail panel matches the header. This is a
  one-function touch; drop it if it ripples further than expected.

### Initial expansion: explicit fold intent (persistence schema v3)

Today `ace_agents_fold_state.json` (schema v2, `src/sase/ace/tui/models/agent_fold_persistence.py`) stores only
`collapsed_panels`; "expanded" is the absence of a record, so "user expanded this" and "user never touched this" are
indistinguishable. That cannot express "config collapses @chop unless the user expanded it", so the schema gains
explicit intent:

- **Snapshot:** add `expanded_panels: frozenset[PanelKey]` to `AgentsFoldStateSnapshot`. The two sets are disjoint
  (overlap is a strict decode error, consistent with the file's existing fail-open-to-empty strictness). Bump
  `SCHEMA_VERSION` to 3; keep decoding v1 and v2 files (both yield empty `expanded_panels`). Encode `expanded_panels`
  only when non-empty with the same deterministic ordering, and bound it with the same limit as `MAX_COLLAPSED_PANELS`.
- **Session state:** alongside `_collapsed_panel_keys` (`_state_init.py`), add `_expanded_panel_keys`. Both sets hold
  **explicit user intent only** — config-driven collapse must never be written into them, otherwise a later config
  change could not take effect and a save triggered by toggling one panel would freeze every seeded panel.
- **Recording intent:** every panel fold mutation path funnels through intent-recording helpers: collapse adds to
  collapsed / discards from expanded; expand does the reverse. Update the handlers in `_folding_panels.py`
  (collapse/expand/isolate and the fold-level layout actions), the hint-driven toggles in `_panel_hint_folding.py`, and
  the pre-load intent journal + replay (`AgentFoldIntent`, `_replay_agents_fold_intent` in `_fold_persistence.py`) so
  intents recorded before hydration survive. Persistence capture (`_capture_agents_fold_state`) writes both sets; loads
  hydrate both in `_install_loaded_agents_fold_state`.
- **Effective collapse is computed, not materialized.** A pure helper decides, for the panel keys currently on screen:
  `collapsed(k) = k in collapsed_intent or (k not in expanded_intent and not tribe_display_for(k).initially_expanded)`.
  Use it wherever `_collapsed_panel_keys` is consumed today (the `_sync_panel_group` →
  `AgentPanelGroup.from_agents(collapsed_panel_keys=…)` hand-off, the render-collapsed/height decisions in
  `_display_panel_widgets.py`, the header's `collapsed` chevron flag, and fold-action state checks). Because nothing is
  materialized, a tribe that first appears mid-session honors its config automatically, and editing the config between
  sessions Just Works for untouched panels.
- **Precedence (the contract):** explicit user intent (this session or persisted) > configured `initially_expanded` >
  default `true`.
- Startup behavior matches today's pattern: config defaults apply from first paint (merged config is already loaded
  during state init), persisted intent lands when the fold snapshot installs — the same brief window v2 already has for
  collapsed panels.

## Testing

- **Config schema** (`tests/test_config_schema.py`): the default-config/schema lockstep test must pass; add
  acceptance/rejection cases for `ace.tribes` (valid entries; bad tribe-name keys; unknown item fields; wrong types)
  mirroring the `telegram.commands` and custom-model-alias suites.
- **Tribe display resolution** (new test module): default/None mapping, unknown-tribe fallback, `icon: ""` override,
  sanitization of hostile values (control chars, absurd length), and per-config-token memoization.
- **Fold persistence** (`tests/ace/tui/test_agent_fold_persistence.py`): v3 round-trip, v2 and v1 files decode with
  empty `expanded_panels`, overlap and bounds rejection, deterministic encoding.
- **Seeding precedence** (fold behavior tests near `test_agent_panel_collapse.py` / `test_agents_panel_fold_mode.py` /
  `test_agent_fold_transitions.py`): untouched panel follows config both ways; explicit expand of a config-collapsed
  tribe survives restart; explicit collapse of a config-expanded tribe survives restart; later config flips do not
  override recorded intent; a tribe appearing mid-session honors its config; toggling panel A does not persist seeded
  state for untouched panel B.
- **Panel titles** (`tests/ace/tui/test_agent_panel_titles.py`): icons appear in the expected positions; existing
  exact-string assertions (`"@apple · 2 [R2]"`, `"❖ @chop · 21 …"`, `"(no tribe) · 2 [R2]"`) updated where default icons
  now apply.
- **PNG visual snapshots**: existing `agents_tribe_panel_*` goldens render `@epic` and will intentionally change once it
  gains 🗻 — regenerate with `--sase-update-visual-snapshots` and eyeball the diffs in `.pytest_cache/sase-visual/`. Add
  a scenario covering a configured icon plus a config-collapsed tribe. Verify each shipped emoji renders
  deterministically under the pinned fontconfig/Fira Code fixtures (🐍/🐚 prove the pipeline handles emoji); if any
  chosen glyph renders as tofu in the golden pipeline, substitute a glyph that renders and note the swap.

## Documentation

- `docs/configuration.md`: new `ace.tribes` subsection (fields, defaults table, the user-intent-beats-config rule, the
  "" -removes-icon override, and that the TUI reads user-level config only).
- `docs/ace.md`: brief mention under the Agents-tab/tribes material.
- Review the `?` help modal's fold/tribe sections; add a line only if the existing text now under-describes initial
  expansion (per the ace help-sync rule).

## Non-goals

- Renaming `(no tribe)` to `@default` (separate change; the `default` config key is already forward-compatible with it).
- Icons in in-panel group banners, agent rows, or per-tribe colors (possible follow-ups).
- Any `../sase-core` change — the Rust config field model is schema-driven.

## Risks and mitigations

- **Seeded state leaking into persistence** is the main correctness trap; it is designed out by computing effective
  collapse instead of materializing it, and the precedence tests pin the behavior.
- **Emoji width/rendering drift**: border-title width uses `cell_len`, chosen glyphs are single-codepoint, and visual
  goldens catch renderer surprises.
- **Perf regressions**: resolution is memoized per config token; no new refresh paths, no render-path I/O, fold
  load/save stays off-thread.

## Verification

Run `just install`, then `just check` and `just test-visual`; smoke-test `sase ace` manually with a user-config override
(custom icon, `initially_expanded: false`, then toggle and relaunch to confirm intent persistence).
