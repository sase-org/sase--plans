---
tier: tale
title: Improve the Help panel Guide-tab content for all three ACE tabs
goal: The Help panel's Guide (2nd) tab reads as an accurate, well-scoped orientation
  for each ACE tab — the Artifacts guide covers all four sub-tabs instead of only
  PRs, and the Agents and Axe guides are tightened for accuracy and scannability.
create_time: 2026-07-23 13:28:14
status: wip
---

- **PROMPT:** [202607/prompts/help_guide_tab_content.md](prompts/help_guide_tab_content.md)

# Plan: Improve the Help panel Guide-tab content

## Context

The Help panel (opened with the leader `,?` keymap) has two panel tabs: **Keymaps** (1st) and **Guide** (2nd).
`HelpModal` (`src/sase/ace/tui/modals/help_modal/modal.py`) builds the Guide tab via `build_guide_view()`
(`src/sase/ace/tui/modals/help_modal/guide_view.py`), which returns one of three onboarding widgets depending on the
active ACE tab:

- Artifacts tab (`changespecs`) -> `ChangeSpecOnboarding` (`src/sase/ace/tui/widgets/changespec_onboarding.py`)
- Agents tab (`agents`) -> `AgentOnboarding` (`src/sase/ace/tui/widgets/agent_onboarding.py`)
- Axe tab (`axe`) -> `AxeOnboarding` (`src/sase/ace/tui/widgets/axe_onboarding.py`)

**Scope is clean.** These three widget classes are referenced **only** from `guide_view.py` (confirmed by grep). The
full-tab empty-state onboarding uses a different widget, `TabQuickStart`, so editing these three widgets changes
**only** the Help panel's Guide tab and does **not** touch the empty-state panels. This work is purely copy/content plus
the tests and PNG snapshots that pin that copy.

Shared Rich-text helpers live in `src/sase/ace/tui/widgets/_onboarding_common.py` (`append_section_heading`,
`append_keycap`, `append_doc_link`, `key_sequence_display`) and should be reused; do not invent new formatting
primitives. All keybinding hints must keep coming from the `KeymapRegistry` (via `key_display_name(app.<action>)` /
`leader_key_display(...)`) so the copy can never hardcode a key.

## Problems with the current content

### 1. Artifacts guide only talks about PRs (highest-value fix)

The ACE tab whose internal id is `changespecs` is user-facing **"Artifacts"** (`src/sase/ace/tui/widgets/tab_bar.py`,
`binding_common.TAB_DISPLAY_NAMES`). It has **four sub-tabs**, ordered Commits (default), Plans, Bugs, PRs
(`src/sase/ace/tui/artifact_tabs.py`: `ARTIFACTS_SUBTAB_ORDER`), navigated with `1`/`2`/`3`/`4` (jump), `[`/`]` (cycle),
and `p` (pick scope) — see the "Artifact Sub-tabs" section in
`src/sase/ace/tui/modals/help_modal/changespecs_bindings.py`.

But `ChangeSpecOnboarding` is framed entirely around PRs: the hero says "Your agents' work, shipped as PRs", and every
card describes ChangeSpecs/PRs only. A user who opens `,?` -> Guide while viewing the Commits, Plans, or Bugs pane gets
no orientation for what they are looking at. The guide should first map the whole Artifacts tab (its four sub-tabs) and
then keep the existing PR-specific depth.

Note the Agents guide already describes this tab correctly as "Artifacts / Browse commits, plans, bugs, and PRs in one
place" (`agent_onboarding._TAB_ROWS`), so the Artifacts guide is the one that has drifted.

### 2. Axe guide's "Lumberjacks own chops" card is overloaded

`_build_chops_card` in `axe_onboarding.py` packs three long dim explanatory paragraphs (source scope + raw YAML +
generated rows; save-&-restart vs save-only; per-run history) between the keycap lines. It reads like reference docs
rather than an orientation card and is hard to scan. Trim to the essential actions and one short supporting line each;
leave the deep detail to the `https://sase.sh/axe/` doc link that the "Learn more" card already provides.

### 3. Agents guide "Inspect the results" jumps straight to keys

