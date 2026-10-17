---
tier: epic
title: Decompose the legacy root docs pages into Reference and Contribute
goal: 'Every legacy root page under docs/ lives in reference/ or contribute/ as a
  lookup-sized, template-shaped page. Old URLs keep working through redirects and
  anchor stubs, history lives only under reference/upgrading/, the legacy allowlist
  in docs/_meta/budgets.yml is empty, and the URL contract stays green throughout.

  '
decisions:
  reference_hygiene:
    ask: Lint hand-written reference/ pages for bead IDs and epic labels once the
      edit phases clear them?
    default: true
    why: About 340 bead IDs grew into the legacy pages; without a rule they grow back
    answer: true
  docs_pointer_notes:
    ask: Update the docs/ path pointers in the tui_perf, dispatch, macros, and cli_rules
      notes when their pages move?
    memory:
    - tui_perf.md
    - dispatch.md
    - macros.md
    - cli_rules.md
    default: true
    corrects: Command plugins section in `docs/plugins.md`.
    answer: true
  docs_routing_note:
    ask: Drop the legacy-root-page wording from docs_writing.md once no legacy root
      pages remain?
    memory:
    - docs_writing.md
    default: true
    corrects: update its Reference page (today, the legacy root page that owns the
      fact)
    answer: true
phases:
- id: gates
  title: Restore the green docs gates this epic owns
  depends_on: []
  size: medium
  description: 'gates: fix the clean-base docs failures recorded on epic sase-1le
    (renderer/Prettier drift, the over-budget Reference hub, render_cli mypy errors,
    PDF/RSS and generated-page terminology tests) so every later phase verifies against
    a green check.'
- id: split-tool
  title: Make docs_split, render_moves, and check_docs safe for monolith splits
  depends_on: []
  size: medium
  description: 'split-tool: add H4 and nested selections, drop mappings, intro handling,
    anchor recording in target context, partial-split pointers, a dry-run budget report,
    and --finish to docs_split; stop render_moves clobbering hand-written pages; let
    check_docs accept root stubs and a homepage rule.'
- id: move-tool
  title: Whole-page move tool and the upgrading history home
  depends_on:
  - gates
  - split-tool
  size: medium
  description: 'move-tool: add tools/docs_move for whole-page moves, pilot it on acknowledgements.md,
    and pilot the hardened docs_split by splitting content_layout.md into Reference
    and the new reference/upgrading/ hub.'
- id: tui-agents-move
  title: Move the Agents tab sections of ace.md to reference/tui/
  depends_on:
  - move-tool
  size: medium
  description: 'tui-agents-move: move-only partial split of the Agents tab cluster
    of ace.md (about 35K words) into budget-sized reference/tui/ pages.'
- id: tui-prompt-move
  title: Move the prompt, snippet, and Launch Control sections of ace.md
  depends_on:
  - tui-agents-move
  size: medium
  description: 'tui-prompt-move: move-only partial split of the prompt input, prompt
    completion, prompts overlay, snippets, macro browser, and Launch Control sections
    of ace.md (about 33K words).'
- id: tui-move
  title: Finish ace.md and move pager.md into reference/tui/
  depends_on:
  - tui-prompt-move
  size: medium
  description: 'tui-move: move the remaining tab, pane, and global-control sections
    of ace.md, move visual-grammar remnants to contribute/, move pager.md, add the
    reference/tui/ hub, and finish ace.md as an anchor stub.'
- id: config-move
  title: Move configuration.md prose except its CLI flag tables
  depends_on:
  - move-tool
  size: medium
  description: 'config-move: move-only partial split of configuration.md into hand-written
    reference/config/ companion pages, reference/environment-variables.md, the Admin
    Center TUI page, and upgrading notes, leaving the CLI Flags section for the cli
    phase.'
- id: macros-move
  title: Move macros.md and workflow_spec.md into reference/macros/
  depends_on:
  - config-move
  size: medium
  description: 'macros-move: split macros.md into reference/macros/ pages beside the
    generated catalogs, move workflow_spec.md to reference/workflows.md, keep the
    decision-cited macros.md stub, and update the macros.md memory pointer.'
- id: work-units-move
  title: Move beads, plans, goals, Patches, and ProjectSpec into reference/work-units/
  depends_on:
  - macros-move
  size: medium
  description: 'work-units-move: split beads.md and sdd.md and move goals.md, change_spec.md,
    and project_spec.md into reference/work-units/ with a hub.'
- id: providers-move
  title: Move provider and memory pages into reference/providers/ and reference/memory/
  depends_on:
  - move-tool
  size: medium
  description: 'providers-move: split llms.md (retargeting render_model_docs), agent_providers.md,
    instruction_bundles.md, memory.md, and memory_history.md into reference/providers/
    and reference/memory/, with invocation internals going to contribute/.'
- id: automation-move
  title: Move scheduler, notifications, monitors, tools, mentors, and sudo pages
  depends_on:
  - providers-move
  size: medium
  description: 'automation-move: split notifications.md, axe.md, monitors.md, and
    the decision-cited tool.md, and move mentors.md and sudo.md into reference/automation/,
    with epic-labelled internals going to contribute/.'
- id: contribute-move
  title: Move the contributor pages into contribute/
  depends_on:
  - move-tool
  size: medium
  description: 'contribute-move: whole-page moves of development, rust_backend (stub),
    fakey, telemetry, perf_runbook, mobile_mvp_runbook, integrations, the two artifacts-pane
    pages, and sdd_storage into contribute/, and update the tui_perf.md memory pointer.'
- id: agents-vcs-move
  title: Move agent, workspace, init, and VCS pages into reference/agents/ and reference/vcs/
  depends_on:
  - contribute-move
  size: medium
  description: 'agents-vcs-move: split or move agent_sessions, workspace, agents_sidecar,
    init, vcs, and commit_workflows into reference/agents/ and reference/vcs/, with
    provider APIs, orchestrator, and sync internals going to contribute/.'
- id: integrations-move
  title: Move integration pages and the single-topic reference pages
  depends_on:
  - agents-vcs-move
  size: medium
  description: 'integrations-move: move plugins, editor, completion, mobile_gateway,
    and remote_dispatch into reference/integrations/ and the six single-topic pages
    into reference/, then update the cli_rules.md and dispatch.md memory pointers.'
- id: cli
  title: Retire the hand-maintained CLI flag tables and cli.md
  depends_on:
  - config-move
  - tui-move
  - work-units-move
  - automation-move
  - integrations-move
  size: medium
  description: 'cli: compare configuration.md CLI Flags and cli.md against the generated
    CLI pages, port missing facts into argparse help or subsystem pages, delete the
    tables with anchor targets, and finish both pages as stubs.'
- id: tui-edit
  title: History and duplicate sweep for reference/tui/
  depends_on:
  - tui-move
  size: medium
  description: 'tui-edit: strip history, bead IDs, and epic labels from the TUI reference
    pages into reference/upgrading/, replace keymap and registry restatements with
    generated links, and confirm template shape.'
