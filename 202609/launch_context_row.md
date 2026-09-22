---
tier: epic
title: Launch-context cluster on each tab's status row
goal: 'The launch-default model/effort and current-project chips leave the crowded
  top bar and appear, clearly labeled and with precise tooltips, at the far right
  of every tab''s status row, backed by one shared state source.

  '
phases:
- id: launch-context-source
  title: One shared launch-context source
  depends_on: []
  size: medium
  description: 'launch-context-source: move launch-default and current-project polling
    and resolution into one app-scoped LaunchContextSource and make both indicators
    render-only views, with no visible change.'
- id: launch-context-bar
  title: Labeled launch-context cluster on every tab's status row
  depends_on:
  - launch-context-source
  size: medium
  description: 'launch-context-bar: build the labeled, density-aware LaunchContextBar,
    mount it on the Agents, Artifacts, and Services status rows, drop the chips from
    the top bar, rewrite tooltips, and refresh tests, goldens, and docs.'
proposed_by: bbugyi200.apollo.0s.f0.f0.w2
create_time: 2026-09-20 22:21:47
status: done
bead_id: sase-14y
---

- **PROMPT:** [prompts/202609/launch_context_row.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/launch_context_row.md)
- **BEAD:** [sase-14y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14y/README.md)

# Move the launch-default model and current-project chips into a labeled "launch context" cluster on each tab's status row

## Goal

The top bar's right cluster (the row with the `Agents | Artifacts | Services` tabs) is
crowded: `⚙ 3  CLAUDE(opus)@high  CODEX +1  +sase  ✱ 2  78 ◆104`. Two of those chips —
the launch-default model/effort (`opus@high`, rendered by `LLMOverrideIndicator`) and
the current project (`+sase`, rendered by `CurrentProjectIndicator`) — are "context"
rather than "alerts", and today they carry no visible hint of what they mean.

Move both chips one row down, to the **far right of each tab's status row** (the row
that carries the agent status counts on the Agents tab), and give them room to explain
themselves with short dim labels plus excellent tooltips. The top bar keeps only the
alert-style indicators (procs, monitors, updates, the violet non-`default` alias
override pill, provider disables, stashed prompts, notifications).

## Design

### Why a shared state source first

The status row is tab-specific: the Agents tab shows `AgentInfoPanel`, Artifacts shows
the `#artifacts-header` sub-tab strip, and Services shows `AxeInfoPanel`. The launch
context is global — launches (the `+` picker, leader keys) work from every tab — so it
must appear on every tab's status row. Textual cannot share one widget instance across
three parents, and reparenting on tab switch would repaint placeholders (`...`) on every
switch. Running three independent copies of today's self-polling indicators would triple
the resolve workers and let hidden copies disagree.

So phase 1 extracts the polling/resolution that currently lives inside the two indicator
widgets into **one app-scoped, non-rendering `LaunchContextSource`**. The indicator
widgets become render-only views that paint from the source's current state (instantly
on mount, with no placeholder when the source already resolved) and re-render when the
source broadcasts. Phase 2 then mounts one lightweight view cluster per tab.

### Visual grammar (phase 2)

A right-aligned cluster, reading left to right, separated from the host row's own
content by at least two cells and ending at the row's right padding:

```
Full:     default opus@high · current +sase
Override: override  opus@high 2h  · current +sase      (gold lane pill, unchanged styling)
Compact:  opus@high · +sase
No proj:  default opus@high                              (compact: opus@high)
```

- `default` / `current` are dim lowercase micro-labels (the same dim grey the status
  rows use for their own labels). They name the two exact concepts the docs already use:
  the **launch default** and the **current project**. Do NOT use a sentence like "new
  agents opus@high in +sase": the current project is the VCS-MRU head that seeds filters
  and highlights the `+` picker row — it is not silently applied to prompts that name no
  project, so "in +sase" would lie.
