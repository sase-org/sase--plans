---
tier: tale
title: Show member project names on agent clan nodes
goal:
  Every Agents-tab clan row leads with its members' teal project label (for example
  `bob-cli (RUNNING) [R5 W3] research.35`), derived live from the same members as its
  count chip, and the CLAN header echoes it.
size: medium
proposed_by: bbugyi200.athena.0uv
create_time: 2026-10-01 11:44:26
status: wip
---

# Plan: Show member project names on agent clan nodes

## Goal

An agent clan row on the Agents tab currently reads

```
🧩 🤖 🚀 🦋 ⚡ (RUNNING) [R5 W3] research.35                 🏃 3m20s / 4m20s
```

Nothing on the row says which project the clan works in, even though every member row
under it starts with a teal project title (`bob-cli`). Make the clan node start with its
members' project label, in the same slot and the same teal as a member title:

```
🧩 🤖 🚀 🦋 ⚡ bob-cli (RUNNING) [R5 W3] research.35         🏃 3m20s / 4m20s
  └─ 🧩 bob-cli w0.25 c1.5x (RUNNING) ×4 research.35.gem        🏃 3m20s
```

Show the same label in the CLAN detail header, so a selected clan's header matches its
row.

## Design Decisions

1. **Placement: the title slot.** The label goes after the provider badges and before
   the `(STATUS` opener. That is where `append_agent_row_prefix` already prints the
   title of every non-clan row, so the clan row reads like the agent rows: what it works
   on, then status, counts and name. `append_agent_row_status` already writes ` (` when
   the prefix does not end in whitespace, so no new spacing logic is needed.
2. **Style: the existing agent-title teal.** Use `#00D7AF`, and `bold #00D7AF` when the
   row is selected. This matches member row titles and the `Project:` value in the
   non-clan detail header.
   - The per-project accent colors from `src/sase/project_accents.py` are deliberately
     not used. An accent would make the clan's label a different color from the teal
     titles of its own members directly below it. Teal gives one consistent "what is
     this working on" column down the panel.
3. **Source of truth: computed at render time from `clan_members(agent)`.** That is the
   same set of direct members that feeds the `[R5 W3]` count chip, so the label always
   matches the members the chip counts.
   - Each member contributes `member.project_display_name` if set, otherwise
     `project_file_parent_name(member.project_file)`. This is the existing
     display-name-first rule in `agent_groups/_keys.py::_project_name`. Skip members
     that have neither.
   - Do not store a projected field on the container:
     - `project_clan_tree` rebuilds containers on Tier-1 merges, optimistic
       kill/dismiss, and fleet reprojection.
     - On the load path, `attach_project_display_names` runs _after_ projection.
     - So a value stored at projection time can go stale. Computing at render time does
       not depend on call order. It also does no disk I/O: `project_file_parent_name` is
       lru-cached, and members already carry `project_display_name`.
4. **Clans spanning several projects.**
   - Labels are de-duplicated case-insensitively, keeping the first spelling seen.
   - Order is member count descending, then case-folded label. The dominant project
     comes first, and the order stays put while statuses change, because counts only
     change when membership changes.
   - One-line surfaces show at most **2** labels, joined by a dim `, `, then a dim ` +N`
     for the rest. Examples: `bob-cli`, `bob-cli, sase`, `bob-cli, sase +2`. This
     matches the dim ` +N` the CLAN header already uses for extra tribes.
   - The expanded CLAN identity block lists every label.
5. **No projects means no change.** A clan whose members have no project renders exactly
   as it does today: no label and no stray space.
