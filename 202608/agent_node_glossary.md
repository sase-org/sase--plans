---
tier: tale
title: Rename the "Agent Lane" glossary term to "Agent Node" and add "Sase Node"
goal:
  The SASE glossary defines "Agent Node" in place of "Agent Lane" and adds "Sase Node"
  (alias "node") for any Agents-tab agent-tree row, with agent-facing docs and
  user-visible keymap strings using the new vocabulary and the four unrelated meanings
  of "lane" left untouched.
size: medium
proposed_by: bbugyi200.athena.03l
create_time: 2026-08-16 11:01:44
status: wip
---

# Plan: Rename the "Agent Lane" Glossary Term to "Agent Node" and Add "Sase Node"

## Goal

1. Rename the SASE glossary term **Agent Lane** to **Agent Node**.
2. Add a new glossary term **Sase Node** (alias `node`) covering any row of ACE's Agents
   tab agent tree.
3. Propagate the rename to the places that use "lane" in _that_ sense — agent-facing
   docs and user-visible keymap descriptions — without disturbing the four unrelated
   meanings of "lane" that also live in this repo.

## Background The Implementer Must Know First

### The glossary is generated, not hand-written

`sase/memory/glossary.md` is **generated output**. Its source of truth is the
`memory.glossary` mapping in `sase/sase.yml`. `src/sase/main/init_memory/glossary.py`
reads that mapping, validates it through the Rust core
(`sase_core::glossary::validate_glossary_entries`), and renders the note with a
`sase_generated: glossary` frontmatter marker.

**Never hand-edit `sase/memory/glossary.md`.** Edit `sase/sase.yml`, then run:

```bash
just install        # workspaces are ephemeral; deps may be stale
sase memory init
```

That regenerates `sase/memory/glossary.md`, `sase/memory/README.md`, `AGENTS.md`, and
the provider instruction shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). The
per-term bullet list that appears in the tier-2 section of `AGENTS.md` / `CLAUDE.md` is
derived from the same catalog, so it updates automatically — do not edit it by hand.

The user has explicitly authorized this memory-file change in the originating
conversation, so running `sase memory init` afterwards is mandatory and needs no further
permission.

### "Lane" has FIVE unrelated meanings in this repo

This is the single largest risk in this task. A blind find-and-replace will corrupt four
subsystems. Only sense (1) is being renamed.

| #   | Sense                                  | Meaning                                                                                                                                             | Representative sites                                                                                                                                                                                                                               | Action     |
| --- | -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| 1   | **Agents-tab row** (the glossary term) | The display container for a sase agent on ACE's Agents tab; the unit that folds, counts, and dismisses                                              | `docs/ace.md` fold/grouping/counting sections, `docs/agent_families.md` fold sections, `src/sase/ace/tui/bindings.py`, `src/sase/ace/tui/keymaps/metadata.py`, `src/sase/default_config.yml` keymap description                                    | **RENAME** |
| 2   | **Detail-panel section**               | A labelled row group inside an agent's detail panel: `PLAN` lane, `BEAD` lane, `ARTIFACTS` lane, `COMMITS` lane, per-member model lanes, wait lanes | `docs/ace.md` SASE CONTEXT sections, `docs/agent_families.md` "Per-member model lanes", `src/sase/ace/tui/widgets/prompt_panel/*`, `docs/perf_runbook.md`, `docs/agent_images.md`, `docs/sdd.md`, `docs/cli.md`, `docs/beads.md`                   | **LEAVE**  |
| 3   | **Axe / lumberjack lane**              | A scheduled lane of background work (hooks lane, checks lane, fast lane)                                                                            | `docs/axe.md`, `docs/configuration.md`, `src/sase/default_config.yml` lumberjack descriptions, `src/sase/config/sase.schema.json` (`wait_runners`), `src/sase/external_mirror/state.py`, `docs/blog/posts/why-coding-agents-need-orchestration.md` | **LEAVE**  |
| 4   | **Monitor lane**                       | The durable agent family a monitor attaches to; also the deprecated `--lane` CLI alias                                                              | `src/sase/monitor/*`, `src/sase/bead/epic_launch.py`, `src/sase/main/parser_monitor.py`, `docs/monitors.md`, `src/sase/xprompts/skills/sase_monitor.md`                                                                                            | **LEAVE**  |
| 5   | **Misc**                               | Test lane (`just test-scoped`), routing/role-alias lane, pill "lane color", CI visual lanes                                                         | `sase/memory/build_and_run.md`, `docs/development.md`, `docs/ace.md` lines about model-alias routing and top-bar pills                                                                                                                             | **LEAVE**  |

