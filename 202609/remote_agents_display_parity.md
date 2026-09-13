---
tier: epic
title: Remote agents render as real agent nodes
goal: 'Remote agents in the ACE Agents tab are displayed identically to local agents
  — same family/clan nodes, member shells, counts, timestamps, and project names —
  except that remote agent nodes carry their host''s name, and local nodes never carry
  a `here` indicator.

  '
parent_bead: sase-xe.16.11.7
phases:
- id: no-here-chip
  title: Host chips on remote nodes, never a here chip
  depends_on: []
  size: small
  description: 'no-here-chip: invert the machine-chip policy — delete the here fallback
    so local rows never carry a chip, render the host alias chip on remote agent nodes
    in every grouping mode, and update render-cache keys, tests, and fleet PNG snapshots.'
- id: remote-node-synthesis
  title: Remote rows become family and clan nodes
  depends_on: []
  size: large
  description: 'remote-node-synthesis: consume row_kind/family_role/current_instance/container_projected_concrete_agent,
    link members under family containers via host-qualified parent lineage, dedupe
    container and non-current instances, route remote rows through the shared node-synthesis
    and clan-projection pipeline, and fix the reproject/refilter fold asymmetry.'
- id: wire-parity-fields
  title: Core wire carries the missing presentation facts
  depends_on: []
  size: large
  description: 'wire-parity-fields: renderer-driven audit of Agent fields versus wire
    sources, then extend ResolvedAgentSummaryWire and the owner projection with start/stop
    timestamps, workspace number, clan/tribe identity, and a human project label;
    bump contract schema, bindings, and fixtures with full core checks.'
- id: published-core-adoption
  title: Publish, ratchet, and verify the new core surface
  depends_on:
  - wire-parity-fields
  size: small
  description: 'published-core-adoption: wait for release-plz to publish the new core
    surface, ratchet the pin and dependency floor with supported tools, just install,
    and verify the installed wheel exposes the new fields before any consumer lands.'
- id: remote-render-integration
  title: Viewer consumes the new facts and drops noisy chrome
  depends_on:
  - remote-node-synthesis
  - published-core-adoption
  size: medium
  description: 'remote-render-integration: map timestamps, workspace numbers, clan
    identity, and human project labels into remote rows; suppress online/aging chrome
    on healthy rows and align viewer freshness thresholds with the worker poll cadence;
    keep render-cache keys honest.'
- id: parity-proof
  title: Mechanical local-versus-remote parity proof
  depends_on:
  - no-here-chip
  - remote-render-integration
  size: medium
  description: 'parity-proof: same-body-of-work equality regression rendering one
    population through the local pipeline and through serialized wire payloads, asserting
    identical rows modulo the host chip; refreshed PNG snapshots and fleet navigation
    benches within budget.'
- id: live-acceptance
  title: Live Athena-to-Apollo before/after acceptance
  depends_on:
  - parity-proof
  size: medium
  description: 'live-acceptance: reproduce the original defect scenario live from
    Athena viewing Apollo on builds with every prior phase, capture pane evidence
    of family/clan-grouped remote nodes with host chips and honest chrome, and leave
    the phase open on any unmet gate.'
proposed_by: bbugyi200.apollo.v
create_time: 2026-09-13 18:37:54
status: wip
bead_id: sase-xe.16.11.7.15
---

- **PROMPT:** [prompts/202609/remote_agents_display_parity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_agents_display_parity.md)
- **BEAD:** [sase-xe.16.11.7.15](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.15.md)

# Remote agents render as real agent nodes — display parity across machines

## Problem

With machines enrolled, the unified Agents list renders remote agents as flat
shell-level rows while local agents render as rich agent nodes. Live evidence (Athena
viewing Apollo, SASE v0.17.1+565, screenshot retained on Apollo at
`~/tmp/screenshots/20260913_180154.png`):

- Every remote row carries a literal `[agent]` bracket prefix; local family and clan
  nodes never show it.
- Remote rows are agent shells (`u--plan`, `sase-zr.1--mon`), not agent nodes: no family
  containers, no `×N` collapsed member counts, no clan containers or tribe chips, no
  expandable member shells.
- One logical agent renders multiple duplicate rows (two identical `sase-zr.1` rows;
  nine rows under a "6 agents" Waiting banner).
