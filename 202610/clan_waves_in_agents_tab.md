---
tier: epic
title: Clan waves in the Agents tab
goal: 'Every agent clan with two or more dependency layers (epics, research swarms,
  toobig chains, custom clans) reads top to bottom in execution order. The Agents
  tab derives clan waves from declared waits in sase-core and shows them through a
  zero-row wave gutter, per-wave progress pips, a live WAVES ladder, and a wave-divided
  JUMP legend. One-wave clans look exactly as they do today.

  '
decisions:
  glossary_agent_clan:
    ask: Correct the glossary's Agent Clan definition, which says clan members run
      in parallel?
    memory:
    - glossary:agent-clan
    default: true
    corrects: An agent clan is a named, rootless container for agents that run in
      parallel.
    answer: true
  glossary_clan_wave:
    ask: Add a Clan Wave glossary term for the derived dependency layers the TUI now
      shows?
    memory:
    - glossary:clan-wave
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: core
  title: sase-core clan wave layering and binding
  depends_on: []
  size: medium
  description: 'core: extract a total dependency-layering kernel from the epic work
    planner and add project_clan_waves with its edge rules, wire types, sase_core_rs
    binding, and Rust tests on real clan shapes.'
- id: model
  title: Python facade and clan wave model adapter
  depends_on:
  - core
  size: medium
  description: 'model: ratchet the core pin, add the Python facade, and attach memoized
    wave layouts plus folded wave states to clan containers and member rows in project_clan_tree,
    with no visible change yet.'
- id: list
  title: Wave order and wave gutter in the agent list
  depends_on:
  - model
  size: medium
  description: 'list: order layered clans by (wave, launch), draw the five-cell wave
    gutter and lanes, switch the clan patch guard and roster numbering to wave order,
    and update docs, help, and goldens.'
- id: pips
  title: Wave pips on clan rows, tribe rosters, and the clan header
  depends_on:
  - list
  size: small
  description: 'pips: add the shared wave-pip renderer with its 12-wave overflow form
    and show it on clan rows, tribe Summary clan units, and the compact clan identity
    header.'
- id: ladder
  title: Live WAVES ladder in the clan Summary card
  depends_on:
  - pips
  size: medium
  description: 'ladder: add the foldable in-memory WAVES section at the top of the
    clan Summary card, sharing the JUMP legend''s cell builder, with fold-level detail
    and hint-cache coverage.'
- id: jump
  title: Wave-divided JUMP legend and roster group labels
  depends_on:
  - ladder
  size: medium
  description: 'jump: draw dim wave dividers and a waves title suffix in the collapsed
    JUMP legend and label wave groups in the expanded roster, keeping the packer''s
    fit and overflow rules.'
- id: verify
  title: Live verification, golden sweep, and glossary
  depends_on:
  - jump
  size: small
  description: 'verify: review live screenshots of real clans, run the full visual
    golden sweep and perf checks, fix polish issues, apply the glossary decisions,
    and record v2 follow-ups.'
proposed_by: bbugyi200.athena.research.4j.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-15 15:02:45
status: done
bead_id: sase-1lc
---

- **PROMPT:** [prompts/202610/clan_waves_in_agents_tab.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/clan_waves_in_agents_tab.md)
- **BEAD:** [sase-1lc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lc/README.md)

# Plan: Clan waves in the Agents tab

## Background

The research report
`research:202610/clan_waves_in_agents_tab/clan_waves_in_agents_tab.md` (read with
`sase artifact read`) is the design source, and the user accepted all of its
requirements. Its final mockup is `file:default:373c1eccd1612db7b8aa60fb`. This plan
turns that report into a buildable epic and pins down details the report left open.

What is true today:

- Direct clan members are stable-sorted by status priority (Failed, Stopped,
  Running/Starting, Queued, Waiting, Done) in
  `src/sase/ace/tui/models/_agent_tree_clan.py::_sort_clan_member_units`. As a result,
  an epic's waiting block reads in reverse (`.land, .7, .6, .5, .4`).
- Waves are visible only as a frozen `Counts: 7 phases · 5 waves` line inside an epic's
  launch-time `clan_summary` markup.
- 87% of the 1,241 clan generations recorded this year have two or more dependency
  layers: 99% of epics and 97% of research swarms.