`_build_inspect_card` lists keybindings without first anchoring what an agent is or the live-vs-finished distinction
(you can watch a running agent, or read a finished agent's transcript). One short orienting line makes the keycaps land
better.

## Editorial principles (apply to all three guides)

- Lead each guide with a one- or two-line "what this tab is" map, then drill in.
- Every actionable line pairs a registry-driven keycap with a short imperative.
- Keep long-form reference in the doc links, not the cards.
- Preserve the existing card/hero structure, accent colors, and helper functions; this is a content revision, not a
  widget rewrite.

## Changes by guide

### A. `ChangeSpecOnboarding` (Artifacts guide) — `changespec_onboarding.py`

1. **Reframe the hero** (`_build_hero`) from PR-only to the whole Artifacts tab. Proposed: a headline like "Everything
   your agents produce, in one place" with a subtitle naming the four views (commits, plans, bugs & PRs). Keep the
   centered gold-star styling.

2. **Add a new "sub-tabs overview" card** as the first card after the hero, modeled on the Agents guide's
   `_build_tabs_card` (reuse `append_section_heading` plus a per-row helper). List the four sub-tabs in
   `ARTIFACTS_SUBTAB_ORDER` (Commits, Plans, Bugs, PRs) with their `ARTIFACTS_ACCENTS` colors and a one-line description
   each, then the navigation hints: `1`/`2`/`3`/`4` jump, the `cycle_artifacts_subtab_reverse`/`cycle_artifacts_subtab`
   (`[`/`]`) cycle, and `pick_artifacts_project` (`p`) scope picker — all pulled from the registry. Prefer sourcing the
   order/labels/accents from `artifact_tabs.py` so the card cannot drift from the real sub-tab set.

3. **Keep and lightly re-scope the PR-specific cards.** Retain "What is a ChangeSpec?" (keep the "One ChangeSpec = one
   PR" line and the WIP->...->Submitted lifecycle), "How ChangeSpecs get here", and "Work the queue", but make clear
   they describe the **PRs** sub-tab specifically (e.g. retitle the queue card to name the PRs pane). Do not remove the
   lifecycle or `~/.sase/projects/` storage detail.

4. **"Learn more"** card: keep the three existing doc links (`change_spec`, `vcs`, `plugins`) and the
   keybinding-reference keycap.

### B. `AgentOnboarding` (Agents guide) — `agent_onboarding.py`

1. **Anchor the "Inspect the results" card** (`_build_inspect_card`) with one short opening line distinguishing a
   running agent (watch it live) from a finished agent (read its transcript / jump to its PR) before the existing keycap
   lines. Keep all existing keycaps and their descriptions.

2. Verify the other cards remain accurate against current keymaps; make only small wording tightenings if needed. The
   `_TAB_ROWS` map is already correct — do not regress it. Do not disturb the plugin-card conditional visibility or the
   step numbering logic (`numbered_step_titles` / `_apply_step_numbers`).

### C. `AxeOnboarding` (Axe guide) — `axe_onboarding.py`

1. **Slim `_build_chops_card`.** Keep the lumberjack/chops definition line and the action keycaps (navigate sidebar,
   `run_workflow` = run now, `add_axe_item` = add, `edit_spec` = edit config, next/prev run, `edit_panel` = open
   output). Collapse the three dense dim paragraphs into at most one short supporting line and defer the config-editing
   depth to the Axe doc link.

2. Leave "What is Axe?", "Background commands", and "Learn more" essentially intact (only minor tightening if
   warranted). Keep the status-bar-anatomy line.

## Tests to update

These tests assert on exact guide copy and will need to move in lockstep with the content. Update assertions to match
the new copy (and add assertions for the new Artifacts sub-tabs card):

- `tests/ace/tui/widgets/test_changespec_onboarding.py` — the hero assertion `"Your agents' work, shipped as PRs"`
  changes; add coverage that the new sub-tabs card names Commits, Plans, Bugs, and PRs and that its nav keys come from
  the registry (extend `test_changespec_onboarding_uses_active_keymap_registry` or add a focused test). Keep the
  lifecycle/storage/doc-link assertions.
- `tests/ace/tui/widgets/test_agent_onboarding.py` — adjust any inspect-card assertions affected by the new anchoring
  line; keep the existing keymap/tab-order/step-numbering assertions passing.
- `tests/ace/tui/widgets/test_axe_onboarding.py` — this test asserts several exact phrases from the dense paragraphs
  ("Choose a source scope", "raw YAML for compound fields", "Generated rows edit their base chop", "save & restart
  reconciles immediately", "open recorded output"). Reconcile these assertions with the slimmed card: keep the phrases
  that survive, drop/replace those that are removed.
- `tests/ace/tui/modals/test_help_modal_guide.py` — keeps asserting the widget types and the launch-card copy; update
  only if a referenced phrase changes.

## PNG visual snapshots to regenerate

Only the Guide-tab goldens are affected (the `*_onboarding_*` goldens belong to the empty-state `TabQuickStart` and must
NOT change):

- `tests/ace/tui/visual/snapshots/png/help_guide_changespecs_120x40.png`
- `tests/ace/tui/visual/snapshots/png/help_guide_agents_120x40.png`
- `tests/ace/tui/visual/snapshots/png/help_guide_axe_120x40.png`

`tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py` also asserts on inline phrases via
`assert_page_svg_contains` (e.g. `"shipped as PRs"`, `"Axe starts automatically"`, `"Read what happened"`). Update those
string assertions to match the revised copy, then regenerate the three goldens with
`just test-visual --sase-update-visual-snapshots` and eyeball the accepted diffs.

## Files to change

- `src/sase/ace/tui/widgets/changespec_onboarding.py` (hero + new sub-tabs card + re-scoped cards; may read
  order/labels/accents from `artifact_tabs.py`)
- `src/sase/ace/tui/widgets/agent_onboarding.py` (inspect-card anchor line)
- `src/sase/ace/tui/widgets/axe_onboarding.py` (slim chops card)
- `src/sase/ace/tui/widgets/_onboarding_common.py` (only if a small shared row helper is warranted for the new sub-tabs
  card)
- The four test files listed above
- The three `help_guide_*_120x40.png` goldens (regenerated)

## Validation

- `just install` first (ephemeral workspace), then `just check` (ruff + mypy + fast tests).
- `just test-visual` to confirm the three regenerated Guide goldens pass and no other visual snapshot moved.
- Manually skim each guide with `,?` -> `]` on each tab to confirm the copy reads well and every keycap resolves.

## Out of scope

- The Keymaps (1st) panel tab and its 57-char box formatting.
- The empty-state `TabQuickStart` panels and their `*_onboarding_120x40.png` goldens.
- Any keymap, tab, or behavior changes — this is content only.
- Renaming the internal `changespecs` tab id (deliberately retained; see `tab_order.py`). </content> </invoke>