- id: config-macros-edit
  title: History and duplicate sweep for configuration, macros, and workflows pages
  depends_on:
  - config-move
  - macros-move
  size: medium
  description: 'config-macros-edit: run the edit checklist on the config companion
    pages, environment variables, Admin Center, reference/macros/, and reference/workflows.md.'
- id: work-units-providers-edit
  title: History and duplicate sweep for work-units, providers, and memory pages
  depends_on:
  - work-units-move
  - providers-move
  size: medium
  description: 'work-units-providers-edit: run the edit checklist on reference/work-units/,
    reference/providers/, and reference/memory/.'
- id: automation-edit
  title: History and duplicate sweep for automation pages
  depends_on:
  - automation-move
  size: medium
  description: 'automation-edit: run the edit checklist on reference/automation/.'
- id: agents-integrations-edit
  title: History and duplicate sweep for agents, VCS, integrations, and single pages
  depends_on:
  - agents-vcs-move
  - integrations-move
  size: medium
  description: 'agents-integrations-edit: run the edit checklist on reference/agents/,
    reference/vcs/, reference/integrations/, and the single-topic reference pages.'
- id: finish
  title: Final hubs, empty allowlist, repo-wide pointer sweep, and success metrics
  depends_on:
  - cli
  - tui-edit
  - config-macros-edit
  - work-units-providers-edit
  - automation-edit
  - agents-integrations-edit
  - contribute-move
  size: medium
  description: 'finish: finalize the Reference and Contribute hubs and nav, empty
    budgets.yml, sweep remaining docs path references, refresh docs_writing.md, apply
    the reference_hygiene decision, and record the success metrics.'
proposed_by: bbugyi200.athena.sase-1le.20
parent_bead: sase-1le.20
decided_by: auto
create_time: 2026-10-16 08:04:32
status: done
bead_id: sase-1le.20.1
---

- **PROMPT:** [prompts/202610/docs_decompose.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/docs_decompose.md)
- **PARENT:** [202610/sase_sh_docs_redesign.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_sh_docs_redesign.md)
- **BEAD:** [sase-1le.20.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1le/sase-1le.20.1.md)

# Decompose the legacy root docs pages into Reference and Contribute

This sub-epic carries out phase `decompose` (bead `sase-1le.20`) of the sase.sh redesign
epic `sase-1le`. The parent plan (`plan:202610/sase_sh_docs_redesign.md`) is the spec
for the target information architecture, the URL compatibility policy, the documentation
contract, and the decompose exit criteria. This plan turns that phase into units that
each fit one coding agent. Where the two plans differ, this one is more specific, but it
may not loosen the parent's exit criteria.

## Starting state (measured at `b94cffe0d3`)

- **Fifty pages** are allowlisted in `docs/_meta/budgets.yml`: 49 legacy root pages
  totaling about 455K counted words, plus the homepage `index.md` (357 words, no
  `description`).
- **Concentration.** Five pages hold 233K words: `ace.md` (98.6K: 46 H2s and 141 H3s),
  `configuration.md` (54.3K, including a 19.8K-word hand-maintained "CLI Flags" table
  section), `macros.md` (29K), `beads.md` (26.3K), and `llms.md` (24.9K, carrying ten
  `render_model_docs` generated blocks).
- **Oversized sections.** Single sections exceed the 6,000-word Reference cap, for
  example `ace.md` › Prompt Input Widget › Completion (an H3 of 10.6K words) and
  `configuration.md` › Configuration Sections › `ace` (8.2K words).
- **History debt.** A regex count in the legacy pages finds about 540 history phrases,
  about 340 bead-ID-shaped tokens (some are false positives such as `sase-core` or CSS
  classes), and 14 epic labels. `ace.md`, `configuration.md`, `beads.md`, `macros.md`,
  `rust_backend.md`, and `perf_runbook.md` carry the most.
- **Moves already made.** Earlier phases moved six pages (`docs/_meta/moves.yml`, all
  whole-page redirects). `reference/` already holds generated CLI, config, keymap,
  directive, discovery-order, glossary, and design-decision pages, plus three
  hand-written pages (`agent-auto-restart.md`, `agent-revive-log.md`,
  `runner-slots.md`). Those three stay where they are; moving them again would create
  redirect chains.
- **Inbound references.** About 110 links from typed pages point at root pages. Root
  pages link each other hundreds of times (`ace.md` 110, `configuration.md` 107,
  `macros.md` 81, and so on). About 60 `src/`, `tests/`, `tools/`, `Justfile`, and
  README locations name a root `docs/*.md` path. README carries relative `docs/*.md`
  links (`llms.md#implicit-role-aliases`, `macros.md#project-tags`,
  `acknowledgements.md`), and the frozen contract (`docs/_meta/url_contract.txt`) pins
  `/ace/#prompt-history-modal`, `/development/#your-sase-versus-this-checkouts-venv`,
  `/macros/#auto-directive`, and `/sdd/#plan-decisions`.
- **Text this epic may not edit** cites `docs/tool.md` and `docs/macros.md` (decision
  records) and `docs/rust_backend.md` (the core note `rust_core_backend_boundary.md`),
  so those three keep stubs at their old paths.

### Tooling gaps found while planning

These gaps make a safe monolith split impossible today; `split-tool` and `move-tool`
close them before any page moves.

1. **Clobbered remainders.** `tools/render_moves` rewrites every `kind: stub` source
   page on every `just fix`. Because `tools/docs_split` marks a page `stub` on its first
   split, any partial split — and any page intro above the first H2 — is overwritten at
   the next `just fix`.
2. **Limited selection.** `docs_split` selects only H2 and H3 headings and rejects
   nested selections, so a page cannot be cut under budget where one H3 alone is over
   6,000 words.
3. **Lost anchors.** `docs_split` records only the selected heading's own anchor, as
   computed in the source page. Deep links to nested headings are lost, and a
   duplicate-suffixed slug (`filtering-1`) can render differently on its new page.
4. **Unrewritten link forms.** Only inline `](…)` links are rewritten. Reference-style
   definitions (`[x]: url`, used in `artifact_references.md`) and HTML `href`/`src` are
   not. Inbound rewriting also edits generated pages (for example
   `reference/tui/keymaps.md` links `../../ace.md`), which their renderers then revert.
5. **No whole-page tool.** A whole-page move needs a hand-run `git mv`, outgoing and
   inbound link rewrites, a nav path edit, and a `moves.yml` entry.
6. **Root stubs fail the lint.** `tools/check_docs` applies its typing rule before it
   recognizes generated files, so a generated stub at a root path that is no longer
   allowlisted fails as "untyped location".
7. **No homepage rule.** Root `index.md` can leave the allowlist only once it has an
   explicit rule (title, description, and a word cap); otherwise it is unbudgeted and
   lacks a description.
