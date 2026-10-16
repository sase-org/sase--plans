---
tier: epic
title: Finish the clan waves landing
goal: 'The clan wave feature matches its approved design end to end. sase-core owns
  the whole wave layout through the planned wire, and the Python adapter only consumes
  it. Wave states, the gutter, the WAVES ladder, and the JUMP legend behave as specified.
  The TUI import budget passes again, and the clan-wave glossary term renders correctly.

  '
decisions:
  glossary_clan_wave_strand:
    ask: Fix the clan-wave glossary strand's keyword, stray heading, and layered-clan
      rule?
    memory:
    - glossary:clan-wave
    default: true
    corrects: The Agents tab shows waves only for clans with two or more layers.
    answer: true
phases:
- id: wire
  title: Realign the sase-core clan wave wire and switch the adapter to it
  depends_on: []
  size: medium
  description: 'wire: replace project_clan_waves'' partial wire with the planned key-based
    layout wire in sase-core (allow_threads, serde defaults, the missing tests). Make
    the Python facade and adapter consume it and delete the Python-side rank, layered,
    ghost-split, and ordering derivations, with no visible change.'
- id: states
  title: Wave states, gutter lanes, the WAVES ladder, and the glossary strand
  depends_on:
  - wire
  size: medium
  description: 'states: fold wave state over anchors and loaded members only, fix
    the workflow-step lane, show ladder rows at COLLAPSED, share one after-phrase
    helper between the ladder and roster, fix singular and zero wave wording, prove
    fleet-row repaint, fix the clan-wave strand, and regenerate the affected goldens.'
- id: legend
  title: JUMP legend dividers, legend performance, and the TUI import budget
  depends_on:
  - states
  size: medium
  description: 'legend: scope dividers to layered clan rosters and draw them only
    between same-row neighbors, restore pre-epic legend layout cost, fix the title
    suffix budget, and take the five clan-wave modules off the TUI startup import
    path. Then regenerate and inspect the jump-panel goldens.'
proposed_by: bbugyi200.athena.sase-1lc.land
parent_bead: sase-1lc
decided_by: auto
create_time: 2026-10-15 23:30:14
status: done
bead_id: sase-1lc.8
---