- The project column shows the raw project id `gh_sase-org__sase` instead of the human
  name `sase`.
- Rows show no start/stop timestamps or durations (a running agent shows `0s`), and
  every row repeats `online · aging` chrome that the design vision says healthy rows
  must not carry.
- Local rows in mixed lists carry a `here` machine chip.

User direction (2026-09-13, binding for this epic): remote agents must be displayed
**exactly** as local agents are, except that the owning host's name is shown in front of
the agent nodes (agent, family, and clan nodes) for remote agents; and the `here`
indicator must **never** be shown in front of local agent nodes. This deliberately
revises the design-vision line in `plan:202609/unified_agents_across_machines.md` that
called for machine chips "on every row of a mixed list" including `here` — local rows
now never carry a machine chip, and remote agent nodes always carry their host alias
chip in every grouping mode, including under `BY_MACHINE` group headers.

## Root causes (audited 2026-09-13 on master 52c80c9528, core master 3fa0a54)

There is one shared renderer for local and remote rows; the divergence is upstream of
it:

1. **Remote rows skip node synthesis.** Local agents pass through
   `normalize_loaded_agents` (`src/sase/ace/tui/models/_agent_loader_normalization.py`),
   whose `apply_status_overrides` populates `followup_agents` — the linkage that makes
   `Agent.is_family_container_row` true — and through `project_clan_tree`
   (`src/sase/ace/tui/models/_agent_tree.py`). Remote rows are built by
   `rows_from_response`/`_agent_from_summary`
   (`src/sase/ace/tui/models/_fleet_agents_rows.py`) and concatenated flat after that
   pass by `_agents_source_for_current_mode`
   (`src/sase/ace/tui/actions/agents/_fleet_projection.py`). With no container flags,
   the generic unknown-type badge branch in `append_agent_row_prefix`
   (`src/sase/ace/tui/widgets/_agent_list_render_agent_prefix.py`) renders the `[agent]`
   bracket, and no family/clan chrome ever applies.
2. **Structural wire fields are ignored.** The wire already serves `row_kind`
   (`agent_shell`/`container_header`/`historical_shell`/monitor/gate/ proc),
   `family_role` (`root`/`member`/`monitor`/`gate`/`proc`/ `historical_shell`),
   `parent_timestamp`, `current_instance`, and `container_projected_concrete_agent`.
   `_agent_from_summary` maps lineage fields onto the `Agent` but nothing consumes
   `current_instance` or `container_projected_concrete_agent` anywhere in `src/` — so
   container projections and non-current instances render as duplicate independent rows.
3. **The fleet reproject path diverges from the filter path.**
   `_reproject_agents_from_current_mode` (`_fleet_projection.py`) rebuilds the list
   without the clan projection that `_refilter_agents`
   (`src/sase/ace/tui/actions/agents/_loading_filter.py`) applies.
4. **The wire lacks presentation facts the local renderer consumes.**
   `ResolvedAgentSummaryWire` (sase-core `crates/sase_core/src/fleet_contract.rs`) has
   `observed_at_unix` but no per-row `started_at_unix`/`stopped_at_unix` (the viewer
   falls back to observation time, so durations are wrong), no workspace number, and no
   clan/tribe identity. `labels.project_label` is projected from the record's raw
   `project_name`, so the owner serves `gh_sase-org__sase` instead of its human project
   name `sase`.
5. **Chip policy.** `_append_machine_chip` falls back to `here` for local rows, and
   `_agent_row_chrome_mode` (`src/sase/ace/tui/widgets/_agent_list_build_rebuild.py`)
   enables chips on **all** rows of a mixed list while suppressing them entirely under
   `BY_MACHINE` — the exact inverse of the user's direction.
6. **Freshness chrome.** `_append_fleet_summary`
   (`src/sase/ace/tui/widgets/_agent_list_render_agent.py`) renders `online`/`offline`
   and freshness on every remote row, and the viewer-side cache classifier
   (`_FLEET_VIEWER_FRESH_SECONDS = 5.0` in `_fleet_agents_rows.py`) is stricter than the
   federation worker's poll cadence, so steady-state rows are perpetually `aging`.

## Scope and relationship to in-flight work

