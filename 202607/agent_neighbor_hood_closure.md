---
tier: tale
title: Agents-tab neighbor closure with kinship-grouped chooser
goal: 'The ~ keymap on the Agents tab surfaces every related agent — prefix relations
  in either direction plus all agents sharing one or more hoods — and the chooser
  modal presents them in intuitive, beautiful kinship groups ordered from closest
  to farthest relation.

  '
create_time: 2026-07-18 20:40:47
status: done
prompt: 202607/prompts/agent_neighbor_hood_closure.md
---

# Plan: Agents-tab neighbor closure with kinship-grouped chooser

## Context

On the Agents tab, `~` (`start_sibling_mode`) opens `AgentNeighborModal` for the selected agent. Today the related set
is too narrow:

- **Ancestors**: visible agents whose dotted name is a proper dotted prefix of the selected agent's name.
- **Descendants**: visible agents (plus same-session dismissed agents) nested under the selected name by dotted
  boundary.
- **Neighbors**: only agents in the selected agent's _immediate_ hood (same dotted namespace up to the final segment). A
  nephew (`foo.baz.qux` while on `foo.bar`) or cousin (`a.z.1` while on `a.b.c`) is invisible to `~`, and a dotless
  agent gets no neighbor group at all.

Per the SASE glossary, an agent hood `H` contains every agent named `H` or `H.*`, and an agent belongs to every hood
along its dotted prefix chain. The requirement is: `~` must show **all other agents whose name the current agent's name
starts with, or vice versa, or that share one or more agent hoods with the current agent** — and the chooser should
separate these neighbors into sensible groups.

Relevant code (all presentation-layer Python; per the Rust-core boundary litmus test this is Agents-tab navigation over
currently _rendered_ TUI rows, so it stays in this repo — no `sase-core` change):

- `src/sase/ace/tui/models/agent_hoods.py` — `agent_name_key`, `is_agent_descendant`, `agent_hood`, `AgentNeighborRow`,
  `AgentNeighborIndex` (built from visible rows, cached).
- `src/sase/ace/tui/actions/agents/_neighbors.py` — `AgentNeighborMixin`: cached index (`_agent_neighbor_index`),
  `_start_agent_neighbor_navigation`, `_agent_neighbor_choices`, dismissed-descendant handling, hood label helpers.
- `src/sase/ace/tui/modals/agent_neighbor_modal.py` — `AgentNeighborChoice`, `AgentNeighborModal` (section headers, a–z
  quick-select, title summary).
- Count consumers: `_selected_agent_neighbor_count` in `src/sase/ace/tui/actions/agents/_display_detail.py` (info-panel
  badge + footer `~ neighbors (N)` conditional binding).

## Relation semantics (design decisions)

All matching stays casefolded and operates on the same population as today: currently visible Agents-tab rows, plus
same-session dismissed agents for the descendant group only.

**1. Boundary-aware prefix relation.** Agent `Y` is prefix-related to the selected agent `X` when one name is a proper
prefix of the other _ending at a name-structure boundary_: the very next characters after the prefix are `.` (dotted
child) or `--` (family chain). Rationale:

- Raw `startswith` would relate `deploy` to `deployer2` and `foo.bar` to `foo.barbaz` — coincidental string overlaps,
  not kinship. Boundary awareness drops only these accidental matches. (`foo.barbaz` still appears for `foo.bar` because
  they share the `foo` hood — it lands in a hood group, which is exactly right.)
- The `--` boundary makes family chains first-class: `fam--plan` ↔ `fam--plan--check`, and `clan.fam--plan` ↔
  `clan.fam--plan--check`, count as ancestor/descendant pairs.

Concretely: extend `is_agent_descendant` (and the ancestor enumeration inside `AgentNeighborIndex.from_visible_rows`) to
accept both separators. Ancestor candidates for `X` are every prefix of `X` cut at a `.` or `--` boundary, ordered
nearest-first (longest prefix first), preserving current ordering behavior. The dismissed-inclusive `descendant_count`
must count both the `name.` and `name--` sorted-prefix ranges.