- **PROMPT:** [prompts/202610/clan_waves_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/clan_waves_landing_repairs.md)
- **PARENT:** [202610/clan_waves_in_agents_tab.md](https://github.com/sase-org/sase--plans/blob/main/202610/clan_waves_in_agents_tab.md)
- **BEAD:** [sase-1lc.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lc/sase-1lc.8.md)

# Plan: Finish the clan waves landing

## Background

Epic `sase-1lc` ("Clan waves in the Agents tab", plan
`plan:202610/clan_waves_in_agents_tab.md`) had all seven phases closed. Its land agent
then reviewed the code against that plan's normative **Design** section and found work
that was reported complete but is not. This child epic finishes that work. When it
lands, its land agent resumes and closes `sase-1lc` through the `parent_bead` link.

Every phase must read the parent plan's Design section first. Its wave model, visual
language, surfaces, and invariants still bind every phase here. Use
`sase artifact read plan:202610/clan_waves_in_agents_tab.md "<why>"`.

The landing agent verified these defects. Each is cited by file and line at master
`bfa78040eb`.

**Core wire deviates from the plan (Rust core boundary).** The parent plan's core phase
specified a different wire from the one sase-core commit `ce9779f7` shipped in
`crates/sase_core/src/clan_waves.rs`.

| Wire           | Specified by the plan                                                      | Shipped in `ce9779f7`                                                                           |
| -------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Request        | `clan_names: Vec`; each member has a unique `key`                          | `ClanWaveRequestWire { clan_name: String, ... }`; no `key`                                      |
| Response       | `layered`, counts, `ClanWaveWire` per wave, per-member placements in order | `waves: Vec<Vec<String>>`, `wave_of`, `preds`, `external_waits`, `ghosts`, `unresolved`, `seal` |
| Response serde | defaults on optional and list fields                                       | no serde defaults                                                                               |
| Binding        | runs inside `py.allow_threads`                                             | holds the GIL (`crates/sase_core_py/src/clan_waves/mod.rs`)                                     |

Shipped member identity is `name`, and `retry_of` holds a name.

As a result, Python re-derives core layout logic in
`src/sase/ace/tui/models/_agent_clan_waves.py:426-638`: display rank, the `layered`
flag, ghost-vs-member splitting, wave index, and predecessor ordering. That violates the
Rust core backend boundary rule.

The current wire also has data and correctness problems:

- Ghost names are mixed into `waves` and `preds` lists, and sort first.
- Predecessors are ordered by launch key, not display order. Example: members r, p
  (waits on r), q, and m (waits on p and q). m's predecessors must be `[q, p]`; the wire
  gives `[p, q]`.
- Two units with the same presented name collide. The earlier unit loses its slot and
  sorts last.

Several of the plan's required Rust tests are missing:

- a land that shares its final layer and stays numbered;
- a ghost-only wave 1;
- the equality pin with a land member and a seal assertion;
- `layered == false` for a flat fan-out;
- diamond and asymmetric overlap in `clan_waves.rs`;
- retry ordering with a same-layer member between the root and its retry, and a
  multi-step retry chain;
- a sub-hood (`<clan>.<sub>`) hood wait.

**Wave state fold (`_agent_clan_waves.py:337-376`):**

- It folds over every row of a unit, including session turns and descendants. A Done
  member with a failed earlier session turn therefore paints its wave failed, while the
  clan container (direct members only) shows otherwise.
- A wave whose loaded members are all Done, plus ghosts, returns `IN_FLIGHT` with
  `open=False`. The spec folds state over _loaded_ members only. Once a user dismisses
  one Done member, that wave reads "in flight" forever while its number renders dim.

**Gutter (`src/sase/ace/tui/widgets/_clan_wave_gutter.py:98`):**

- `rows_by_suffix` keeps the last row per `raw_suffix`. Workflow step rows share their
  parent's suffix, so a step child of a layered-clan member resolves to a sibling step
  and gets no lane.
- It renders the old `"  │  "` guide instead of `"   │ "`.
- The model side already prefers the non-child row (`_rows_by_suffix` in
  `models/_agent_tree_clan.py:27`).

**WAVES ladder (`widgets/prompt_panel/_agent_display_clan_waves.py:308`):**

- The default COLLAPSED level returns right after the heading. The spec's Surface 3
  shows the wave rows at COLLAPSED (`1 ◐  .1 ✓  .2 ▶  .3 ▶`). EXPANDED only adds the
  `after` column.
- `tests/ace/tui/widgets/test_summary_fold_contracts.py:373` pins the deviation.

**Two after-phrase implementations disagree:**

- The ladder (`_agent_display_clan_waves.py:150-208`) says `after 4 members` for
  non-contiguous predecessor waves.
- The roster (`_agent_display_clan_roster.py:40-70`) says `after waves 1, 3`, and falls
  back to `after every member` when no predecessor has a wave.

**Wording:**

- A one-numbered-wave epic (one phase plus land) gets the JUMP title suffix
  `1 WAVES + LAND` (`_agent_display_clan_roster.py:~99`).
- A cycle-only clan renders `WAVES · 0 waves ·  · not started` (empty pip segment) and
  the suffix `0 WAVES` (`_agent_display_clan_waves.py:~142`).

**JUMP legend (`src/sase/ace/tui/widgets/_agent_jump_legend.py`):**

- **Dividers leak into other rosters.** `_member_roster.py:272` sets
  `legend_group=entry.group_label` for every roster. The NEIGHBORS map already sets
  group labels (`ancestors`, `descendants`, hoods), so it now draws `┊` dividers. The
  plan says only `clan_roster_entries` sets them.
- **Dividers are grid cells.** They pad to the column width and land at row edges. See
  the committed golden `agents_jump_panel_layered_wave_dividers_80x40.png`: row 1 ends
  with `┊`, and in the 60-column golden row 2 starts with one.
- **Layout cost regressed 4–25×.** For 100 targets with no groups it went from 2.4–6.9
  ms to 58–88 ms. `_collapsed_selection_for_columns` searches `shown` from `count`
  downward inside every column candidate, and `_largest_collapsed_columns` raised its
  column cap from `width // 8` to `width // 3`.
- **The title suffix drops too early.** `agent_jump_panel.py:~90` subtracts the subtitle
  width, but the subtitle is drawn in the bottom border.

**TUI import budget, caused by this feature:**
`tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
fails `3524 < 3520`. `tools/tui_import_closure --diff 91350ad5b6^` shows 3519 before
`sase-1lc` and +5 modules after:

- `sase.ace.tui.models._agent_clan_waves`
- `sase.ace.tui.widgets._clan_wave_gutter`
- `sase.ace.tui.widgets._clan_wave_presentation`
- `sase.ace.tui.widgets.prompt_panel._agent_display_clan_waves`
- `sase.core.agent_clan_waves`

Do not raise `_MAX_MODULE_COUNT`.

**Glossary strand (`sase/memory/glossary/clan-wave.md`):**

- It declares `keyword: clan-wave` (other strands use Title Case, for example
  `Agent Clan`), carries `aliases: []`, and has a body `# Clan Wave` H1.
- The generated term list reads `clan-wave;`, and `docs/reference/glossary.md` renders
  an H1 inside the `## clan-wave` section.
- Its last sentence omits that an unresolved wait cycle also makes a clan layered.

**Known failures that are not this epic's:**

- `just fmt-docs-check` drift (`render_glossary` / `render_design_decisions` vs
  prettier) is tracked on docs epic `sase-1le`.
- So are `test_pdf_check_recipe_keeps_strict_mkdocs_export` and
  `test_rss_plugin_images_use_the_default_social_image`.
- Do not fix these here. Cite `sase-1le` when `sase tool run check` reports them.

## Rules for every phase

- **Before touching code:**
  - Read `sase memory read tui.md -r "<why>"` before any projection, patch, render,
    screenshot, or golden work.
  - Read `sase memory read lint_and_test.md -r "<why>"` before finishing.
- **Invariants:** keep the parent plan's invariants. No file I/O on render or projection
  paths; members are never hidden; selection is keyed by identity; waves are not sase
  nodes; every cache key covering wave chrome includes the wave inputs it draws.
- **No new modules on the TUI startup path:** add no new module to the
  `sase.ace.tui.app` import closure. Prefer extending existing clan-wave modules over
  creating new ones.
- **Verification:**
  - Run `just fix`, then `sase tool run check`. Never run `just check-full`.
  - For visual work, run targeted `just fix-tui-screenshots -- <selectors>` through
    `/sase_monitor`, then inspect the retained report and every changed golden.
  - Prove host-independence with two `--check` runs, one under `TZ=UTC`.
- **Follow-ups:** record discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on
  your phase bead. Never create beads.

## Phase: wire

This phase changes both the linked `sase-core` checkout
(`sase repo open sase-core -r "<why>"`; read its `AGENTS.md`) and sase in one turn. The
host commits the sase-core sibling first and writes its pushed SHA into
`sase-core-revision.txt`. Do not hand-edit the pin.

1. **Wire types** (`crates/sase_core/src/clan_waves.rs`): replace the current request
   and response with the parent plan's core wire, implementing every Wave model rule as
   today.
   - **Request:**
     `ClanWavesRequestWire { clan_names: Vec<String>, members: Vec<ClanWaveMemberWire> }`.
     Clan-name waits match any spelling in `clan_names`. Hood containment is checked
     against every spelling.
   - **Member:**
     `ClanWaveMemberWire { key, name, aliases, launch_key, retry_of: Option<key>, wait_for, wait_for_beads, wait_for_hoods, phase_bead_id: Option<String>, seal: bool }`.
     - `key` is the unique member identity.
     - A wait naming a `name` or alias that several members share resolves to every such
       member.
     - No member is ever dropped.
     - `retry_of` names the chain root by `key`.
   - **Response:**
     `ClanWavesWire { layered, numbered_wave_count, has_seal, waves: Vec<ClanWaveWire>, members: Vec<ClanWaveMemberPlacementWire>, unresolved: Vec<key> }`.
     - `waves` lists numbered layers in order, including ghost-only layers, followed by
       the seal layer when present.
     - `members` lists every loaded member in final display order.
     - `layered` is true when there are at least two layers (ghost-only and seal layers
       included) or any unresolved member.
   - **Wave:**
     `ClanWaveWire { index, number: Option<u32>, seal: bool, member_keys: Vec<key>, ghost_names: Vec<String> }`.
     - `index` is the 1-based layer position.
     - `number` is `None` only for the seal.
     - Ghosts appear only in `ghost_names`, sorted, never in `member_keys`.
   - **Placement:**
     `ClanWaveMemberPlacementWire { key, wave_index: Option<u32>, rank: u32, predecessor_keys, ghost_predecessors: Vec<String>, external_wait_count: u32 }`.
     - `wave_index` is `None` only for unresolved members.
     - `rank` is the member's position in `members`.
     - `predecessor_keys` is in display order.
     - `ghost_predecessors` is sorted.
   - **Serde:** put `#[serde(default)]` on every optional and list field of all wires,
     matching `agent_clan_tribe.rs`.
   - **Kernel:** keep the shared `dependency_waves.rs` kernel and the `bead/work.rs`
     caller unchanged.
2. **Binding** (`crates/sase_core_py/src/clan_waves/mod.rs`): keep the name
   `sase_core_rs.project_clan_waves(request: dict) -> dict`. Run the projection inside
   `py.allow_threads`, like `agent_scan/mod.rs:37`. Update the error text and the
   binding round-trip test. Extend `crates/sase_core/tests/python_wire_parity.rs` if it
   covers comparable wires.
3. **Rust tests**: keep the existing shape tests passing on the new wire, and add every
   missing case listed in Background:
   - a land sharing its final layer (stays numbered, `has_seal == false`);
   - ghost-only wave 1 with honest numbering;
   - the equality pin with a land member: numbered waves equal `plan.waves`, and the
     land is the seal;
   - flat fan-out with `layered == false`;
   - diamond and asymmetric overlap;
   - retry ordering with an interleaved same-layer member, plus a two-step retry chain;
   - a sub-hood wait;
   - duplicate names keep both members;
   - predecessor display order (the r/p/q/m example gives `[q, p]`);
   - a cycle-only clan (`layered`, zero numbered waves).
4. **Python facade** (`src/sase/core/agent_clan_waves.py`):
   - Mirror the new wire in frozen dataclasses, keeping the cached request hash and the
     `lru_cache` memo.
   - Call `require_rust_binding`, with no fallback.
   - Parse defensively, as today.
5. **Adapter** (`src/sase/ace/tui/models/_agent_clan_waves.py`):
   - **Build members:**
     - `key` is `clan_wave_identity_key(anchor)`.
     - `aliases` keeps every raw and presented spelling.
     - `clan_names` holds both `agent_clan` and `presented_clan_reference_name()`. Drop
       `_canonical_clan_name` folding wherever core now handles it.
     - `retry_of` is the root unit's key.
   - **Read layout from the wire:** take rank, display order, wave membership, ghost
     names, predecessors, the `layered` flag, counts, and seal straight from the wire.
     Delete the Python re-derivations, including `rank_by_name`, the `layered` formula,
     the ghost filters, and the predecessor re-sorting.
   - **Keep the fast path:** the dependency-free `_simple_*` path stays, built on the
     new wire.
   - **Keep the presentation types:** `ClanWaveSlot`, `ClanWaveSummary`, and the view
     types keep their fields and meaning. `ClanWaveSlot.wave_index` keeps today's
     values, so the patch guard and gutter grouping do not change.
   - **Leave the state fold alone:** the `states` phase changes it.
6. **No visible change:**
   - Existing adapter, gutter, ladder, roster, jump-legend, and diff tests pass
     unchanged. Add an adapter test for the duplicate-name case.
   - The existing clan, clan-panel, jump-panel, and tribe goldens pass a targeted
     `--check` run unchanged.
   - The warm-memo clan benchmark in `tests/perf/bench_tui_trace.py` stays within noise
     of the current tree.
7. **Verify:**
   - `sase tool run check` inside the sase-core checkout.
   - `just fix`, then `sase tool run check` in sase.
   - Confirm the TUI import closure did not grow:
     `.venv/bin/python tools/tui_import_closure --diff HEAD`.

## Phase: states

1. **State fold** (`_agent_clan_waves.py::_wave_state`):
   - Fold each wave's state over its members' anchor rows only (each unit's direct
     member row), never over session turns or other descendants. This matches the clan
     container's direct-member aggregate.
   - When every loaded member is Done, the state is `done` with `open=False`, even if
     the wave also has ghosts.
   - A ghost-only wave stays `dismissed only`.
   - Keep the rest of the state table and the retried-failure rule.
   - Tests:
     - a Done anchor with a failed earlier session turn is not `failed`;
     - Done members plus a ghost give `●` and a dim label;
     - a ghost-only wave gives `·`;
     - every other state-table row still holds.
2. **Workflow-step lane** (`_clan_wave_gutter.py:98`): build the suffix map with the
   prefer-non-child rule. Share or reuse `_rows_by_suffix` from
   `models/_agent_tree_clan.py` without adding a module. Test: a workflow member with
   step children, plus a later same-wave peer, renders `"   │ "` lanes on the step rows.
3. **Ladder levels** (`_agent_display_clan_waves.py`):
   - **COLLAPSED (the default):** the heading plus one row per wave (label, state pip,
     member cells), with no `after` column.
   - **EXPANDED:** adds the aligned `after` column.
   - **FULLY_EXPANDED:** adds the per-member `waits on` lines.
   - Update `test_summary_fold_contracts.py` so COLLAPSED asserts the rows and adjacent
     levels still differ.
4. **One after-phrase:**
   - Replace the ladder's `_after_summary` predecessor logic and the roster's
     `_predecessor_summary` with one shared helper in an existing module, so the ladder
     row and the roster group label say the same thing for the same wave.
   - Rules:
     - no predecessors gives `no dependencies`;
     - up to three gives `after .a .b .c`;
     - more than three gives `after wave k` (one wave), `after waves a–b` (contiguous),
       or `after waves a, c` (non-contiguous);
     - the seal gives `after every phase` for an epic land and `after every member`
       otherwise;
     - the unresolved group gives `wait cycle` and never lists its own members.
   - Roster labels stay `wave N · <phrase>`, `land · after every phase`, and
     `unresolved · wait cycle`.
5. **Wording:**
   - A single numbered wave reads `1 wave` in the ladder and `1 WAVE` / `1 WAVE + LAND`
     in the JUMP title suffix.
   - A clan with zero numbered waves (cycle only) shows no `0 waves` count, no empty pip
     segment (no `·  ·`), and no title suffix. Its heading reads
     `WAVES · wait cycle · <phrase>`.
   - Cover both cases in tests.
6. **Fleet-row repaint (test first):**
   - Add an incremental-patch test for a mixed local+fleet layered clan. The wave's
     label row is an unchanged fleet row, and a local member's completion finishes the
     wave. The label number must repaint dim through the in-place patch path, with no
     list rebuild.
   - If it fails (fleet rows are reused and mutated in place, so `previous != next`
     misses the slot change), make projection publish fresh row objects for reused rows
     whose `clan_wave_slot` changed. Do not fall back to a rebuild.
7. **Glossary strand** (decision below), then `sase memory init` and `just fix`:

   > [!decision] glossary_clan_wave_strand In `sase/memory/glossary/clan-wave.md`, set
   > `keyword: Clan Wave`, remove `aliases: []` and the body `# Clan Wave` heading, and
   > change the last sentence to: "The Agents tab shows waves only for layered clans:
   > clans with two or more layers, or with any member caught in a wait cycle."
   > `just fix` regenerates `docs/reference/glossary.md`; the generated term list must
   > read `Clan Wave`.

   > [!decision] glossary_clan_wave_strand = no Leave the strand unchanged and record a
   > `PROPOSED FOLLOW-UP:` note on this phase bead instead.

8. **Docs:** update the WAVES ladder passage in `docs/ace.md` for the fold levels and
   the shared after-phrases, staying within `docs/_meta/budgets.yml` (`just fix` runs
   `fix-docs-budgets`).
9. **Goldens:**
   - Regenerate the clan-panel ladder goldens (all three fold levels), the clan and
     tribe goldens whose pips or labels change, and any jump-panel golden whose Summary
     body now shows ladder rows.
   - Inspect every change.
   - Run the clan j/k bench:
     `pytest -s -m slow tests/ace/tui/bench_tui_jk_agents.py -k clan`. Next/prev p95
     must stay under 16 ms.

## Phase: legend

1. **Scope dividers to layered clans:**
   - Add a dedicated wave-group field (for example `MemberRosterEntry.legend_group`)
     that only `clan_roster_entries` sets, and only for layered clans.
   - `_member_roster.py` must copy that field, not `group_label`, into
     `_MemberJumpTarget.legend_group`.
   - Test: the NEIGHBORS, session, and tribe maps render no `┊`, even though they set
     group labels.
2. **Divider placement** (`_agent_jump_legend.py`):
   - Draw `┊` only between two shown cells that sit next to each other on the same
     rendered row and differ in group. It is never first or last on a row, and never a
     padded grid column.
   - For example, after choosing the column count, fold the divider into the left cell's
     trailing separator. Fitting (`_largest_collapsed_columns`, `_grid_fits`) must use
     the widths actually drawn.
   - Keep the in-flight-first selection, the always-shown selection, and the `+N ▶M`
     overflow chip.
   - Test: overflow, row wrap at 24, 60, and 80 cells, and no divider at a row edge.
3. **Restore layout cost:**
   - With no legend groups, layout must equal the pre-epic algorithm: the `width // 8`
     column cap and the closed-form shown count. Prove output equality across a width ×
     count sweep against today's no-group output.
   - With groups, start the shown search at `min(count, 2 * columns)` and bound it by
     the divider count.
   - Add a deterministic test that a no-group map calls `_collapsed_selection` at most
     once per column candidate.
   - Record before/after timings for 30 and 100 targets at widths 80, 120, and 200 in
     the phase note. No-group timings must be within noise of the pre-epic numbers
     (2.4/3.5/6.9 ms at 100 targets).
4. **Title suffix budget** (`agent_jump_panel.py`): budget the border title against the
   panel width minus the border chrome only. The subtitle is in the bottom border. Test:
   the suffix is kept at a width where only the old subtraction dropped it, and is still
   dropped first when the title genuinely does not fit.
5. **Import budget:**
   - Make the five clan-wave modules listed in Background load on first use (function-
     level or `TYPE_CHECKING`-only imports) instead of at `sase.ace.tui.app` import
     time.
   - A deferred import on a hot path must cost only a `sys.modules` hit after first use.
   - `tools/tui_import_closure --diff 91350ad5b6^` must list none of them, and
     `test_tui_app_import_stays_under_startup_budget` must pass without changing
     `_MAX_MODULE_COUNT` or the deferred-module list.
   - If unrelated growth that landed later still trips the cap, leave it and record that
     in a note citing `sase-1ic`.
6. **Docs:** update the JUMP legend passage in `docs/ace.md` if divider behavior
   changes.
7. **Goldens and sweep:**
   - Regenerate `agents_jump_panel_layered_wave_dividers_{60,80,120}x40` and any other
     jump-panel golden that changes. Inspect each: no row-edge dividers and no wide
     gaps.
   - Run two targeted `--check` runs (one under `TZ=UTC`) over every clan, clan-panel,
     tribe, and jump-panel selector, and rerun the clan j/k bench.
   - `sase tool run check` must show no failures except the `sase-1le`-tracked docs
     failures named in Background.