8. **Stubs are searchable.** Stubs carry no `search: {exclude: true}` frontmatter, even
   though the URL policy excludes them from search. Stale-stub detection also reads only
   the first line, so it would miss a stub that gains frontmatter.
9. **Stub destination.** A split page's stub points at its first moved target rather
   than a hub.
10. **Stale allowlist entries.** Nothing flags a `budgets.yml` entry whose page is gone
    or is now a stub.
11. **A constraint, not a gap:** `tools/render_cli` prunes every non-generated `*.md` in
    `reference/cli/`, so hand-written CLI prose cannot live there.

### Known clean-base check failures

Recorded on `sase-1le` notes #1–#4 and reproduced while planning:

- `just fmt-docs-check`: `render_glossary` and `render_design_decisions` output differs
  from Prettier.
- `tools/check_docs`: `reference/index.md` is a 424-word hub (cap 400).
- `mypy tools/render_cli`: 8 errors.
- `tests/test_docs_pdf_tools.py::test_pdf_check_recipe_keeps_strict_mkdocs_export`.
- Reported but not re-run here:
  `tests/test_docs_social_card.py::test_rss_plugin_images_use_the_default_social_image`
  and three terminology tests that flag generated glossary, design-decision, and
  `cli/gate.md` text.

A failing `sase tool run check` stops at the first UNKNOWN stage. Unless these are fixed
first, every sub-phase would verify against a red gate that masks its own drift. The
epic owns them, so `gates` fixes them.

## Design

### Move phases, then edit phases

The parent plan requires a move-only commit first and content commits after. Each phase
lands as one commit, so the rule maps onto separate phases:

- **A move phase** moves text verbatim, using `tools/docs_split` or `tools/docs_move` to
  rewrite links and normalize heading levels. Its only new prose is page scaffolding
  from `docs/_templates/reference.md`: frontmatter (`title`, a description of at most
  160 characters), the H1, a one-sentence opening, and closing "Related" links, plus hub
  pages, nav, and pointer updates. It may delete only pure navigation (hand-written
  tables of contents). It may not rewrite moved text.
- **An edit phase** changes content inside the new pages, following the edit checklist
  below. It never moves pages between directories or renames files. URLs are fixed at
  move time.

### Partial splits

After `split-tool`, a page can be split across several phases:

- `docs_split` leaves a small pointer in place of each moved section: the old anchor ids
  plus a "Moved to …" link. The URL contract therefore stays green while the page is
  half-split.
- `render_moves` writes the generated stub only after `docs_split --finish` has verified
  that nothing unmapped remains and has removed the source page.

`ace.md` (three phases) and `configuration.md` (`config-move`, then `cli`) depend on
this.

### Target layout

Use lowercase hyphenated file names, one directory per product unit, and an `index.md`
hub of at most 400 words in each new directory:

| Source pages                                                                                                                                                                                                                              | Destination                                                                                                                                                                                    |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ace.md`, `pager.md`                                                                                                                                                                                                                      | `reference/tui/`: one page per tab or coherent surface; rendering and visual-grammar remnants → `contribute/`                                                                                  |
| `configuration.md`                                                                                                                                                                                                                        | hand-written companion pages in `reference/config/` beside the generated family pages; `reference/environment-variables.md`; Admin Center → `reference/tui/admin-center.md`; CLI Flags → `cli` |
| `macros.md`, `workflow_spec.md`                                                                                                                                                                                                           | `reference/macros/` (beside the generated `directives.md` and `discovery-order.md`); `reference/workflows.md`                                                                                  |
| `beads.md`, `sdd.md`, `goals.md`, `change_spec.md`, `project_spec.md`                                                                                                                                                                     | `reference/work-units/`                                                                                                                                                                        |
| `llms.md`, `agent_providers.md`, `instruction_bundles.md`, `memory.md`, `memory_history.md`                                                                                                                                               | `reference/providers/`, `reference/memory/`; invocation, delivery, and manifest internals → `contribute/`                                                                                      |
| `notifications.md`, `axe.md`, `monitors.md`, `tool.md`, `mentors.md`, `sudo.md`                                                                                                                                                           | `reference/automation/`; epic-labelled internals → `contribute/`                                                                                                                               |
| `agent_sessions.md`, `workspace.md`, `agents_sidecar.md`, `init.md`, `vcs.md`, `commit_workflows.md`                                                                                                                                      | `reference/agents/`, `reference/vcs/`; provider APIs, orchestrator, and sync internals → `contribute/`                                                                                         |
| `plugins.md`, `editor.md`, `completion.md`, `mobile_gateway.md`, `remote_dispatch.md`                                                                                                                                                     | `reference/integrations/`; API details → `contribute/`                                                                                                                                         |
| `query_language.md`, `agent_images.md`, `prompt.md`, `artifact_references.md`, `artifact_links.md`                                                                                                                                        | one `reference/<name>.md` each (`agent-attachments.md`, `prompt-history.md`, …)                                                                                                                |
| `cli.md`                                                                                                                                                                                                                                  | routing role → `reference/index.md`; behavior prose → subsystem pages; flags → generated `reference/cli/`                                                                                      |
| `development.md`, `rust_backend.md`, `fakey.md`, `telemetry.md`, `perf_runbook.md`, `mobile_mvp_runbook.md`, `integrations.md`, `artifacts_pane_contract.md`, `artifacts_pane_visual_grammar.md`, `sdd_storage.md`, `acknowledgements.md` | `contribute/` (whole moves; `rust_backend.md` keeps a stub)                                                                                                                                    |
| `content_layout.md`                                                                                                                                                                                                                       | layout facts → `reference/content-layout.md`; migration steps → `reference/upgrading/`                                                                                                         |
| history from every page                                                                                                                                                                                                                   | `reference/upgrading/index.md` hub plus one `reference/upgrading/<area>.md` page per area, created on first use                                                                                |

`reference/upgrading/index.md` serves the parent plan's `reference/upgrading.md` URL
(`/reference/upgrading/`). Per-area pages keep it under budget, and parallel edit phases
write separate files.

Every whole-page move is a 301 redirect, except the decision-cited `tool.md` and
`rust_backend.md`, which are stubs. Every split page becomes a stub whose primary link
is its new hub. Page lists suggested in the phase sections are starting points. The
worker confirms or adjusts them with the `docs_split --dry-run` budget report, never
exceeding 6,000 hand-written words per `reference/` page.

### Ordering

The tooling phases run first: `gates` and `split-tool` in parallel, then `move-tool`.
The move phases then run in four lanes, so at most four move phases touch the shared
files (`moves.yml`, `mkdocs.yml`, the Reference hub, and generated maps) at once:

- **TUI:** `tui-agents-move` → `tui-prompt-move` → `tui-move`.
- **Config and authoring:** `config-move` → `macros-move` → `work-units-move`.
- **Providers and automation:** `providers-move` → `automation-move`.
- **Contributor and agent pages:** `contribute-move` → `agents-vcs-move` →
  `integrations-move`.

Each edit phase starts when its move phases land. `cli` waits for every lane, because
its CLI prose needs the subsystem pages to exist. `finish` runs last.

### Memory edits mirror the parent's final decisions

The parent epic already answered yes, finally, to memory edits for exactly these notes:

- `docs_pointer_notes`: `tui_perf.md`, `dispatch.md`, `macros.md`, and `cli_rules.md`;
- `docs_routing_note`: `docs_writing.md`.

This plan restates both, with the same ids, so that phase workers one level down see
coverage on their own beads and the memory guard stays quiet. These decisions are not
new questions. The phases below apply them through `/sase_memory_write` and then run
`sase memory init`. No other memory note may be edited. Decision records and
`rust_core_backend_boundary.md` stay untouched; their stubs keep the old paths working,
and the core note's pointer gets a `PROPOSED FOLLOW-UP:` note.

## Rules for every phase

- **Read first:** the parent plan's "Rules for every phase" and "Phase decompose"
  sections, `docs/contribute/writing-docs.md`, and through `/sase_memory_read` both
  `docs_writing.md` and `lint_and_test.md`. Read `generated_skills.md` before editing
  anything under `src/sase/macros/skills/`, and `cli_rules.md` before changing argparse
  help.
- **Hot pages:**
  - Rebase right before splitting a hot page (`ace.md`, `configuration.md`, `macros.md`,
    `beads.md`, `llms.md`), and land promptly.
  - After a rebase conflict, regenerate generated outputs with `just fix` instead of
    hand-merging them: the `_redirects` and `not_in_nav` blocks, stubs,
    `docs/README.md`, and `docs/llms.txt`.
  - In `moves.yml`, `mkdocs.yml` nav, and the hubs, keep both sides.
- **Mapping discipline:**
  - Commit the `docs_split` mapping file you used under `docs/_meta/splits/`, so
    reviewers and later phases can see where every heading went.
  - Map every heading of a page you finish, and run `--dry-run` until no target is over
    budget.
- **Same-commit pointer updates.** When a path moves, update every literal reference in
  the same phase:
  - `src/` help text, comments, schema descriptions, and `default_config.yml` comments;
  - `tests/`, including the terminology allowlists the parent plan lists;
  - `tools/`, `Justfile`, `README.md`, `CONTRIBUTING.md`, and `mkdocs*.yml`;
  - links in other pages.

  Before finishing, run `git grep -n "docs/<old>.md"` and `git grep -n "sase.sh/<old>/"`
  for every page you moved. Generated pages are never hand-edited: when one links a
  moved page, change its renderer (`render_keymap_macro_reference`, `render_config`,
  `render_model_docs`, …) and regenerate.

- **Nav and hubs:**
  - Replace your own Reference sub-group in `mkdocs.yml`, hub first. Nest sub-groups
    when a group passes about 12 entries.
  - Repoint your section of `reference/index.md` (or `contribute/index.md`) to the new
    hub, and keep it at or under 400 words.
- **Budgets:**
  - Remove a page's `budgets.yml` entry when the page becomes a stub or disappears;
    `check_docs --fix` removes stale entries after `split-tool`.
  - Never raise a ceiling.
- **Verify:**
  - Run `just fix`, then `sase tool run check`. It includes the docs lint and, when docs
    change, the strict build and URL contract.
  - Iterate with `just docs-lint` and `just docs-check` (through `sase tool run` where
    the recipe is guarded).
  - Never run `just check-full` unless explicitly told to.
- **Accuracy:** when moved text disagrees with the code, record a `PROPOSED FOLLOW-UP:`
  (bug) note on your phase bead. Never add a sentence explaining the disagreement.
- **Phase workers file nothing.** Record discovered work as `PROPOSED FOLLOW-UP:` notes
  on your own phase bead. A check failure that reproduces identically on the clean base
  tree is recorded and does not keep the phase open.

### Edit checklist (edit phases)

Run these over the phase's pages only:

1. **History sweep.** Search for
   `\b(formerly|no longer|legacy|renamed|still mentions?|previously|used to|deprecated)\b`,
   bead IDs (`\bsase-[0-9a-z]{2,}(\.[0-9a-z]+)*\b`, ignoring real names such as
   `sase-core` or `sase-github`), and epic or phase labels (`\bE\d+\b`, `phase sase-…`).
   Resolve each hit:
   - **Current behavior with a historical aside:** state the current behavior only.
   - **A change an upgrading reader needs:** add a concise entry to
     `reference/upgrading/<area>.md`, newest first, and keep the reference sentence
     current-state.
   - **Internal provenance, or already in `CHANGELOG.md`:** drop it.
   - **A current feature or flag name** (`--legacy`, a "legacy" status the code still
     shows): keep it.
2. **Registry duplicates.** Delete hand-maintained tables and rows that restate a
   generated page (CLI flags, config keys and defaults, keymap rows, directive
   signatures, model tables) and link the generated anchor instead.
   - If a row holds a fact the generated page lacks, port it into the source (argparse
     help, schema description, keymap metadata, or the directive contract) and
     regenerate.
   - Record a `PROPOSED FOLLOW-UP:` note when porting is large or the source lives in
     `sase-core`.
3. **No re-teaching.** Replace how-to narrative that an existing guide already covers
   with a link to that guide. Keep narrative that has no guide home; if it serves a real
   user goal, record a "guide candidate" `PROPOSED FOLLOW-UP:`.
4. **Template shape.** Each page opens with one sentence saying what it is for and
   closes with "Related" links. Titles name the thing, and descriptions stay one line of
   at most 160 characters.
5. **Record counts.** Note before and after counts for (1) and (2) on the phase bead.

## Phase gates

Fix each item that still reproduces at phase start. Skip any item another change has
already fixed, and note which on the bead.

- **Renderer/Prettier drift.** Make `tools/render_glossary` and
  `tools/render_design_decisions` emit Markdown that the repo's Prettier configuration
  leaves unchanged: `_…_` emphasis and wrapped prose. Alternatively, pass their output
  through the same formatter `just fmt` uses inside the renderer. Either way,
  `just fmt-docs-check` must pass right after `just fix`, and `abbreviations.md` must
  stay stable.
- **Reference hub.** Trim `docs/reference/index.md` to at most 400 words without
  dropping a product unit. Later phases repoint its links.
- **Type errors.** Fix the 8 mypy errors in `tools/render_cli` without changing its
  output.
- **PDF and RSS tests.** Reconcile `test_pdf_check_recipe_keeps_strict_mkdocs_export`
  and `test_rss_plugin_images_use_the_default_social_image` with the curated-PDF build
  that `pdf_scope = curate` produced:
  - fix the configuration if the test guards a real requirement;
  - otherwise update the test to the curated design.
- **Terminology tests on generated pages.**
  `test_macro_docs_and_memory_avoid_xprompt_terms`,
  `test_current_docs_skills_and_memory_avoid_stale_shell_phrases`, and
  `test_current_docs_skills_and_memory_avoid_stale_family_phrases` flag generated
  Reference text (glossary aliases, design-decision bodies, `cli/gate.md`). The source
  text is already scanned where it lives, so either:
  - exempt whole-file generated pages, or
  - fix the source wording when it is stale rather than a recorded alias.

**Exit:** on a clean tree after `just fix`, `sase tool run check` has none of the five
failures above.

## Phase split-tool

Harden the split path. Close tooling gaps 1–4 and 6–10, and add tests for each (in
`tests/test_docs_moves.py` and `tests/test_check_docs_tool.py`).

- **`tools/docs_split` selection:**
  - Accept H2–H4 selections.
  - Allow nested selections, where the most specific selection wins: a parent keeps its
    intro text and unselected children; a selected child goes to its own target.
  - Accept a pseudo-section `_intro` for text between the H1 and the first H2.
  - Accept `drop: true` for pure navigation (tables of contents), with an optional
    `anchor_target:` for sections replaced by a generated page. Their anchors then map
    to that target.
- **`docs_split` normalization and anchors:**
  - Normalize heading levels so the shallowest moved heading becomes H2.
  - Compute each anchor as it renders on the target page, adding an explicit `{#id}`
    where needed to keep the old id.
  - Record every nested heading id, explicit `{#id}`, and `<a id>` inside a moved block
    in the `moves.yml` anchors map.
- **`docs_split` link rewriting:**
  - Rewrite reference-style link definitions and HTML `href`/`src` as well as inline
    links.
  - Skip whole-file generated pages and `BEGIN GENERATED` blocks during inbound
    rewriting, and print them as "update the renderer" warnings.
- **`docs_split` partial splits:** after a partial split, leave a compact pointer in
  place of each moved section: the old ids as `<a id>` plus a "Moved to [title](link)"
  line.
- **`docs_split --dry-run`:** print, per target, the hand-written word count (using
  `check_docs`'s counter) against its budget, plus every heading left unmapped.
  Over-budget targets exit nonzero.
- **`docs_split --finish`:**
  - Fail if hand-written content would remain; otherwise delete the source page.
  - Set the move entry's primary destination: a mapping-level `to:`, defaulting to the
    hub `index.md` when one exists in the target directory.
- **`tools/render_moves`:**
  - Never overwrite a page with hand-written content. Write a stub only when the source
    is absent or already a generated stub.
  - Give stubs `search: {exclude: true}` frontmatter.
  - Detect stale stubs by the marker anywhere in the head of the file.
  - Keep the stub title stable (store it in the move entry or derive it before the
    source is deleted).
- **`tools/check_docs`:**
  - Accept generated stubs at any path in the typing rule.
  - Treat root `index.md` as the homepage, requiring `title` and `description` and
    applying the 400-word hub cap, so its allowlist entry can go in `finish`.
  - Report `budgets.yml` entries whose page is missing or is now a stub, and have
    `--fix` remove them.
- **Docs:** add a short "Splitting and moving pages" section to
  `docs/contribute/writing-docs.md` that names the tools, the mapping files under
  `docs/_meta/splits/`, and the partial-split behavior.

**Exit:** a fixture round-trip that splits one page across two runs (partial, then
`--finish`) keeps every old anchor resolvable through `check_docs_urls`, and `just fix`
never alters a half-split page.

## Phase move-tool

- **`tools/docs_move`** (new, sharing link-rewriting helpers with `docs_split` in
  `tools/docs_moves.py` or a sibling module):
  - Move one page: `git mv` semantics, rewrite its outgoing relative links, rewrite
    inbound links across `docs/` with the same generated-page exclusions as
    `docs_split`, and rewrite its nav path in `mkdocs.yml`.
  - Add `title` and `description` frontmatter when they are missing (from flags), and
    drop the page's `budgets.yml` entry.
  - Append a `moves.yml` entry: `kind: redirect` by default, or `--stub`, which records
    every heading anchor so decision-cited paths keep a stub.
  - Add tests.
- **Pilot 1:** `acknowledgements.md` → `contribute/acknowledgements.md` (redirect).
  Update README's relative link, `contribute/index.md`, and the homepage footer link.
- **Pilot 2:**
  - Create `reference/upgrading/index.md`: a hub saying it is the single home for
    upgrade and history notes, organized one page per area, newest first.
  - Split `content_layout.md` with the hardened `docs_split`: the canonical-layout facts
    go to `reference/content-layout.md`, and the migration and compatibility sections go
    to `reference/upgrading/content-layout.md`. Finish it as a stub.
  - Replace the "Migration" Reference nav group with "Upgrading" plus "Content layout".
- **Writing docs:** record the `docs_move` usage in the section `split-tool` added.

**Exit:**

- both pilots pass the URL contract;
- `/content_layout/` resolves through its stub;
- the upgrading hub is in nav and the Reference hub.

## Phase tui-agents-move

This is a partial, move-only split of the Agents tab cluster of `ace.md`. It takes these
H2s:

- Keybindings: Agents Tab (19.5K);
- Agent Statuses (5.7K);
- Agents Tab Main Deck;
- Agents Tab LLM Calls Panel;
- Agents Tab Tool Runs;
- Agents Zoom and Node Rail;
- Agent Auto-Naming;
- Plan Workflows;
- Launch Approval;
- Agent Holds;
- Linked Chats in Multi-Step Workflows;
- Retry/Fallback Display.

Suggested pages under `reference/tui/`:

- `agents-tab.md`: tabs, navigation, actions, forking, the wait modal, artifacts,
  search, and leader mode;
- `agents-tab-panels.md`: clan and session detail panels, neighbors, tribe side panels,
  and the refresh panel;
- `agents-tab-grouping.md`: group banners, grouping modes, and row glyphs;
- `agent-statuses.md`;
- `agents-main-deck.md`: the main deck, LLM calls, tool runs, and zoom;
- `agent-workflows.md`: plan workflows, launch approval, holds, linked chats,
  retry/fallback, and auto-naming.

Create the `reference/tui/index.md` hub listing these pages (and `admin-center.md` if
`config-move` already added it); later TUI phases extend it. Add a "TUI" nav sub-group.
Do not finish `ace.md`.

**Exit:**

- every moved section lives in a budget-clean page;
- `ace.md` keeps pointer anchors for all moved ids;
- the URL contract is green.

## Phase tui-prompt-move

This is a partial, move-only split of these `ace.md` H2s:

- Prompt Input Widget (20.3K; its Completion H3 needs an H4 split);
- Prompts Overlay (keep the frozen `#prompt-history-modal` id);
- Snippets;
- Macro Browser;
- Launch Control (7.8K, split by H3).

Suggested pages:

- `prompt-input.md`: insert and normal modes, alt-brace syntax, and macro editing;
- `prompt-stacks.md`;
- `prompt-completion.md` and a second completion page, split by H4 group;
- `prompts-overlay.md`;
- `snippets.md`, which may absorb the macro browser;
- `launch-control.md` and `launch-control-overrides.md`.

Extend the TUI hub and nav.

**Exit:** the same as `tui-agents-move`, and `/ace/#prompt-history-modal` resolves.

## Phase tui-move

Move every remaining `ace.md` section and finish the page with `--finish`, setting its
primary destination (`to:`) to `reference/tui/index.md`. Suggested grouping:

- **The hub:** Overview and Launching become `reference/tui/index.md` (within 400 words)
  plus `launching.md`.
- **Tab System (9.9K):** `tabs-and-panes.md` and `filtering.md`, plus a
  files-and-preview page if the budget needs it.
- **Artifacts:** Keybindings: Artifacts / Patches, mentor comment stats, and the PR
  origin chip → `artifacts-tab.md`.
- **One page per tab:** Services, Procs, Updates, and Projects (with Current project);
  Machines, Statistics, and Tools tabs plus the Config Flags pane in one page.
- **Smaller surfaces:**
  - Query System → `query-system.md`;
  - Global Keybindings, Custom Keymaps, Command Palette, Command Line, and Auto-Refresh
    → `global-controls.md`;
  - Notifications Modal and Notification Actions → `notifications-modal.md`;
  - Jump All Modal and Node Finder → `finders.md`.
- **Rendering remnants:** Tab Bar Display, File Panel Rendering, and Image Preview
  Foundation → `contribute/tui-rendering.md`.
- **Pager:** move `pager.md` whole to `reference/tui/pager.md` with `docs_move`.

Then finish the page and its references:

- Update `tools/render_keymap_macro_reference`'s `../../ace.md` link and regenerate.
- Update the tests that name `docs/ace.md` (for example
  `test_agent_tribe_terminology.py`, `test_docs_urls.py`, and
  `test_agent_wait_dependency_status_counts.py`).
- Update `pager.md` references in `tests/pager/` and `tests/perf/README.md`.
- Remove `ace.md` and `pager.md` from `budgets.yml`.

**Exit:**

- `ace.md` is a generated stub mapping every old anchor;
- every `reference/tui/` page is within budget;
- the TUI nav group shows the hub first;
- the URL contract is green.

## Phase config-move

This is a partial, move-only split of `configuration.md`. Do not touch "## CLI Flags",
which `cli` owns, and do not finish the page.

- **Drop:** Table of Contents.
- **Files and merging:** Config File Location, Owner Identity, Deep-Merge System, and
  Directory Sharding → `reference/config/files-and-merging.md`.
- **Admin Center:** SASE Admin Center → `reference/tui/admin-center.md`. Add it to the
  TUI nav group. Link it from `reference/tui/index.md` if that hub exists; otherwise
  link it from the TUI section of `reference/index.md`, and the TUI phases fold it into
  their hub. On conflict, keep both sides.
- **Configuration Sections:** move to hand-written companion pages in
  `reference/config/`, named for their subject (for example `ace-behavior.md`,
  `llm-provider-behavior.md`, `repos-behavior.md`, `feature-flags.md`). Split the
  8.2K-word `ace` H3 by H4. Before writing there, confirm that `render_config` never
  prunes hand-written pages in `reference/config/`.
- **Environment Variables** → `reference/environment-variables.md`.
- **Local State Cutover** → `reference/upgrading/local-state.md`, moved verbatim.
- **Renderer links:**
  - Teach `tools/render_config` to link the companion pages from its generated
    `reference/config/index.md` (a small static map in the renderer is fine).
  - Repoint its `../../configuration.md#config-file-location` link.
- **References:** update `sase.schema.json` descriptions, `default_config.yml` comments,
  `src/sase/doctor/checks_config_tribes.py`, and the tests that name
  `docs/configuration.md`, so they point at the new pages. `configuration.md` itself
  stays as the CLI Flags remainder.

**Exit:**

- only the CLI Flags section and pointer anchors remain in `configuration.md`;
- every new page is within budget;
- the URL contract is green.

## Phase macros-move

- **Split `macros.md` into `reference/macros/`:**
  - an `index.md` hub;
  - syntax pages: file format, reference syntax, arguments, shorthand, typed inputs,
    output specification, Jinja2, raw placeholders, tags, command substitution,
    protected content, aliases, and recursive expansion, as two pages if over budget;
  - `fields.md`: the snippet, skill, and memory fields;
  - `built-ins.md`: built-in, config-based, and local-file macros;
  - directive behavior prose (12.3K words, two or three pages linking the generated
    `directives.md`);
  - `multi-agent-prompts.md`, which includes the relationship to workflows;
  - troubleshooting;
  - the CLI subcommands and editor LSP sections, kept with macros.
- **Discovery Order:** drop it with an `anchor_target` to the generated
  `discovery-order.md` when it only restates that page; otherwise move it.
- **History:** "Renamed from xprompts" and "Legacy Placeholders" →
  `reference/upgrading/macros.md`, moved verbatim.
- **Finish:**
  - Finish `macros.md` as a stub; the decision record `hold-pull-fail-open` cites it.
  - `/macros/#auto-directive` must resolve.
  - Move `workflow_spec.md` whole to `reference/workflows.md` with `docs_move`.
- **References:**
  - Update `render_keymap_macro_reference`'s `../../macros.md` link, README's
    `docs/macros.md#project-tags` link, `tests/_macro_terminology_docs.py`, and the
    other tests that name these paths.
  - Under `docs_pointer_notes`, update `sase/memory/macros.md`'s `docs/macros.md` (Hold
    Directive) pointer through `/sase_memory_write`, then run `sase memory init`.

**Exit:** both pages are stubbed or redirected, every new page is within budget, and the
URL contract is green.

## Phase work-units-move

- **`beads.md` (26.3K)** → `reference/work-units/`. Suggested pages:
  - bead IDs and data model, split by H3 (types, lifecycles, follow-up triage, external
    mirroring, attachments, creation metadata);
  - storage, pages, the Rust backend, and the checkout source of truth;
  - the CLI commands section (12.9K), split by H3, with `sase bead work` (3.5K) as its
    own page;
  - TUI integration, which stays with beads.
- **`sdd.md` (7.5K)** → plans pages; `/sdd/#plan-decisions` must resolve.
- **Whole moves:** `goals.md` → `goals.md`, `change_spec.md` → `patches.md`, and
  `project_spec.md` → `project-spec.md`, all under `reference/work-units/`.
- **Hub and references:**
  - Add `reference/work-units/index.md` and the nav sub-group.
  - Update `src/sase/bead/cli_admin_doctor.py`,
    `src/sase/doctor/checks_attachment_store.py`, `src/sase/patch_stitch_audit.py`, and
    the tests that name these paths.

**Exit:** every source is stubbed or redirected, every page is within budget, and the
URL contract is green.

## Phase providers-move

- **`llms.md` (24.9K)** → `reference/providers/`:
  - an `index.md` hub;
  - per-provider integration pages (grouped so each stays within budget);
  - external provider plugins;
  - models: tiers, reasoning effort, role aliases, and the generated tables;
  - routing: per-prompt switching and temporary overrides, disables, and priority;
  - usage: subscription usage and usage-limit auto-disable;
  - retry and fallback;
  - environment variables: merge them into `reference/environment-variables.md` if it
    exists, otherwise keep them in a providers page and let `finish` consolidate.
- **`llms.md` internals:** provider architecture, preprocessing, subprocess streaming,
  postprocessing, chat history, token usage, and invocation lifecycle →
  `contribute/llm-invocation.md` (split if needed). Legacy Thinking Metadata →
  `reference/upgrading/providers.md`.
- **Retarget `tools/render_model_docs`:** each generated block moves with its section.
  Make the renderer find blocks by marker in their new pages (or through an explicit
  block-to-file map), keep its YAML-only guarantee, and update
  `tests/test_render_model_docs.py`, the `models.yml` comments, and the doctor checks
  that name `docs/llms.md`.
- **Other moves:**
  - `agent_providers.md` → `reference/providers/agent-clis.md` (whole).
  - `instruction_bundles.md` → `reference/memory/instruction-bundles.md`, with manifest
    and delivery internals → `contribute/`.
  - `memory.md` → a `reference/memory/` page plus the `reference/memory/index.md` hub.
  - `memory_history.md` → `reference/memory/history.md`.
- **References:** README's `docs/llms.md#implicit-role-aliases` link, the tests, and
  `src/` references.

**Exit:**

- `just fix` regenerates the model blocks in place with no drift;
- every source is stubbed or redirected;
- the URL contract is green.

## Phase automation-move

- **`notifications.md` (18K)** → `reference/automation/`: viewing notifications,
  notification types, fields and tags, the CLI, storage, and command-backed interaction
  gates (5.2K, its own page).
- **`axe.md` (15.8K)** → scheduler pages; split Default Routines and Configuration by
  H3.
- **`monitors.md` (8K)** → two pages.
- **`tool.md` (6.4K):** stub it, since the decision record
  `explicit-handoff-fails-closed` cites it. Move its epic-labelled sections to
  `contribute/` and the rest to `reference/automation/tools.md`.
- **Whole moves:** `mentors.md` and `sudo.md`.
- **Hub:** add `reference/automation/index.md` and the nav sub-group.
- **References:**
  - Update `Justfile` and `tools/require_tool_run` (`docs/tool.md`),
    `src/sase/monitor/tool_wrap.py`, and `src/sase/macros/skills/sase_final.md`
    (`docs/monitors.md`; follow `generated_skills.md`).
  - Update `tests/test_plugins_required_gate_preview.py`,
    `tests/monitor/test_monitor_followup.py`, and the others found by grep.

**Exit:** every source is stubbed or redirected, every page is within budget, and the
URL contract is green.

## Phase contribute-move

Use `docs_move` for whole moves into `contribute/`:

- `development.md` → `development.md`;
- `rust_backend.md` → `rust-backend.md` (`--stub`, since the core note cites it);
- `fakey.md`;
- `telemetry.md`;
- `perf_runbook.md` → `perf-runbook.md`;
- `mobile_mvp_runbook.md` → `mobile-mvp-runbook.md`;
- `integrations.md` → `integration-apis.md`;
- `artifacts_pane_contract.md` → `artifacts-pane-contract.md`;
- `artifacts_pane_visual_grammar.md` → `artifacts-pane-visual-grammar.md`;
- `sdd_storage.md` → `sdd-storage.md`.

Contribute budgets are warning-only, so the pages move whole.

- **Provenance.** Bead IDs in contributor runbooks stay as provenance. Only user-visible
  migration notes move to `reference/upgrading/`.
- **Hub.** Update `contribute/index.md`.
- **References:**
  - README's `/development/` links (the frozen `#your-sase-versus-this-checkouts-venv`
    fragment carries through the redirect).
  - `Justfile` and `src/` references to `docs/perf_runbook.md`,
    `docs/artifacts_pane_visual_grammar.md`, and `docs/rust_backend.md` (including
    `tools/render_applicability` and `src/sase/ace/tui/fonts/README.md`).
  - The test-selection and perf tests that name these paths.
