---
tier: epic
title: NEIGHBORS section for agent lane metadata panels
goal: 'Every agent lane metadata panel (family container rows, and single agents that
  own their lane) renders a numbered, fold-aware NEIGHBORS section listing exactly
  the related agents the `~` neighbors modal offers for that lane, digit keys jump
  to (or revive) them, and single agents gain real fold-level support.

  '
phases:
- id: lane-model
  title: Lane identity, lane fold scale, and the shared neighbor projection
  depends_on: []
  size: medium
  description: '''Phase lane-model'' section: add `agent_owns_lane`, a lane fold scale
    helper, and a pure `AgentLaneNeighborProjection` builder, then refactor the `~`
    neighbors modal to build its choices from that same projection so the panel section
    and the modal can never drift.

    '
- id: roster-shared
  title: Shared roster numbering, group labels, and entry limits
  depends_on: []
  size: medium
  description: '''Phase roster-shared'' section: extend the shared numbered member-roster
    renderer with a document-wide digit allocator, per-entry group labels, a fold-driven
    entry limit, dismissed-row styling, target roles, and jump-map merging, leaving
    every existing roster byte-identical.

    '
- id: neighbors-render
  title: NEIGHBORS section rendering and lane panel wiring
  depends_on:
  - lane-model
  - roster-shared
  size: medium
  description: '''Phase neighbors-render'' section: render the NEIGHBORS section as
    the last metadata-region section of every lane panel, wire the lane fold level
    through `build_header_text` for single agents, and publish one merged member-jump
    map per lane document.

    '
- id: lane-jump
  title: Digit jump, revive, footer, and help for lane neighbors
  depends_on:
  - neighbors-render
  size: medium
  description: '''Phase lane-jump'' section: extend digit-key member jump to lane
    containers and neighbor/dismissed target roles, revive dismissed neighbors, replace
    the stale regular-agent fold notification, and update the footer and help modal.

    '
