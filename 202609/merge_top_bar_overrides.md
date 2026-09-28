---
tier: tale
title: Merge top-bar override indicators
goal:
  Show alias, priority, and provider-disable pills together under one overrides label on
  the top bar.
size: medium
proposed_by: bbugyi200.apollo.2n
create_time: 2026-09-28 08:24:08
status: wip
---

# Plan: Merge top-bar override indicators

## Outcome

The TUI's second row from the top is `#top-bar`: the tab strip on the left and the
labeled indicator cluster on the right. That cluster currently renders seven groups:

`procs:` · `updates:` · `overrides:` · `priority:` · `disabled:` · `stash:` · `inbox:`

`overrides:`, `priority:`, and `disabled:` are one family of temporary launch overrides.
Merge them into the single existing `overrides:` group. The three pills stay side by
side, in today's left-to-right order, and each keeps its own colors, glyphs, wording,
and tooltip detail. The `priority:` and `disabled:` labels, and the `·` separators that
sat between these three groups, go away.

A busy full-density cluster reads:

`procs: … · updates: … · overrides:  @medium@max ∞  CODEX ★ ∞  CLAUDE off ∞  · stash: … · inbox: …`

The per-tab status rows under that bar (`#agent-info-row`, `#artifacts-header`,
`#axe-info-row`) are a different surface. Their gold launch-default pill and `project:`
chip stay where they are.

## Rendering contract

`AliasOverridesIndicator` (`#alias-overrides-indicator`, `GROUP_LABEL = "overrides"`)
remains the only mounted widget for this slot. Its body is the concatenation of
whichever of these three pills is non-empty:

1. The violet non-default alias pill, built exactly as
   `AliasOverridesIndicator._build_content` builds it today (single
   `@alias[@effort] <remaining>`, `epic lander` / `big epic lander` subjects, or
   `@first +N`). The default-launch setting stays excluded.
2. The priority pill, built exactly as `ProviderPriorityIndicator._build_content` builds
   it today (`CODEX ★ <remaining>`, plus `soft-disabled` or `unavailable` only in those
   states). Still one provider, never `+N`.
3. The disable pill, built exactly as `ProviderDisablesIndicator._build_content` builds
   it today (`CLAUDE off <remaining>`, `CLAUDE soft <remaining>`, or `CLAUDE +N`, soft
   palette only when every active disable is soft).

Join non-empty pills with `Text.append_text` and no extra characters. Each pill already
carries its leading and trailing pad space, so adjacent accents are separated the same
way the proc gear chips are (`" ⚙ 2  ⚙ 1 "`): the trailing pad of one pill plus the
leading pad of the next. Omit an empty pill entirely so a missing fact does not leave a
hole.

`TopBarGroup` still supplies the dim label, so full-density plain text is `overrides: `
plus that body. Because each pill starts with a space, the rendered text has two spaces
after the colon, matching `procs:` and the current override pills. Examples, including
that padding:

- Alias only: `overrides:  @medium@max ∞ `
- Priority only: `overrides:  CODEX ★ ∞ `
- Hard disable only: `overrides:  CLAUDE off ∞ `
- All three: `overrides:  @medium@max ∞  CODEX ★ ∞  CLAUDE off ∞ `
- None: empty body, group hidden, width 0. Neighboring cluster separators collapse
  through the existing `separator_visibility` rules.

Compact density drops the `overrides:` label and renders the same body. `priority:` and
`disabled:` appear nowhere in either density.

Palettes stay per pill: violet alias, cyan priority, orange hard-disable, yellow
soft-disable. A priority provider that is soft-disabled or unavailable keeps the palette
it uses today. Do not recolor the three pills onto one accent.

## Tooltip and click

The group has one tooltip. Build it from the three existing tooltip builders, in the
same order as the pills, including a section only when that fact is active. Each builder
today ends with `Press ,m for Config > Launch.`; keep those builders' return values
stable for their unit tests, strip that trailing footer from each section while
composing, join the remaining sections with one blank line, and append the footer once.
A single active fact therefore still matches today's tooltip exactly. Hovering the label
or any pill shows that combined tooltip.

`CLICK_ACTION` stays `open_models_panel`. One click on the group opens Launch Control
once. There are no longer separate click targets for priority and disables.