- **Memory:**
  - Under `docs_pointer_notes`, update both `docs/perf_runbook.md` pointers in
    `sase/memory/tui_perf.md` through `/sase_memory_write`, then run `sase memory init`.
  - Record a `PROPOSED FOLLOW-UP:` note: `rust_core_backend_boundary.md` still cites
    `docs/rust_backend.md`, which now resolves through a stub; updating it needs a
    core-memory decision.

**Exit:** all ten pages live under `contribute/`, `/rust_backend/` serves a stub, and
the URL contract is green.

## Phase agents-vcs-move

- **`agent_sessions.md` (7.9K)** → `reference/agents/` session pages, split by H3.
- **`workspace.md` (6.6K)** → workspace pages, two if over budget.
- **`agents_sidecar.md`** → `reference/agents/hood-sync.md`, with sync internals →
  `contribute/`.
- **`init.md` (7.2K)** → initialization pages; migration notes → `reference/upgrading/`,
  moved verbatim.
- **`vcs.md`** → `reference/vcs/` providers, with provider APIs → `contribute/`.
- **`commit_workflows.md` (8.2K)** → `reference/vcs/` commit-workflow pages, with
  orchestrator internals → `contribute/`.
- **Hubs:** add `reference/agents/index.md` (also linking the existing
  `agent-auto-restart.md`, `agent-revive-log.md`, and `runner-slots.md`) and
  `reference/vcs/index.md`, plus their nav sub-groups.
