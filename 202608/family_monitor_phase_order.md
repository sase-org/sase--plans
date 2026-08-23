---
tier: tale
title: Order family monitor phases after the shell that started them
goal:
  A monitor shell renders immediately after the agent shell that started it in every
  family projection — the metadata panel's AGENT REPLY phase stream, the FAMILY SHELLS
  roster, the shell/model lanes, and digit-jump numbering — including a monitor started
  by the family root itself.
size: small
proposed_by: bbugyi200.athena.0bn
---

- **AGENTS:**
  - [bbugyi200.athena.0bn](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0bn.md)
- **COMMITS:**
  - [0ccfd7a](https://github.com/sase-org/sase/commit/0ccfd7a6ff3c52b163c8a05c1fa3065e1588db97)
    — fix(ace): order family monitor phases after the shell that started them

# Plan: Order family monitor phases after the shell that started them

## Symptom

In the ACE Agents tab, select a sequential agent family container whose family root
started a `sase monitor` (the `<family>--mon` shell). The metadata panel's consolidated
`AGENT REPLY · N` stream renders the amber `⚙ MONITOR` phase **first**, above the
`AGENT (plan)` phase of the root shell that started it — even though the monitor started
18 minutes _after_ that shell did.

Observed shape (family `0bh`, root started 11:43:19, monitor started 12:01:06):

```text
AGENT REPLY · 2
  ── ⚙ MONITOR ──── 08:01:29        <- rendered first, but started last
     Command:  sase bead work .../procs_filter.md ...
     Status:   EPIC APPROVED → EPIC CREATED
  ── AGENT (plan) ──── 07:43:19     <- the shell that started that monitor
```

The same misordered sequence feeds four other surfaces, because they all read one
projection:

- the `FAMILY SHELLS` roster and its numbered digit-jump targets
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py:106`,
  `src/sase/ace/tui/actions/navigation/_member_jump.py:198`)
- the per-member `Model:` shell lanes
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_shell_section.py:77`)
- the family panel's hint-cache digest
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_cache.py:199`)
- `_attach_family_containers` roster back-pointers
  (`src/sase/ace/tui/models/_agent_ordering.py:237`)

The Agents-tab **tree** is not affected: `sort_and_reorder` emits the main workflow step
before `_append_followup_subtree`, so the expanded tree already reads root → main step →
monitor. Only the family _shell projection_ is wrong.

## Root cause

Everything below lives in `src/sase/ace/tui/models/agent_family_members.py`.

`concrete_family_shell_rows()` (line 276) is a two-stage projection:

1. `_family_shell_anchors()` (line 208) builds the ordered **agent** shell chain. For a
   plan-family root it calls `_concrete_planner_child()` (line 364): when the root's
   concrete `main` ace-run workflow step is loaded, that **step** becomes the first
   anchor and the container row itself is dropped from the anchor list.
   (`_root_represents_member()` at line 381 only adds the container when no planner step
   was found.)
2. `_expand_nested_monitor_shells()` (line 222) walks the loaded subtree and splices
   each monitor in after its causal starter.

Stage 2 opens with an unconditional pre-walk (lines 269-272):

```python
if container.identity not in anchor_identities:
    walk_monitors(container)
for anchor in anchors:
    emit(anchor)
```

`Agent.identity` is `(agent_type, cl_name, raw_suffix)`. The container row and its
planner step share a `raw_suffix` and an artifacts dir — they are the same underlying
agent process — but differ in `cl_name`:

```text
container identity: (WORKFLOW, 'gh_sase-org__sase', '20260823114319')
planner  identity: (WORKFLOW, 'main',              '20260823114319')
anchors:           ('20260823114319',)   # the planner step only
```

So `container.identity not in anchor_identities` is **True** whenever the planner step
is loaded, the pre-walk fires, and every monitor attached directly to the container row
is emitted at index 0 — ahead of every agent shell.

Monitors started by mid-family continuations (`--1`, `--2`, …) are placed correctly,
because those starters _are_ anchors and `emit()` calls `walk_monitors(row)` right after
appending the row. Only the root-attached `--mon` shell is hoisted.

This also contradicts the function's own docstring, which promises to "insert nested
monitor shells immediately after their causal starter".

### Evidence

Projecting every family container in the local ace-run history through
`concrete_family_shell_rows()` and checking `start_time` monotonicity:

```text
27 out-of-order of 31 monitor-bearing family containers
```

In all 27 the defect is identical — the root-attached monitor sits at index 0:

```text
=== OUT-OF-ORDER sase-p4.3
    20260817204622 MON       2026-08-17 20:46:22 --mon      <- hoisted
    20260817185435 agt step  2026-08-17 18:54:35 --plan
    20260817204843 agt       2026-08-17 20:48:43 --1
    20260817205843 MON       2026-08-17 20:58:43 --mon-0    <- correct
    20260817213435 agt       2026-08-17 21:34:35 --2