## Data and refresh

One 30-second interval on `AliasOverridesIndicator` replaces the two intervals that
exist today. `refresh()` with no arguments rebuilds content; `refresh` with arguments
still forwards to `Widget.refresh`, as both widgets do now.

Each rebuild does this from one snapshot:

- Call `get_active_alias_overrides()` once and drop the default-launch setting, via the
  existing `_active_non_default_overrides` helper.
- Call `peek_provider_routing_context` once. Import it into `alias_overrides_indicator`
  so that module is the only patch point for the mounted widget. Use that same context
  for the priority pill and the disable pill.
- Classify priority with `ProviderPriorityIndicator._priority_availability`, which keeps
  calling `provider_routing_facts` inside `provider_priority_indicator`. Visual tests
  patch that name there; leave that call path in place.
- Store the joined body with one `_set_body` call, and replace the tooltip only when it
  changed.

`__init__` still paints the correct body before mount, and `on_mount` reapplies before
starting the single interval. The text-signature short-circuit stays: an unchanged body
does not call `Static.update`.

Delete the sibling push. `ProviderDisablesIndicator._apply_priority_sibling` exists only
to copy one peek onto `#provider-priority-indicator`. After the merge that query is
gone.

`LeaderModeMixin._refresh_launch_indicators` refreshes `#alias-overrides-indicator`
once, plus the usage indicator and the launch-context source. That single refresh covers
alias, priority, and disable pills. Stop querying `#provider-disables-indicator`.

## Classes that stop being widgets

Keep `ProviderPriorityIndicator` and `ProviderDisablesIndicator` as the owners of their
static pill and tooltip builders so the existing plain-text and palette tests can keep
calling `_build_content`, `_build_tooltip`, and `_priority_availability`. They are no
longer `TopBarGroup` subclasses and are not composed into `TopBarIndicators`. Remove
their widget lifecycle: mount-time content, `on_mount`, `refresh`, `_apply_content`, the
sibling push, `GROUP_LABEL`, and `CLICK_ACTION`.

`TopBarIndicators` composes five groups. Update `_TOP_BAR_GROUP_IDS`, the compose order,
and the comments that say seven groups and six separators:

`proc-indicator`, `updates-indicator`, `alias-overrides-indicator`,
`stashed-prompts-indicator`, `notification-indicator`

Four separators remain. `sync_top_bar_groups` zips separators with `strict=True`, so the
id tuple, compose order, and separator count have to move together. Keep the widget id
`#alias-overrides-indicator`; the visible change is the label contents, not a renamed
query id.

Leave the lazy exports in place while those classes still exist. No CSS change:
`styles.tcss` has no per-id rules for these three indicators.

## Leave unchanged

- Pill grammar in `_override_pill.py`: subjects, effort suffix, remaining-time text,
  `∞`, `+N` collapse, and the soft / hard / unavailable words.
- The gold default-model pill and current-project chip on each tab's launch-context
  cluster.
- Launch Control's own title line (`priority: CODEX ★ <time>`, `disabled providers: …`,
  `CLAUDE soft <time>`). That modal is not this cluster.
- Routing behavior, persistence, and the other top-bar groups (`procs`, `updates`,
  `stash`, `inbox`).
- `CHANGELOG.md`. It is generated from commits.

## Docs

Update the two top-bar descriptions in `docs/ace.md`:

- The provider-routing paragraph that says active priority and disables render as
  labeled `priority:` and `disabled:` groups beside the violet `overrides:` group.
  Describe one `overrides:` group whose pills sit side by side. Keep the examples of
  state words, `off` / `soft`, and `+N`, with the `overrides:` label in front. Say that
  each pill keeps its color and its tooltip section, and that clicking the group opens
  Launch Control.
- The non-default alias paragraph that says the violet pill is the `overrides:` group.
  Say that pill is the alias slot inside that group, beside the priority and disable
  pills when those are active. The single-alias and `@first +N` rules stay.

Match the existing prose style in that file (`overrides: CODEX ★ 42m`), which does not
spell the widget's pad spaces.

## Tests

Retarget every top-bar monkeypatch of `peek_provider_routing_context` onto
`sase.ace.tui.widgets.alias_overrides_indicator`. Patches of `provider_routing_facts`
stay on `provider_priority_indicator`. Alias-override patches stay on
`alias_overrides_indicator`.