- Declared waits are durable and already loaded for every row (`Agent.waiting_for`,
  `waiting_for_beads`, `waiting_for_hoods`, from `agent_meta.json`). `waiting.json`
  rewrites them only for active rows, with the same declared lists. Deriving waves
  therefore needs no new persistence and no render-path I/O.
- A member status-bucket change cannot be patched in place today. It falls back with
  `fallback_reason="clan_member_order_change"`
  (`actions/agents/_display_panel_patches.py::_clan_row_patch_preserves_order`), and in
  STANDARD/BY_MACHINE that becomes a full list rebuild.

## Design

These sections are normative for every phase. A phase implements its slice and must not
re-decide anything defined here.

### Wave model (owned by sase-core)

> A **clan wave** is a dependency layer of one clan generation's direct members. A
> member is in wave 1 if none of its declared waits resolves to another member of the
> same generation. Otherwise it is in 1 + the highest wave among the members it waits
> on.

- **Derived, never authored.** There is no `%wave` directive and no stored wave index.
  It works retroactively on every recorded clan.
- **A wave is a layer, not a barrier.** `.4` can start while `.3` (wave 1) is still
  running. There is never a single "current wave".
- **Scope.** Each clan generation is scoped separately, which `project_clan_tree`
  already does. A recreated clan gets fresh waves.

Edge resolution, where `p → m` means "m waits on p". A member's waits are the union over
every row in its unit: the member plus its session turns and other descendants.

- A `wait_for` name equal to a member's name or alias resolves to that member. This
  covers session turns and every spelling of the name.
- A `wait_for` name ending in a session-turn suffix `x--<suffix>` that does not match
  directly is retried as `x`.
- A `wait_for` name equal to the clan name, and every `wait_for_hoods` entry `h` that is
  the clan or lies inside its hood, gets launch-snapshot semantics. It resolves to every
  member whose name is `h` or starts with `h.` and whose launch key sorts before m's.
- A `wait_for` name inside the clan hood (`<clan>.<suffix>`) that matches no loaded
  member becomes a **ghost**: an unloaded, dismissed member.
  - Ghosts have no known waits, so they always sit in wave 1.
  - They render no row.
- `wait_for_beads: b` resolves to the member whose `phase_bead_id == b`. This repairs
  re-worked epics such as `sase-z3`.
- A retry shares its chain root's slot (`retry_of`): same wave, ordered right after the
  root. Waits naming the retry resolve to that slot.
- Everything else (other clans, tribes, foreign beads, epic follows) adds no edge.
  - Agent, bead, and hood targets count toward the member's `external_wait_count`.
  - Time floors (`wait_duration`, `wait_until`) are not dependencies and are not
    counted.
- Self-edges are dropped and duplicate edges are deduplicated.

Seal, unresolved members, and ordering:

- **Seal.** Members flagged `seal` (the epic `.land`: some unit row has `epic_bead_id`
  and no unit row has `phase_bead_id`) form an **unnumbered seal stage `∎`** when they
  alone make up the final layer and at least one other layer exists. Otherwise they stay
  ordinary numbered members. Live counts therefore agree with the plan's "5 waves".
- **Unresolved.** Members in a wait cycle, or depending on one, are **unresolved**. They
  form a trailing `?` group and are never hidden.
- **Order.** Layers come first, ordered by (slot launch key, launch key, name) within a
  layer. Unresolved members come last.
- **Layered.** A clan is _layered_ when it has at least two layers or any unresolved
  member. Layer count includes ghost-only and seal layers. Only layered clans get any
  wave chrome.

### Visual language

The design is restrained on purpose. Its beauty comes from alignment, the brace shape,
calm execution order, and one vocabulary repeated on every surface. It is **not** a hue
ramp:

- `TREE_DEPTH_COLORS` already encodes hierarchy depth;
- green and gold already mean Done and Running.

**Glyphs and styles** (all in the bundled Fira Code except `✗`, which falls back to
DejaVu exactly as the Failed glyph already does):

| Element                               | Glyph                  | Style                                                                    |
| ------------------------------------- | ---------------------- | ------------------------------------------------------------------------ |
| wave number, wave has unfinished work | `1`…`99`, `…` above 99 | bold `TREE_DEPTH_COLORS[0]` (sky)                                        |
| wave number, every loaded member Done | same                   | dim `TREE_DEPTH_COLORS[0]`                                               |
| brace / lane                          | `╭ ├ ╰ ─` / `│`        | `TREE_DEPTH_COLORS[0]`, bold on the selected row like today's connectors |
| seal label                            | `∎`                    | like a wave number (bold/dim by state)                                   |
| unresolved label                      | `?`                    | bold `#FFAF00`                                                           |
| JUMP wave divider                     | `┊`                    | dim                                                                      |

