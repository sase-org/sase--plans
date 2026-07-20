---
tier: tale
title: Configurable Agents-tab tribe panel colors
goal: 'ACE users can assign a safe, per-tribe color to the icon and @tribe identity
  in Agents-tab panel headers, while all semantic title chrome keeps its existing
  colors. The bundled default, epic, research, and chop tribes ship with four distinct,
  readable colors and unconfigured or explicitly cleared colors retain the current
  gold fallback.

  '
create_time: 2026-07-19 21:53:58
status: wip
prompt: 202607/prompts/tribe_panel_colors.md
---

# Plan: Configurable Agents-tab tribe panel colors

## Product context

The Agents tab already resolves `ace.tribes.<name>` display settings for each split panel, caches them by the merged
configuration token, and uses them for icons and initial expansion. Panel identities still all render in the same gold,
so the new icons improve shape recognition without giving each high-traffic tribe a stable color identity.

Add one optional `color` field to each existing tribe display entry. It controls the bold foreground color of the
tribe's icon and `@<tribe>` label in the split-panel border title. It does not recolor the title's interaction and
status signals: jump hints, selected-panel marker, collapsed chevron, isolation marker, counts, metric labels, and
metric values retain their existing semantic palette. The merged `All agents` title is not a tribe identity and remains
unchanged.

The four tribes requested in the original display-config work receive distinct bundled colors:

| Config key | Header identity | Color     | Intent                                           |
| ---------- | --------------- | --------- | ------------------------------------------------ |
| `default`  | `⌂ @default`    | `#87D7FF` | Familiar sky blue for the home/unassigned bucket |
| `epic`     | `▲ @epic`       | `#AF87FF` | Lavender-purple for large, multi-phase work      |
| `research` | `∴ @research`   | `#5FD7AF` | Teal-green for investigation and discovery       |
| `chop`     | `† @chop`       | `#FFAF5F` | Warm amber-orange for AXE chop work              |

`pinned` and `review`, which were added as extra icon defaults by the previous implementation, do not receive colors in
this focused change; they continue to use the established gold fallback. All four selected colors come from the TUI's
existing bright palette, remain legible on the pinned dark snapshot theme, and differ in both hue and role association.

## Configuration contract and validation

Extend each closed object under `ace.tribes` in `src/sase/config/sase.schema.json` with:

```yaml
ace:
  tribes:
    research:
      icon: "∴"
      color: "#5FD7AF"
```

`color` is an optional string containing either an empty value or a six-digit hexadecimal RGB foreground color. Use a
schema pattern equivalent to `^(?:|#[0-9A-Fa-f]{6})$`, document the default as `""`, and keep unknown entry fields
rejected. Restricting the public format to exact RGB hex makes the value portable and prevents a user-provided Rich
style fragment from injecting backgrounds, links, or text attributes. An absent or empty value means “use ACE's existing
tribe-title fallback” (`bold #FFD75F`). The explicit empty form is necessary because merged defaults are deep: it lets a
user clear one of the four bundled colors without replacing the rest of that tribe's default entry.

Update `src/sase/default_config.yml` with the four values above and comments that explain the empty-value fallback. This
remains a user-level ACE setting, with the same overlay and project-local exclusion behavior as `icon` and
`initially_expanded`. The schema-driven Config Center requires no Rust-core or binding change.

## Cached display resolution

Extend the frozen display value in `src/sase/ace/tui/models/tribe_display.py` with a `color` field whose resolved
fallback is empty. Add a defensive sanitizer that strips surrounding whitespace, accepts only exact six-digit hex, and
returns empty for wrong types or malformed values. Schema validation is the normal user-facing guard, but the resolver
must still fail safely when called from rendering against a malformed or partially loaded mapping.

Resolve `icon`, `color`, and `initially_expanded` together inside the existing `_tribe_displays_for_token()` cache. Do
not add config reads, filesystem checks, Rich parsing, or a second cache to the title path. A config-token change must
continue to invalidate the complete display value atomically, and an unconfigured tribe must retain the current
iconless, expanded, gold-title behavior.

