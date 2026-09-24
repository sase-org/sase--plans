---
tier: tale
title: Restore Agents-only "node" and add Nav Section / Nav Item glossary terms
goal:
  The glossary defines "sase node" as an Agents-tab-only row again, adds generic "nav
  section" and "nav item" terms for every main tab's left-side navigation, drops the
  obsolete Service Node strand, and nothing in memory or Services-tab code still calls
  non-Agents rows nodes.
size: small
proposed_by: bbugyi200.athena.0rj
create_time: 2026-09-24 17:38:51
status: wip
---

# Plan: Restore Agents-only "node" and add Nav Section / Nav Item glossary terms

## Problem

Commit `3016e92d2` (service-host glossary strands) widened `glossary:sase-node` from
"one row of the Agents tab's agent tree" to "one selectable row of a hierarchical TUI
view", and added `glossary:service-node` so Services-tab rows were "nodes" too
(`3f5d34e9f` then re-touched both for the two-panel Services sidebar). That was a
mistake: "node" is an Agents-tab concept (the code's `agent_nodes.py`, `NodeSpine`,
agent-node folding), and we have no shared name for the thing all three main tabs
actually share — a left-side navigation panel and its selectable rows.

Fix: put "node" back to Agents-tab only, add two generic terms — **nav section** (a
left-side navigation panel on a main tab) and **nav item** (a selectable row in one) —
delete the Services-only node strand, and repair the strands and Services-tab code that
lean on the widened meaning.

The user asked for these memory changes in the prompt that produced this plan, and
approving this plan authorizes them. The implementer must still invoke
`/sase_memory_write` first and follow its "Edit And Republish" path: edit the canonical
strand files under `sase/memory/glossary/`, never hand-edit `AGENTS.md`, `CLAUDE.md`, or
other generated shims, then run `sase memory init`.

## Facts the definitions rest on (verified against the code)

- The TUI has exactly three main tabs: Agents, Artifacts, Services
  (`src/sase/ace/tui/tab_order.py`, `docs/ace.md` "Tab System"). Admin Center "tabs"
  (Procs, Projects, ...) belong to a modal, not the main tab bar.
- **Agents tab left column:** stacked tribe panels, one `AgentList` per effective tribe
  (`@default`, `@<tribe>`), or a single merged `All agents` panel
  (`actions/agents/_display_panel_titles.py`). Rows form a foldable tree: clan
  containers, agent nodes (standalone roots and family containers), family-member sase
  shells (agent shells, monitor `⚙` proc shells, gate `⋔` shells), workflow
  `python`/`bash`/`parallel` step children, and stand-alone `▣` proc shells.
  `models/agent_nodes.py::is_agents_tab_agent_node` says clan containers, family-member
  shells, step children, and durable shell rows "are rendered nodes but not ... agent
  nodes". A collapsed grouping banner can be selected. An expanded banner is skipped. A
  whole tribe panel can be selected (`❖`, "whole-panel focus").
- **Services tab left column:** two `BgCmdList` panels, **Service Procs** (service proc
  rows plus oneshot rows under a `── oneshots ──` divider) and **Scheduled Routines**
  (routine rows with nested job rows). `j`/`k` cross the panels and `J`/`K` jump between
  them.
- **Artifacts tab left column:** only one pane shows at a time (Agent, Stitch, Patch,
  Bead, a document provider such as Plan, File). Each pane has one entry list, with a
  relation panel rail under it, and the Patch pane also has a `PatchInfoPanel` above its
  list. Neither of those is a navigable list. The Patch pane's `j`/`k` stops include
  collapsed banners (`actions/patch/_grouping_nav.py`). The other panes skip headers
  such as the Beads `Tasks`/`Flags`/`Epics` headers.
- The word "section" is already used for in-list header groups: Beads/Plans headers,
  relation-panel sections, and `docs/ace.md` calling routine rows "top-level sections".
  The Nav Section definition must say a nav section is a whole panel.
