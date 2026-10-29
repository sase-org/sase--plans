---
tier: tale
title: Agents tab Monitoring status group
goal:
  Group agent nodes whose session is running a monitor under a new Monitoring group in
  by-status grouping.
size: medium
decisions:
  machine_subgroups:
    ask: Should BY_MACHINE status subgroups also split out a Monitoring subgroup?
    choices:
      include:
        BY_MACHINE status subgroups gain Monitoring too, via the shared
        status_bucket_for helper
      status_only:
        Only BY_STATUS splits out Monitoring; BY_MACHINE keeps it under Running via a
        second helper
    default: include
    why:
      Docs say BY_MACHINE reuses the BY_STATUS groups, and one helper keeps patch
      signatures simple
    answer: include
  clans:
    ask:
      Should a clan row that mirrors its lone monitoring member also move into
      Monitoring?
    choices:
      include:
        A clan row joins Monitoring whenever it shows its lone member's live monitor
        label
      sessions_only:
        Clan rows stay in Running, so monitoring sessions inside clans never appear
        there
    default: include
    why: The group then always matches the status word the clan row already shows
    answer: include
  position:
    ask: Should Monitoring sit directly after Running in the BY_STATUS order?
    choices:
      after_running:
        Order becomes Stopped, Failed, Running, Monitoring, Queued, Waiting, Done,
        Starting
      after_waiting:
        Order becomes Stopped, Failed, Running, Queued, Waiting, Monitoring, Done,
        Starting
    default: after_running
    why:
      A running monitor is live work, so it sits beside Running, above not-yet-started
      work
    answer: after_running
proposed_by: bbugyi200.athena.15v
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 12:53:48
status: wip
---

# Agents Tab: "Monitoring" status group

## Goal

When the Agents tab is grouped **by status** (`GroupingMode.BY_STATUS`), agent nodes
whose agent session is currently running a monitor should render under a new
**`Monitoring`** L0 group instead of being mixed into `Running`. The group's banner
should read `⚙ Monitoring`, using the existing monitor glyph and color.

## Background (how it works today)

- `BY_STATUS` L0 keys come from `status_bucket_for(target)` in
  `src/sase/ace/tui/models/agent_groups/_buckets.py`, evaluated on each row's
  **presentation anchor**: the outer session, clan, or workflow root. See
  `_l0_value_for` / `grouping_keys_for` in `agent_groups/_keys.py`. Structural
  descendants, including the `--mon` monitor member row, inherit the anchor's group, so
  the whole session subtree moves together.
- `status_bucket_for` is currently just `agent_status_bucket(agent)` (from
  `sase.agent.status_buckets`). It returns the shared status bucket: an explicit
  `status_bucket` override, or else the Rust `status_bucket_for_status` table.
- Agent-session roots mirror their active child in `_mirror_root_from_child`
  (`src/sase/ace/tui/models/_agent_status_apply.py`). When the mirrored child is a
  monitor, the root copies `status` (for example `MONITORING`, or a custom start label
  such as `TESTING`), `monitor_start_status`, `monitor_stop_status`, and
  `monitor_state`. It also takes the lane bucket (`monitor_row_lane_bucket`), which is
  `Running` while the monitor runs. Clan containers do the same for a lone running
  member (`_copy_turn_status_presentation` in `models/_agent_clan.py`). See
  `tests/ace/tui/models/test_agent_tree_clan_status_monitor.py`.
- Row styling already uses this rule: `monitor_status_presentation`
  (`widgets/_agent_list_styling.py`) treats a row as "displaying a monitor status" when
  its `status` matches one half of its recorded monitor pair.
- Every BY_STATUS / BY_MACHINE incremental path already goes through
  `status_bucket_for`: `status_grouping_signature`, `machine_grouping_signature`,
  `rendered_group_keys`, the row-insert/remove guards in
  `widgets/_agent_list_build_{insert,remove}.py`,
  `actions/agents/_display_panel_patches.py`, `_display_panel_widgets_keys.py`, and
  `_display_diff.py`. Changing that one function therefore keeps full rebuilds and
  in-place patches consistent.

## Design

### Scope: a grouping-only bucket, not a new shared status bucket