### The Agents-tab node taxonomy (verified against the code)

The user's proposed list was checked against `src/sase/ace/tui/models/` and
`src/sase/ace/tui/widgets/_agent_list_*`. Findings:

Confirmed node kinds (all are `Agent`-model rows in the agent tree):

- **agent clan node** — `Agent.is_clan_container`; a selectable synthetic container,
  never an agent.
- **agent node** — a sase agent's row. Either an **agent family node** (a real
  multi-member family root) or an **agent shell node** (a sase agent with no family).
  Family _member_ rows are also agent shell nodes, nested one level in.
- **agent step node** — a workflow child row (`AgentChildLinkage.WORKFLOW_STEP`). Step
  types are `python`, `bash`, `parallel`, `prompt_part`, and `agent`. Only `python` and
  `bash` get their own 🐍 / 🐚 glyph; `agent` steps are ordinary agent nodes, and
  `prompt_part` rows are invisible by default.
- **proc shell node** — a monitor shell: a family member whose work is a supervised
  command (amber `⏱` glyph, `sase monitor start`).

Kinds the user's list did not mention — resolve these as directed:

- **Grouping banner rows** (`GroupRow`: project / Patch / date / status / name buckets)
  and **split-panel titles**. These are rows on the Agents tab but are not part of the
  agent tree; they are a separate model type and a separate folding layer. **Decision:
  exclude them from "sase node" and say so in one clause**, so the term stays useful.
  Flag this to the approver if you disagree.
- **Synthetic planner rows** (`Agent.is_synthetic_planner`, set in
  `src/sase/ace/tui/models/_agent_status_family_planner.py`) — display-only child rows
  standing in for the planner segment of a promoted plan family. These are `Agent` rows,
  so they fall under agent shell nodes. No separate glossary kind needed.
- **Prior-attempt rows** (`↳ Attempt N`, rendered by
  `src/sase/ace/tui/widgets/_agent_list_render_attempt.py`). **Verify** whether these
  are selectable/navigable rows. If they are, they are sub-rows of an agent shell node,
  not nodes in their own right — keep them out of the type list and do not add a clause
  for them unless the definition would otherwise read as wrong.

## Task 1 — Edit the glossary source (`sase/sase.yml`)

### 1a. Replace the `Agent Lane:` entry

Delete the `Agent Lane:` block (currently sits between `Agent Instruction File:` and
`Agent Neighbor:`) and insert an `Agent Node:` entry **after** `Agent Neighbor:` — the
mapping is maintained in alphabetical order and `Neighbor` sorts before `Node`.

```yaml
Agent Node:
  definition: >-
    An agent node is the Agents-tab node for a non-dismissed sase agent: an agent family
    node, or an agent shell node when the agent has no family. A family node's member
    rows are agent shell nodes as well. Dismissal removes the node, not the sase agent's
    identity.
```

This is a 1:1 translation of the old definition plus one clarifying sentence, so every
existing doc sentence about "lanes" translates directly to "agent nodes".

### 1b. Add the `Sase Node:` entry

Insert **after** `Sase Agent:` and before `Sase Project:`.

```yaml
Sase Node:
  aliases:
    - node
  definition: >-
    A sase node is one row of the Agents tab's agent tree: an agent clan node, an agent
    node (with its member agent shell nodes), an agent step node — a workflow `python`,
    `bash`, or `parallel` step — or a proc shell node. Grouping banners and tribe-panel
    titles are chrome, not nodes.
```

Notes on the alias:

- Author only `node`. Plurals are derived automatically by the Rust catalog and are
  dropped from `display_aliases`; authoring `nodes` would be redundant (compare the
  `Proc` entry, whose authored `procs` / `background tasks` plurals never render).