```

The 4 in-order families are exactly those with no root-attached monitor.

## Fix

Give the projection an explicit notion of the anchor that **stands in for the container
row**, and walk the container's monitors immediately after that anchor instead of before
all of them.

All edits are in `src/sase/ace/tui/models/agent_family_members.py`.

1. Add a private frozen dataclass beside the existing ones:

   ```python
   @dataclass(frozen=True, slots=True)
   class _FamilyShellAnchors:
       """Ordered agent-shell anchors plus the anchor standing in for the root.

       ``container_proxy`` is the shell that represents the container row's own
       agent process: the concrete planner step when one is loaded (same
       ``raw_suffix`` and artifacts dir as the container, different
       ``identity``), the container itself when it represents a member, and
       ``None`` when nothing in the sequence represents it.
       """

       anchors: tuple[Agent, ...]
       container_proxy: Agent | None
   ```

2. Change `_family_shell_anchors()` to return `_FamilyShellAnchors`. It already computes
   the proxy — set `container_proxy` to `planner` when `_concrete_planner_child()`
   returned one, to `agent` when `_root_represents_member(agent)` added the container,
   and to `None` otherwise. Anchor construction is otherwise unchanged.

3. Change `_expand_nested_monitor_shells()` to take the proxy and place the container
   walk relative to it. `emit()` and `walk_monitors()` keep their current bodies:

   ```python
   proxy = container_proxy
   if proxy is not None and proxy.identity == container.identity:
       proxy = None  # emit(container) already walks the container's monitors
   pending_container_walk = proxy is not None
   if proxy is None and container.identity not in anchor_identities:
       walk_monitors(container)  # nothing represents the container
   for anchor in anchors:
       emit(anchor)
       if pending_container_walk and anchor.identity == proxy.identity:
           walk_monitors(container)
           pending_container_walk = False
   if pending_container_walk:
       walk_monitors(container)  # proxy absent from anchors: never drop a row
   ```

4. `concrete_family_shell_rows()` passes both fields through.

5. Update the docstrings of `_expand_nested_monitor_shells()` and
   `concrete_family_shell_rows()` to state the contract explicitly: a monitor is emitted
   immediately after the shell that started it, and a monitor attached to the container
   row is emitted after the anchor that represents that container row (its planner step
   when one is loaded). Keep the existing notes about identity dedupe, `id()`
   cycle-guarding, and not sorting by timestamp — the fix is causal placement, not a
   timestamp sort.

### Design notes for the implementer

- **Do not sort by timestamp.** The chain order already encodes chronology, and a sort
  would break rows whose `start_time` is `None`.
- **Do not key `anchor_identities` on `raw_suffix`** to make the container and its
  planner step compare equal. `identity` is the module-wide dedupe key and weakening it
  here would let unrelated same-timestamp rows collide.
- The `container_proxy is None` fallback (pre-walk, current behavior) occurs zero times
  across the full local history. Keep it as the documented conservative fallback rather
  than inventing a new placement for it.
- The final `pending_container_walk` guard exists so a proxy that is somehow absent from
  `anchors` cannot silently drop a monitor row from the roster.
- In practice `walk_monitors(container)` finds exactly the container's monitor children:
  `_attach_runtime_children` only attaches the main agent step plus family-member
  follow-ups, and `_concrete_continuations` already claimed every non-monitor follow-up
  as an anchor.

### Prototype result

The design above was prototyped in-memory against the real local history before this
plan was written:

```text
monitor-bearing containers: 32; still out-of-order: 0;
sequences changed: 27; membership changed: 0
```

Every misordered family is fixed, no correctly ordered family changes, and no row is
added or dropped.

## Tests

### `tests/ace/tui/models/test_agent_family_members.py`

Add a helper to `tests/ace/tui/models/_agent_family_members_helpers.py` that builds a
plan root **with a loaded concrete `main` workflow step child** — the shape none of the
current tests cover, and the one that triggers the bug. The step needs `parent_workflow`
set, `step_type="agent"`, `step_index=0`, `parent_step_index=None`, the same
`raw_suffix` as the root, a different `cl_name` (production uses the step name,
`"main"`), and must be present in the root's `runtime_children` so
`_concrete_planner_child()` finds it.

New tests:

- `test_root_monitor_follows_its_planner_step_anchor` — root + main step + monitor whose
  `parent_timestamp` is the root's `raw_suffix` projects to `(main_step, monitor)`, not
  `(monitor, main_step)`.
- `test_root_monitor_precedes_later_continuations` — root + main step + root monitor + a
  `--1` continuation carrying its own monitor projects to
  `(main_step, root_monitor, continuation, continuation_monitor)`.
- `test_planner_step_projection_keeps_every_monitor` — the projected identity set equals
  the loaded shell identity set, so the placement change can never drop a row.
- `test_root_monitor_follows_root_when_no_step_is_loaded` — regression guard that the
  existing `(root, monitor)` shape (no planner step loaded, container is its own anchor)
  is unchanged.

Keep every existing test in the file passing unmodified — in particular
`test_nested_monitor_follows_mid_family_continuation` and
`test_monitor_family_member_rows_do_not_count_as_agents`.

### `tests/ace/tui/widgets/test_agent_display_family_render.py`

Add one panel-level test proving the rendered phase stream is fixed, using the existing
`make_family` / `FakePromptPanel` / `plain_of` harness in
`tests/ace/tui/widgets/_agent_display_family_helpers.py`: build a family with a loaded
planner step and a root-attached monitor, render through `_update_family_display`, and
assert the `AGENT (plan)` divider appears before the `⚙ MONITOR` divider in the plain
text (and that the `AGENT REPLY · N` count is unchanged).

### Snapshot suites

The PNG family fixtures in
`tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py` build the root as a
plain `AgentType.RUNNING` row and call `sort_and_reorder(rows, [])` with no workflow
steps, so no planner step is ever loaded and the container is its own anchor. Their
goldens are expected to be **unchanged**; confirm with `just test-visual` rather than
assuming, and do not regenerate goldens unless a real rendering change is understood and
intended.

## Documentation

- `docs/ace.md`, the `AGENT REPLY` bullet (~line 4196): after the sentence about phase
  dividers, state that phases follow the family's chain order and that a monitor phase
  renders immediately after the shell that started it — including a monitor started by
  the family root, which renders after the root's own phase.
- `docs/agent_families.md`, "Family detail folding" (~line 330): the `FAMILY SHELLS`
  roster's "stable chain order" is agent shells in chain order with each monitor spliced
  in directly after its starter shell.
- `docs/monitors.md` (~line 363): the inline `MONITOR` phase in the AGENT REPLY stream
  appears at the starter's position in the family conversation.

Keep the edits to the existing prose voice; do not restructure the sections.

## Out of scope

- Any change to how monitors are attached (`parent_timestamp`), nested in the Agents-tab
  tree, or counted in the `⚙N` running/settled lanes.
- Any change to `concrete_family_member_rows()` or agent-only status counts — monitors
  are filtered out of those and the fix must leave them byte-identical.
- Timestamp-based sorting of the family shell chain.

## Verification

```bash
just install
just check
just test-visual
```

Then, before landing, hand the exhaustive suite to a monitor:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED \
  --next 'Report just check-full results for the family monitor phase-order fix.'
```

Manual confirmation: open ACE, select a family container whose root started a monitor
(any family with a `⚙1` badge on the container row), and check that the `⚙ MONITOR`
phase now renders below the root's `AGENT (plan)` phase and that the `FAMILY SHELLS`
roster lists it in the same position.
