---
tier: tale
title: Rename the H-ladder "houses" vocabulary to the real SASE term "lanes"
goal:
  The ACE footer, help modal, command palette, keymap descriptions, docs, and internal identifiers all name the first
  step of the uppercase-H collapse ladder "lanes" — the actual glossary term — instead of the metaphor-only word
  "houses", with H's behavior unchanged.
create_time: 2026-07-29 06:35:39
status: wip
---

- **PROMPT:** [202607/prompts/h_collapse_lanes_label.md](prompts/h_collapse_lanes_label.md)

# Rename the `H` "houses" vocabulary to the real SASE term "lanes"

## Problem

The ACE Agents-tab footer advertises the uppercase `H` ladder's first step as `H collapse houses`.

"House" is not a real SASE term. It has no glossary entry of its own. It appears exactly once in the glossary, inside
the **Agent Lane** definition, and only as an explanatory metaphor:

> We think of an agent lane like an agent's house (i.e. where they live). When agents are single, they live in their own
> lane.

The user-facing vocabulary for the Agents tree is **lane / hood / family / clan / tribe**. So the footer chip, the help
modal row, the command-palette entry, the `Binding` descriptions, and the prose in `docs/` are all naming a concept with
a word that a user cannot look up.

The concept the `H` ladder actually operates on _is_ an agent lane. `_is_canonical_house_owner()` in
`src/sase/ace/tui/actions/agents/_folding_houses.py` accepts a row that is not a clan container and not a child row, and
that uniquely owns its fold key — i.e. a standalone workflow row, a standalone agent row, or a sequential-family
container row. That is the same set `agent_owns_lane()` recognizes in `src/sase/ace/tui/models/agent_hoods.py`, and it
matches the definition already written in `docs/ace.md`:

> A lane is a multi-member family container or a single agent that owns its own lane.

`docs/ace.md` even mixes the two words in one paragraph today — "A grouping banner, standalone **lane**,
already-collapsed clan..." sits four lines below "the footer advertises `H collapse **houses**`".

## Goal

Replace the "house" vocabulary with "lane" everywhere it names this concept: user-facing strings first (the reported
bug), then the docs prose, then the internal identifiers that seeded the wrong word. After this change, `grep -ri house`
over `src/sase/ace/` and `docs/ace.md` should return nothing about the agent tree.

The `H` ladder's **behavior** does not change. This is a pure terminology change.

## Terminology decision

Use **`lanes`**, matching the glossary term. The footer ladder becomes:

`H collapse lanes` → `H collapse clan` → `H collapse clans` → `H collapse group`

One tradeoff was weighed and accepted: "lanes" and "clans" look similar at a glance in a footer chip. They are never
shown at the same time — they are consecutive, mutually exclusive steps of the same ladder — and inventing a
non-glossary synonym to avoid the visual similarity would recreate exactly the problem being fixed here. Use `lanes`.

## Non-goals — do NOT touch these