**2. Hood-sharing relation.** Hoods remain dot-defined (`agent_hood` and the glossary are unchanged). An agent's hoods
are every dotted prefix of its name _including the full name itself_ (the glossary: agent `foo` is part of the `foo`
hood). Two agents share ≥1 hood iff their first dotted segments are equal, and their _deepest shared hood_ is the
longest common dotted prefix. The related set for `X` is the union of the prefix relation and the hood-sharing relation.
Note the union matters in both directions: `foo` vs `foobar` is excluded (no boundary, no shared hood), while `fam--a`
vs `fam--a--b` is included by prefix even though neither has a dotted hood.

**3. Deliberate non-relation.** Top-level family mates with no shared prefix and no dotted hood (`fam--plan` vs
`fam--fix`) are _not_ related under the stated requirements and stay out of the panel; the Agents-tab family container
nesting already covers that relationship. Call this out in a code comment/test so it reads as intentional.

## Grouping design

Every related agent appears in **exactly one** group, assigned by closest kinship, and groups are ordered closest →
farthest:

1. **Ancestors** — boundary-prefixes of `X`, nearest first (unchanged placement, now also including `--` family-chain
   ancestors).
2. **Descendants** — `X` is a boundary-prefix of them; visible plus dismissed (revivable) rows, name-sorted as today.
3. **Hood groups, deepest shared hood first.** Walk `X`'s hood chain from its own full name down to its root segment;
   each remaining related agent lands in the group for the deepest hood it shares with `X`:
   - _Own-name hood_ (`X`'s full name): rows whose name equals `X`'s name (duplicate-name rows, e.g. relaunches visible
     across panels). Usually empty; when present it is the closest kinship after descendants.
   - _Immediate hood_ (e.g. `a.b` for `a.b.c`): today's "siblings" plus their subtrees (`a.b.d`, `a.b.d.e`) — the
     current neighbor group, widened from exact-hood members to the sibling subtree.
   - _Shallower hoods_ (e.g. `a`): cousins — everything else under the shared root (`a.z`, `a.z.1`).

Within each hood group, keep visible **render order** (as the current neighbor group does) so the modal ordering matches
the panel layout the user is jumping around.

### Modal presentation

- Section headers keep the existing `-- Label ----` style and gain member counts: `Ancestors (2)`, `Descendants (3)`,
  and one section per non-empty hood group titled `Neighbors - <hood> hood (4)`. Hood display labels are sliced from the
  selected agent's raw (non-casefolded) name so displayed case matches what the user sees in the tab.
- `AgentNeighborChoice` gains a `hood: str = ""` field. The action layer emits neighbor choices already ordered (deepest
  hood group first, render order inside); the modal starts a new section whenever `hood` changes. Drop the modal's
  single `hood_label` constructor parameter (and the `_agent_neighbor_hood_label` helper's role in the title) in favor
  of the per-choice field.
- The title summary keeps its three-bucket shape — `[N ancestors - M descendants - K neighbors]` — where `K` sums all
  hood groups.
- Quick-select keys stay a–z minus `j/k/q`; overflow rows simply have no selector (existing behavior). Keep the
  single-related-agent direct jump (skip the modal) and the dismissed-descendant revive flow exactly as today.

## Index and action-layer changes

`src/sase/ace/tui/models/agent_hoods.py`:

- Build hood-chain membership at index construction: each visible row contributes to every hood along its dotted prefix
  chain (including its full name). Precompute per-row grouped neighbors — ancestors and descendants excluded — exposed
  as a new accessor, e.g. `hood_neighbor_groups_for(global_idx) -> tuple[tuple[str, tuple[int, ...]], ...]` (hood key,
  member indices in render order), deepest-first.
- `neighbors_for` becomes the flattened union of those groups so `neighbor_count` (badge/footer/title consumers)
  automatically reflects the widened set; ancestors/descendants accessors keep their names with the boundary extension
  above.
- Complexity stays O(rows × name depth) to build memberships plus the per-row group extraction; visible-row counts are
  small (tens to low hundreds), matching the existing precompute pattern.

`src/sase/ace/tui/actions/agents/_neighbors.py`:

- `_start_agent_neighbor_navigation` consumes the grouped accessor; the related-emptiness check, single-target direct
  jump, artifact-viewer guard, and modal payload wiring keep their current structure.
- `_agent_neighbor_choices` emits ancestor, descendant (visible + dismissed, merged name-sorted as today), then
  hood-group choices carrying `hood` display labels.
- `_selected_agent_neighbor_count` needs no semantic rework beyond the index change (it already sums ancestors +
  descendants + neighbors).

### Performance constraints (tui_perf)

- Reuse the existing `_agent_neighbor_index_cache` keys unchanged (agents identity, panel keys, merge flag, grouping
  mode, fold version, dismiss epoch); no new refresh paths, no disk I/O, no per-keypress rebuilds. The only hot-path
  consumer is `neighbor_count` on the debounced detail path — it must stay a dict lookup on the prebuilt index.
- The full grouped structure is computed once per index build (same cadence as today's neighbor precompute), never
  inside the modal or per keystroke.

## Edge cases to cover

- Dotless selected agent (`foo`): descendants (`foo.*`, `foo--*`) and duplicate `foo` rows are related; `foobar` is not.
- `foo.bar` vs `foo.barbaz`: hood-related (group `foo`), not ancestor/descendant.
- Nephew/cousin closure: on `a.b.c`, `a.b.d.e` lands in the `a.b` group and `a.z.1` in the `a` group; on `foo.bar`,
  `foo.baz.qux` is now reachable.
- Family chains: `clan.fam--plan` sees `clan.fam--plan--check` as descendant and `clan.fam--fix` in the `clan` hood
  group; top-level `fam--plan` / `fam--fix` remain unrelated.
- Malformed dotted names (empty segments) keep producing no relations; case-insensitive matching preserved throughout.
- Collapsed/hidden rows stay excluded (index is built from rendered rows); panel-switch focus jumps and back-jump
  anchors keep working.

## Testing

- `tests/ace/tui/models/test_agent_neighbors.py`: rework `test_parent_and_sub_hood_descendant_are_not_all_neighbors` and
  `test_index_groups_by_exact_hood_not_first_segment` to the new closure + closest-kinship grouping; add cases for
  boundary-aware prefixes (`--` chains, `deployer2` exclusion), deepest-shared-hood assignment, duplicate names, dotless
  agents, render-order preservation, and dismissed `--` descendant counting.
- `tests/ace/tui/test_agent_neighbor_navigation.py`: nephew/cousin jump scenarios, widened direct-jump/no-op conditions,
  revive flow intact.
- `tests/ace/tui/modals/test_agent_neighbor_modal.py`: multi-hood sections in order, headers with counts, per-choice
  `hood` sectioning, title summary aggregation.
- `tests/ace/tui/test_agent_neighbor_index_cache.py`: cache keying unchanged with the new build.
- PNG snapshots (`tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py`, fixtures in
  `_ace_png_snapshot_fixtures.py`): the neighbor modal and badge goldens will change (header counts, wider sets). Extend
  `hood_neighbor_agents()` (or add a fixture) so one snapshot shows at least two hood sections; accept intentional diffs
  via `--sase-update-visual-snapshots` after eyeballing `.pytest_cache/sase-visual/` artifacts.
- Help modal: the `~` line ("Jump ancestor/neighbor/desc") stays accurate; if any wording references only the immediate
  hood, refresh it per the ace help sync rule.

## Validation

Run `just install` first (ephemeral workspace), then `just check` and `just test-visual` before finishing; update visual
goldens only for verified intentional changes.