Files that drive or assert the mounted groups:

- `tests/ace/tui/test_top_bar_indicators.py`. Wide busy cluster contains `overrides:`
  once, contains the three pills, and does not contain `priority:` or `disabled:`. `·`
  still joins the remaining groups and is still a single-space separator (`  ·  ` stays
  absent). Compact density drops `overrides:` along with the other labels and keeps the
  glyphs (`⚙`, `⬆`, `≡`, `★`). The click test clicks `#alias-overrides-indicator` once
  and records one `open_models_panel`.
- `tests/ace/tui/test_top_bar_order.py`. Cluster child ids drop
  `provider-priority-indicator` and `provider-disables-indicator`. The narrow bounds
  test finds both the alias pill and the disable pill on the one widget, with no
  `priority:` or `disabled:` label, and the top bar still stays inside its width.
- `tests/test_provider_priority_indicator.py`. Keep the pill and tooltip unit tests on
  the static builders. Rewrite `test_single_peek_drives_both_routing_groups` for one
  widget and one peek: both pills visible, no `+N` on the priority pill, and clearing
  priority removes only that pill while the disable pill remains.
- `tests/test_provider_disables_indicator.py`. Keep pill and tooltip unit tests. Remove
  the assertion that the mounted group label is `disabled`.
- `tests/test_provider_disables_indicator_widget.py`. Mount, click, and the
  unchanged-`update` short-circuit move to `#alias-overrides-indicator`. The initial
  peek test can call the static disable builder.
- `tests/test_alias_overrides_indicator.py`. Existing alias-only `_build_content` tests
  stay on that alias-only builder. Add combined-body coverage for alias-only,
  priority-only, disable-only, all three, and none.
- `tests/test_models_panel_leader_mode.py`. The refresh query list is the alias
  indicator, the usage indicator, and the launch-context source. One `refresh()` on the
  alias indicator is the routing refresh.
- `tests/ace/tui/test_usage_header.py`. A disable pill click goes through
  `#alias-overrides-indicator`, still opens Launch Control, and still does not open the
  usage modal.
- `tests/ace/tui/test_top_bar_palette.py`. It can keep calling
  `ProviderPriorityIndicator._build_content`.

The merged group is narrower by the two dropped labels and the two dropped separators
(about 26 cells). `test_busy_cluster_compacts_narrow` assumes a 120-column terminal is
compact. If the busy cluster now fits there at full density, shrink that case until
`free_cells < full_cells` instead of deleting the density assertion. Wide 220 stays
full. Restoring the wide size restores the `overrides:` label.

Visual fixtures and snapshots that paint these pills:

- `tests/ace/tui/visual/test_ace_png_snapshots_top_bar_indicators.py` (full and compact
  busy cluster).
- `tests/ace/tui/visual/test_ace_png_snapshots_alias_overrides_indicator.py`, including
  its autouse empty-routing patch and the priority, disable, soft-disable, and
  unavailable frames. Point those peeks at the alias module. Frames whose only change is
  the shared label and the missing inter-pill `·` need new goldens. Alias-only frames
  stay the same when routing is empty.
- `tests/ace/tui/visual/_provider_usage_indicator_fixtures.py` `quiet_top_bar`, so
  `disables=` still reaches the mounted widget.
- `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py` and
  `test_ace_png_snapshots_provider_usage_indicator_states.py` wherever a disable or
  alias pill is in the frame (`top_bar_usage_attention_80x24`,
  `top_bar_disable_pill_usage_dark_160x24`, `top_bar_disable_pill_usage_light_160x24`).
- `tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py` if a neighbor frame
  includes an alias pill.

## Verification

Run the non-visual tests above with the repo's pytest runner. Then update the affected
TUI goldens with a targeted `just fix-tui-screenshots -- <selectors>` for the snapshot
modules listed above. Read that command's report before treating goldens as current:
`partial` means some frames were skipped. Inspect every golden creation, removal, and
update. The expected pixel change is the shared `overrides:` label and the pills sitting
next to each other without `priority:`, `disabled:`, or a `·` between them. Alias-only
frames with empty routing should not move.