Never write `W1`/`w3/5`: `W` already means Waiting in the count chip. Text surfaces
spell out "wave".

**Wave states**, folded over the wave's _loaded_ members, worst first. One pip per
numbered wave, plus `∎` for the seal, which takes the seal stage's state color.

| State          | Condition                                                                            | Pip | Style                     |
| -------------- | ------------------------------------------------------------------------------------ | --- | ------------------------- |
| failed         | any member in the Failed bucket (retried failures are not Failed)                    | `✗` | `RAIL_FAILED_STYLE` red   |
| needs you      | any Stopped bucket member, or one asking / awaiting plan review                      | `▲` | bold `#FFAF00` amber      |
| in flight      | any Running/Starting/Queued/Finalizing member, or any other mix of Done and not Done | `◐` | bold `RUNNING_COLOR` gold |
| done           | every member Done                                                                    | `●` | `#5FD75F` green           |
| not started    | every member Waiting                                                                 | `○` | neutral chip grey         |
| dismissed only | ghost-only wave                                                                      | `·` | dim neutral grey          |

Ghosts never count as Done, so a ghost-only wave is `·`, never `●`.

With more than 12 numbered waves the strip compacts to `<worst pip> <done>/<total>` (for
example `◐ 8/14`), followed by ` ∎` when sealed. The fraction counts completed waves. It
is not a current-wave index.

### Surface 1: wave gutter and order (agent list)

Layered clans order their direct-member units by the core order; descendants stay
attached to their unit. Single-wave clans keep today's status order, byte for byte.

The gutter replaces today's five-cell `"  └─ "` depth-1 connector on direct members with
`f"{label:>2} {bracket} "`. That is the same footprint, so no member content moves
horizontally.

```text
▾ bob-cli (RUNNING) [R2 W5 D1] ◐○○○○ ∎ bob-cli-7v                  12m54s / 13m36s
 1 ╭ bob-cli (DONE) ×5 ◆ bob-cli-7v.1                          12:40:51 · 8m10s
   ├ bob-cli (TESTING) ◆ bob-cli-7v.2                              22s / 13m36s
   │ └─ (DONE) bob-cli-7v.2--plan                               12:31:30 · 3m01s
   │ └─ (TESTING) bob-cli-7v.2--1                                           22s
   ╰ bob-cli (RUNNING) ×4 ◆ bob-cli-7v.3                                 12m54s
 2 ─ bob-cli (WAITING ▶1 ◐1 ✓1 ●1) ◆ bob-cli-7v.4
 3 ─ bob-cli (WAITING ▶1 ◐2 ⏳1) ◆ bob-cli-7v.5
 4 ─ bob-cli (WAITING ⏳1 ○1) ◆ bob-cli-7v.6
 5 ─ bob-cli (WAITING ⏳1 ○1) ◆ bob-cli-7v.7
 ∎ ─ bob-cli (WAITING ▶2 ◐6 ⏳4 ✓1 ●1) ◆ bob-cli-7v.land
```

Other shapes:

```text
▾ sase (RUNNING) [R1 W3 D4] ●◐○○ research.4j        researchers done: their "1" is dim
 1 ╭ sase (DONE) research.4j.cdx
   ├ sase (DONE) research.4j.cld
   ├ sase (DONE) research.4j.grk
   ╰ sase (DONE) research.4j.gem
 2 ─ sase (RUNNING) research.4j.final
 3 ╭ sase (WAITING) research.4j.image
   ╰ sase (WAITING) research.4j.audio
 4 ─ sase (WAITING) research.4j.linker
▸ sase (RUNNING) [R1 W5 D8] ◐ 8/14 toobig-5y         collapsed 14-wave chain
▸ sase (DONE) [D4] review.2                           one-wave clan: unchanged
▾ sase (RUNNING) [R1 W2] ·◐○○ research.4k             researchers dismissed (ghosts)
 2 ─ sase (RUNNING) research.4k.final                 numbering stays honest: starts at 2
```

Gutter rules:

- **Brace shape comes from the visible row sequence**, so it stays correct under live
  queries and folds. A widget build pass groups each clan's consecutive visible depth-1
  anchors by wave and assigns brackets:
  - `╭`/`├`/`╰` across a wave with two or more visible members;
  - `─` for a single visible member.
  - The label goes on the first visible member of each wave.
- **Lanes.** A depth-2+ row (an unfolded session turn, for example) replaces today's
  leading `"  │  "` with `"   │ "` while later visible members of the same wave follow,
  and with `"     "` otherwise. Its remaining guides and `└─ ` are unchanged.
- **Rail density** keeps today's connectors in v1.
- **Grouping.** BY_STATUS still files the clan under its aggregate status, and banners
  are still never emitted inside a clan subtree.
- **Patching.** Wave order does not depend on status. A layered member's status change
  therefore patches that row (plus the container header) instead of rebuilding.
  - The clan patch guard compares the wave order key in place of the status bucket.
  - A wave finishing dims its number through the same in-place patch path.

### Surface 2: wave pips

- **Clan row:** right after the count chip (and its `?N` unknown-wait glyph), before the
  `⚙N`/`⋔N` lane counts.
- **Tribe Summary rosters:** after a layered clan unit's count chip.
- **Compact clan identity header:** after the chip on row 1, so every deck shows
  progress.
- Pips are off in the 19-cell rail.

### Surface 3: live WAVES ladder (clan Summary card)

The ladder is a foldable section with id `waves`, built only from in-memory rows. It
sits directly after the identity block:

- in the detached live layout, at the top of the Summary body, before the frozen
  `clan_summary`;
- otherwise, just before CLAN MEMBERS.

It is absent for non-layered clans.

```text
▾ WAVES · 5 + land · ◐○○○○ ∎ · wave 1 in flight          (COLLAPSED, the default)
   1 ◐  .1 ✓  .2 ▶  .3 ▶
   2 ○  .4 ⏳
   3 ○  .5 ⏳
   4 ○  .6 ⏳
   5 ○  .7 ⏳
   ∎ ○  .land ⏳
```

- **EXPANDED** adds an aligned `after` column to each wave row:
  - `no dependencies`;
  - `after .1 .2`;
  - `after wave 3` / `after waves 2–4` when there are more than three predecessors;
  - `after every phase` (epic seal) / `after every member` (other seals).
  - Suffixes: `· 4 dismissed` (ghosts) and `· +N outside` (external waits).
- **FULLY_EXPANDED** adds one indented line per member that has predecessors or outside
  waits. The line names each predecessor with its live status glyph, for example
  `.5  waits on .3 ▶ · .4 ⏳ · +1 outside`. This answers "why is this still waiting?".

Heading phrase, worst first:

- `wave k failed` / `land failed`;
- `wave k needs you`;
- `wave 1 in flight` / `waves 1, 2 in flight` / `3 waves in flight`;
- `all waves done`;
- `not started`.

Member cells reuse the JUMP legend's cell builder (label, status glyph, lit tint for
in-flight members), so the ladder and the legend read as one system.

### Surface 4: JUMP legend and roster

Digits follow the core (wave, launch) order, so tree row _N_, ladder position _N_, and
JUMP digit _N−1_ name the same member. For epics this equals today's launch order.
Digits still never change with status; they move only when membership or waits change.

```text
JUMP · CLAN MEMBERS 0–7 · 5 WAVES + LAND
 0 .1 ✓  1 .2 ▶  2 .3 ▶ ┊ 3 .4 ⏳ ┊ 4 .5 ⏳ ┊ 5 .6 ⏳ ┊ 6 .7 ⏳ ┊ 7 .land ⏳
```

- **Divider:** a dim `┊` between consecutive _shown_ cells of different waves.
- **Packer:** the two-row packer counts divider cells when fitting, and keeps its
  in-flight-first overflow rule and `+N ▶M` chip.
- **Title suffix:** dropped first when the title does not fit.
- **Expanded roster (`.`):** wave group labels such as `wave 2 · after .1 .2`,
  `land · after every phase`, and `unresolved · wait cycle`.
- **CLAN NEIGHBORS:** continues the numbering with no dividers.

### Invariants

Every phase upholds these and tests the ones it touches:

- No file I/O, stat, or glob on render or projection paths. Layouts are memoized by a
  wave signature, so refilters and refresh ticks cost a hash.