6. **Left unchanged on purpose:**
   - Member rows keep their own titles.
   - Rail density has no room: it is a fixed 19 cells. Its tooltip reuses the expanded
     row text, so it picks up the label automatically.
   - Node-finder rows.
   - Where clans are grouped (still by the anchor member's `project_file`).
7. **Rust core boundary.** This stays in the Python TUI model layer next to
   `clan_member_counts`. The clan container projection (`_agent_tree_clan.py`) and clan
   count aggregation (`_agent_clan.py`) are presentation code owned by the TUI. No other
   frontend consumes them, so `sase-core` does not change.

## Implementation

### 1. Model helper: `src/sase/ace/tui/models/_agent_clan.py`

Add a public `clan_project_labels(agent: Agent) -> tuple[str, ...]` next to
`clan_member_counts`:

- Return `()` unless `agent.is_clan_container`.
- Walk `clan_members(agent)`, de-duplicating by `identity` the same way
  `clan_member_counts` does.
- Get each member's label with the rule from decision 3. Import
  `project_file_parent_name` from `.agent`; `agent.py` does not import `_agent_clan`, so
  there is no cycle. Skip empty labels.
- Count members per case-folded label and keep the first spelling seen.
- Return the labels sorted by `(-count, casefolded_label)`.

### 2. Shared text helper: new `src/sase/ace/tui/widgets/_clan_project_label.py`

A small, focused module in the style of `_owner_badge.py` / `_queue_weight_badge.py`:

- `CLAN_PROJECT_LABEL_COLOR = "#00D7AF"`, with a comment that it matches agent-row
  titles and the header's `Project:` field.
- `CLAN_PROJECT_LABEL_LIMIT = 2`.
- `append_clan_project_label(text: Text, labels: tuple[str, ...], *, bold: bool = False, limit: int | None = CLAN_PROJECT_LABEL_LIMIT) -> None`
  - Appends nothing when `labels` is empty.
  - Otherwise appends each visible label in `bold <color>` when `bold` is true, else
    `<color>`, with `, ` separators in style `dim`.
  - If labels were dropped, appends ` +N` in style `dim`.
  - `limit=None` shows every label (used by the expanded header).

### 3. List row rendering

- `src/sase/ace/tui/widgets/_agent_list_render_agent_prefix.py::append_agent_row_prefix`:
  - Add a keyword parameter `clan_projects: tuple[str, ...] = ()`.
  - The `if not agent.is_clan_container:` block (title, tab chip, tribe label) stays as
    it is. Add an `else:` branch that calls
    `append_clan_project_label(text, clan_projects, bold=is_selected)`.
- `src/sase/ace/tui/widgets/_agent_list_render_agent.py::format_agent_option`:
  - Add `clan_projects: tuple[str, ...] | None = None`.
  - When it is `None` and the row is a clan container, compute
    `clan_project_labels(agent)`. This follows the existing `clan_counts` fallback, so
    the direct caller in `_agent_list_widget.py` keeps working.
  - Pass the result to the prefix.
- `cached_format_agent_option` (same file): compute `clan_project_labels(agent)` once
  for clan containers, next to `visible_clan_counts`, and pass it to both
  `agent_render_key` and `format_agent_option`.
- `src/sase/ace/tui/widgets/_agent_list_render_cache.py::agent_render_key`:
  - Add `clan_projects: tuple[str, ...] | None = None`.
  - When it is `None` on a clan container, compute it, mirroring the `clan_counts`
    handling.
  - Add it to the explicit `lanes` key tuple. Then a member joining or leaving, or a
    member's project display name changing, invalidates the cached clan row. The key is
    deliberately explicit (see its docstring), so this edit is required.

### 4. CLAN detail header: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_identity.py`

- `append_clan_identity_fields` (expanded form):
  - Right after the `Name:` line, add `Project: bob-cli` (one label) or
    `Projects: bob-cli, sase, chezmoi` (several labels, all shown, `limit=None`).
  - The field label uses `CLAN_FIELD_LABEL_STYLE`; the values go through the shared
    helper.
  - Leave the line out when there are no labels.
- `build_clan_compact_lines` (compact form): start the second line with the capped label
  (limit 2), followed by `·` in `CHIP_SEPARATOR_STYLE`, before the tribes. Result:
  `bob-cli · @research · 8 agents · 4m12s · ▸ 1/3`. Leave it out when there are no
  labels. The node-finder preview (`modals/node_finder_preview.py`) reuses this builder,
  so it gets the label too.
- Check that the clan document re-renders its header when membership changes. It should,
  because `agent_count` and the counts change. If you find a header or document cache
  key that would not notice a member's project changing, add
  `clan_project_labels(agent)` to it.

### 5. Docs: `docs/ace.md`

In the clan-row paragraph (around "Clan and session rows add an agent-tree hierarchy
inside those grouping banners…"), document:

- A clan row starts with the teal project label of its direct members in the title slot,
  for example `bob-cli (RUNNING) [R5 W3] research.35`.
- A clan spanning several projects shows the dominant project first, at most two labels,
  then `+N`.
- The CLAN header repeats the label as a `Project:`/`Projects:` field and at the start
  of the compact second line.

## Tests

### Unit tests

- **New `tests/ace/tui/models/test_agent_clan_projects.py`** for `clan_project_labels`:
  - a single project;
  - `project_display_name` wins over the key, and the key from `project_file` is the
    fallback;
  - dominant project first, with ties broken by case-folded alphabetical order;
  - case-insensitive de-duplication;
  - members with no project are skipped, and an empty clan gives `()`;
  - a non-clan agent gives `()`;
  - rows from another clan or generation are excluded;
  - members nested inside a session are not counted separately: only direct members
    count, matching the count chip.
- **New `tests/ace/tui/widgets/test_agent_list_clan_project_label.py`**, using
  `format_agent_option`:
  - in the plain text, `bob-cli (RUNNING)` comes before the count chip and the orchid
    clan name;
  - the two-project form `bob-cli, sase (` and the three-project form `… +1 (`;
  - span styles: label `#00D7AF`, `bold #00D7AF` when selected, separators and overflow
    `dim`;
  - a clan with no project members is unchanged, with `(` straight after the badges;
  - a non-clan member row is unchanged.
- **Cache test** in `tests/ace/tui/widgets/test_agent_render_cache_clan.py`:
  - the cached clan row is rebuilt when a member in a new project is added to
    `runtime_children`;
  - it is rebuilt when a member's `project_display_name` changes;
  - it is reused when nothing relevant changed.
- **Header tests** in `tests/ace/tui/widgets/test_identity_header_compact.py`, plus an
  expanded-identity test next to the existing clan identity tests:
  - the compact second line starts with the project label, then `·`, before `@tribe`;
  - the expanded block has `Project:` (one label) or `Projects:` (several);
  - both are left out when there are no labels;
  - the existing tribe-overflow `+2` assertions still hold.
- **Existing tests:** update any assertion that pins the exact text of a clan row or
  clan header that now gains a label. Do not weaken unrelated assertions.

### Visual goldens

- **New fixture and snapshot:**
  - Add `multi_project_clan_agents()` to
    `tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py`. It is one clan
    with four direct members: two with `project_file` under a `bob-cli` project
    directory, one under `sase`, and one under `chezmoi`. The row should read
    `bob-cli, chezmoi +1 (…)`: the tie between chezmoi and sase breaks alphabetically.
  - Add `test_multi_project_clan_png_snapshot` to
    `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`, with golden
    `agents_clan_multi_project_120x40`. Select the clan so both the row and the CLAN
    header (`Projects: bob-cli, chezmoi, sase`) are in frame, and assert the key text
    with the existing `assert_page_svg_contains` helpers.
- **Regenerate existing goldens:** existing clan fixtures use
  `/workspace/sase/visual_project.sase`, so their clan rows and CLAN headers gain
  `sase`. Regenerate the affected goldens with
  `just fix-tui-screenshots -- <targeted selectors>`. It is long-running, so run it
  through `/sase_monitor`. The likely affected snapshot files under
  `tests/ace/tui/visual/` are:
  - `test_ace_png_snapshots_agents_clans.py`
  - `..._agents_clan_panel.py`
  - `..._agents_group_clan_collapse.py`
  - `..._agents_panel_clan_collapse.py`
  - `..._agents_tribe_clan_summaries.py`
  - `..._agents_node_rail.py`
  - `..._agents_node_finder.py`
- **Review the result:** read the command's report, including any `partial` WARNING
  block. Open each changed PNG and confirm the only difference is the new teal label and
  its spacing.

## Verification

- Run `sase tool run check`; it must pass.
- After the golden update, run `just test-visual` on the targeted selectors, through
  `/sase_monitor` if it runs long. It must report no diffs.
- Optional live check: `sase screenshot` of the Agents tab with a running clan, to
  confirm `bob-cli (RUNNING)` renders in the title slot.

## Acceptance Criteria

- A clan row whose members all run in `bob-cli` reads
  `… bob-cli (RUNNING) [R5 W3] research.35`. The label is teal, and bold when the row is
  selected.
- A clan spanning several projects shows the dominant project first, at most two labels,
  and a dim `+N` for the rest.
- A clan with no project members renders exactly as it does today.
- The CLAN header shows `Project:`/`Projects:` in its expanded block and starts its
  compact second line with the label.
- A member joining or leaving, or a member's project display name changing, updates the
  cached clan row.
- No disk I/O is added to any render path, and no `sase-core` change is made.
- Member rows, rail cells, and node-finder rows are unchanged, apart from the rail
  tooltip picking up the expanded-row text.
- Unit tests, the new multi-project golden, and the regenerated goldens all pass;
  `sase tool run check` passes.
