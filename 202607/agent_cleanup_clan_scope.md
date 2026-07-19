---
tier: epic
title: Clan-scoped Agent Cleanup
goal: 'The Agent Cleanup panel can target whole clans or individual clan members within
  the current tribe, with planner-backed previews, a beautiful clan chooser modal,
  and the same confirm/execute funnel that every other cleanup scope already uses.

  '
phases:
- id: core-clan-scope
  title: Clan scope in the cleanup planner core
  depends_on: []
  description: '''Clan scope in the cleanup planner core'' section: add a first-class
    clan scope to the Rust cleanup wire and planner in sase-core, mirror it in the
    Python wire, target mapping, and reference planner, and prove parity with tests
    on both sides.'
- id: tui-clan-chooser
  title: Clan chooser modal and panel integration
  depends_on:
  - core-clan-scope
  description: '''Clan chooser modal and panel integration'' section: add the C row
    and clan counts to the Agent Cleanup panel, build the AgentCleanupClanModal with
    member drill-down and live planner previews, route selections through the shared
    planned-cleanup funnel, and style the new modal.'
- id: hardening-polish
  title: End-to-end hardening and visual polish
  depends_on:
  - tui-clan-chooser
  description: '''End-to-end hardening and visual polish'' section: end-to-end cleanup
    flow tests, a PNG visual snapshot for the clan chooser, and an audit of help modal,
    footer labels, and hint lines for the new action.'
create_time: 2026-07-19 08:16:32
status: done
bead_id: sase-74
---

# Plan: Clan-scoped Agent Cleanup

## Context

The `sase ace` Agents tab has an Agent Cleanup panel (`X`) with eight actions: dismiss completed (panel `d` / everywhere
`D`), kill+dismiss (panel `k` / everywhere `K`), marked `m`, focused group `g`, tag chooser `t`, and custom selection
`c` (`src/sase/ace/tui/modals/agent_cleanup_panel_modal.py`). Agent clans — named, rootless containers whose members run
in parallel (`<clan>.<suffix>`) — have no first-class presence in this panel. Today the only clan-shaped cleanup is the
Agents-tab footer `x` on a focused clan container, which flattens the clan into member identities and opens a bulk
confirm modal (`src/sase/ace/tui/actions/agents/_kill_action.py`, clan branch of `action_kill_agent`).

The goal: from the cleanup panel, the user can target clans directly — or drill into a clan and target members
individually — scoped to the current tribe (the focused tag panel, e.g. `@epic`). The feature must be planner-backed
(counts shown are the counts executed), converge on the existing confirmation and tracked-persistence funnel, and look
like it was always part of the panel.

### Load-bearing existing machinery (reuse, do not reinvent)

- **Shared funnel**: every cleanup path converges on `_present_planned_cleanup(request, header, targets)` →
  `_present_bulk_kill_modal(...)` → `ConfirmKillAllModal` / `ConfirmDismissAllModal` → `_do_bulk_kill_agents(...)`
  (`src/sase/ace/tui/actions/agents/_kill_action.py`, `_marking_kill.py`, `_kill_flow.py`), with optimistic in-memory
  updates and tracked background persistence tasks.
- **Chooser pattern**: `AgentCleanupTagModal` (`src/sase/ace/tui/modals/agent_cleanup_tag_modal.py`) shows one row per
  tag with a per-tag preview plan computed by the core planner; multi-selected tags union their planned identities into
  a `custom_selection` request (`_present_multi_tag_cleanup` in `_kill_action.py`). The clan chooser follows this exact
  shape.
- **Clan model**: members carry `agent_clan` + `agent_clan_generation` (`src/sase/ace/tui/models/_agent_state.py`);
  synthetic container rows are built by `project_clan_tree` (`src/sase/ace/tui/models/_agent_tree.py`);
  `clan_members_for_container` (`src/sase/ace/tui/actions/agents/_clan_cleanup.py`) is the single shared member
  enumerator used by every kill/dismiss path; clan containers never reach the cleanup wire (they are stripped everywhere
  before planning).
- **Tribe = panel**: panels are tag buckets; the focused panel key is the current tribe
  (`src/sase/ace/tui/models/agent_panels.py`, `_focused_panel_label` in `_kill_action.py`). A clan's resolved tribe is
  the container's `tag`, so a clan is never split across panels.
- **Presentation helpers**: aggregate clan status via `aggregate_clan_status`, member bucket counts via
  `clan_member_counts` + `format_agent_count_chip` (the `[D4]`-style chips), clan-relative member labels via the clan
  section helpers in `src/sase/ace/tui/models/` — reuse these for chooser rows.
- **Cleanup core**: the Rust planner lives in the `sase-core` linked repo (open it via the `/sase_repo` skill),
  `crates/sase_core/src/agent_cleanup/{wire.rs,planner.rs}`. Python mirrors: `src/sase/core/agent_cleanup_wire.py`,
  `agent_cleanup_targets.py`, `agent_cleanup_python.py` (pure-Python fallback planner used when the Rust binding is
  missing or its plan fails validation), `agent_cleanup_facade.py`.

## Product design

### Main panel: one new row, richer context

A single new action row, keyed `C`, inserted after `t Choose tag`:

```
C  Choose clan
   4 clans in @epic · focused: sase-72
```

- Enabled iff at least one clan in the focused panel has cleanable members (killable or dismissable). Disabled rows
  render dim, like the other rows.
- The detail line names the focused clan when the Agents-tab cursor is on a clan container or any of its members;
  otherwise it is just `4 clans in @epic` (or `... in (untagged)` / `... in all panels`, matching
  `_focused_panel_label`).
- The Context summary card gains a clan count on its second line, e.g. `3 tag panels · 4 clans`, keeping the three-card
  layout intact.
- The hints line gains the new key: `... m marked  g group  t tag  C clan  q close`.

Key choice rationale: `c` is taken (custom) and `d/D`, `k/K` reserve the lower/upper-case pairing for
panel-vs-everywhere scope. `C` is the strongest remaining mnemonic for Clan, and no existing action pairs with `c`, so
there is no scope-pair ambiguity to collide with.

### The clan chooser: `AgentCleanupClanModal`

A new modal, visually a sibling of the tag chooser but two-level:

```
Clan Cleanup — @epic

 ● sase-72   DONE [D4]      4 members · kill 0 · dismiss 4
 ◐ sase-70   RUNNING [R2 D3] 5 members · kill 2 · dismiss 3
     ● .1    DONE   42m
     ○ .2    RUNNING 10m
     ● .3    DONE   38m
 ○ sase-6z   DONE [D7]      7 members · kill 0 · dismiss 7
 ○ sase-6v   FAILED [F1 D2] 3 members · kill 0 · dismiss 3

 Selected: 2 clans + 2 members → kill 2 · dismiss 9
 <space> toggle  l/h expand/collapse  a all  <enter> clean up  q close
```

- **Rows**: one per clan container currently in the focused panel (current generation only, i.e. exactly what
  `clan_members_for_container` would target). Each clan row shows: selection glyph, clan name (existing clan name
  style), aggregate status word in its status color, the member-count chip (`format_agent_count_chip`, zero suppressed),
  and a dim planner-preview segment (`kill n · dismiss m`, plus `· cascade w` when workflow children cascade). Clans
  with nothing cleanable render disabled/dim and cannot be selected.
- **Drill-down**: `l` expands the highlighted clan into indented member rows — clan-relative label (`.1`, `.2`, …),
  status word, runtime — and `h` collapses. This mirrors the Agents-tab nesting convention (`l`/`h` fold navigation) so
  it costs the user nothing to learn. Member rows that are not cleanable render dim/disabled.
- **Selection**: `space` on a clan row toggles the whole clan; on a member row it toggles just that member. Glyphs: `○`
  none, `◐` partial, `●` all. `a` toggles all enabled clans. Multi-select across clans and members is allowed.
- **Confirm**: `enter` with a selection acts on the union; `enter` with no selection acts on the highlighted clan
  (tag-modal convention, so `X → C → enter` is the three-keystroke fast path for the focused clan). `esc`/`q` cancels.
- **Live preview footer**: recomputed on every selection change from the core planner over the targets snapshot captured
  at open, so the numbers shown always match what the confirm modal will execute.
- **Pre-highlight**: the modal opens with the Agents-tab focused clan highlighted when the cursor is on a clan container
  or member; otherwise the first enabled row.

### Execution semantics

- Exactly one whole clan selected (and its members carry `agent_clan`) → build a `CLEANUP_SCOPE_CLAN` request (clan
  name + generation), header `Clan: <name>`, and hand it to `_present_planned_cleanup`.
- Anything else — multiple clans, member subsets, or legacy parallel-family containers whose members lack `agent_clan` —
  union the planned member identities into a `CLEANUP_SCOPE_CUSTOM_SELECTION` request, exactly as multi-tag cleanup does
  today.
- Both paths converge on the existing confirm modal and `_do_bulk_kill_agents` tracked execution; clan containers
  themselves never reach the wire (preserving the existing invariant), and the post-cleanup toast is the existing bulk
  summary notification.

## 'Clan scope in the cleanup planner core' section

Per the Rust-core-boundary rule, "clean up clan X" is domain behavior any frontend would need to match, so the scope
lands in `sase-core` first. Open the repo with the `/sase_repo` skill; work in `crates/sase_core/src/agent_cleanup/`.

- **Wire** (`wire.rs`): add `CLEANUP_SCOPE_CLAN = "clan"`; add optional request fields `clan_name` and `clan_generation`
  to `AgentCleanupRequestWire`; add optional `agent_clan` and `agent_clan_generation` fields to
  `AgentCleanupTargetWire`. All new fields default to absent so existing encoded payloads stay valid; keep
  `AGENT_CLEANUP_WIRE_SCHEMA_VERSION` at 2 if the decoders tolerate the additive fields (they enforce version equality,
  not field exhaustiveness) — bump only if core conventions or tests demand it.
- **Planner** (`planner.rs`): extend `selected_by_scope` — the clan scope selects targets whose `agent_clan` equals
  `request.clan_name`, and whose `agent_clan_generation` matches `request.clan_generation` when the request carries one.
  Everything downstream (kill/dismiss partitioning, parallel-family cascade, "parallel family still active" skip in
  dismiss-completed mode, skip reasons, counts, confirmation severity, summary lines) is reused untouched.
- **Python mirrors** (this repo): add the scope constant and new fields to `src/sase/core/agent_cleanup_wire.py`; emit
  `agent_clan`/`agent_clan_generation` from `agent_to_cleanup_target` in `src/sase/core/agent_cleanup_targets.py`; teach
  the pure-Python reference planner in `src/sase/core/agent_cleanup_python.py` the identical selection rule so the
  facade fallback stays in lockstep; `agent_cleanup_facade.py` needs only passthrough.
- **Tests**: Rust unit tests in the core crate (clan selection, generation mismatch excluded, clan scope composed with
  dismiss-completed's active-sibling skip and the kill/dismiss cascade); Python parity tests alongside the existing
  suites in `tests/test_core_facade/` (same request → same plan from Rust binding and Python reference planner; target
  mapping emits the clan fields). Follow the sase-core repo's own build/test conventions for the Rust side, and
  rebuild/refresh the `sase_core_rs` binding however that repo's workflow prescribes so the Python facade exercises the
  new scope.

## 'Clan chooser modal and panel integration' section

All in this repo, building on the shipped core scope.

- **Types** (`src/sase/ace/tui/modals/agent_cleanup_types.py`): add `"clan"` to `AgentCleanupAction`; add `clan_count`
  (and the optional focused-clan label) to `AgentCleanupPanelState`; add an `AgentCleanupClanResult` dataclass carrying
  the selected whole-clan keys (name + generation) and individually selected member identities.
- **Panel modal** (`agent_cleanup_panel_modal.py`): new `C` binding + `_ActionRow` (title "Choose clan", detail per the
  product design), Context card clan count, and the updated hints line. Modal-internal keys stay hardcoded like the
  tag/custom modals, so no `default_config.yml` change is needed.
- **State build** (`_build_agent_cleanup_panel_state` in `src/sase/ace/tui/actions/agents/_kill_action.py`): count clans
  in the focused panel with at least one cleanable member, using the panel slice for containers and
  `clan_members_for_container` for membership; derive the focused-clan label from the selected row (container or
  member).
- **New modal** (`src/sase/ace/tui/modals/agent_cleanup_clan_modal.py`):
  `AgentCleanupClanModal(clans, targets, focused_panel_label, initial_clan)` per the product design. Per-clan preview
  plans are computed at construction the way the tag modal computes per-tag plans (single whole-clan previews via the
  new clan scope; member-level and union previews via custom-selection requests over explicit identities).
  Selection-change previews re-plan over the targets snapshot captured at open — pure in-memory planner calls, no disk
  I/O in construction or render paths. Respect the OptionList programmatic-highlight guard gotcha when rebuilding rows
  on expand/collapse.
- **Routing** (`_kill_action.py`): `"clan"` branch in `_run_agent_cleanup_panel_action` →
  `_open_clan_cleanup_selector()` (warn when no clans, mirroring the tag/custom guards); on dismiss, translate the
  result into a single-clan `CLEANUP_SCOPE_CLAN` request or a unioned custom-selection request and call
  `_present_planned_cleanup` with the `Clan: <name>` / `Clans: <n> selected` header. Stale selections resolve through
  the existing funnel, which already warns when nothing actionable remains.
- **Styling** (`src/sase/ace/tui/styles.tcss`): a new `AgentCleanupClanModal` section consistent with the tag/custom
  modal blocks (container sizing, list, preview footer, selection glyph emphasis).
- **Tests**: a new `tests/ace/tui/test_agent_cleanup_clan_modal.py` (row building and preview counts, disabled empty
  clans, l/h expand/collapse, space toggles with partial glyph transitions, toggle-all, enter-on-highlighted,
  pre-highlight of the focused clan) plus routing tests in the style of the tag-flow tests in
  `tests/ace/tui/test_panel_scoped_bulk.py` (single clan uses the clan scope; multi/mixed selections union into custom
  selection; containers stripped from the wire; tribe scoping honored, including the untagged panel; generation
  filtering; workflow-child cascade) and panel-state additions to `tests/ace/tui/test_agent_cleanup_modal.py`.

## 'End-to-end hardening and visual polish' section

- **End-to-end flow tests**: drive `X → C → select → confirm` against a synthetic agent tree with clans across two
  tribes plus an untagged clan: verify killed/ dismissed member sets, the bulk summary notification counts,
  marks/dismissed-index bookkeeping, and that a clan spanning running + done members partitions correctly into the kill
  and dismiss sections of the confirm modal. Mirror the structure of the existing e2e dismissal tests in
  `tests/ace/tui/`.
- **Visual snapshot**: add a PNG snapshot of the clan chooser (expanded clan with a partial selection, preview footer
  visible) to the visual suite under `tests/ace/tui/visual/`, following the existing snapshot fixture conventions — the
  first visual coverage for any cleanup modal, anchoring the "beautiful" bar.
- **Surface audit**: help modal (`?`) cleanup-panel entry mentions the clan action
  (`src/sase/ace/tui/modals/help_modal/agents_bindings.py`, respecting the 57-char box width rule); footer `X cleanup`
  label and the clan-container footer labels in `src/sase/ace/tui/widgets/_keybinding_bindings.py` remain correct; panel
  hints line matches the final key set; keybinding-footer tests updated accordingly.

## Testing strategy

Each phase lands with its own tests (core parity, modal/routing units, then e2e + visual). Full gates: `just check` in
this repo for every phase that touches it, plus the sase-core repo's own checks for the core phase. Perf discipline per
the TUI perf rules: previews are pure in-memory planner calls with the targets snapshot captured once at modal open;
execution reuses the tracked cleanup task mixins; no new refresh paths; re-capture UI state after awaits.

## Risks and edge cases

- **Generation drift**: agents can finish or respawn between chooser open and confirm. Selections resolve to identities
  and re-plan inside the existing funnel, which already warns on nothing-actionable; the clan scope's generation match
  prevents a respawned clan generation from being swept accidentally.
- **Legacy parallel families**: containers whose members lack `agent_clan` (old `agent_family_parallel` rows) still
  appear in the chooser but always execute via explicit member identities — behavior identical to today's footer `x`
  path.
- **Planner fallback parity**: the Python reference planner must implement the clan scope in the same phase as the Rust
  planner, or the facade's fallback would silently select nothing; parity tests are the guard.
- **Symvision**: new public symbols (modal, result type, scope constant) may trip unused-symbol lint during intermediate
  phases; consult the symvision memory before adding pragmas or whitelists.
- **Untagged / all-panels focus**: the chooser is scoped by the focused panel key, which may be the untagged panel or
  the all-panels fallback; labels and counts follow `_focused_panel_label` so wording stays consistent.