This epic is presentation parity for the Agents tab only. It does not own
launch/approval repair, snapshot correctness, dismissal propagation, or the
release-cohort acceptance that in-flight epic `sase-xe.16.11.7.14.6.7` and open phases
`sase-xe.16.11.7.13` / `sase-xe.16.11.7.14.6.7.4–.6` carry; workers must not close or
absorb those beads. Where both need a core pin/floor ratchet, coordinate on one ratchet
rather than racing. Action-vocabulary parity landed with the unified-actions phase of
`sase-xe.16.11.7` and is out of scope here except where a display fix touches it.

Baseline honesty: before changing anything, each Python phase records the current
failing-node baseline for the fleet/agents test files it touches (after `just install`;
a stale `sase_core_rs` extension fabricates dozens of `fleet_*` binding
AttributeErrors). Pre-existing failures are compared, never silently absorbed or "fixed"
by loosening assertions.

## Constraints (binding for every phase)

- Open sase-core and any other repository with `/sase_repo`; read plans and research
  with `sase artifact read`; read reference memory with `/sase_memory_read`:
  `tui_perf.md` before any TUI phase, `lint_and_test.md` before finishing any sase
  change, `symvision.md` for lint failures, `sase_flags.md` before touching any flag.
- Shared policy stays in Rust core per the rust_core_backend_boundary memory: which rows
  are containers, members, current instances, or historical shells is the core
  contract's decision; Python consumes it through bindings and must not fork a second
  membership/dedupe policy. Textual presentation, row construction, and folding stay in
  Python.
- No new feature flags: this is a repair of the already-unconditional unified surface,
  landing whole per phase. If a phase worker believes a flag is unavoidable, read
  `sase_flags.md` first and record the reasoning.
- Preserve laziness and performance budgets: zero enrolled machines do no remote work;
  hidden surfaces stay lazy; keystroke paths stay read-only; j/k p95 stays under 16 ms
  on every tab; route refreshes through the existing fast path and selective-update APIs
  (`patch_row`/`try_remove_rows`); keep the render-cache key honest for every field that
  can change visible state.
- Remote rows never pass through local PID checks, filesystem hydration, or cleanup side
  effects; mutating local actions keep treating them as read-only.
- Reuse the split fixtures (`tests/ace/tui/_fleet_summary_fixture.py`,
  `_fleet_response_fixture.py`, `_fleet_locator_fixture.py`, `_fleet_facade_fixture.py`)
  and serialized real wire shapes; do not reintroduce invented envelope schemas.
- Core changes run the complete core `just check`/`scripts/check.sh` including PyO3;
  SASE changes run `just check`; long commands go through `/sase_monitor`. Release-plz
  owns core versions — never hand-pin an unpublished version; ratchet with the supported
  tools and verify the published wheel contains the new surface before consuming it.
- Any changed named control updates `src/sase/default_config.yml`, the keybinding
  footer, and the help modal together.
- Phase workers never create beads; record discoveries as `PROPOSED FOLLOW-UP:` notes on
  their own phase bead. A commit's automatic close is not evidence; leave genuinely
  unmet work open.

## Phases

### no-here-chip

size: small dependencies: none

Invert the machine-chip policy to the user's direction:

- `_append_machine_chip` (`src/sase/ace/tui/widgets/_agent_list_render_agent_prefix.py`)
  renders nothing for rows without `fleet_origin_alias` — delete the `here` fallback.
  Local rows never carry a machine chip in any grouping mode.
- `_agent_row_chrome_mode` (`src/sase/ace/tui/widgets/_agent_list_build_rebuild.py`)
  stops special-casing `BY_MACHINE`: chips are enabled whenever any loaded row has a
  fleet origin, and render only on rows that have one. Remote agent nodes (standalone
  agent rows, family containers, clan containers) always show their host alias chip,
  including under an unambiguous `BY_MACHINE` machine header. Member shell rows indented
  under an already-chipped parent node do not repeat the chip.
- `BY_MACHINE` group banners are chrome, not nodes: the local group header label
  (`_machine_name` in `src/sase/ace/tui/models/agent_groups/_keys.py`) and the
  `machine:here` query vocabulary are unchanged.
- If the `show_fleet_badge`/`_append_fleet_badge` star path is confirmed unreachable
  (its only producer already returns False), delete it in this phase rather than leaving
  a dead branch.