- **References:** `src/` references to `docs/init.md` (service platform, ssh_agent,
  workspace lease, plan approval side effects) and the tests that name these paths.

**Exit:** every source is stubbed or redirected, every page is within budget, and the
URL contract is green.

## Phase integrations-move

- **`plugins.md` (9.4K)** → `reference/integrations/plugins*.md`, with the plugin
  authoring API → `contribute/plugin-api.md`.
- **Whole moves:** `editor.md`, `completion.md`, and `remote_dispatch.md` →
  `reference/integrations/`.
- **`mobile_gateway.md`** → `reference/integrations/`, with API details → `contribute/`.
- **Single-topic pages**, each moved whole:
  - `query_language.md` → `reference/query-language.md`;
  - `agent_images.md` → `reference/agent-attachments.md`;
  - `prompt.md` → `reference/prompt-history.md`;
  - `artifact_references.md` → `reference/artifact-references.md` (its reference-style
    link definitions must be rewritten);
  - `artifact_links.md` → `reference/artifact-links.md`.
- **Hub:** add `reference/integrations/index.md` and nav.
- **References:**
  - Update `src/sase/main/parser_machine.py`, `src/sase/plugins/_required_gate_spec.py`,
    `tools/render_applicability`, `src/sase/sdd/artifact_links.py`,
    `src/sase/agents_sync/publication_snapshot.py`, and the tests found by grep.
  - Many test hits on `prompt.md` and `memory.md` are unrelated fixture names; change
    only real docs paths.