- `node` is a common word, so it will highlight in prompts that mean a graph node or
  Node.js. This matches existing practice (`agent`, `shell`, `project`, `repo`,
  `workspace`, `ref` are already aliases) and is what the user asked for. Do not
  second-guess it.
- Validation only rejects _exact_ alias collisions (`alias_conflict`), and `node`
  collides with nothing. The matcher prefers the longest span at a given start offset,
  so `agent node` and `sase node` both win over the bare `node` alias.

### 1c. Sanity-check the entries

Both definitions must stay on the `>-` folded-scalar form, wrap inside the file's
existing column budget, and avoid literal line breaks inside a single term or alias
(`multiline_term` / `multiline_alias` are hard validation errors).

## Task 2 — Regenerate the derived memory files

```bash
just install
sase memory init
```

Confirm the regenerated `sase/memory/glossary.md` contains `## Agent Node` and
`## Sase Node` with `ALIASES: node`, contains no `## Agent Lane`, and that `AGENTS.md`,
`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and `sase/memory/README.md` all
picked up the new term list. Read the regenerated glossary with
`sase memory read glossary.md`, not with a direct file read.

## Task 3 — `docs/ace.md`

`docs/ace.md` has ~76 `lane` occurrences split roughly evenly between sense (1) and
senses (2)/(5). Rename **only** sense (1).

Sense-(1) regions to convert (line numbers are from the pre-change file and will drift —
classify by meaning, not by number):

- The jump-key and fold-mode key tables (~858, 860, 933, 936).
- The `h` / `l` / `H` / `L` / `-` fold-ladder prose and its three-layer table
  (~1447–1494, 1563–1574).
- The clan/family tree section (~1596).
- Grouping and ordering prose — "standalone lanes precede name subgroups" (~1684,
  1700–1702).
- The count chip and status-chrome paragraphs (~1727, 1750, 1753).

Sense-(2)/(5) regions that MUST keep the word "lane": the `SASE CONTEXT` / `BEAD` /
`PLAN` / `ARTIFACTS` / `COMMITS` lane prose (~390, 1018, 1054–1123, 1198, 1308,
3546–3620), the top-bar pill "lane color" sentence (~2800), and the role-alias routing
"lane" sentence (~2833).

### Watch for sentences whose meaning shifts

The old term meant _top-level_ container only. The new term also names nested member
rows. Where a sentence counts or folds only top-level units, add the qualifier rather
than substituting blindly. Examples:

- "The clan row's fold count and status chrome count those direct clan **lanes** once;
  nested family or workflow members do not inflate them." → keep the contrast explicit,
  e.g. "those direct clan **agent nodes**".
- "The **lane** total is followed by an always-visible capacity chip" → "The
  **agent-node** total" only if it still means top-level rows; verify against
  `src/sase/ace/tui/models/agent_summary*` / the count-chip implementation before
  wording it.
- "counts terminal **lanes** that still need acknowledgement" and the `STARTING`
  exclusion sentence — same check.

## Task 4 — `docs/agent_families.md`

Rename sense (1) at ~180, 181, 186, 230, 523, 539, 548.

Keep sense (2) at ~95 (`PLAN-lane presentation`), ~213 (`context-lane summaries`), and
the whole "Per-member model lanes" subsection (~378–401) including its heading.

Two sites need real editing, not substitution:

- **~360**: `See [Lane Neighbors Section](ace.md#lane-neighbors-section)` is an
  **already-broken anchor** — the heading in `docs/ace.md:1159` is
  `### Sase Agent Neighbors Section`. Fix the link text and anchor to match the real
  heading while you are here.
- **~375**: "A member row owns no **lane**, so its …" asserts that member rows are not
  lanes. Under the new taxonomy a member row _is_ an agent shell node, so reword to say
  what is actually true — the member row owns no fold of its own (verify the surrounding
  `Fold: N/M` header claim before settling on wording).

## Task 5 — User-visible strings and keymap config

Rename sense (1) in:

- `src/sase/ace/tui/bindings.py` — the `H` binding description
  `"Collapse Selected Workflow/Family / Scoped Lanes/Clans/Groups / Hint Panel Fold / Compact Tools / All"`.
- `src/sase/ace/tui/keymaps/metadata.py` — the same string, kept in sync.
- `src/sase/default_config.yml` — the `H` keymap description (~line 367, "fully
  collapses remaining lanes in the …"). Per the repo's keymap convention, a keymap
  description change is not complete until `default_config.yml` matches.

Do **not** touch `src/sase/default_config.yml`'s lumberjack descriptions (sense 3) or
`src/sase/config/sase.schema.json`'s `wait_runners` description (sense 3).

## Task 6 — Sense-(1) internal docstrings and private symbols

Bounded, mechanical, and worth doing so the code reads consistently with the glossary.
Confirm each hit is sense (1) before changing it.

- `src/sase/ace/tui/models/sase_agent_neighbors.py` — module and class docstrings
  ("lane-relative projection", "one lane's shared modal/panel projection", "one
  lane-owning agent").
- `src/sase/ace/tui/actions/agents/_folding_sase_agents.py` — docstrings and comments
  about lane-collapse actions and lane owners.
- `src/sase/ace/tui/actions/navigation/_fold.py` — the three docstrings at ~294, ~315,
  ~336.
- `src/sase/ace/tui/models/fold_scale.py` — the docstring at ~48.
- `src/sase/agents_sync/rendering_kinship.py` — module docstring plus the private
  symbols `_Lane`, `_LaneKinshipProjection`, `_LaneKinshipGroup`, `_lane_projection`,
  `_lane_sort_key`, and the `lanes` locals/fields. All are private to the module; rename
  them and their references in `tests/` together. Verify nothing outside the module
  imports them before renaming.

## Explicitly Out Of Scope

Do not change these, and do not file them as defects — they are deliberate:

- `src/sase/agent_lanes.py`, `AgentLaneRef`, and the `lane_name` / `lane_page_path` /
  `lane_ref_for_agent` / `lane_ref_for_lane_name` aliases in `src/sase/sase_agent.py`.
  These are a frozen legacy compatibility surface that also backs **serialized `lane_*`
  field aliases**; renaming them would break persisted data. Their docstring
  ("Historical imports used the agent-lane vocabulary") stays accurate after this
  change.
- `tests/test_agent_lanes.py`, which exists to pin that compatibility surface.
- Every sense (2), (3), (4), and (5) site listed in the table above.
- `CHANGELOG.md` history.

## Verification

```bash
just install
just check
```

Then confirm by inspection:

1. `rg -i 'agent[ _]lane' -- ':!CHANGELOG.md' ':!src/sase/agent_lanes.py' ':!src/sase/sase_agent.py' ':!tests/test_agent_lanes.py'`
   returns nothing.
2. Every remaining `rg -in '\blanes?\b' docs/ace.md docs/agent_families.md` hit is a
   detail-panel section, a per-member model lane, a pill-color mention, or a role-alias
   routing lane — i.e. no hit describes an Agents-tab row.
3. `sase memory read glossary.md` shows `## Agent Node` and `## Sase Node`
   (`ALIASES: node`) and no `## Agent Lane`.
4. `git status` shows `sase/memory/glossary.md`, `sase/memory/README.md`, `AGENTS.md`,
   `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md` as regenerated, not
   hand-edited.
5. The ACE Agents tab still renders and folds correctly — the code changes are strings
   and private symbols only, so `just check` plus the ACE TUI test suite is sufficient
   evidence; no manual TUI run is required.

If `just check`'s scoped test lane escalates or reports an unusual selection (this
change touches `default_config.yml` and generated instruction files), run
`just check-full` through `/sase_monitor` with a `--next` action rather than inline.

## Decisions For The Approver

1. **Grouping banners and tribe-panel titles are excluded from "sase node."** The user's
   phrasing was "any row on the agents tab," which would literally include them, but
   they are a distinct model type (`GroupRow`) and a distinct folding layer, and the
   user's own type list omits them. Say the word if they should be folded in instead.
2. **"Agent step node" is the user's name for workflow child rows.** These are _xprompt
   workflow_ steps, so "workflow step node" would be marginally more precise; the plan
   keeps the user's chosen name since the parent is always a workflow agent.
3. **`node` as an alias will highlight ordinary English "node"** in the ACE prompt
   editor and LSP. Consistent with existing generic aliases; called out so it is not a
   surprise.