Update the render-cache key tests (`tests/ace/tui/widgets/test_agent_render_key_core.py`
machine-chip cases), status-indicator/grouping/gutter tests, and the fleet PNG snapshots
(`tests/ace/tui/visual/test_ace_png_snapshots_agents_fleet.py`) so a local row with a
chip is asserted to be impossible.

### remote-node-synthesis

size: large dependencies: none

Make remote rows real agent nodes using fields already on the wire, in the shared
pipeline:

- Consume `row_kind`, `family_role`, `current_instance`, and
  `container_projected_concrete_agent` when projecting summaries
  (`src/sase/ace/tui/models/_fleet_agents_rows.py`): one logical agent yields one agent
  node; member/monitor/gate/proc and historical shells become member rows of their
  family node via `followup_agents`/`runtime_children` linkage built from the
  host-qualified `parent_timestamp` (`_resolve_host_parent_lineage`); container
  projections and non-current instances stop rendering as duplicate top-level rows. This
  fixes the duplicate rows and count/row mismatches in the live evidence.
- Synthesize a family container when members exist but the root row is outside the page,
  following `materialize_imported_family_containers`
  (`src/sase/ace/tui/models/_agent_imported_family.py`) as precedent, with
  origin-qualified identity so selection survives refresh.
- Route host-scoped remote row sets through the same node-synthesis pass local agents
  get (`normalize_loaded_agents` or a shared extraction of it) instead of concatenating
  flat rows in `_agents_source_for_current_mode`
  (`src/sase/ace/tui/actions/agents/_fleet_projection.py`). Local-only steps (PID
  checks, filesystem hydration, status overrides that read local state) must not run
  against remote rows.
- Fix the fold-path asymmetry: `_reproject_agents_from_current_mode` must produce the
  same tree as `_refilter_agents`
  (`src/sase/ace/tui/actions/agents/_loading_filter.py`), including `project_clan_tree`,
  so a fleet refresh and a query refilter render identically.
- Where a membership/promotion decision is not already stated by the wire, consume the
  existing shared promotion bindings
  (`src/sase/ace/tui/models/_fleet_agents_promotion.py` and the core policy it wraps)
  rather than inventing Python-side policy.

Acceptance: with real serialized wire fixtures, a remote family renders as a family
container node (no `[agent]` bracket, `×N` fold annotation, member shells nested with
monitor/gate lane glyphs) exactly as the equivalent local family does; regression tests
cover container-plus-concrete duplicates, historical shells, unresolved-parent pages,
and selection stability across refresh. Extend
`tests/ace/tui/test_fleet_agents_projection.py`, the grouping-mode tree tests
(`tests/ace/tui/models/test_agent_groups_grouping_mode_tree_machine.py`), and add a
remote-container assertion mirroring
`tests/ace/tui/models/test_agent_tree_rendering.py`.

### wire-parity-fields

size: large dependencies: none

In sase-core, close the wire gaps a renderer-driven audit confirms. First enumerate
every `Agent` field the local row renderer consumes
(`_agent_list_render_agent_prefix.py`, `_agent_list_render_agent_status.py`,
`_agent_list_render_agent.py`, `_agent_list_render_layout.py`, plus banner summaries)
and map each to its wire source; record the audit table in the phase note. Known gaps to
close in `ResolvedAgentSummaryWire`, the owner-facts projection, and validation
(`crates/sase_core/src/fleet_contract.rs`):

- Per-row `started_at_unix`/`stopped_at_unix` from the owner record's lifecycle facts,
  so viewers stop fabricating start times from observation time.
- Workspace number.
- Clan identity: clan name, clan generation, and tribe labels, sourced from the owner's
  records with the vocabulary core already owns (`agent_clan_tribe.rs`).
- Human project label: `labels.project_label` must carry the owner's human project
  display name, not duplicate the raw record `project_name`. Investigate where the
  owner's id-to-display-name mapping lives (the project registry the local loaders
  consume) and thread it into the snapshot the gateway builds from `--sase-home`; keep
  the portable project id in the locator so `machine:` and `project:` queries and
  cross-machine project grouping still work.
- Any further facts the audit proves the renderer needs (for example plan-chain/tale and
  auto-approve presentation flags currently read by `_agent_from_summary` but absent
  from the wire). Facts that are strictly owner-local (for example local artifact
  directory paths) are explicitly out of scope — record them as non-goals in the audit
  table.