- When a temporary override on the `default` lane is active the model label reads
  `override` (styled in the gold lane's dim tone) and the chip keeps today's gold
  override pill (subject, `@effort`, remaining time / `∞`).
- The model chip keeps its provider-palette two-tone coloring (subject in the provider's
  subject style, `@effort` in its detail style); the project chip keeps its accent
  coloring (dim `+`, bold name). Drop the chips' built-in leading/trailing pad spaces
  where the labels now supply spacing, so the cluster is evenly spaced (one space
  between label and chip; `·` between the two groups).
- The ` · current +sase` group disappears entirely (zero width, no dangling separator)
  when no project resolves or `ace.current_project.indicator` is false.
- Density is chosen per host from the cells actually free on that row: the richest of
  `full` then `compact` that fits without truncating the host's own content; if even
  `compact` does not fit, render `compact` anyway and let the host's primary content
  clip on its right (the model chip is never hidden). The rule lives in one pure
  function, e.g.
  `choose_launch_context_density(free_cells: int, *, full_cells: int, compact_cells: int) -> Literal["full", "compact"]`,
  so it is unit-testable.

### Tooltips (phase 2)

Concise, factual, and consistent. Keep the existing `PROVIDER(model) @ effort` long form
in tooltips. Before writing the directive names below, verify the actual override
spellings in `sase.xprompt.directives` (e.g. whether effort has its own directive) and
use the real ones.

Model chip, calm state:

```
Launch default: CLAUDE(opus) @ high
The model and effort a new agent uses when its prompt sets no %model.
@default rotates across 3 models; CLAUDE(opus) is next.        <- only for round_robin
Click (or ,m) to change it in Config › Launch.
```

Model chip, override state:

```
Temporary override: CLAUDE(opus) @ high · 2h left              <- or "· until cleared"
New agents use this instead of the launch default until it lapses.
Click (or ,m) to change or clear it in Config › Launch.
```

Unresolved/failed states keep a first line of `Launch default: resolving…` /
`Launch default: unavailable` plus the same second and last lines.

Project chip:

```
Current project: sase
Your working project: it seeds project filters and is preselected in the + launch picker.
Set by your last launch (#gh:sase)                              <- or "Set via Patch <name>"
Click to launch an agent on a project · press c on the Projects tab to switch.
```

Labels (`default` / `override` / `current`) carry short tooltips of their own:
`The model new agents launch with by default.` /
`A temporary override is replacing the launch default.` /
`The project you are currently working in.` The leader key text (`,m`) should come from
the keymap registry if a display helper is already available to the widget; otherwise
keep today's literal.

### Placement per tab (phase 2)

- **Agents**: wrap `#agent-info-panel` in a `Horizontal#agent-info-row` (height 1,
  `$surface` background, the panel at `width: 1fr`, the cluster `width: auto`). Move the
  onboarding hide rule (`#agents-view.-onboarding-active #agent-info-panel`) to the row.
  `AgentInfoPanel` click spans (filter segment) must keep working — they are computed
  against the panel's own content, which does not move.
- **Artifacts**: append the cluster to `#artifacts-header` after
  `#artifacts-split-badge`. Keep the sub-tab strip visually centered when the row has
  room (grow the left spacer to mirror the right cluster's width); when it does not, let
  the strip center in the remaining space and pick `compact`. The strip already reflows
  (`reflow_to_fit=True`); the cluster must not push it into a worse reflow than
  `compact` would.
- **Services**: wrap `AxeInfoPanel`'s first line host in a `Horizontal#axe-info-row`
  inside `#axe-container`, moving the `border-bottom` from `#axe-info-panel` to the row
  so the divider still spans the full column; the cluster sits at the far right of the
  first line.

## Constraints and conventions

- Read `sase memory read tui.md` and its children `tui_perf.md` and `tui_screenshot.md`
  before starting. The periodic tick must stay peek-only (config-token + `os.stat`);
  real resolves stay on worker threads, never the UI thread, timers, or render paths —
  the existing comments in `llm_override_indicator.py` and
  `current_project_indicator.py` explain why; carry those comments over to the source.
- Presentation-only work; no Rust core changes are needed (the resolvers already live
  behind existing Python adapters).
- Multiple view instances exist after phase 2, so every caller that today does
  `query_one(LLMOverrideIndicator)` / `query_one(CurrentProjectIndicator)` or queries
  `#llm-override-indicator` / `#current-project-indicator` must go through the source
  (for invalidation) or `query(...)` (for fan-out).
- Update `src/sase/default_config.yml` comments that say the chip lives "in the top
  bar".
- Read `sase memory read lint_and_test.md` before finishing each phase; `just check` is
  the verification recipe.

## Phases

### Phase 1: One shared launch-context source

Extract launch-default and current-project polling/resolution into a single app-scoped
`LaunchContextSource`; turn `LLMOverrideIndicator` and `CurrentProjectIndicator` into
render-only views. No visible change.

- Add `src/sase/ace/tui/widgets/launch_context_source.py` with a non-rendering
  `LaunchContextSource(Widget)` (CSS `display: none`), mounted exactly once by
  `AppLayoutMixin._compose_layout` (id `launch-context-source`), plus an immutable
  `LaunchContextState` dataclass holding: the resolved `_LaunchDefaultSnapshot` (or
  resolving/failed flags), the current-project snapshot (project + accent, or
  resolving/failed), and a `now`-independent view of the active `default`-lane override
  (views compute remaining time at render).
- Move into the source, preserving behavior exactly: the 5s peek-only tick; the
  launch-default change token, cold-start, failure-retry and override-lapse
  re-resolution rules; the current-project change token and
  `ace.current_project.indicator` gating; both worker groups;
  `invalidate_cached_default()` (rename on the source to `invalidate_launch_default()`)
  and the current project `invalidate()` (as `invalidate_current_project()`).
- Broadcast: after any state change (and on every tick, so override countdowns advance),
  the source calls a single method (e.g. `apply_launch_context(state)`) on every mounted
  view. Views pull `source.state` on mount so a view mounted after resolution paints
  resolved content immediately.
- `LLMOverrideIndicator` / `CurrentProjectIndicator`: remove `set_interval` and worker
  code; keep their static content and tooltip builders and click actions; a no-arg
  `refresh()` re-renders from the source state. Keep
  `LLMOverrideIndicator._build_content` (synchronous test helper) working.
- Route callers through the source: `LeaderModeMixin._refresh_launch_indicators`
  (`src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`) and
  `_invalidate_current_project_indicator` in
  `src/sase/ace/tui/modals/project_management_actions.py`; grep for any other
  `query_one` / `#…-indicator` use of these two widgets (including `tests/`).
- Tests: add `tests/ace/tui/test_launch_context_source.py` covering the moved resolution
  rules, one resolve per token change regardless of how many views are mounted, two
  mounted views always painting identical content, and a late-mounted view painting
  resolved content with no placeholder. Update `tests/test_llm_override_indicator.py`,
  `tests/test_llm_override_indicator_provider_color.py`,
  `tests/test_launch_default_indicator_pool_rotation.py`,
  `tests/ace/tui/test_current_project_indicator.py`,
  `tests/ace/tui/test_projects_pane_set_current.py`, and
  `tests/test_models_panel_leader_mode.py` so polling assertions target the source and
  rendering assertions target the views.
- Verify no golden changes: `just fix-tui-screenshots --check` passes unchanged, then
  `just check`.

### Phase 2: Labeled launch-context cluster on every tab's status row

Build the `LaunchContextBar` cluster described under **Design**, mount one per tab on
the status row, remove the two chips from the top bar, write the tooltips, and refresh
goldens and docs.

- Add `src/sase/ace/tui/widgets/launch_context_bar.py`: `LaunchContextBar(Horizontal)`
  composing the model label (`default`/`override`), an `LLMOverrideIndicator` view, the
  `·` separator, the `current` label, and a `CurrentProjectIndicator` view; implement
  the full/compact densities, the empty-project collapse, the label tooltips, and the
  pure `choose_launch_context_density` helper. Export it from
  `src/sase/ace/tui/widgets/__init__.py` / `__init__.pyi`.
- Rewrite both view tooltips to the texts under **Tooltips** (verifying directive names
  first); the override tooltip helpers live in
  `src/sase/ace/tui/widgets/_override_pill.py`.
- Mount the cluster on the three hosts exactly as described under **Placement per tab**
  (`src/sase/ace/tui/_app_layout.py`, `src/sase/ace/tui/widgets/artifacts/view.py`,
  `src/sase/ace/tui/styles.tcss`). Each host computes free cells from its own primary
  content width on resize and whenever its content or the cluster's content changes, and
  sets the cluster density only when it changes (no layout thrash).
- Remove `LLMOverrideIndicator` and `CurrentProjectIndicator` from `#top-bar`; update
  `EXPECTED_TOP_BAR_ORDER` and its comment in `tests/ace/tui/test_top_bar_order.py` (the
  gold/violet pairing note no longer applies) and add a test pinning the cluster's
  presence and child order at the far right of each tab's status row.
- Unit tests: density helper boundaries; empty-project collapse (no dangling `·`);
  override label switch; tooltip text for calm, round-robin, override, resolving,
  failed, and each project origin.
- Visual goldens: repoint
  `tests/ace/tui/visual/test_ace_png_snapshots_launch_default_pill.py` and
  `test_ace_png_snapshots_current_project_indicator.py` at the new location, and add
  focused snapshots for the Agents, Artifacts, and Services rows at 120x40 plus an
  override-state and a compact 80x24 case. Visually inspect only these new/targeted
  PNGs; then run `just fix-tui-screenshots` to refresh the many incidental golden
  changes (these do not need individual review).
- Docs: update `docs/ace.md` wherever it places the model pill or `+<project>` chip in
  the top bar (e.g. the "top-bar `+<project>` chip" wording and the Current project
  section), and the `ace.current_project.indicator` comment in
  `src/sase/default_config.yml`.
- Finish with `just check`.