- Members are never hidden. Selection stays keyed by identity, never by row index.
- Waves are not sase nodes:
  - they are not nav items, `j`/`k` stops, fold, kill, or mark targets, or JUMP numbers;
  - they add no tree rows and no banners.
- Every render, patch, and paint cache key covering a surface that shows wave chrome
  includes the wave inputs it draws.
- Query, fold, and tab filters run after `project_clan_tree` and never re-project, so
  they cannot renumber waves. Dismissal re-projects, and the ghost rule keeps numbering
  honest.
- No feature flag. Each phase leaves the Agents tab coherent:
  - the riskiest change (order) lands complete with its gutter, patch path, and JUMP
    numbering;
  - later phases are additive.

There is no permanent config knob.

## Phase: core

> [!NOTE] This phase works in the linked `sase-core` repo. Open it with
> `sase repo open sase-core` and use the printed path. Nothing changes in sase here, so
> the sase-side pin moves in `model`.

1. **Extract a total layering kernel.** Add a generic `layer_dependency_waves` module,
   for example `crates/sase_core/src/dependency_layers.rs`.
   - It takes nodes in priority order plus a dependency map.
   - It returns `layers` (each in input priority order) and `unresolved`. It never
     errors.
   - Rewrite the Kahn loop in `crates/sase_core/src/bead/work.rs` (about 165–190) to
     call it. Pre-sort phases by `(created_at, id)`. Map a non-empty `unresolved` to the
     existing `cycle` `BeadError` with the identical message.
   - Every existing work-plan test must pass unchanged.
2. **Add `crates/sase_core/src/agent_clan_waves.rs`**, exported from `lib.rs` next to
   `agent_clan_tribe`, with `project_clan_waves(ClanWavesRequestWire) -> ClanWavesWire`.
   Request:
   - `ClanWavesRequestWire { clan_names: Vec<String>, members: Vec<ClanWaveMemberWire> }`.
     `clan_names` carries every spelling of the clan name (raw and presented).
   - `ClanWaveMemberWire { key, name, aliases: Vec<String>, launch_key, retry_of: Option<key>, wait_for, wait_for_beads, wait_for_hoods: Vec<String>, phase_bead_id: Option<String>, seal: bool }`.

   Response:
   - `ClanWavesWire { layered: bool, numbered_wave_count: u32, has_seal: bool, waves: Vec<ClanWaveWire>, members: Vec<ClanWaveMemberPlacementWire>, unresolved: Vec<key> }`.
   - `members` is listed in final display order.
   - `ClanWaveWire { index: u32, number: Option<u32>, seal: bool, member_keys: Vec<key>, ghost_names: Vec<String> }`.
     `number` is `None` only for the seal.
   - `ClanWaveMemberPlacementWire { key, wave_index: Option<u32>, rank: u32, predecessor_keys, ghost_predecessors: Vec<String>, external_wait_count: u32 }`.

   Implement every rule in the Wave model section exactly. The output must be fully
   deterministic: sorted ghosts, and predecessors in display order. Use serde defaults
   on optional and list fields, matching `agent_clan_tribe.rs`.

3. **Binding:** `sase_core_rs.project_clan_waves(request: dict) -> dict`, beside
   `resolve_clan_tribe` in `crates/sase_core_py/src/agent_scan/mod.rs`. Use
   `py.allow_threads` and the same dict↔JSON helpers and error style, and register it in
   the module.
4. **Rust tests.** Base them on real stored metadata. Inspect a real `bob-cli-7v.*` /
   `research.4j.*` `agent_meta.json` on this host to confirm wait spellings before
   writing fixtures. Cover:
   - the `bob-cli-7v` epic: 3-member wave 1, chain, land seal;
   - the `research.4j` swarm with image and audio in one wave;
   - a 14-wave toobig chain;
   - a flat fan-out (`layered == false`);
   - diamond and asymmetric overlap;
   - session-turn folding (`x--plan`) and aliases;
   - retry slot sharing and ordering;
   - ghosts after dismissal, including the ghost-only wave 1 and honest numbering;
   - bead-only edges of a re-worked epic (`sase-z3` shape);
   - clan-name and hood waits with launch-snapshot semantics;
   - external counts, and time floors not counted;
   - a late joiner with no waits (end of wave 1);
   - a forced cycle (`unresolved`, nobody dropped);
   - a land that shares its final layer (stays numbered);
   - an equality pin: build an `EpicWorkPlanWire` from an issues fixture, convert its
     assignments and land into clan member wires, and assert the numbered waves equal
     `plan.waves` and the land is the seal.

   If `crates/sase_core/tests/python_wire_parity.rs` or a docs binding inventory covers
   wire types, extend it.