## Panel-title rendering

Teach `agent_panel_border_title()` in `src/sase/ace/tui/actions/agents/_display_panel_titles.py` to accept an optional
resolved tribe color while preserving the current call-site default. Derive one bold identity style from the sanitized
color, falling back to `_PANEL_TRIBE_STYLE` when empty, and apply it only to the icon and `agent_panel_label(key)`.

Keep the selected `❖` marker on `_PANEL_SELECTED_CHROME_STYLE` even when a tribe has a custom color. This preserves the
documented selection accent and avoids making arbitrary tribe colors carry interaction meaning. Likewise, retain the
neutral collapsed chevron and count separator, the yellow jump/fold hints, the restore marker, and every status metric
style. Plain title text and cell width must be identical to today's icon-enabled output; only Rich spans change.

In `src/sase/ace/tui/actions/agents/_display_panel_collection.py`, resolve `tribe_display_for(key)` once per title build
and pass both its icon and color. The existing token cache makes repeated title refreshes cheap. Do not color the merged
`All agents` title, agent rows, grouping banners, detail-panel `TRIBE` summary, modal text, or prompt metadata; those
are separate surfaces with their own semantic style systems and are outside this request.

## Tests and visual verification

Add focused coverage at each boundary:

- Extend `tests/test_config_schema_tribes.py` with accepted uppercase/lowercase RGB and explicit-empty cases, plus
  rejection of short hex, missing `#`, named colors, style fragments, wrong types, and unknown fields. Keep the default
  config/schema lockstep test green.
- Extend `tests/ace/tui/models/test_tribe_display.py` to cover `None` → `default` mapping with color, unconfigured and
  empty fallback, defensive malformed-value handling, and color updates across config-token changes without extra config
  loads.
- Extend `tests/ace/tui/test_agent_panel_titles.py` to assert exact Rich spans: custom color covers only the icon and
  `@tribe`; selected markers and totals keep the focus accent; collapsed/hinted titles keep their semantic styles; empty
  color preserves the legacy gold; and `All agents` ignores tribe display color. Preserve all existing plain-text
  expectations.
- Update the dedicated six-panel visual scenario in `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`
  so it verifies the four configured identity colors in the rendered Rich titles as well as the icon/collapse contract.
  Regenerate every Agents-tab PNG golden affected by the new bundled colors, inspect the dedicated multi-tribe image for
  contrast and differentiation, and accept only color changes attributable to this feature. The pinned and review titles
  should demonstrate the unchanged fallback.

Run `just install` before repository commands, then run the focused schema/model/title tests, `just test-visual`, and
the mandatory `just check`. Also run `git diff --check` and audit the final diff so generated snapshots are the only
binary changes.

## Documentation

Update the `ace.tribes` example, field table, bundled-default description, and empty-override semantics in
`docs/configuration.md`. Update the Agents-tab tribe-panel prose in `docs/ace.md` to say that icons, identity colors,
and initial expansion are configurable, while explicit fold intent still outranks initial expansion. The help modal does
not enumerate appearance fields today, so no help text change is expected unless implementation inspection finds a newly
inaccurate statement.

## Risks and boundaries

- **Style injection:** accept only `#RRGGBB`, sanitize again before constructing a Rich style, and treat malformed
  runtime values as the legacy fallback.
- **Lost interaction cues:** split tribe identity from selected/fold/status chrome and pin the distinction with span
  tests plus selected/collapsed visual coverage.
- **Render-path regression:** extend the existing config-token-cached value and resolve it once per title build; add no
  I/O, parsing, refresh lane, or data-scaled work to Textual's event loop.
- **Snapshot churn:** bundled colors intentionally change affected Agents-tab goldens. Use the dedicated multi-tribe
  image as the design oracle and reject unrelated pixel changes.
- **Non-goals:** no configurable background or arbitrary Rich style, automatic palette assignment for unknown tribes,
  per-theme color variants, recoloring of tribe names outside the split-panel header, persistence changes, or
  `sase-core` work.