- id: polish-verify
  title: Fold contracts, visual snapshots, docs, and full check
  depends_on:
  - lane-jump
  size: small
  description: '''Phase polish-verify'' section: add lane cases to the cross-kind
    fold contract suite, add PNG visual snapshots for the NEIGHBORS section at two
    fold levels, update docs, and run `just check`.

    '
create_time: 2026-07-25 08:39:30
status: wip
---

# Plan: NEIGHBORS section for agent lane metadata panels

## 1. What we are building

Agent lane metadata panels gain a `NEIGHBORS` section, styled and numbered exactly like the existing `TRIBE MEMBERS` /
`CLAN MEMBERS` / `FAMILY MEMBERS` rosters.

An **agent lane** is a family, or a single agent that does not belong to a family (see the glossary in `CLAUDE.md`). So
the section appears on:

- **family container rows** (`agent.is_family_container_row`) — below their `FAMILY MEMBERS` roster, and
- **single agents that own their lane** — a regular agent row that is not a clan container, not a family member child,
  and not a workflow step.

It does **not** appear on clan container panels (a clan is not a lane, and clan containers have no dotted name key so
they have no computable neighbors), on tribe panel summaries, on family _member_ child rows (the family container owns
the lane), or on workflow aggregate rows (see §9 Out of scope).

The rows listed are exactly the rows the `~` (neighbors) modal offers for that lane: ancestors, descendants (including
same-session dismissed descendants), and hood neighbors grouped by hood. Each row carries a numeric keymap on the left;
pressing that digit does what `<enter>` does in the modal (jump, or revive for a dismissed row).

## 2. Current state (read this before touching anything)

**Numbered rosters.** `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py` owns the shared renderer:
`append_member_roster()` draws a `━`-ruled `❖ <TITLE> · <count>` heading plus one numbered line per `MemberRosterEntry`,
and returns a `MemberJumpMap` (`container_identity` + `_MemberJumpTarget(number, member_identity, kind)`). Callers:

- clan: `_agent_display_clan.build_clan_detail_text()` (`title="CLAN MEMBERS"`, `CLAN_FOLD_SCALE`)
- family: `_agent_display_family.append_family_member_roster()` (`title="FAMILY MEMBERS"`, `FAMILY_FOLD_SCALE`), called
  from `_agent_display_header.build_header_text()`
- tribe: `_agent_display_tribe.build_tribe_detail_text()`

`member_jump_map_publisher_for(app)` returns a callback that stores the map in the app's
`_member_jump_maps: dict[MemberJumpContainerIdentity, MemberJumpMap]` (`actions/_state_init.py`), **keyed by container
identity, replacing any previous entry for that key**. Today each panel renders at most one roster, so replacement is
harmless — that changes in this epic (see D5).

Note: `roster_level` never hides entries today. It only gates per-entry annotations and nested children.

**Digit navigation.** `actions/navigation/_member_jump.py` handles digits. `_selected_member_jump_container()` returns a
focused panel, else the selected agent **only when** `agent.is_clan_container or agent.is_family_container_row`.
`_member_jump_map_for()` requires the stored map's `container_identity` to equal the live container's.
`_current_member_target_is_valid()` revalidates a target against clan `runtime_children` or
`concrete_family_member_rows()`. Two-digit mode is chosen by `len(jump_map.targets[0].number) == 2`. Reveal goes through
`_reveal_agent_row()` → `prepare_agent_navigation_target(..., require_current=False)` →
`reveal_agent_navigation_target()`.

**Neighbors (`~`).** `actions/agents/_neighbors.py` `AgentNeighborMixin`:

- `_agent_neighbor_index()` returns a **cached** `AgentNeighborIndex` (`models/agent_hoods.py`) built from rendered rows
  plus clan-revealable prospective rows, invalidated by a wide signature (roster, panel keys, grouping, folds, query,
  dismissed set, panel collapses). Moving the cursor does **not** invalidate it.
- `_start_agent_neighbor_navigation()` gathers `ancestor_targets_for`, `descendant_targets_for`,
  `hood_neighbor_target_groups_for`, `neighbor_targets_for`, plus `_dismissed_descendant_agents(selected)`, and builds
  modal rows in `_agent_neighbor_choices()`: **ancestors**, then **descendants + dismissed descendants** sorted by
  `(agent_name_key, dismissed, display_name.casefold())`, then **hood groups in index order** (nearest hood first).
  Helpers: `_agent_neighbor_display_hoods()` (case-preserving hood labels), `_agent_neighbor_panel_label()`,
  `_agent_neighbor_time_hint()`.
- Selecting a dismissed row calls `_do_revive_agent(agent)` instead of jumping.

**This index is already on the j/k hot path.** `actions/agents/_display_detail_info.py`
`_selected_agent_neighbor_count()` calls `_agent_neighbor_index()` on every info-panel update, and
`actions/agents/_display_detail_footer.py` passes `neighbor_count=` to the footer. Reusing the cached index during
detail render therefore adds no new index builds — only O(k) extraction. Keep it that way (`sase/memory/tui_perf.md`
rules 7 and 8: debounce detail panels, render paths never stat/glob).

**Fold state.** `panel_fold_level` is an app reactive defaulting to `FoldLevel.COLLAPSED` (`ace/tui/app.py`);
per-section overrides live in `_panel_fold_overrides` (`SectionFoldStateManager`, `models/fold_state.py`).
`_agent_display_clan.panel_fold_state_from_widget()` reads both. `models/fold_scale.py` defines
`FAMILY_FOLD_SCALE = (EXPANDED, FULLY_EXPANDED)`, `CLAN_FOLD_SCALE = (COLLAPSED, EXPANDED, FULLY_EXPANDED)`,
`TRIBE_FOLD_SCALE` (+ `EXHAUSTIVE`), `effective_fold_level()` (clamps a shared level into a kind's scale), and
`fold_scale_position()`.

Fold keys (`src/sase/default_config.yml`, `modes.fold_mode.keys.agents`): `zz` cycle level, `zZ` toggle all, `za` cycle
section under the cursor, `zA` toggle section, `z1`–`z4` direct levels.

`resolve_summary_fold_scale()` **already** returns `CLAN_FOLD_SCALE` for a selected regular agent, so fold mode is
already reachable there — but `build_header_text()` gates every fold-aware section behind
`family_fold_enabled = agent.is_family_container_row and family_fold_level is not None`, so nothing happens.
`actions/navigation/_fold.py::_notify_regular_agent_fold_scope()` exists purely to explain that ("Fold levels currently
shape clan and family summaries"). That is the "single agents don't support fold levels" gap this epic closes.

**Panel document order for a non-clan agent** (`build_header_text`): metadata fields → (family: `Fold: N/M` line +
`FAMILY MEMBERS` roster) → legacy parallel members → output variables → `WORKFLOW VARIABLES` → context section
(memory/skills/workspaces/BEAD/PLAN/deltas/artifacts) → slow tool calls → `ERROR` → `──────` divider → prompt/reply
body.

## 3. Design decisions

**D1 — One data source for `~` and the panel.** Phase `lane-model` adds a pure `build_agent_lane_neighbor_projection()`
and then rewrites `_agent_neighbor_choices()` to consume it. The modal and the section therefore share one ordering,
grouping, and dismissed-handling implementation, which is what makes "the section lists the same neighbors the modal
shows" true by construction rather than by convention. A parity test locks it (§8).

**D2 — Lane-member suppression.** For a family container row the projection is asked to suppress the identities of the
lane's own `concrete_family_member_rows()`. Reason: a dotted family (e.g. a clan-hosted family `myclan.worker`) has its
container row keyed by the first member's name (`presented_identity_name`, e.g. `myclan.worker--plan`), so sibling
members like `myclan.worker--impl` land in the `myclan` hood group and would otherwise be listed twice in one panel —
once under `FAMILY MEMBERS` and again under `NEIGHBORS`. Worse, they'd appear only while the family happened to be
tree-expanded, making the section flicker. Suppressing by identity (not by name math) removes both problems and is
stable regardless of expansion. Descendants _of_ a sibling member (`myclan.worker--impl.child`) are genuinely separate
lanes and stay listed. The projection reports `suppressed_lane_member_count` so the renderer can print a dim
`… +N also listed under FAMILY MEMBERS` tail — honest about the difference from the `~` modal and from the info panel's
`neighbors:` badge, which both remain unchanged.

**D3 — Placement: last section of the metadata region.** `NEIGHBORS` is appended after `ERROR` and immediately before
the `──────` divider that separates metadata from the prompt/reply body — identical relative position for family and
single-agent lanes. Rationale: neighbors are navigational context, not primary content, so they must not push `BEAD` /
`PLAN` / `CONTEXT` down; and one placement rule for both lane kinds is what makes the feature predictable. Digit numbers
still ascend in document order (family members near the top, neighbors at the bottom).

**D4 — Fold ladder keyed to scale _position_, not to the `FoldLevel` enum.** Family lanes have a 2-level scale whose
first level is `EXPANDED`; single-agent lanes have a 3-level scale whose first level is `COLLAPSED`. Keying off the enum
would make "first level shows three" true for one and false for the other. So the rule is positional, via
`fold_scale_position(level, scale)`:

| position in lane scale | rows shown                                       |
| ---------------------- | ------------------------------------------------ |
| first                  | 3 (`NEIGHBORS_FIRST_LEVEL_LIMIT`)                |
| any middle position    | 10 (`NEIGHBORS_MID_LEVEL_LIMIT`)                 |
| last                   | all (capped only by the document digit capacity) |

Concretely: family `EXPANDED` → 3, `FULLY_EXPANDED` → all. Single agent `COLLAPSED` (the default) → 3, `EXPANDED` → 10,
`FULLY_EXPANDED` → all. This satisfies "only show the first three in the first fold level" for both lane kinds with one
rule and no per-kind special cases.

**D5 — One merged jump map per document.** A family lane panel renders two numbered rosters, and `_member_jump_maps`
keeps one map per container identity. `build_header_text()` therefore merges the family map and the neighbors map and
publishes **once**. Digits are allocated from a single document-wide `MemberJumpNumbering` so numbers are unique,
contiguous, ascend in document order, and share one width (1 digit while the document has ≤10 numbered rows, else 2 —
the existing rule, now computed from the document total instead of per roster). This also keeps
`_handle_member_jump_key`'s two-digit detection (`len(targets[0].number) == 2`) correct.

**D6 — Only visible rows get numbers.** The digit allocator is fed the _shown_ slice, so a digit never jumps to a row
the user cannot see. Numbers extend as the section unfolds; every fold action re-renders and re-publishes the map, so
the map and the document never disagree.

**D7 — Target roles.** `_MemberJumpTarget` gains `role: "member" | "neighbor" | "dismissed"`. Revalidation branches on
the role (family/clan membership vs. the live neighbor index vs. the live dismissed set), so a stale map can never jump
to something that stopped being related. `"dismissed"` rows revive instead of jumping — the same action `<enter>`
performs in the `~` modal, so one mental model covers both surfaces. Dismissed rows are explicitly marked (`⊘` glyph +
`dismissed` annotation) so the effect is discoverable rather than surprising.

**D8 — Visual language reuse.** The section reuses `append_member_roster` verbatim for rows, so status glyphs, model,
duration, marks, unread ticks, and annotations match the other rosters exactly. Its accent is `#00D7AF` — the teal
already used for the info panel's `neighbors:` badge and for agent names in the `~` modal — which visually ties the
three neighbor surfaces together without colliding with the clan or family accents. Group boundaries are dim italic
label lines (`  ancestors`, `  descendants`, `  myclan hood`), not rules, so the section stays quiet at the bottom of
the panel. Row labels are shortened relative to their group (strip `<hood>.` inside a hood group, strip the lane name
for ancestors/descendants) so the eye reads the part that differs.

**D9 — No new index work on the keystroke path.** The projection is built from the already-cached
`_agent_neighbor_index()` and returns lightweight row references; `MemberRosterEntry` objects are materialized only for
the shown slice. Both the immediate (`update_header_only`, cheap) and debounced (`update_display`) render paths build
it, so the section never flickers in after a delay.

**D10 — Rust core boundary.** Checked against the `rust_core_backend_boundary` memory: this epic adds no new domain
behavior. Hood/ancestor/descendant kinship already lives in this repo's presentation layer (`models/agent_hoods.py`) and
everything added here is Textual panel presentation, fold state, and keymap routing. Nothing crosses into
`../sase-core`.

## 4. Rendered shape

Single-agent lane `myclan.review`, default fold (position 1 of 3):

```
Fold: 1/3                                   (family lanes only; see phase neighbors-render)
...metadata / context / error sections...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
▸ ❖ NEIGHBORS · 9
  ancestors
 0  myclan · agent · ◍ Running · opus-5 · 21m
  myclan hood
 1  .code · agent · ✔ Done · sonnet-5 · 12m
 2  .plan · agent · ◈ Waiting · opus-5 · 2m
… +6 more neighbors (zz / za to show more)
──────────────────────────────────────────────────
```

Family lane `myclan.worker` (3 members), fold position 2 of 2, one dismissed descendant:

```
▼ ❖ FAMILY MEMBERS · 3
 0  --plan · plan · ✔ Done · opus-5 · 8m · ws 12
 1  --impl · agent · ◍ Running · opus-5 · 33m · ws 12
 2  --review · agent · ○ Stopped · sonnet-5 · —
...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
▼ ❖ NEIGHBORS · 4
  descendants
 3  --impl.helper · agent · ✔ Done · sonnet-5 · 4m
 4  ⊘ --impl.scratch · agent · ✔ Done · haiku-4.5 · 1m · dismissed
  myclan hood
 5  .code · agent · ✔ Done · sonnet-5 · 12m · folded
 6  .plan · agent · ◈ Waiting · opus-5 · 2m
… +3 also listed under FAMILY MEMBERS
```

`folded` marks a prospective row (`AgentNeighborRow.is_prospective`) that lives inside a collapsed clan; jumping to it
expands that clan, exactly as the `~` modal already does.

---

## 5. Phase lane-model

Files: `src/sase/ace/tui/models/agent_hoods.py`, `src/sase/ace/tui/models/fold_scale.py`, new
`src/sase/ace/tui/models/agent_lane_neighbors.py`, `src/sase/ace/tui/actions/agents/_neighbors.py`.

**5.1 `agent_owns_lane(agent)` in `models/agent_hoods.py`.** Returns whether `agent` is the metadata-panel owner of an
agent lane:

```python
def agent_owns_lane(agent: Agent) -> bool:
    if agent.is_clan_container:
        return False
    if agent.is_workflow_child or agent.is_hidden_step:
        return False
    if agent.is_family_member_child:
        return False
    return agent_name_key(agent) is not None
```

Family container rows satisfy this (they are roots, not children). Agents with no usable dotted name key are excluded
because they can have no neighbors anyway.

**5.2 Lane fold scale in `models/fold_scale.py`.** Add `AGENT_FOLD_SCALE: FoldScale = CLAN_FOLD_SCALE` (alias, not a
copy — a duplicated literal would drift) and:

```python
def lane_fold_scale(agent: object) -> FoldScale:
    """Return the fold scale owned by one agent lane's summary document."""
    return (
        FAMILY_FOLD_SCALE
        if getattr(agent, "is_family_container_row", False)
        else AGENT_FOLD_SCALE
    )
```

Rewrite the tail of `resolve_summary_fold_scale()` to `return lane_fold_scale(agent)` for the non-clan branch (clan
containers keep `CLAN_FOLD_SCALE`) and update its docstring: regular-agent lanes now own a real 3-level scale rather
than "legacy" scope. Behaviour is unchanged for every existing caller — this phase only gives the scale a name the
renderer can import. Export the new names from `__all__`.

**5.3 New `models/agent_lane_neighbors.py`** — pure, no app/widget imports:

```python
type NeighborRelation = Literal["ancestor", "descendant", "neighbor"]

@dataclass(frozen=True, slots=True)
class LaneNeighborRow:
    agent: Agent
    relation: NeighborRelation
    group_label: str        # "ancestors" | "descendants" | "<hood> hood"
    label_prefix: str       # stripped from the row label ("" when nothing to strip)
    is_prospective: bool
    is_dismissed: bool

@dataclass(frozen=True, slots=True)
class AgentLaneNeighborProjection:
    lane_identity: AgentIdentity
    rows: tuple[LaneNeighborRow, ...]
    suppressed_lane_member_count: int

    @property
    def is_empty(self) -> bool: ...

def build_agent_lane_neighbor_projection(
    *,
    lane_identity: AgentIdentity,
    lane_name: str | None,
    index: AgentNeighborIndex,
    dismissed_descendants: Sequence[Agent] = (),
    suppressed_identities: Collection[AgentIdentity] = (),
    hood_labels: Mapping[str, str] | None = None,
) -> AgentLaneNeighborProjection: ...
```

Row order **must** reproduce today's `_agent_neighbor_choices()` order exactly:

1. `index.ancestor_targets_for(lane_identity)` in index order → `relation="ancestor"`, `group_label="ancestors"`,
   `label_prefix=""`.
2. `index.descendant_targets_for(lane_identity)` merged with `dismissed_descendants`, sorted by
   `(agent_name_key(agent) or "", is_dismissed, (agent.display_name or "").casefold())`, skipping agents whose
   `agent_name_key` is `None` → `relation="descendant"`, `group_label="descendants"`, `label_prefix=lane_name or ""`.
3. `index.hood_neighbor_target_groups_for(lane_identity)` in index order; per group,
   `group_label=f"{hood_labels.get(hood, hood)} hood"`, `label_prefix=hood_labels.get(hood, hood)`,
   `relation="neighbor"`.

`is_prospective` comes from `AgentNeighborRow.is_prospective`; dismissed rows are never prospective. Rows whose
`agent.identity` is in `suppressed_identities` are dropped and counted into `suppressed_lane_member_count`.

**5.4 Refactor `_agent_neighbor_choices()`** in `actions/agents/_neighbors.py` to build from the projection
(`suppressed_identities=()`), mapping each `LaneNeighborRow` to one `AgentNeighborChoice` + `_AgentNeighborPayload` and
preserving today's `group=` / `hood=` / `dismissed=` values and the existing `panel_label` / `time_hint` helpers.
`AgentNeighborChoice.hood` must keep receiving the bare hood label (strip the trailing `" hood"` from `group_label`, or
carry the raw hood on `LaneNeighborRow` — prefer the latter: add a `hood: str = ""` field so no string surgery is
needed). Existing modal snapshots and `tests/ace/tui/modals/test_agent_neighbor_modal.py` must pass unchanged; the
modal's rendered output must not change at all in this phase.

**5.5 Also add on `AgentNeighborMixin`** (used by later phases):

```python
def lane_neighbor_projection_for(self, agent: Agent) -> AgentLaneNeighborProjection | None:
    """Return the lane-relative neighbor projection for a lane-owning row."""
```

It returns `None` unless `agent_owns_lane(agent)`; otherwise it reads the cached `self._agent_neighbor_index()`,
resolves `lane_identity=agent.identity`, `lane_name=agent.presented_identity_name`, gathers
`self._dismissed_descendant_agents(agent)` and `self._agent_neighbor_display_hoods(agent)`, and passes
`suppressed_identities={m.identity for m in concrete_family_member_rows(agent)}` when `agent.is_family_container_row`
(else `()`). It must never build a fresh index of its own beyond the normal cached accessor, and must never touch disk.

**Tests** (`tests/ace/tui/models/test_agent_lane_neighbors.py`, plus additions to
`tests/ace/tui/test_agent_neighbor_navigation.py` or a new sibling): ordering across all three relations; dismissed
descendants interleaved by name; suppression drops family members and counts them; prospective flag preserved; empty
projection for an agent with no relations; `agent_owns_lane` truth table (clan container, family container, family
member child, workflow step, hidden step, plain agent, unnamed agent); `lane_fold_scale` returns the family scale for a
family container and the 3-level agent scale otherwise.

---

## 6. Phase roster-shared

File: `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py` (plus the two-line call-site adjustments needed to keep
it compiling). **Every existing roster must render byte-identical output after this phase** — all new parameters default
to today's behaviour. `tests/ace/tui/widgets/test_member_roster.py` and
`tests/ace/tui/widgets/test_summary_fold_contracts.py` are the regression net; do not update their expectations.

**6.1 Document-wide digit allocator.**

```python
type MemberJumpRole = Literal["member", "neighbor", "dismissed"]

@dataclass(slots=True)
class MemberJumpNumbering:
    """Digit allocator shared by every numbered roster in one panel document."""
    total: int
    capacity: int = MEMBER_ROSTER_LIMIT
    _taken: int = 0

    @property
    def width(self) -> int:
        return 1 if self.total <= 10 else 2

    def take(self) -> str | None:
        """Return the next zero-padded number, or None when capacity is spent."""
```

`append_member_roster()` gains `numbering: MemberJumpNumbering | None = None`. When `None` it creates a private
`MemberJumpNumbering(total=len(ordered_entries))`, which reproduces today's
`number_width = 1 if len(ordered_entries) <= 10 else 2` and the `MEMBER_ROSTER_LIMIT` cutoff exactly.

**6.2 Group labels.** Add `group_label: str | None = None` to `MemberRosterEntry`. Before rendering an entry whose
`group_label` differs from the previously rendered one, emit `f"  {group_label}\n"` styled `"dim italic #8787AF"`. Group
lines are plain lines — **not** section anchors (do not route them through `append_section_heading`), so `[`/`]` section
navigation and `za`/`zA` keep landing on the roster heading and on entry anchors only.

**6.3 Fold-driven entry limit.** Add `entry_limit: int | None = None`. When set, only `ordered_entries[:entry_limit]`
are rendered and numbered. Generalize the existing hidden tail with `hidden_tail_label: str = "members"` and
`hidden_tail_hint: str = "not numbered"`, producing `… +N more {label} ({hint})`; the hidden count is
`len(ordered_entries) - len(rendered_entries)`. The heading count stays the **total** entry count so the section always
tells the truth about how many neighbors exist. Add an optional `extra_tail: str | None = None` rendered after the
hidden tail in `"dim italic"` (used for D2's `… +N also listed under FAMILY MEMBERS`).

**6.4 Dismissed rows.** Add `is_dismissed: bool = False` to `MemberRosterEntry`. In `_append_member_fields()`, when set,
prefix `"⊘ "` styled `"dim #FFAF00"` before the label (alongside the existing `is_marked` / `is_unread` prefixes) and
append `"dismissed"` as the final annotation regardless of fold level.

**6.5 Target roles and merging.** Add `role: MemberJumpRole = "member"` to `_MemberJumpTarget`, a
`target_role: MemberJumpRole = "member"` parameter to `append_member_roster()`, and:

```python
def merged_member_jump_map(
    container_identity: MemberJumpContainerIdentity,
    *maps: MemberJumpMap | None,
) -> MemberJumpMap:
    """Concatenate same-container maps into the one map a document publishes."""
```

It ignores `None` maps, asserts/filters mismatched `container_identity`, and preserves target order. Export the new
public names from `__all__`.

**Tests** (extend `tests/ace/tui/widgets/test_member_roster.py`): shared numbering across two rosters produces
contiguous, unique, single-width numbers below the threshold and two-digit numbers at/above it; `entry_limit` renders
exactly N rows, numbers exactly N rows, keeps the total in the heading, and emits the configured tail; group labels
appear once per run and never as section markers (`SECTION_MARKER_META_KEY` assertions); dismissed prefix and
annotation; `merged_member_jump_map` ordering and container-mismatch rejection; capacity exhaustion stops numbering
without raising.

---

## 7. Phase neighbors-render

Files: new `src/sase/ace/tui/widgets/prompt_panel/_agent_display_neighbors.py`; `_agent_display_header.py`,
`_agent_display_family.py`, `_agent_display_render.py`, `_agent_display.py`, `_agent_display_hints.py`.

**7.1 New `_agent_display_neighbors.py`:**

```python
NEIGHBORS_SECTION_ID = "neighbors"
NEIGHBORS_IDENTITY_COLOR = "#00D7AF"
NEIGHBORS_ROSTER_TITLE = "NEIGHBORS"
NEIGHBORS_FIRST_LEVEL_LIMIT = 3
NEIGHBORS_MID_LEVEL_LIMIT = 10

def neighbor_entry_limit(level: FoldLevel, scale: FoldScale) -> int | None:
    """Rows shown at ``level``: 3 at the first position, all at the last, else 10."""

def neighbor_roster_entries(
    rows: Sequence[LaneNeighborRow],
    *,
    unread_agent_ids: Collection[AgentIdentity] = (),
    marked_identities: Collection[AgentIdentity] = (),
    now: datetime | None = None,
) -> tuple[MemberRosterEntry, ...]:
    """Adapt lane neighbor rows into shared roster entries."""

def append_lane_neighbors_section(
    text: Text,
    *,
    projection: AgentLaneNeighborProjection,
    panel_level: FoldLevel,
    scale: FoldScale,
    section_fold_overrides: Mapping[str, FoldLevel] | None = None,
    numbering: MemberJumpNumbering | None = None,
    unread_agent_ids: Collection[AgentIdentity] = (),
    now: datetime | None = None,
) -> MemberJumpMap | None:
    """Append the lane's NEIGHBORS roster and return its jump map."""
```

- `neighbor_entry_limit` uses `fold_scale_position(level, scale)`: position 1 → `NEIGHBORS_FIRST_LEVEL_LIMIT`; position
  == size → `None` (all); otherwise `NEIGHBORS_MID_LEVEL_LIMIT`.
- Entry fields: `label` = presented name with `label_prefix` stripped when it is a strict prefix (keep a leading `.` or
  `--` so the row reads `.code` / `--impl.helper`), falling back to the full presented name; `kind` from
  `get_phase_label(agent)` (`"agent"` when it returns `"AGENT"`), overridden to nothing extra for dismissed rows;
  `status` = `agent.display_status`; `model` = `agent.model or "default"`; `duration` from the same leaf-runtime helper
  the family roster uses; `digest` from the shared digest builder (see 7.2); `group_label` from the row; `is_dismissed`;
  `is_unread` from `unread_agent_ids`. Append `"folded"` to the annotations for prospective rows — do this by extending
  the digest the entry carries, not by hacking the roster renderer.
- Entry limit is applied by passing `entry_limit=` to `append_member_roster`, with `hidden_tail_label="neighbors"`,
  `hidden_tail_hint="zz / za to show more"`, and `extra_tail` set to `f"… +{n} also listed under FAMILY MEMBERS"` when
  `projection.suppressed_lane_member_count` is non-zero.
- `section_id=NEIGHBORS_SECTION_ID`, `member_anchor_prefix="neighbor:"` (distinct from the family roster's `"member:"`
  so anchors can never collide inside one document), `target_role="neighbor"`. Dismissed rows need `role="dismissed"`:
  give `append_member_roster` a per-entry override by adding `target_role: MemberJumpRole | None = None` to
  `MemberRosterEntry` (phase roster-shared) and letting the roster prefer the entry's role over the call-level default —
  specify this in roster-shared's 6.5 work.
- Returns `None` (appending nothing) when `projection.is_empty`.

**7.2 Share the roster digest.** `_agent_display_family._family_member_digest()` /`_waiting_digest()` /`_retry_digest()`
already build exactly the in-memory digest the shared roster consumes. Move them to a new `_member_roster_digest.py`
(public `agent_roster_digest(agent, *, extra_annotations=())`) and have both the family roster and the neighbors roster
call it. Do not duplicate the logic.

**7.3 `build_header_text()` wiring** (`_agent_display_header.py`):

- Rename `family_fold_level` → `lane_fold_level` and `family_section_fold_overrides` → `lane_section_fold_overrides`.
  Update the call sites: `_agent_display.py` (`update_header_only`), `_agent_display_render.py`
  (`_update_display_impl`), `_agent_display_hints.py`, plus `tests/ace/tui/widgets/test_agent_display_family.py` and
  `tests/ace/tui/widgets/test_summary_fold_contracts.py`. Keep `family_fold_enabled` internally for the family-only
  sections so no existing family behaviour shifts.
- Add `lane_neighbors: AgentLaneNeighborProjection | None = None`.
- Compute once: `lane_scale = lane_fold_scale(agent)`;
  `resolved_lane_level = effective_fold_level(lane_fold_level or FoldLevel.COLLAPSED, lane_scale)`;
  `neighbors_level = effective_fold_level(overrides.get(NEIGHBORS_SECTION_ID, resolved_lane_level), lane_scale)`;
  `limit = neighbor_entry_limit(neighbors_level, lane_scale)`.
- Build the document numbering **before** rendering either roster:
  `MemberJumpNumbering(total=family_entry_count + shown_neighbor_count)` where
  `shown_neighbor_count = min(len(rows), limit)` (or `len(rows)` when `limit is None`). This requires the family entries
  up front: expose `family_roster_entries(agent, *, now=None)` publicly from `_agent_display_family.py` and give
  `append_family_member_roster()` optional `entries=` and `numbering=` parameters (defaulting to today's internal
  computation) so it can share the allocator.
- Append the neighbors section **after** the `ERROR` block and before the trailing `header_text.append("─" * 50 ...)`
  divider (D3).
- Stop publishing from `append_family_member_roster()`. Instead collect both maps and publish once:
  `member_jump_map_publisher(merged_member_jump_map(agent.identity, family_map, neighbors_map))`, only when
  `member_jump_map_publisher is not None` and at least one map exists. Keep the clan branch's single publish as-is.
- Keep the `Fold: N/M` line family-only. For single-agent lanes the `NEIGHBORS` heading's own fold glyph (`▸`/`▾`/`▼`)
  is the indicator; do not add a top-of-panel fold line to regular agent panels.

**7.4 Render-path wiring.** In `_agent_display_render._update_display_impl()`, `_agent_display.update_header_only()`,
and `_agent_display_hints.py`, replace the `if agent.is_family_container_row` gates on `member_jump_map_publisher` /
fold-level arguments with `agent_owns_lane(agent)`, and pass `lane_neighbors=<app>.lane_neighbor_projection_for(agent)`
(resolved defensively via `getattr(app, "lane_neighbor_projection_for", None)` + `callable` check, matching the
surrounding style; the projection is `None` when the app is unavailable, e.g. in widget unit tests). Pass
`unread_agent_ids=getattr(app, "_unread_completed_agent_ids", set())` on the non-clan path too so unread ticks render on
neighbor rows. Build the projection on **both** the cheap and the full path (D9) so the section does not appear late.

**Tests** (`tests/ace/tui/widgets/test_agent_display_neighbors.py`, using the existing
`tests/ace/tui/widgets/_agent_display_helpers.py` `FakePromptPanel` / `plain_of` helpers): heading, count, group lines,
and the 3/10/all ladder for both lane scales; the section is absent for an empty projection, for clan containers, and
for family member child rows; it sits between `ERROR` and the metadata divider; a family panel's merged jump map numbers
family members first and neighbors second with one shared width; the suppression tail appears only when something was
suppressed; per-entry `"dismissed"` targets carry `role="dismissed"`.

---

## 8. Phase lane-jump

Files: `src/sase/ace/tui/actions/navigation/_member_jump.py`, `actions/navigation/_fold.py`,
`actions/agents/_display_detail_footer.py`, `widgets/_keybinding_bindings.py`, `modals/help_modal/agents_bindings.py`.

**8.1 Container selection.** In `_selected_member_jump_container()`, replace
`agent.is_clan_container or agent.is_family_container_row` with `agent.is_clan_container or agent_owns_lane(agent)`.
Panel focus keeps priority over the selected agent, so digits inside a focused tribe panel still address panel members.

**8.2 Role-aware revalidation.** Change `_current_member_target_is_valid()` to take the resolved `_MemberJumpTarget`
(not just its identity) and branch on `target.role`:

- `"member"` → today's logic unchanged (panel slice / clan `runtime_children` / `concrete_family_member_rows`).
- `"neighbor"` → `self._agent_neighbor_index().related_target_identities_for(container.identity)` must contain
  `target.member_identity`.
- `"dismissed"` → `target.member_identity` must still be present in `_active_dismissed_agent_objects()` and still be a
  descendant of the container's presented identity name (`is_agent_descendant`), i.e. still listed by
  `_dismissed_descendant_agents(container)`.

Update `_member_jump_target()` to return the target (not just the identity) so `_handle_member_jump_key()` can dispatch
on role, and keep the existing "roster changed; jump cancelled" behaviour for stale targets (use neighbor-specific
wording — "Neighbor list changed; jump cancelled" — when `role != "member"`).

**8.3 Dispatch.** In `_handle_member_jump_key()` (both the single-digit and pending-second-digit paths):
`role == "dismissed"` → resolve the `Agent` from `_active_dismissed_agent_objects()` by identity and call
`self._do_revive_agent(agent)` (guarded `getattr` + `callable`, as `_neighbors.py` does); otherwise
`self._reveal_agent_row(target.member_identity)` unchanged. `_reveal_agent_row` already uses `require_current=False`,
which is what lets a prospective (clan-folded) neighbor be revealed.

**8.4 Replace the stale fold notification.** `_notify_regular_agent_fold_scope()` currently always says fold levels do
nothing for regular agents. Fold levels now shape their `NEIGHBORS` section, so notify **only** when the selected lane
agent has nothing foldable — i.e. when `self._selected_agent_neighbor_count(agent) == 0` — and reword to
`"Fold levels shape clan, family, and neighbor summaries"`. Stay silent otherwise. Rename the method to
`_notify_lane_fold_scope()` and update its call site in `_handle_panel_fold_key()`.

**8.5 Footer.** `widgets/_keybinding_bindings.py` currently appends `("0-9", "member")` when
`not panel_focused and not group_focused and (agent.is_clan_container or agent.is_family_container_row)`. Add a
`lane_neighbor_jump_available: bool = False` keyword and append `("0-9", "neighbor")` when it is true and the container
branch did not already fire. Compute it in `actions/agents/_display_detail_footer.py` next to the existing
`neighbor_count=` argument:
`agent_owns_lane(current_agent) and not current_agent.is_family_container_row and neighbor_count > 0`. This satisfies
the footer convention in `src/sase/ace/CLAUDE.md` (conditional, sometimes-true bindings only) and keeps the existing
`"member"` label for containers.

**8.6 Help modal.** In `modals/help_modal/agents_bindings.py` change `("0-9", "Jump to numbered member")` to
`("0-9", "Jump numbered member/neighbor")` (29 chars, within the 32-char limit documented in `src/sase/ace/CLAUDE.md`)
and add one line documenting the panel section near the existing `"Jump ancestor/neighbor/desc"` entry, e.g.
`("~", "Neighbors modal (see NEIGHBORS)")`, keeping every box at `_BOX_WIDTH = 57`.

**Tests**: extend `tests/ace/tui/test_agent_neighbor_navigation.py` (or add a sibling
`test_agent_lane_neighbor_jump.py`) using the existing `_agent_neighbor_navigation_helpers.py` harness — a digit on a
single-agent lane jumps to the numbered neighbor; a digit on a family lane addresses family members first and neighbors
after; a dismissed row's digit revives instead of jumping; a target that stopped being related is rejected with the
neighbor wording; digits still do nothing on rows that own no lane (family member child, workflow step). Add the
**parity test** for D1: for a set of fixture agents, the identities in `lane_neighbor_projection_for(agent).rows` equal
the identities in the `~` modal's choices for the same row, minus suppressed family members. Extend
`tests/ace/tui/widgets/` footer/help coverage for the new `("0-9", "neighbor")` binding and the reworded help entry.

---

## 9. Phase polish-verify

**9.1 Fold contracts.** Extend `tests/ace/tui/widgets/test_summary_fold_contracts.py` with lane cases so the new section
joins the cross-kind contract: for a single-agent lane (3-level scale) and a family lane (2-level scale), assert the
fold glyph per position, the 3 / 10 / all row counts, that the heading count is always the total, and that the published
jump map's numbers match the rendered digits at every level.

**9.2 Visual snapshots.** Add cases to `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py` (reuse
`hood_neighbor_agents` from `tests/ace/tui/visual/_ace_png_snapshot_helpers.py`, add a clan-hosted family fixture if
needed):

- `agents_lane_neighbors_section_first_level_<W>x<H>.png` — single-agent lane, default fold, exactly 3 rows + hidden
  tail.
- `agents_lane_neighbors_section_expanded_<W>x<H>.png` — same lane after `zz` (or `z3`), all rows.
- `agents_family_lane_neighbors_<W>x<H>.png` — family lane showing `FAMILY MEMBERS` then `NEIGHBORS` with one continuous
  digit ladder and the suppression tail.

Generate goldens with `just test-visual -- --sase-update-visual-snapshots` and then re-run `just test-visual` clean.
Inspect `.pytest_cache/sase-visual/` artifacts on any failure. Confirm the digit chips, group labels, and teal accent
read well at the snapshot size — this is the acceptance check for "beautiful".

**9.3 Docs.** Update `docs/ace.md` (Agents-tab metadata panel sections, the fold ladder, and digit jump — say explicitly
that regular-agent lanes now respond to `z*`) and `docs/agent_families.md` (family panels list neighbors below members,
and lane members are not repeated). Grep `docs/` for existing neighbor/`~` documentation and keep the wording
consistent. Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims (`CLAUDE.md`, `GEMINI.md`,
`OPENCODE.md`, `QWEN.md`) — that needs explicit user permission per this repo's memory rules.

**9.4 Full check.** Run `just install` first (ephemeral workspaces may have stale deps), then `just check`. If
`symvision` flags any new public symbol, follow `sase/memory/symvision.md` via the `/sase_memory_read` skill — prefer
deleting or privatizing over pragmas, and note that test-only references never keep a public symbol alive. Sanity-check
perf with `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` and confirm the Agents-tab p95 stays under 16 ms; the
neighbor index is already built on that path, so a regression means the projection is doing more than O(k) work per
render.

## 10. Out of scope (state these as follow-ups, do not build)

- **Workflow aggregate rows.** A top-level workflow row is an agent lane, but it renders through
  `_update_workflow_display()`, which never calls `build_header_text()`. Wiring `NEIGHBORS` into the workflow document
  is a separate change.
- **Clan and tribe panels.** Clans are not lanes and clan containers have no dotted name key; tribe panels summarize a
  panel, not a lane. Neither gains a `NEIGHBORS` section.
- **Other single-agent sections becoming fold-aware.** This epic makes exactly one section on a regular-agent panel
  respond to fold levels. `WORKFLOW VARIABLES`, `CONTEXT`, `ERROR`, and the slow-tool section stay fold-inert for
  non-family agents; making them fold-aware is a natural follow-up now that the lane fold level is plumbed through
  `build_header_text()`.
- **The info panel's `neighbors:` badge and the `~` modal's row set** stay exactly as they are. Only the panel section
  applies lane-member suppression (D2).