- **Memory:** under `docs_pointer_notes`, update `sase/memory/cli_rules.md` (the Command
  plugins section pointer) and `sase/memory/dispatch.md` (the `docs/remote_dispatch.md`
  pointer) through `/sase_memory_write`, then run `sase memory init`.

**Exit:** every source is stubbed or redirected, and the URL contract is green.

## Phase cli

- **`configuration.md` "## CLI Flags" (19.8K):** for each `### sase …` subsection,
  compare its table with `reference/cli/<group>.md`.
  - Port facts the generated page lacks into argparse help or descriptions (per
    `cli_rules.md`), and regenerate with `just docs-gen`.
  - Move behavior prose to the owning subsystem reference page. It cannot live in
    `reference/cli/`, which `render_cli` prunes.
  - Delete the table, using a `drop` mapping whose `anchor_target` is the generated
    group page (or its subcommand anchor).
  - Finish `configuration.md` as a stub whose primary link is
    `reference/config/index.md`.
- **`cli.md` (6.9K):**
  - Fold its routing role into `reference/index.md`, which stays at or under 400 words,
    plus a short sentence in `render_cli`'s generated index intro if useful.
  - Move its behavior sections (autonomy, agent auto-restart/search/wait/hold, service,
    goal, doctor support reports) to subsystem pages.
  - Finish it as a stub whose primary link is `reference/cli/index.md`.