5. **Verify** with `sase tool run check` from inside the sase-core checkout.

## Phase: model

1. **Move the pin.** Run `just ratchet-core-revision` to move `sase-core-revision.txt`
   past the `core` commit. Rebuild the local binding with `just rust-install`. It is
   long, so run it through `/sase_monitor`.
2. **Add the facade** `src/sase/core/agent_clan_waves.py`, mirroring
   `src/sase/core/agent_clan_tribe.py`: frozen dataclasses for the request and response
   plus `project_clan_waves(...)` through `require_rust_binding`. There is no Python
   fallback.
3. **Add the adapter** `src/sase/ace/tui/models/_agent_clan_waves.py`.
   - Build one member wire per direct-member unit. Refactor unit construction out of
     `_sort_clan_member_units` so units exist before sorting. Field sources:
     - **key:** the anchor identity string (the same shape as
       `ClanTribeMemberWire.identity`).
     - **name and aliases:** every spelling the TUI knows for each unit row (raw
       `agent_name` and `presented_identity_name`). `refresh_presented_agent_name`
       rewrites `waiting_for` to presented spellings in place, so both spellings must be
       present. `clan_names` gets both `agent_clan` and
       `presented_clan_reference_name()`.
     - **launch_key:** encode the exact key `ordered_clan_members` sorts by today
       (`start_time` present first, then ISO time), so single-wave numbering cannot
       move.
     - **retry_of:** the unit whose anchor `raw_suffix` equals
       `retry_chain_root_timestamp`.
     - **waits:** the union over the unit's rows.
     - **phase_bead_id:** the first non-empty value in the unit.
     - **seal:** set per the Wave model section.
   - Memoize the binding call with a bounded, thread-safe cache (`functools.lru_cache`
     on a hashable wave signature works). Projection runs on worker threads and the UI
     thread.
   - Define frozen presentation types:
     - `ClanWaveState` (the six states);
     - `ClanWaveSlot` for depth-1 anchors: wave index, label kind (number/seal/
       unresolved), number, rank, and `open` (the wave has unfinished loaded work);
     - `ClanWaveSummary` for the container: layered flag, numbered count, seal flag,
       per-wave views (number, kind, member identities, ghost names, state, predecessor
       identities), per-member views (identity, relative label, wave index,
       predecessors, ghost predecessors, external count), and unresolved identities.
   - **States** are folded per projection from the current rows (only the layout is
     memoized), using `agent_status_bucket`, `agent_is_asking`, and the retried-failure
     rule.
4. **Hook projection.** In `project_clan_tree`, attach `clan_waves` to each container
   and `clan_wave_slot` to each anchor. Declare both fields beside the other
   tree-projection fields in `models/_agent_state_session.py` and reset them in
   `_reset_tree_projection`. **Ordering and rendering do not change in this phase.**
5. **Tests:**
   - the adapter over the real binding for the epic, swarm, chain, flat, and cycle
     shapes;
   - the owner-prefixed/fleet spelling case;
   - a memo hit, with no second binding call for an unchanged signature;
   - state folding for every row of the state table;
   - dismissal producing ghosts;
   - a live query hiding a blocker without renumbering;
   - `layered == False` for one-wave clans.
6. **Perf:** a warm-memo projection of a 2,000-row clan roster must stay within noise of
   today. Use a focused micro-benchmark next to `tests/perf/bench_tui_trace.py`'s clan
   case.

## Phase: list

1. **Order.** In `_sort_clan_member_units`, order layered clans' units by
   `ClanWaveSlot.rank` (unresolved last). Keep the status sort for everything else.
2. **Gutter.** Add `src/sase/ace/tui/widgets/_clan_wave_presentation.py` (glyph and
   style constants from the Visual language section) and a pure build pass,
   `src/sase/ace/tui/widgets/_clan_wave_gutter.py`.
   - The pass derives each row's gutter (label, bracket, lane, open) from the visible
     row sequence.
   - Store the gutter in the row render context, so `patch_row` reuses it, and add it to
     `agent_render_key`.
   - Extend `_append_tree_indent` in `_agent_list_render_agent_prefix.py` to draw the
     gutter for depth-1 anchors and the lane for deeper rows. Rows outside layered clans
     render byte-identically.
