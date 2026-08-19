---
tier: tale
title: Show Agents-tab row titles only for workflow bash/python steps
goal:
  Family member agent shells and monitor proc shells no longer show a step-style
  left-side title on the Agents tab; only xprompt workflow bash and python steps keep
  those names, while shell identity stays on the right-hand %id annotation.
size: medium
proposed_by: bbugyi200.athena.08d
create_time: 2026-08-19 18:35:52
status: wip
---

# Plan: Show Agents-tab row titles only for workflow bash/python steps

## Problem

The Agents tab always paints a left-side title on every non-clan row. That title is
correct for xprompt workflow `bash` / `python` steps (`setup`, `prepare`, `checkout`,
`diff`), where the step name is the only identity the row has. The same path also titles
**sase shells** — agent shells and proc shells — so family members look like named
workflow steps.

The `08b` family in `.sase/artifacts/home/tmp/screenshots/20260819_180741.png` shows the
three unwanted titles next to the titles that should stay:

| Row                                                | What is painted today                                     | What it actually is          | Wanted left-side title               |
| -------------------------------------------------- | --------------------------------------------------------- | ---------------------------- | ------------------------------------ |
| `main (TALE APPROVED) 08b--plan`                   | workflow agent step name `main` (`cl_name` / `step_name`) | planner agent shell          | none — keep `08b--plan` on the right |
| `sase (TALE DONE) 08b--code`                       | `Agent.display_name` → project display name               | coder agent shell            | none — keep `08b--code` on the right |
| `research-swarm priority check (TESTING) 08b--mon` | `monitor_label`                                           | monitor proc shell           | none — keep `08b--mon` on the right  |
| `setup` / `prepare` / `checkout` / `diff`          | workflow step name                                        | bash/python agent-step nodes | keep the step name                   |

`docs/ace.md` and the glyph comment in `src/sase/ace/tui/widgets/_agent_list_styling.py`
encode the wrong assumption: "agent rows already carry a meaningful display name." They
do not. A shell's identity is the `%id` / family-member name (`08b--plan`), already
rendered after the status in `_AGENT_NAME_ANNOTATION_STYLE`.

## Do we need these names on sase shells?

Audit conclusion: **no extra shell title is required**. Do **not** delete the stored
fields that people might confuse with that title.

Keep writing and reading:

- `agent_meta.json` `name` / `agent_name` / `presented_agent_name` (`08b--plan`,
  `08b--code`, `08b--mon`). These are the sase-shell identity used by `%id`, `%wait`,
  attach, restart, CLI `sase agent list`, completions, and the right-hand Agents-tab
  annotation. They are not the titles this plan removes.
- Workflow `step_name` on **every** YAML step, including `agent` steps. The executor
  keys markers (`prompt_step_{name}.json`), outputs (`{{ step_name.field }}`), and HITL
  on that name. The planner's `main` step must keep `step_name="main"` on disk even
  though the tree stops showing it.
- `monitor_label` (and `monitor_command`) on monitor members. The detail panel,
  `sase agent list --json`, stop/kill confirmations, and completions still need a short
  command description. The tree must stop using `monitor_label` as the row title. Do
  **not** drop `monitor_label` from `create_monitor_member()` or the scan wire.
- `cl_name`, `project_display_name`, and `Agent.display_name` for **roots, family
  containers, and clan containers**. Top-level `sase (TESTING) 08a` stays. Only child
  **shells** lose the left-side title.

This is presentation in this repo. Do not change `sase-core`, the scan wire, or
`agent_meta.json` schema. No feature flag: the old titles must not remain a
user-selectable branch.

## Behavioral contract

Introduce one predicate used by every Agents-tab **tree** title:

A row shows a left-side title if and only if:

1. it is a workflow step child whose `step_type` is `bash` or `python`, or
2. it is **not** a sase shell child: a clan container, a sequential-family container, or
   a standalone root / clan-member agent node.

A row shows **no** left-side title when it is:

- a family-member child (`parent_timestamp` set, not a workflow step),
- a monitor / proc shell (`is_monitor`),
- a workflow `agent` step (including the planner `main` step that
  `concrete_family_member_rows` treats as the first family member),
- any other agent shell nested under a parent.

After the status, keep painting `presented_agent_name or agent_name` exactly as today
(`08b--plan`, `visual-nav--code`). Bash/python steps typically have no `%id`; their step
name remains the only label, plus the existing `❯` glyph and step-type color.

Concrete `08b` / visual-nav targets:

```text
08b
  🎭 (TALE APPROVED) 08b--plan
  🤖 (TALE DONE) 08b--code
  ⚙ (TESTING) 08b--mon
  ❯ setup (DONE) ▲#gh
  ❯ prepare (DONE) ▲#gh
```

(Provider emoji and `#gh` badges stay on the rows that already have them.)

### Parallel and `prompt_part`

`parallel` steps are agent-step nodes, not sase shells. They already have no `❯` glyph
(children carry the structure). Do **not** invent a new title for them. If a parallel
row already uses `display_name`/`step_name` as its title today, leave that as-is; it is
out of the "sase shell" set. `prompt_part` rows stay hidden by default.

### Other surfaces

Apply the same "no shell title" rule anywhere a **tree-style** title is copied from
`display_name` / `monitor_label` / `step_name` for a shell:

- `format_agent_option` in `src/sase/ace/tui/widgets/_agent_list_render_agent.py`
  (required).