- Glossary linking is `link_reference: implicit` with inline rendering. Any strand whose
  body mentions another strand's keyword or alias links it. So each new strand must
  mention "nav item", "nav section", and "sase node" in the singular somewhere to be
  sure the link is caught, and must not take a generic alias (`row`, `item`, `section`,
  `panel`) that would match unrelated text.

## Design decisions

1. **Keywords and aliases.** Use the keywords `Nav Section` and `Nav Item` (the user's
   terms), with the spelled-out aliases `navigation section` and `navigation item`.
   Their slugs are `nav-section` and `nav-item`.
2. **A nav item is a navigation stop, not an "object row".** It is any row that row
   navigation (`j`/`k`) can land on. That includes a collapsed grouping banner (Agents
   tab, Patch pane) while it is collapsed. This keeps statements like "`J` lands on the
   first nav item of the next nav section" literally true, which matches `docs/ace.md`
   ("Collapsed grouping banners count as rows"). Such a banner is still never a node.
   Panel titles, expanded banners, headers, dividers, and empty-state rows are chrome.
3. **Nodes are nav items. The reverse is false.** Every sase node is a nav item. Only
   the Agents tab has nodes.
4. **Sase Node goes back to the pre-`3016e92d2` text, with two fixes.** It names member
   rows as "member sase shell nodes", so gate shells (added later, `7bc0c0d98`) and
   monitors are covered. It says "stand-alone proc shell node" for `▣` roots. It keeps
   the old "Grouping banners and tribe-panel titles are not nodes" rule, adds the nav
   item relation, and drops the Services clause.
5. **Deletion audit.** `glossary:service-node` is the only strand that the rollback
   makes obsolete: it is the only strand defining a non-Agents "node". Every other
   strand was reviewed and kept. `glossary:agent-node` is an Agents-tab node subtype
   used all through the code (`agent_nodes.py`, agent-node folding, unread counts). It
   stays, but gets a one-sentence accuracy fix so it agrees with the restored Sase Node.
   The one Service Node fact worth keeping, that a service proc's row is stable across
   process restarts, moves into `glossary:service-proc`.

## Steps

### 1. Invoke `/sase_memory_write`

Record the skill use, then follow its Edit And Republish route for the following edits.

### 2. Rewrite `sase/memory/glossary/sase-node.md`

Keep the frontmatter (`keyword: Sase Node`, alias `node`). Replace the body with:

```markdown
A sase node is one row of the Agents tab's agent tree: an agent clan node, an agent node
(with its member sase shell nodes), an agent step node — a workflow `python`, `bash`, or
`parallel` step — or a stand-alone proc shell node. Every sase node is a nav item, but
nodes exist only on the Agents tab: Services and Artifacts nav items are not nodes.
Grouping banners (including selectable collapsed ones) and tribe-panel titles are not
nodes.
```

### 3. Add `sase/memory/glossary/nav-section.md`

```markdown
---
keyword: Nav Section
aliases:
  - navigation section
---

A nav section is one panel in the left-side navigation column of a main TUI tab; each of
its selectable rows is a nav item. On the Agents tab, every agent tribe panel
(`@default`, `@<tribe>`, or the merged `All agents` panel) is a nav section; on the
Services tab, the Service Procs and Scheduled Routines panels are; on the Artifacts tab,
each pane's entry list (Agent, Stitch, Patch, Bead, a document provider such as Plan, or
File) is one, and only the visible pane's list shows. A nav section is a whole panel:
grouping banners, in-list headers and dividers (Beads `Tasks`, the Services
`── oneshots ──` divider), the Artifacts relation and Patch info panels, right-side
detail panels, and Admin Center lists are not nav sections.
```

### 4. Add `sase/memory/glossary/nav-item.md`

```markdown
---
keyword: Nav Item
aliases:
  - navigation item
---

A nav item is one selectable row in a nav section — a stop for row navigation such as
`j`/`k`. On the Agents tab, every sase node is a nav item; on the Services tab, every
service proc, oneshot, routine, and job row is; on the Artifacts tab, every entry in the
visible pane's list (an agent, stitch, Patch, bead, document, or file) is. A collapsed
grouping banner on the Agents tab or Patch pane is a nav item while collapsed, yet never
a node. Panel titles, expanded banners, headings, dividers, and empty-state rows are
chrome, not nav items, and a selected whole tribe panel (`❖`) is a selected nav section,
not a nav item.
```