3. **Patch path.**
   - `_clan_row_patch_preserves_order`: when both rows carry a slot, compare
     `(wave_index, rank)`; otherwise keep the bucket rule.
   - A wave finishing must dim its number on the wave's first visible row through the
     incremental patch path. Make the slot participate in the display diff (for example
     `compare=True`) and prove it with a test.
   - BY_STATUS keeps its existing membership-rebuild path; do not widen scope.
4. **Roster numbering.** `ordered_clan_members` (`_agent_display_clan_roster.py`) uses
   the layout order when present. Existing single-wave roster tests must pass unchanged.
5. **Help and docs.**
   - Add a "Clan Waves" box to the `?` help (`help_modal/agents_reference_sections.py`)
     with the gutter entries: number and brace, `∎`, `?`. Respect the 57/50-cell box and
     32-char description limits.
   - In `docs/ace.md`, rewrite the clan order passages (about 1682–1686, 2448–2470, and
     6096–6100) and the JUMP numbering promise.
   - In `docs/agent_sessions.md`, update about 194–196 and 240–243, and explain that
     `%wait` edges become waves.
6. **Tests:**
   - wave vs status order;
   - the gutter for every shape: unfolded session lane, query-split brace, ghost-skipped
     numbering, cycle `?`, labels above 99;
   - a status change patching in place with no `clan_member_order_change` fallback
     (`tests/ace/tui/test_agent_display_diff_grouping.py`);
   - a wave completion repaint;
   - render-key invalidation;
   - selection surviving a refresh that reorders;
   - rail output unchanged;
   - JUMP digits equal to tree order.
7. **Goldens and bench.**
   - Add a `wave_clan_agents` fixture to
     `tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py`: an epic
     mid-flight with an unfolded session in wave 1, a research swarm with media, a
     14-wave chain, a one-wave clan, and a cycle. Pin time, zone, and host.
   - Add goldens and regenerate affected ones (`clan_tree_agents` has a wait, so it
     becomes layered) with targeted `just fix-tui-screenshots -- <selectors>` through
     `/sase_monitor`. Inspect every golden change.
   - Prove host-independence with two `--check` runs, one with `TZ=UTC`.
   - Run `pytest -s -m slow tests/ace/tui/bench_tui_jk_agents.py -k clan`; p95 must stay
     under 16 ms.

## Phase: pips

1. **Renderer.** Extend `_clan_wave_presentation.py` with the state glyph table and
   `format_clan_wave_pips(summary) -> Text`. It handles ≤ 12 pips, the `◐ 8/14` compact
   form, the ` ∎` seal, and `·` ghost-only waves, and returns empty for non-layered
   clans.
2. **Placements:**
   - the clan row in `_agent_list_render_agent.py::format_agent_option`, after the chip
     and `?N`, before lane counts;
   - tribe Summary clan units: a new `MemberRosterEntry.wave_pips`, set by
     `tribe_roster_entries` and drawn after the chip by `_append_member_fields`;
   - `build_clan_compact_lines` row 1, plus the non-detached Status line in
     `_agent_display_clan_identity.py`.

   Add pip states to every affected render, paint, and header cache key.

3. **Help and docs.** Extend the "Clan Waves" help box with the pip legend and the
   compact form, and update the docs' clan row shape.
4. **Tests and goldens.**
   - Cover the renderer table, overflow, seal, ghost, and cache-key invalidation when a
     wave changes state.
   - Regenerate and inspect the clan, tribe-panel, and tribe-clan-summary goldens.

## Phase: ladder

1. **Builder.** Add `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_waves.py`
   with an append function for the `waves` section, matching the Surface 3 spec for
   every fold level. Use `append_fold_heading` with section id `waves`.
2. **Placement.** Insert it in `build_clan_detail_text`
   (`prompt_panel/_agent_display_clan.py`) as specified for the detached and
   non-detached layouts.
3. **Shared cell builder.** Factor the label + status glyph + lit tint cell out of
   `_agent_jump_legend.py::_build_cell` so the legend and the ladder share it. Legend
   output must stay byte-identical.
4. **Hint cache.** Add the wave summary signature (wave states, member status glyphs,
   predecessors) to the clan hint-render cache key (`_agent_display_hint_cache.py`).