- Detail-header `Step:` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`: keep
  `Step: setup` for bash/python; do **not** show `Step: main` for an `agent` step.
  `Name:` already uses `presented_agent_name` and is correct.
- Family roster (`family_member_label`) already prefers the `--plan` / `--code` suffix
  after stripping the family name. Leave it unless a test shows it still falls through
  to `step_name="main"`.
- Jump-all / zoom / cleanup / revive lists: only change them if they currently present
  `main`, the project display name, or `monitor_label` as the primary label **for a
  shell**. Prefer `presented_agent_name or agent_name` there. Do not rewrite project
  pickers or plugin `display_name`s.

Do **not** change `Agent.display_name` itself. Too many callers (kill labels, cleanup
modals, jump-all path suffixes, project-scoped roots) depend on "project name or
humanized `cl_name`". Add a dedicated helper instead.

CLI `sase agent list` already prints `agent.name` (`08b--code`), not the tree title.
Leave it.

## Implementation

### 1. Shared helper

Add a focused helper next to the Agents-tab model (preferred:
`src/sase/ace/tui/models/agent.py` or a tiny sibling used by the renderer), e.g.
`agent_tree_title(agent) -> str | None`:

- `bash` / `python` workflow step child → `step_name or display_name`
- clan container / family container / non-child root → `display_name` (today's string)
- monitor, family-member child, workflow `agent` step, other child shells → `None`

Keep it allocation-light and side-effect-free: `format_agent_option` and the render
cache run on the UI path (`sase/memory/tui_perf.md`). The helper may only read
already-loaded `Agent` fields.

Update `_agent_list_render_cache.agent_render_key` only if the painted title is no
longer a pure function of fields already in the key (`display_name`, `monitor_label`,
`step_type`, `is_workflow_child`, `is_monitor`, `agent_name`). Prefer deriving the title
from those existing inputs so the cache key does not grow.

### 2. Tree renderer

In `format_agent_option`:

- Stop using `agent.monitor_label if agent.is_monitor else agent.display_name` as an
  always-on title.
- Append the helper's title only when it is a non-empty string.
- Keep reverted strike styling, tribe suffix, and the
  `status_opener = "(" if row_prefix ends with space else " ("` rule so title-less
  children render as `  ⚙ (TESTING)` rather than `  ⚙(TESTING)`.
- Keep monitor **glyph**, provider emoji (agent shells only), bash/python `❯`, identity
  annotation, and `#embedded` badges.

Update the stale comment in `_agent_list_styling.py` (lines 94–97) and the matching
paragraph in `docs/ace.md` (~1931–1937): bash/python steps are named because that name
is their identity; sase shells are identified by the right-hand `%id` / family-member
name, not a step-style title.

### 3. Detail header

In `_append_project_fields`, gate `Step:` on
`is_workflow_step_child and step_type in {"bash", "python"}` (plus a non-empty
`step_name`). Agent-step shells then fall through to Patch/Project like other shells.
`Name:` is unchanged.

### 4. Tests

Add or rewrite focused `format_agent_option` tests (alongside
`tests/ace/tui/widgets/test_agent_list_monitor_rows.py` and
`tests/ace/tui/widgets/test_agent_list_provider_emoji_badges.py`):

- Family planner workflow `agent` step titled `main` with `agent_name="08b--plan"`: left
  text has no `main`; does contain `(TALE APPROVED)` (or whatever status) and
  `08b--plan`.
- Family coder `RUNNING` child that is a project agent (`display_name == "sase"`): left
  text has no project display name; does contain `08b--code`.
- Monitor with `monitor_label="research-swarm priority check"`: no label and no command
  on the row; still has `⚙` and `08b--mon`. Flip
  `test_monitor_row_uses_glyph_and_label_without_the_command` to
  `test_monitor_row_uses_glyph_without_label_or_command`.
- Bash and python children: still `❯` + step name + status.
- Family container and standalone project root: still show `08b` / `sase`.
- Title-less child: status parenthesis is separated from the indent/glyph.

Keep monitor **detail-panel** tests that still expect `just check-full` /
`monitor_label` in the prompt panel.

Update PNG goldens that show family trees with `main`, a project-name child title, or a
monitor label. Known fixtures include
`tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py`
(`workflow_step("main", "agent", 0)`, `cl_name="visual-house-navigation-code"`) and
snapshots such as `agents_python_step_parent_family_120x40.png`,
`agents_auto_approve_workflow_child_alignment_120x40.png`,
`agents_waiting_family_child_120x40.png`, and other family/monitor goldens that fail
after the renderer change. Accept intentional diffs with
`--sase-update-visual-snapshots`. Inspect `.pytest_cache/sase-visual/` before accepting.

### 5. Verification

From the workspace: `just install`, then `just check`. If scoped selection misses the
new render tests, run them directly as well. Run `just test-visual` when goldens change.
If `just check` escalates or this touches the broadening set, hand `just check-full` to
`/sase_monitor` (`TESTING` / `TESTED`).

## Out of scope

- Renaming family members, changing `%id` / `--<suffix>` allocation, or renaming the
  planner workflow step off `main`.
- Removing `monitor_label` from metadata or the CLI JSON envelope.
- Hiding titles on standalone roots or clan-member agent nodes (those rows _are_ the
  agent; the left title is the project or workflow name, not a fake step name).
- Changing family roster numbering, fold glyphs, or unread projection (`agent_nodes.py`
  already documents that the `main` step's `cl_name` is the step name for completion-key
  matching — leave that).
- Rust core / scan-wire changes.
- A feature flag or config knob to restore shell titles.
