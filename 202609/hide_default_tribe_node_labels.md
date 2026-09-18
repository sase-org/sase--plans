---
tier: tale
title: Hide the implicit default tribe from Agents nodes
goal:
  Agents nodes never show an inline @default label while named tribes and the reserved
  split-panel identity remain visible where they add information.
size: small
proposed_by: bbugyi200.apollo.0a.f0.f0
create_time: 2026-09-18 03:07:56
status: wip
---

# Hide the implicit default tribe from Agents nodes

## Goal

Never render the synthesized `@default` tribe annotation on an Agents-tab node. The
default tribe remains a real internal/panel identity, and the split layout may still
title its reserved panel `@default`; only inline node annotations are suppressed. In the
merged `All agents` layout, named tribes such as `@research` must remain visible so
users can still distinguish them.

## Root cause and boundaries

Merged-panel rendering calls `effective_tribe_per_agent()` and passes every effective
tribe through `AgentList.tribe_labels`. That helper intentionally materializes the
reserved fallback as the semantic string `default`, so the common row renderer treats it
like any user-managed tribe and appends `@default`. The value is also retained in the
row render context used by selective patches.

Keep the semantic helpers and panel plumbing unchanged: `default` is still needed for
panel membership, effective-clan inheritance, cleanup filtering, display diffs, and the
split-panel title. Enforce the presentation invariant at the shared agent-node renderer,
which covers full list rebuilds, cached renders, and row patches without adding I/O or
new refresh work to the keystroke path.

## Implementation

1. In `src/sase/ace/tui/widgets/_agent_list_render_agent.py`, add one small private
   normalization helper for inline tribe labels. Reuse the existing
   `normalize_panel_key()` semantics so `None`, an empty value, and the reserved
   `default` identity normalize to no inline annotation, while public named tribes are
   preserved.
2. Normalize `tribe_label` before constructing both uncached and cached agent rows. Feed
   the normalized value to `append_agent_row_prefix()`, the clan-container suffix logic,
   and `agent_render_key()`. This prevents `@default` for standalone/family/tree rows
   and clan containers, and makes `default` and `None` share the same cached
   presentation instead of retaining duplicate cache entries for identical output.
3. Do not change `effective_tribe_per_agent()`, panel keys, split-panel titles, tribe
   configuration, cleanup semantics, or named-tribe annotations. In particular,
   `@default` remains correct in a reserved panel title and in non-node surfaces where
   the reserved identity is intentionally discussed.
4. Extend focused renderer tests:
   - In `tests/ace/tui/widgets/test_agent_list_runtime_rendering_layout.py`, cover an
     ordinary row passed `tribe_label="default"` and assert that it omits `@default`,
     while the existing named-label behavior remains intact.
   - In `tests/ace/tui/widgets/test_agent_render_cache_clan.py`, cover a merged clan row
     whose synthesized label is `default`, including explicit/default clan metadata, and
     assert that no `@default` annotation survives while distinct named clan tribes
     still render. Also assert the cached `default` and `None` presentations converge.
   - In `tests/ace/tui/widgets/test_agent_render_cache_patching.py`, retain a
     synthesized default label across a status mutation and selective row patch, and
     assert the patched prompt still omits `@default`.
5. Extend the mounted `oo` regression in `tests/ace/tui/test_agent_grouping_picker.py`:
   after switching a mixed set of implicit-default, explicit-default, and named-tribe
   agents into `All agents`, inspect the rendered `AgentList` prompts. Assert that no
   node contains `@default`, that named labels such as `@alpha`/`@beta` remain present,
   and that toggling back still restores the normal split panel collection. This locks
   the fix to the exact user-visible path from the report.
6. Update the Tribe Side Panels / Grouping Modes prose in `docs/ace.md` to state that
   merged panels annotate only named tribes on nodes; the default tribe is implied and
   therefore omitted from node labels. Preserve the documented `@default` split-panel
   title and reserved-identity behavior.

## Verification

1. Run the focused behavioral suites:

   ```bash
   pytest -q \
     tests/ace/tui/test_agent_grouping_picker.py \
     tests/ace/tui/test_agent_panels_display.py \
     tests/ace/tui/widgets/test_agent_list_runtime_rendering_layout.py \
     tests/ace/tui/widgets/test_agent_render_cache_clan.py \
     tests/ace/tui/widgets/test_agent_render_cache_patching.py
   ```

2. Run `just fix`, then `just check` as the repository-wide required gate. If the
   workspace dependencies are stale, run `just install` before retrying `just check`.
3. Manually inspect the mounted mixed-tribe test state (or an exported SVG from that
   deterministic fixture) to confirm the merged title remains `All agents`, default
   nodes have no inline tribe label, named nodes retain their colored labels, and the
   split layout still titles the reserved panel `@default`. Do not update unrelated PNG
   goldens for ambient renderer/title drift.

## Acceptance criteria

- No Agents-tab node can render an inline `@default` annotation through either a full
  rebuild or a selective patch.
- Named tribe annotations remain visible in the merged panel with their existing
  identity colors and ordering.
- The semantic default identity, explicit `default` assignments, inherited clan/family
  panel membership, `@default` split-panel title, and layout toggle behavior do not
  change.
- Focused tests and `just check` pass.