Bump the contract schema and gateway contract manifest, extend the PyO3 bindings and
their dict-in/dict-out validators, and update core fixtures with real serialized shapes.
Full core `just check`/`scripts/check.sh` including PyO3. Record the exported names and
precise schema in the phase note so the adoption phase consumes a committed surface.

### published-core-adoption

size: small dependencies: wire-parity-fields

Let release-plz publish the core version containing wire-parity-fields; wait through
`/sase_monitor`. Then ratchet the SASE core revision pin and dependency floor with the
supported tools (coordinating with any concurrent cohort ratchet from the in-flight
acceptance epic), run `just install`, and verify the installed wheel actually exposes
the new fields and bindings before any consumer lands. Run the binding/environment
validators (`tools/validate_sase_core_rs*`). Do not hand-pin an unpublished version; do
not characterize a dev build as the published surface.

### remote-render-integration

size: medium dependencies: remote-node-synthesis, published-core-adoption

Consume the new wire facts in the viewer and finish visual parity:

- Map `started_at_unix`/`stopped_at_unix`, workspace number, clan identity, and the
  human project label through `_agent_from_summary` so remote rows get real
  timestamp/duration columns (`build_runtime_suffix` layout), clan containers with tribe
  chips via `project_clan_tree`, and `sase` instead of `gh_sase-org__sase` in the
  project column, with a viewer-side fallback that maps a portable project id through
  the local project registry when the owner label is absent.
- Honest-chrome diet: healthy rows (online, fresh-enough) render no
  `online`/`fresh`/`aging` suffix; only qualified states render (offline, stale,
  `WAS RUNNING · last seen …`), per the design vision. Align the viewer cache-freshness
  thresholds in `_fleet_agents_rows.py` with the federation worker's actual poll cadence
  so steady-state rows are not perpetually `aging`; freshness facts stay available in
  the detail panel.
- Machine group banner counts keep authoritative-count sourcing; banner vocabulary
  matches local banners for the same buckets.
- Keep the render-cache key honest for every newly consumed field.

### parity-proof

size: medium dependencies: no-here-chip, remote-render-integration

Prove "identical except the host chip" mechanically:

- A same-body-of-work equality regression: build one agent population (families with
  monitors/gates, a clan with tribe labels, running/waiting/ done/failed mixes,
  historical shells), render it once through the local loading pipeline and once through
  serialized wire payloads and the remote projection, and assert the rendered row text
  and tree structure are identical modulo the single host-alias chip prefix on remote
  nodes. Anchor it on serialized real wire shapes, not facade shortcuts.
- Refresh PNG snapshots for mixed and `BY_MACHINE` views at 82×28 and wider
  (`tests/ace/tui/visual/test_ace_png_snapshots_agents_fleet.py` and the family/clan
  snapshot suites); inspect intentional golden changes.
- Re-run the fleet navigation benches (`tests/ace/tui/bench_tui_jk_fleet.py`, fault
  benchmarks) and confirm p95 under 16 ms, no stalls, zero-machine and hidden-surface
  laziness intact.

### live-acceptance

size: medium dependencies: parity-proof

Read `tailnet.md` through `/sase_memory_read` first. With both machines on builds
containing every prior phase, reproduce the original defect scenario live from Athena
viewing Apollo (the retained Apollo screenshot `~/tmp/screenshots/20260913_180154.png`
is the "before"): the Apollo machine group renders family- and clan-grouped agent nodes
with host chips, one row per logical agent with expandable member shells, correct human
project names, real timestamps/durations, no `[agent]` brackets, no `online · aging`
chrome on healthy rows, and no `here` chips on local rows in any grouping mode. Capture
`sase ace --tmux` pane evidence on this phase bead. Confine operations to test agents;
do not close `sase-xe.16.11.7.13` or any other epic's bead — cross-reference the
evidence on this phase only. If any item fails, preserve the exact unmet gate and leave
the phase open.

## Verification and landing

Every Python phase: `just check` green (escalating per its rules), with the recorded
pre-existing-failure baseline compared explicitly. Core phase: full core check including
PyO3. Landing runs the combined-tree `just check-full` through `/sase_monitor` with
TESTING/TESTED status, re-runs Symvision after closes, and marks this plan done through
the normal landing flow. The land agent verifies the parity-proof equality test and the
live-acceptance evidence before closing the epic; never force-close a nested bead.
