---
tier: epic
title: Show loaded queue multipliers on TUI capacity surfaces
goal: A persisted queue_capacity_multiplier of 1.5 with no queue_capacity_explicit
  flag renders as c1.5x and as 1.5x budget (7.5 capacity units) on the TUI row, header,
  wait lane, queue ladder, clan digest, and roster digest, while the runner capacity
  record stays explicit-false.
parent_bead: sase-19f.6
phases:
- id: loaded-display
  title: Render loaded multipliers on the remaining TUI surfaces
  size: medium
  depends_on: []
  description: 'loaded-display: Treat a valid multiplier with no integer as authored
    capacity on the badge, header, wait lane, queue ladder, and clan and roster digests,
    and drop the two reintroduced sase-19f epic-symbol lines.'
proposed_by: bbugyi200.apollo.sase-19f.6.land
create_time: 2026-09-26 08:08:30
status: wip
bead_id: sase-19f.6.4
---

- **PROMPT:** [prompts/202609/queue_multiplier_loaded_display.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_multiplier_loaded_display.md)
- **PARENT:** [202609/queue_multiplier_surfaces.md](https://github.com/sase-org/sase--plans/blob/main/202609/queue_multiplier_surfaces.md)
- **BEAD:** [sase-19f.6.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19f/sase-19f.6.4.md)

# Show loaded queue multipliers on TUI capacity surfaces

Epic sase-19f.6's three phases landed the multiplier field, the rich-text badges, and
the editors. Those display tests construct agents with `queue_capacity_explicit=True`.
Real loaders do not. Design decision 6 of `plan:202609/queue_capacity_multiplier.md`
says a multiplier's presence is the explicit signal and writers omit
`queue_capacity_explicit`.

Reproduced on this tree: `enrich_agent_from_meta` of `agent_meta.json`
`{"pid": 1234, "queue_capacity_multiplier": 1.5}` yields
`queue_capacity_multiplier == 1.5`, `queue_capacity is None`,
`queue_capacity_explicit is False`, and `wait_runners is None`.
`append_agent_queue_badges` returns false, the detail header does not contain
`1.5x budget`, `build_wait_lanes` returns no lanes, and both waiting digests are empty.

`set_queue_capacity(None, multiplier=...)` stores `queue_capacity_explicit` from its
`explicit` argument, which defaults to false. Filesystem, wire, and fleet loaders all
call it that way. Do not "fix" this by passing `explicit=True` from the loaders or the
setter. `capacity_record_from_agent` copies that flag into the runner capacity record,
and persisted markers must keep omitting `queue_capacity_explicit` for multipliers.

## Already done

Leave the projection, JSON, and editing work in place. Commits `73c47eb84f`,
`5f676a28e8`, and `2f88d1eaa7` carry it. Agent-list JSON already exposes the multiplier
without the explicit flag. Do not reopen the wait modal, directive persistence, or
prompt-edit paths.

## Loaded display

When no integer capacity is present, a valid multiplier is authored capacity. Integer
capacity still requires `queue_capacity_explicit` or `wait_runners_explicit` and still
wins when both forms are present. Invalid multipliers stay hidden. `M > 1` keeps the
over-limit badge style. Flag-off behavior stays the existing legacy wording
(`waiting for weighted load ×1.5x` in the wait lane); do not invent a new flag-off row
badge.

Use `format_queue_capacity_multiplier` and `resolve_queue_capacity_multiplier` from
`src/sase/xprompt/queue_directive.py`. Do not reimplement formatting.

### Badge and header

In `src/sase/ace/tui/widgets/_queue_weight_badge.py`:

- `format_queue_capacity_badge_value` returns the formatted multiplier when the integer
  is absent, even if `explicit` is false. An integer still returns `None` unless
  `explicit` is true.
- `queue_capacity_badge_number_style` applies the over-limit style for an authored
  multiplier above 1 even when `explicit` is false. When an integer is present, style
  that integer and ignore the multiplier.

`append_agent_queue_badges` and `prompt_panel/_agent_display_header_metadata.py` already
call those helpers. After the helper change, a loader-shaped agent with
`runner_effective_limit == 5` shows `c1.5x` and
`Capacity: 1.5x budget (7.5 capacity units)`.

### Wait lane and queue ladder

`prompt_panel/_agent_wait_section.py` currently requires
`queue_capacity_explicit or wait_runners_explicit` before it will emit the multiplier
sentence, and `_runner_wait_has_detail` ignores a multiplier-only agent, so the lane is
omitted entirely. Treat an authored multiplier as detail. With the budget flag on and
effective limit 5, the lane text is `capacity budget 1.5x (7.5)`. The flag-off branch
still says `waiting for weighted load ×1.5x`. The free-capacity `capacity_budget`
condition in that file already has a multiplier arm; keep it.

`prompt_panel/_agent_queue_section.py` sizes and draws the ladder badge only when
`entry.wait_runners_explicit` is true. `agent_runner_slots.py` sets that flag from the
integer explicit bits, so a multiplier-only entry is skipped. Include an entry whose
`capacity_multiplier` is the authored form in the capacity-badge width and in
`_queue_entry_capacity_parts`. Do not set `wait_runners_explicit` true to sneak through:
the legacy `≤N` column must stay limited to explicit integers
(`capacity_multiplier is None`). Do not change `capacity_record_from_agent`'s
`queue_capacity_explicit` expression.

### Clan and roster digests

Phase sase-19f.6.1 deferred these and phase sase-19f.6.2 did not update them. Both
`_waiting_digest` helpers append `c{wait_runners}` only:

- `src/sase/ace/tui/models/_agent_clan_sections.py`
- `src/sase/ace/tui/widgets/prompt_panel/_member_roster_digest.py`

When `wait_runners` is None and the multiplier formats, append `c1.5x` (the `c` prefix
plus `format_queue_capacity_multiplier`). Keep the two helpers local. The models module
must not import the widget badge module.

### Justfile integration

Commit `6bfd7103dc` (sase-19x.4), which landed after `2f88d1eaa7`, put these two
consumed exemptions back:

```
--epic-symbol 'sase-19f(format_queue_capacity_multiplier)'
--epic-symbol 'sase-19f(resolve_queue_capacity_multiplier)'
```

Phases 6.1 and 6.2 had already removed them because the TUI calls both functions. Delete
those two lines again. Update the comment that still says the queue multiplier seams
await sase-19f.4 so it says sase-19f.6 consumed `format_queue_capacity_multiplier`,
`resolve_queue_capacity_multiplier`, and `parse_queue_capacity_value`.

Do not add, remove, or re-key any other `--epic-symbol` entry. In particular leave the
`sase-19x.4`, `sase-18i`, and `sase-19x` lines alone.

### Tests

Update the assertions in
`tests/ace/tui/widgets/test_queue_capacity_multiplier_display.py` that currently expect
`explicit=False` to hide a multiplier and to skip the over-limit style. Add:

- A filesystem-enriched agent (`queue_capacity_multiplier: 1.5` only) renders `c1.5x`,
  header `1.5x budget (7.5 capacity units)` when `runner_effective_limit` is 5, and
  wait-lane `capacity budget 1.5x (7.5)`.
- `capacity_record_from_agent` on that agent keeps `queue_capacity_explicit` false and
  `queue_capacity_multiplier == 1.5`.
- A queue-ladder entry with `wait_runners_explicit=False`, `threshold=None`, and
  `capacity_multiplier=1.5` contributes badge width and renders `c1.5x`.
- Clan `build_agent_member_digest(...).waiting` and `agent_roster_digest` contain
  `c1.5x` for that agent, and still contain `c4` when `wait_runners` is 4.
- An integer capacity with both explicit flags false still renders no badge.

### Verification

Read `tui.md`, `tui_perf.md`, `tui_screenshot.md`, `lint_and_test.md`, and
`symvision.md` with `sase memory read` before editing. This change is string content in
existing widgets, not a refresh or navigation change. No PNG golden is expected to
change; if one does, follow the screenshot memory.

Run `sase tool run check`. Do not run `just check-full`. If check fails only on
`--epic-symbol` entries whose bead id is not `sase-19f`, and this phase did not touch
those lines, stop and record the exact symvision text. Do not delete those lines.

## Out of scope

Do not close sase-19f.6, edit its plan status, run its symvision pass, or clean
parent-epic symbols beyond the two lines above. Do not file follow-up beads. The land
agent still owns these notes, which are not this phase:

- sase-19f.6.1's parallel-load flakes
  (`test_busy_cluster_compacts_narrow_and_restores_wide`,
  `test_child_pytest_of_a_stageless_run_records_no_stage_rows`,
  `test_empty_panel_semicolon_hops_to_palette_and_back`)
- sase-19f.6.2's sase-1aa.3 symbols, already absent from the Justfile
- sase-19f.6.3's sase-1aa.4 symbols, removed by `39acc57547`
- sase-19f.6.3's sase-19x.4 symbols, still listed after `6bfd7103dc`

Design decision 9 stays integer-only: plan-approval option capacity, `sase bead work`
capacity flags, and lumberjack `wait_runners` config.