### 5. Delete `sase/memory/glossary/service-node.md`

Remove it with `git rm`. The roster in `sase/memory/glossary.md` sits between the
`<!-- sase:strands -->` markers and is regenerated by `sase memory init`. Do not
hand-edit it.

### 6. Fix the dependent strands

- `sase/memory/glossary/agent-node.md`: replace "A family node's member rows are agent
  shell nodes as well." with "A family node's member rows are sase shell nodes, not
  agent nodes." A family's members include monitor and gate shells, and
  `is_agents_tab_agent_node` excludes all family-member rows.
- `sase/memory/glossary/service-proc.md`: replace "Control one with `sase service proc`
  or its Services-tab node; ..." with "Control one with `sase service proc` or its
  Services-tab nav item, which stays the same row across restarts; ...". Leave the rest
  of the body unchanged.

Before moving on, grep `sase/memory/` for any remaining `service node`, `service-node`,
or Services-tab "node" wording and fix it.

### 7. Align the Services-tab code wording (code and tests only, no behavior change)

Several Services-tab docstrings and comments still call their rows "nodes". Change them
to "nav item" (or "row" where that reads better):

- `src/sase/ace/tui/actions/axe_display/_panel_navigation.py`: the docstrings of the
  `J`/`K` panel-jump helpers ("first/last node", "has nodes").
- `src/sase/ace/tui/actions/axe_display/_panels.py`: the `adjacent_nonempty_panel`
  docstring ("at least one node", "has nodes").
- `src/sase/ace/tui/actions/axe_display/_panel_titles.py`: rename the
  `ServiceProcsPanelStats.nodes` field to `items`, including its constructor kwarg and
  the `stats.nodes` read in the title builder. Update
  `tests/ace/tui/test_services_panel_titles.py` (the two `stats.nodes` asserts).
- `src/sase/ace/tui/widgets/bgcmd_list.py`: in the `empty_placeholder` docstring, change
  "It is not a node" to "It is not a nav item".
- `src/sase/ace/tui/widgets/_bgcmd_list_styles.py`: in the module comment, change
  "daemon service nodes" to "daemon service proc rows".
- `tests/ace/tui/test_axe_service_panel_jumps.py`: rename
  `test_j_from_routines_wraps_to_first_service_node` to
  `test_j_from_routines_wraps_to_first_service_proc`. No other file references it.
- `tests/ace/tui/visual/test_ace_png_snapshots_services_panels.py`: in the docstring,
  change "first routine node" to "first routine row". Change only the docstring, not the
  test name, so PNG goldens are untouched.

Leave Agents-tab "node" wording alone (`agent_nodes.py`, `NodeSpine`, agent-node
folding). It is correct under the restored definition.

### 8. Republish and verify memory

1. Run `sase memory init`. It regenerates the glossary roster, `AGENTS.md`, every
   provider shim, and `sase/memory/README.md`.
2. Check that the Memory Webs glossary roster now lists `Nav Item (navigation item)` and
   `Nav Section (navigation section)`, still lists `Sase Node (node)`, and no longer
   lists `Service Node`.
3. Run
   `sase memory read glossary:sase-node glossary:nav-section glossary:nav-item glossary:agent-node glossary:service-proc -r "Verify revised node/nav glossary strands render and cross-link"`.
   Confirm that all five render, and that Nav Item's closure inlines Nav Section and
   Sase Node (implicit links resolve).
4. Run `sase memory read glossary:service-node -r "Confirm the deleted strand is gone"`
   and confirm it fails as an unknown selector.

### 9. Lint and test

Read `sase memory read lint_and_test.md` and follow it. Run `just fmt` if Markdown
wrapping changed, then `sase tool run check` (not `check-full`). Fix any failures the
changes introduce.

## Out of scope

- Renaming Agents-tab code, and adding nav-section or nav-item wording to `docs/ace.md`.
  The docs already say "panel" and "row" consistently.
- Any glossary strand other than those listed above.