- **Help-text disagreements.** Stale caveats such as the compact-help memory blurb are
  code/docs disagreements. Fix the help text if it is small and clearly wrong; otherwise
  record a `PROPOSED FOLLOW-UP:` (bug) note.
- **Memory:** under `docs_pointer_notes`, update `sase/memory/macros.md`'s `docs/cli.md`
  pointer, then run `sase memory init`.

**Exit:**

- no hand-maintained CLI flag table remains in `docs/`;
- `configuration.md` and `cli.md` are generated stubs;
- the URL contract is green.

## Phase tui-edit

Run the edit checklist on `reference/tui/` and `contribute/tui-rendering.md` (about 120
history hits). Pay particular attention to keymap rows and key lists that restate the
generated `reference/tui/keymaps.md`.

**Exit:**

- the checklist counts are recorded;
- no TUI page grew past budget;
- the URL contract is green.

## Phase config-macros-edit

Run the edit checklist on:

- the `reference/config/` companion pages;
- `reference/environment-variables.md`;
- `reference/tui/admin-center.md`;
- `reference/macros/` (hand-written pages only);
- `reference/workflows.md`.

Config companion pages lose any text that only restates a schema description or default.

**Exit:** the checklist counts are recorded, and the URL contract is green.

## Phase work-units-providers-edit

Run the edit checklist on `reference/work-units/`, `reference/providers/`, and
`reference/memory/`. The bead command pages lose flag tables that restate the generated
`reference/cli/bead.md`.

**Exit:** the checklist counts are recorded, and the URL contract is green.

## Phase automation-edit

Run the edit checklist on `reference/automation/`.

**Exit:** the checklist counts are recorded, and the URL contract is green.

## Phase agents-integrations-edit

Run the edit checklist on `reference/agents/`, `reference/vcs/`,
`reference/integrations/`, and the five single-topic pages moved by `integrations-move`.

**Exit:** the checklist counts are recorded, and the URL contract is green.

## Phase finish

- **Hubs and nav:**
  - Write the final `reference/index.md`, routing by product unit within 400 words: CLI,
    Configuration, TUI, Macros and workflows, Work units, Agents and workspaces, VCS,
    Automation, Providers, Memory, Integrations, single-topic pages, Glossary, Design
    decisions, and Upgrading.
  - Write the final `contribute/index.md`.
  - Audit nav against the success metric: at most 30 visible sidebar items in Guides and
    Reference, and at most 10 elsewhere. Count from a built site or the nav structure,
    and nest the generated CLI command pages if needed.
- **Empty allowlist:**
  - Remove the `index.md` entry from `budgets.yml` (the homepage rule from `split-tool`
    now governs it) and add a homepage `description`. Leave `ceilings: {}`.
  - Update the "legacy" wording in `tools/check_docs` docstrings, the `Justfile` comment
    on `fix-docs-budgets`, and `docs/contribute/writing-docs.md` (the tabs section and
    the budget table row) so nothing claims legacy root pages still exist.
- **Repo-wide sweep.** `git grep` every old root path (`docs/<name>.md` and
  `sase.sh/<name>/`). Hits may remain only in `moves.yml`, generated stubs, decision
  records, the core note, `CHANGELOG.md`, and `reference/upgrading/`. Convert README's
  remaining relative `docs/*.md` links to canonical `https://sase.sh/...` URLs.
- **Docs map:** run `just fix`, then confirm that `docs/README.md` lists every page with
  its description and that its "Moved pages" table covers every move.
- **`docs_writing.md`:** under `docs_routing_note`, remove the now-false parenthetical
  "(today, the legacy root page that owns the fact)" and restate the budget rule without
  legacy pages, through `/sase_memory_write`. Then run `sase memory init`.
- **Reference hygiene decision:**

  > [!decision] reference_hygiene Add a `check_docs` rule that fails hand-written
  > `reference/` pages containing bead IDs or epic or phase labels (generated pages and
  > blocks are exempt). History phrases stay allowed, because Reference documents
  > deprecated flags. Before enabling it, clean any bead IDs that feature agents added
  > during the sub-epic, add fixture tests, and document the rule in `writing-docs.md`.

  > [!decision] reference_hygiene = no Add no rule. Record the remaining bead-ID and
  > epic-label counts in `reference/` with the success metrics.

- **PDF check.** Run `just docs-pdf-check` once to confirm the curated handbook still
  builds after the link rewrites. If it cannot run locally, record why.
- **Metrics.** Measure the parent plan's success-metrics table: Guide words, sidebar
  items, history, bead-ID, and `src/` counts in the Guide, hand-maintained registry
  tables, broken sase.sh URLs, and PDF size where measurable. Record it as a note on
  epic `sase-1le` and on this phase's bead.

**Exit:** the parent decompose criteria hold:

- no root pages remain except `index.md` and generated stubs;
- the `budgets.yml` allowlist is empty;
- no `reference/` page is over budget;
- the URL contract is green;
- the metrics are recorded.

## Landing

The land agent for this plan:

- triages the phases' `PROPOSED FOLLOW-UP:` notes;
- re-verifies the parent decompose exit criteria on master with `sase tool run check`
  and `just docs-check`;
- runs `sase bead epic-symbols sase-1le.20` and resolves any leftovers;
- closes phase `sase-1le.20` with a note naming what it verified.

It never closes the parent epic `sase-1le` or any other ancestor; `sase-1le`'s own land
agent owns that.