5. **Tests:**
   - `test_summary_fold_contracts.py` (absent when not layered; adjacent levels differ);
   - the heading-phrase table;
   - `after` compression;
   - ghosts and outside counts;
   - FULLY_EXPANDED predecessor lines;
   - no artifact access during render (mirror the "full render bypasses artifact paths"
     test).
6. **Docs and goldens.** Document the section in `docs/ace.md`. Add clan-panel ladder
   goldens at fold levels 1–3 using `wave_clan_agents`, then regenerate and inspect.

## Phase: jump

1. **Target metadata.**
   - Add an optional `legend_group` to `_MemberJumpTarget` and an optional
     `title_suffix` to `_MemberJumpSection` (`_member_roster.py`).
   - Only `clan_roster_entries` sets them, for layered clans. The neighbor, session, and
     tribe rosters are unaffected.
   - Also set `MemberRosterEntry.group_label` per the Surface 4 labels.
2. **Legend** (`_agent_jump_legend.py`):
   - Draw dim `┊` between consecutive shown cells whose `legend_group` differs.
   - Include divider cells in `_collapsed_lines` and `_largest_collapsed_columns`
     fitting.
   - Append the title suffix in `jump_legend_title` (`5 WAVES + LAND`, `4 WAVES`) and
     drop it first when the title would not fit.
   - The in-flight-first rule, the overflow chip, and the always-shown selection stay
     unchanged.
3. **Docs.** Update the jump panel passage in `docs/ace.md` (about 1462–1470).
4. **Tests and goldens.**
   - Cover divider placement with overflow, title-suffix dropping, group labels in the
     expanded roster, and numbering parity with the tree.
   - Add jump-panel goldens at 60, 80, and 120 cells, then regenerate and inspect.

## Phase: verify

1. **Live review.** Capture real clans with `sase screenshot` (`--keep` for iteration):
   an epic mid-flight, a research swarm, a long toobig chain, a one-wave clan, and a
   custom `---` clan if one exists on this host.
   - Check alignment, brace and lane continuity under folds and a live query, number
     dimming, pip legibility, and the ladder and JUMP agreeing with the tree.
   - Confirm `∎` is legible at the TUI font size. If it is not, switch the seal label to
     a dim `L` (one constant) and note it.
   - Fix any polish issue found.
2. **Golden sweep.** Run one full `just fix-tui-screenshots` (no selectors) through
   `/sase_monitor`, so stale goldens can be pruned. Inspect the report and every change.
3. **Perf.** A short `SASE_TUI_TRACE=1` session must show no `clan_member_order_change`
   fallbacks for layered clans. Rerun the clan j/k bench.
4. **Glossary.** Apply the memory decisions with `/sase_memory_write`, then run
   `sase memory init`.

   > [!decision] glossary_agent_clan Rewrite the first sentence of the
   > `glossary:agent-clan` strand: a clan is a named, rootless container for related
   > agents whose members may run in parallel or in clan waves derived from their
   > `%wait` edges. Link `[[clan-wave]]` if that term lands.

   > [!decision] glossary_clan_wave Add a `glossary:clan-wave` strand (keyword "Clan
   > Wave"):
   >
   > - the dependency-layer definition from the Wave model section;
   > - layers, not barriers;
   > - the `∎` epic land seal;
   > - shown in the Agents tab only for clans with two or more layers.

5. **Follow-ups.** Record a `PROPOSED FOLLOW-UP:` note on this phase's bead for each v2
   item below.

## Non-goals (v2 candidates)

- A dependency-cone highlight for the selected member (`WAITS ON` / `BLOCKS`).
- Done-prefix compaction for very long chains.
- A Gantt card.
- Optional producer stage labels.
- A wave line in `sase agent show`.
- Wave fields in the editor integration
  (`src/sase/integrations/_agent_list_entry_build.py`).
- Epic phase titles in the ladder via the associated-plan worker cache.
- Wave gutters in rail density.

## Verification (every sase phase)

- Run `just fix` inline, then `sase tool run check`. Never run `just check-full`.
- Visual work uses targeted `just fix-tui-screenshots -- <selectors>` through
  `/sase_monitor` (TESTING/TESTED), with inspection of the retained report and every
  golden change before finishing.
- Read `sase memory read tui_perf.md` before touching projection, patch, or render
  paths. Read `sase memory read tui_screenshot.md` before golden work.