- **`sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, and the other provider shims.** No memory edit is needed: the
  glossary's **Agent Lane** entry is already correct, and its house _metaphor_ sentence is what justifies this rename.
  Memory edits also require explicit user permission that this plan does not carry.
- **`housekeeping`** — the AXE lumberjack interval. Unrelated word. Appears in `src/sase/axe/*`,
  `src/sase/scripts/sase_chop_managed_tmp_reap.py`, `src/sase/telemetry/*`, `docs/axe.md`, `docs/configuration.md`,
  `tests/test_axe_cli.py`, `tests/telemetry/*`.
- **Docstrings that use "houses" as a verb**, e.g. `"""Houses style constants and small layout helpers..."""`. These are
  in `src/sase/ace/tui/widgets/_agent_list_render_layout.py:3`,
  `src/sase/ace/tui/widgets/_changespec_list_helpers.py:3`, and `src/sase/ace/tui/actions/_state_init.py:3`.
- **`CHANGELOG.md`** — historical release notes are a record of what shipped; do not rewrite history.
- **`tinyhouse`** in `tests/ace/tui/widgets/test_history_word_completion.py` — arbitrary completion-test fixture data.
- **The `visual-house-navigation` CL-name fixture strings** in
  `tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py:95,143` and the query in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py:68`. These CL names are rendered _into_ the
  `agents_families_*` PNG goldens; renaming them would force regenerating goldens that have nothing to do with this
  change. Leave them.

## Step 1 — User-facing strings (this is the reported bug)

These five sites produce every string a user can read.

1. `src/sase/ace/tui/widgets/_keybinding_bindings.py:229` — the footer chip from the screenshot.
   `collapse_all_label = "collapse houses"` → `"collapse lanes"`.

2. `src/sase/ace/tui/modals/help_modal/agents_bindings.py:159` — the `?` help modal row.
   `"Fully collapse houses in scope"` → `"Fully collapse lanes in scope"`. Length check per `src/sase/ace/CLAUDE.md`:
   help-modal keybinding descriptions cap at 32 chars. The new string is 29. Fine, no truncation.

3. `src/sase/ace/tui/commands/_app_metadata.py:317-323` — the command-palette entry for `hooks_or_collapse_all`. Both
   the description (`"Collapse Agents group houses, ... panel houses/clans/groups/panel ..."`) and the searchable alias
   `"collapse houses"` → `"collapse lanes"`.

4. `src/sase/ace/tui/bindings.py:35` **and** `src/sase/ace/tui/keymaps/metadata.py:23` — the `Binding` description
   `"Collapse Scoped Houses/Selected Clan/Clans/Groups/Panel / Compact Tools / All"` → `"Collapse Scoped Lanes/..."`.
   **These two strings must stay byte-identical to each other** — `tests/test_keymaps_app_bindings.py:225` asserts the
   `Binding` description matches the keymap metadata description. Change both in the same edit.

5. `src/sase/default_config.yml:257,260` — the explanatory comment above `hooks_or_collapse_all`: "it fully collapses
   houses in the next group" and "it collapses every open house, then every open clan".

Update the assertions that pin these strings:

- `tests/test_keymaps_display_help.py:163` — `("H", "Fully collapse houses in scope")`
- `tests/test_keymaps_app_bindings.py:225` — the `Binding` description
- `tests/test_command_catalog.py:227-233` — description text and the `"collapse houses"` alias
- `tests/ace/tui/widgets/test_keybinding_footer_tools_detail.py:135,188,322,389` — four
  `("H"/"<f3>", "collapse houses")` footer-layout assertions
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_group_house_collapse.py:69` — `("H", "collapse houses")`

## Step 2 — Docs prose

`docs/ace.md` — lines 854-860, 925-926, 934, 968-985, 1012, 1022-1024. Notable spots:

- 934: the `H` row of the key table.
- 968-975: the group-scoped ladder paragraph, including the literal footer sequence "The footer advertises
  `H collapse houses`, then `H collapse clan`, ...". This paragraph already says "standalone lane" one sentence later —
  after this edit the paragraph reads consistently.
- 979-985: the whole-panel ladder paragraph, including "shows the configured `hooks_or_collapse_all` key as
  `collapse houses`, `collapse clans`, or `collapse group`".
- 1012, 1022-1024: the `BY_STATUS` grouping description — "Standalone houses precede name subgroups", "directly
  contained houses precede visible dotted-prefix subgroups".

`docs/agent_families.md` — lines 149, 376, 382: "every open workflow/family house", "including houses hidden by grouping
banners", "closes group-wide houses first".

Prefer "lane" over "house" one-for-one; where the doc says "workflow/family house", "lane" alone is already the union
term, so simplify rather than writing "workflow/family lane".

## Step 3 — Internal identifiers

This is what seeded the wrong word into the UI, so rename it too, in the same change. All of it is mechanical.

**Module rename:** `src/sase/ace/tui/actions/agents/_folding_houses.py` → `_folding_lanes.py`. Use `git mv` so history
follows.

**Within that module** (and its `__all__`):

| Old                                   | New                                  |
| ------------------------------------- | ------------------------------------ |
| `AgentHouseCollapseTarget`            | `AgentLaneCollapseTarget`            |
| `AgentPanelHouseCollapseTarget`       | `AgentPanelLaneCollapseTarget`       |
| `resolve_group_house_collapse_target` | `resolve_group_lane_collapse_target` |
| `resolve_panel_house_collapse_target` | `resolve_panel_lane_collapse_target` |
| `_is_canonical_house_owner`           | `_is_canonical_lane_owner`           |
| `_open_canonical_house_keys`          | `_open_canonical_lane_keys`          |

**Callers:**

- `src/sase/ace/tui/actions/agents/_folding_agent_groups.py:10-14,341-423` — the import block plus
  `_resolve_agent_house_collapse_target` → `_resolve_agent_lane_collapse_target`,
  `_resolve_focused_panel_house_collapse_target` → `_resolve_focused_panel_lane_collapse_target`,
  `_collapse_agent_house_folds` → `_collapse_agent_lane_folds`.
- `src/sase/ace/tui/actions/agents/_folding.py:91-115` — local variables `panel_house_target` / `house_target` and the
  renamed method calls.
- `src/sase/ace/tui/actions/agents/_display_detail_footer.py:133-215` — the `house_collapse_available` keyword and local
  `house_resolver_name` / `resolve_house_collapse`.

  > **Trap:** lines 134-139 resolve the method **by string name** through `getattr(self, house_resolver_name, None)`. A
  > rename that misses these two string literals fails silently — `getattr` returns `None`, `callable(...)` is False,
  > and the chip simply stops appearing rather than raising. Grep for the literal strings, not just the identifiers.

- `src/sase/ace/tui/widgets/_keybinding_bindings.py:114,228` and
  `src/sase/ace/tui/widgets/_keybinding_modes.py:51,116,144` — the `house_collapse_available` parameter →
  `lane_collapse_available`. This sits next to the existing `lane_neighbor_jump_available` parameter in the same
  signature, so the result is more consistent, not less.

**Comments:** `src/sase/ace/tui/models/agent_groups/__init__.py:27,29` and
`src/sase/ace/tui/models/agent_groups/_keys.py:360,363` ("standalone houses render before visible name-root subgroups",
"direct houses precede...").

**Naming-collision check (already done, recorded here so it is not re-litigated):** "lane" is also used for detail-panel
metadata sections (`lane_fold_level`, `lane_section_fold_overrides`, the `SASE CONTEXT` lanes). None of the new names
collide — every new symbol is `*_lane_collapse_*` or `AgentLane*CollapseTarget`, and none of those exist today.

**Test-side renames:**

- `tests/ace/tui/_agent_fold_transition_helpers.py:199-215` — `make_standalone_workflow_house` →
  `make_standalone_workflow_lane`, plus its docstring and the `cl_name="house"` / `raw_suffix="workflow-house"` /
  `workflow="house-workflow"` fixture strings. These are internal fixture values that no golden renders, so renaming
  them is safe — unlike the `visual-house-navigation` names called out under Non-goals.
- Import/call sites: `tests/ace/tui/test_agent_fold_transitions_groups.py` (38 refs, includes local helpers
  `_named_workflow_house` / `_named_family_house` and several test function names),
  `tests/ace/tui/test_agent_fold_transitions_navigation.py`, `tests/ace/tui/test_agent_fold_transitions_tree.py`,
  `tests/ace/tui/test_agent_fold_transitions_group_clans.py:293,309`,
  `tests/ace/tui/test_agent_fold_transitions_panel_clans.py:45`.
- `tests/ace/tui/test_agent_display_defer_detail.py:234-335` — monkeypatches the resolver methods by attribute and
  asserts `call["house_collapse_available"]`. Both must follow the rename.
- `tests/ace/tui/models/test_agent_groups_grouping_mode_tree_status.py:116,233,358` — test names and a comment.
- `tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py:161-210` — `group_house_collapse_precedence_agents` →
  `group_lane_collapse_precedence_agents`, `workflow_house` local → `workflow_lane`, and the docstring "Return three
  Running houses...".

**Visual test + golden rename.** `tests/ace/tui/visual/test_ace_png_snapshots_agents_group_house_collapse.py` →
`..._agents_group_lane_collapse.py`. Its snapshot key `"agents_group_house_collapse_precedence_120x40"` names the golden
file on disk, so `git mv` the golden too:

```
tests/ace/tui/visual/snapshots/png/agents_group_house_collapse_precedence_120x40.png
  → agents_group_lane_collapse_precedence_120x40.png
```

Also update the snapshot `title=` kwarg ("ACE group-wide house collapse before status banner").

## Validation

Workspaces are ephemeral, so install first:

```bash
just install
just check
```

Then the visual suite:

```bash
just test-visual
```

**On the PNG golden.** Read this before reaching for the update flag. The `agents_group_house_collapse_precedence`
snapshot is captured _after_ the `H` press, at which point the test itself asserts the footer reads `collapse group` —
so the renamed chip is most likely **not** in that golden's pixels, and a pure `git mv` of the golden should keep it
passing. If a golden does fail, open the artifacts in `.pytest_cache/sase-visual/` and confirm from the diff that the
only change is footer chip text before accepting anything. Use `--sase-update-visual-snapshots` **only** to accept a
diff you have visually confirmed is this rename; never as a blanket way to make a red suite go green.

**Symvision.** Step 3 renames private symbols and moves a module, which can trip symvision's unused-symbol and
epic-whitelist checks. If `just check` reports symvision failures, use the `/sase_memory_read` skill on
`sase/memory/symvision.md` before touching pragmas or whitelists.

**Final sweep.** Confirm the vocabulary is actually gone:

```bash
grep -rni house src/sase/ace/ docs/ace.md docs/agent_families.md
```

The only surviving hits should be the three "Houses <X>" verb docstrings listed under Non-goals.

## Done when

- The footer chip in the screenshot's state reads `H collapse lanes`.
- Help modal, command palette, `Binding` descriptions, and `default_config.yml` comments all say "lane".
- `docs/ace.md` and `docs/agent_families.md` no longer describe the tree in terms of houses.
- No `house` identifier remains under `src/sase/ace/`.
- `just check` and `just test-visual` both pass.
- `H` ladder behavior is byte-for-byte unchanged: lanes → selected clan → remaining clans → group, with the same
  panel-focus ordering.