`Monitoring` is a **TUI grouping label only**. Do **not** add it to
`AGENT_STATUS_BUCKETS`, `AGENT_STATUS_BUCKET_GLYPHS`, the Rust
`status_bucket_for_status` table, `monitor_state_bucket`, or
`monitor_lane_status_bucket`. Those shared buckets drive query filters, header and
summary counts, wait/completion tallies, `sase agent list`, and the Telegram/Neovim
integrations. All of those must keep reading a monitoring session as `Running`.

This respects the Rust core boundary. Shared status semantics stay unchanged in
`sase_core`, and the change only refines how one TUI grouping mode lays rows out, using
fields the loader already projects. No `sase-core` change and no
`sase-core-revision.txt` bump are needed.

### The "displaying a live monitor" predicate

Add a pure, O(1), row-local predicate to the models layer, next to `status_bucket_for`
in `agent_groups/_buckets.py`, and export it from the `agent_groups` package:

```python
MONITORING_STATUS_GROUP = "Monitoring"


def row_displays_live_monitor(agent: Agent) -> bool:
    """Return whether *agent* currently shows a running monitor's start label."""
    start = agent.monitor_start_status
    stop = agent.monitor_stop_status
    if not start and not stop:
        return False
    if monitor_state_is_terminal(agent.monitor_state):
        return False
    pair = monitor_status_pair(start, stop)
    return (agent.status or "").casefold() == pair.start.casefold()


def status_bucket_for(agent: Agent) -> str:
    bucket = agent_status_bucket(agent)
    if bucket == "Running" and row_displays_live_monitor(agent):
        return MONITORING_STATUS_GROUP
    return bucket
```

Why this shape:

- **Row-local.** It reads only the row's own fields, so it is cheap and it works with
  the per-row grouping signatures that the incremental display paths compare. A subtree
  walk (for example `turn_lane_counts`) would be O(n·depth) on rebuild paths. It could
  also go stale when only a child row changes. Do not use one (see the `tui_perf` memory
  note's rule 6 on incremental grouping).
- **Matches what the row shows.** Using the same "status equals the pair's start half"
  rule as `monitor_status_presentation` means a row lands in `Monitoring` exactly when
  it displays a live monitor label. This holds for the default `MONITORING`, for custom
  labels such as `TESTING`, for session roots that mirror a nested monitor, and for the
  monitor member row itself if it is ever its own anchor.
- **Never steals higher-priority groups.** The `bucket == "Running"` guard means
  needs-you (`Stopped`), `Failed`, `Queued`, `Waiting`, `Done`, and `Starting` rows keep
  their groups. A settled monitor (terminal `monitor_state`) also falls back to its
  existing bucket. A monitor whose continuation was launched (lane bucket `Running`, but
  the row now mirrors the follow-up agent) goes back to `Running`. A missing or unknown
  `monitor_state` counts as live, following the doctrine that a monitor that has not
  reported never reads as finished.

Import `monitor_status_pair` from `sase.monitor_status`. `_buckets.py` already imports
`monitor_state_is_terminal` from `sase.monitor_state`.

> [!decision] clans = sessions_only Add `if agent.is_clan_container: return bucket`
> before the predicate in `status_bucket_for`, and test that a clan with a lone
> `TESTING` member stays in `Running`.

### Group ordering

Insert `"Monitoring"` into `_STATUS_BUCKETS` in `agent_groups/_buckets.py`, directly
after `"Running"` (the `position` decision). Update the comment above the tuple.
`bucket_sort_index` and `_project_sort_key` / `_subgroup_sort_key` then order it without
further changes.

### Banner glyph

Add a `"Monitoring"` entry to `RAIL_BUCKET_GLYPHS` in
`widgets/_agent_list_render_rail.py`: `(RAIL_MONITOR_GLYPH, _MONITOR_GLYPH_STYLE)` (the
`⚙` in bold `MONITOR_GLYPH_COLOR`, already used for monitor rows). Key it by the
`MONITORING_STATUS_GROUP` constant, and add a short comment saying it is a grouping-only
label that `agent_status_bucket` never returns, so the per-row rail glyph lookup is
unaffected. `banner_prefix_segments` in `_agent_list_render_banner.py` then renders the
BY_STATUS L0 banner as `⚙ Monitoring`, and the BY_MACHINE L1 subgroup as
`▎ ⚙ Monitoring`, with no other changes. The rail legend already explains `⚙`, so leave
`RAIL_LEGEND` unchanged.

### BY_MACHINE

With the default `include`, `BY_MACHINE` status subgroups get `Monitoring` automatically
because they share `status_bucket_for`.

> [!decision] machine_subgroups = status_only Keep `status_bucket_for` unrefined. Add
> `status_group_for(agent)`, which applies the refinement, and use it only for the
> BY_STATUS L0 key (`_l0_value_for`) and for `status_grouping_signature`. Leave
> `machine_grouping_signature`, the BY_MACHINE `subgroup` in `grouping_keys_for`, and
> `_clan_row_patch_preserves_order` on `status_bucket_for`. Audit every BY_STATUS-only
> caller listed under Background so it uses `status_group_for`, and add a test that
> BY_MACHINE keeps a monitoring session under `Running`.

### Incremental display paths

No code changes are expected there. Confirm that every caller listed under Background
still compares `status_bucket_for` values, so a monitor starting or settling moves the
row between `Running` and `Monitoring` through the existing rebuild-on-signature-change
logic. `_clan_row_patch_preserves_order` will conservatively force a rebuild when a clan
member flips between Running and Monitoring, even though within-clan rank (which uses
shared buckets) does not change. Monitor start and settle are rare events, so accept
this and leave the guard as is.

## Non-goals

- No change to header/status-strip counts (`sase_agent_status_counts`), Tribe Summary
  counts, clan count chips, or wait-status tallies. Monitoring sessions still count as
  running there.
- No new query/filter value (for example `status:monitoring`).
- No change to the `sase agent list` CLI or to the Telegram/Neovim status-grouped lists
  (`integrations/agent_status_groups.py`).
- No change to within-clan member ordering (`clan_member_status_priority`).
- No change to STANDARD or BY_DATE modes.

## Implementation steps

1. `src/sase/ace/tui/models/agent_groups/_buckets.py`
   - Add the `MONITORING_STATUS_GROUP` constant and the `row_displays_live_monitor`
     predicate, and refine `status_bucket_for` as shown above. Apply the `clans` /
     `machine_subgroups` branches if those defaults were overridden.
   - Insert `"Monitoring"` into `_STATUS_BUCKETS` according to `position`.
   - Update the `GroupingMode` docstring's BY_STATUS bucket list and the
     `status_bucket_for` docstring.
2. `src/sase/ace/tui/models/agent_groups/__init__.py`: export `MONITORING_STATUS_GROUP`
   and `row_displays_live_monitor` (add them to `__all__`). Update the module docstring
   so it says BY_STATUS splits live-monitor rows into `Monitoring`.
3. `src/sase/ace/tui/widgets/_agent_list_render_rail.py`: add the `Monitoring` entry to
   `RAIL_BUCKET_GLYPHS`. Check the import direction: widgets may import from models, not
   the reverse. Update the BY_STATUS / BY_MACHINE glyph notes in the
   `format_banner_option` docstring in `_agent_list_render_banner.py` if they list
   buckets.
4. `src/sase/ace/tui/models/agent_groups/_keys.py`: only docstring touch-ups where they
   enumerate status buckets (`status_grouping_signature`, `machine_grouping_signature`).
5. Docs: before editing, read the `docs_writing` memory note with `/sase_memory_read`.
   Then update `docs/reference/tui/agents-tab-grouping.md`:
   - the `BY_STATUS` row of the grouping-mode table (L0 bucket list);
   - the BY_STATUS paragraph's priority order sentence and its glyph list. That glyph
     list is already stale (it says `▲ … ⏳`, while the code renders the
     `RAIL_BUCKET_GLYPHS` glyphs `? ✗ ▶ ○ ◷ ✓ ◐`), so correct it and add `⚙`;
   - one sentence defining Monitoring: an agent node, or a clan row under the default
     `clans` choice, whose row shows a running monitor's start label. Note that header
     and summary counts still count it as running;
   - the BY_MACHINE paragraph, if `machine_subgroups = include`;
   - the clan paragraph's "The status bucket, precedence, and count chip are unchanged"
     sentence. Clarify that the shared status bucket is unchanged, while BY_STATUS
     grouping places such a clan under Monitoring (under the default `clans` choice).
6. `src/sase/default_config.yml`: no keymap or config changes are needed. Confirm
   nothing there enumerates status buckets.

## Tests

Add or extend tests (keep any new test module under the repo's file-size limits):

- `tests/ace/tui/models/test_agent_groups_grouping_mode_status.py`. Add unit tests for
  `status_bucket_for`:
  - a session root mirroring a running monitor (`status="MONITORING"`,
    `monitor_start_status="MONITORING"`, `monitor_state="running"`,
    `status_bucket="Running"`) → `Monitoring`;
  - a custom pair (`TESTING`/`TESTED`) with status `TESTING` → `Monitoring`; with a
    different status casing (`Testing`) → still `Monitoring`;
  - a settled monitor (`monitor_state="completed"`, status `MONITORED`) → not
    `Monitoring`. A failed one (`status_bucket="Failed"`) → `Failed`;
  - a row with a running `monitor_state` but a status that does not match the start
    label (for example `RUNNING` after the root mirrored a follow-up) → `Running`;
  - a row with no monitor pair → unchanged bucket;
  - `agent_status_bucket` for the same rows still returns `Running`, which proves the
    shared bucket is untouched.
- An end-to-end grouping test (alongside the above or in a sibling module). Build the
  nested-monitor topology from
  `tests/test_agent_loader_status_override_monitor_agent_session.py`: a plan root, a
  completed coder, and a running `--mon` member. Run `_apply_status_overrides`, then
  project the tree and call `build_agent_tree(..., mode=GroupingMode.BY_STATUS)` with an
  ordinary running agent and a done agent. Assert:
  - L0 order is `Running`, `Monitoring`, `Done` (per `position`);
  - the whole session subtree (root, coder, monitor member) sits contiguously under
    `Monitoring`;
  - after flipping the monitor to `completed` and re-applying overrides, the session
    leaves `Monitoring`.
- A clan case using the helpers in
  `tests/ace/tui/models/_agent_tree_clan_status_helpers.py`: a clan with a lone running
  `TESTING` member groups under `Monitoring` (`clans = include`). A clan with two
  running members, one of them monitoring, stays under `Running`.
- A `BY_MACHINE` case: a monitoring session forms a `Monitoring` status subgroup under
  its machine (or stays under `Running` if `machine_subgroups = status_only`).
- `tests/ace/tui/widgets/test_agent_list_grouping_buckets.py`: the BY_STATUS banner for
  a monitoring session renders `⚙ Monitoring` with the monitor glyph style. The existing
  `_status_bucket_banner_labels` helper should pick up the new `RAIL_BUCKET_GLYPHS`
  entry automatically.
- `status_grouping_signature` changes when a row flips between Running and Monitoring,
  so the in-place row-patch guard forces a rebuild. Extend an existing
  `_display_panel_patches` / `_display_diff` grouping test if there is a natural one
  (for example `tests/ace/tui/test_agent_display_diff_grouping.py`).
- Run the existing BY_STATUS fold, insert, remove, and diff suites
  (`tests/ace/tui/test_agent_display_*grouping*`, `test_agent_display_row_insert.py`,
  `test_agent_fold_transitions_groups.py`) as part of `just check`. They should pass
  unchanged.

If any TUI PNG visual snapshot fixture includes a running monitor in BY_STATUS mode, its
golden will change. Read the `tui_screenshot` memory note before touching goldens, and
do not run `just check-full` or `just fix-tui-screenshots` unless explicitly instructed.

## Verification

- Before finishing, read the `lint_and_test` memory note, then run `just check` through
  `sase tool run` as that note prescribes. Fix lint, mypy, and symvision findings (new
  exports must be used or whitelisted per the `symvision` note).
- Optionally confirm visually in a live TUI (`sase screenshot`, per the `tui_screenshot`
  note) that BY_STATUS shows a `⚙ Monitoring` banner holding monitoring sessions.
