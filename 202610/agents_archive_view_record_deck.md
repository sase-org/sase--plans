---
tier: epic
title: Record deck for every agent, built in five bounded phases
goal: The Agents tab offers a fifth agent data deck (RECORD, picker key r) with
  Lifecycle, Provenance, Relations, and Links cards for inbox and archived agents
  alike, ready for the retire-pane phase to absorb the old Details panel.
phases:
- id: record-registry
  title: RECORD deck registry, cycle order, and persistence
  size: small
  depends_on: []
  description: 'record-registry: add DeckId.RECORD to widgets/decks/model.py and its
    _DeckSpec to widgets/decks/spec.py (name RECORD, glyph ≡, picker key r, accent
    #87AFAF, blurb "Lifecycle, provenance, relations, links"). Place Record last in
    DECK_CYCLE and active_deck_cycle. Extend models/agent_deck_persistence.py so
    states without Record still round-trip and unknown decks keep their tolerant
    fallback. Unit-test the spec entry, cycle order, coercion, and persistence
    round-trip.'
- id: record-chrome
  title: Record panel compose, picker, availability, and keys
  size: medium
  depends_on:
  - record-registry
  description: 'record-chrome: wire RECORD through DeckPanel compose
    (widgets/decks/panel.py), titles, the deck picker modal (fifth RECORD row with
    glyph ≡ and key r), widgets/decks/availability.py (a probe that always reports
    at least one card), deck cycling, default_config.yml, the keymap registries,
    the ? help modal Agents section, and footer metadata. Resolve the exact cycle
    keys against the keymaps (cycle_deck_view and Ctrl+J/Ctrl+K); do not invent
    bindings. Test the picker row, picker key, and always-available probe.'
- id: record-facts
  title: Lifecycle and Provenance cards for both views
  size: medium
  depends_on:
  - record-registry
  description: 'record-facts: build the Lifecycle card (@agent: reference, Inbox or
    Archive location, status and outcome, started/ended/dismissed times with time
    basis, revived ×N, capabilities with missing requirements) and the Provenance
    card (project, workspace, model, provider and effort, agent tab, tribe, clan
    tribe, owner, source run id, chat path, artifacts dir, bundle path when
    archived) in a new widgets/decks/record_facts.py module. Use existing Agent
    fields for inbox agents and the sase-1jm.2 archive-projection derivation
    (core/agent_archive_facade.py capabilities and corpus summary) for archived
    agents; never add a parallel status table. Test card content for an inbox
    agent, an archived agent, a container, and a thin agent with no artifacts
    dir.'
- id: record-relations
  title: Relations and Links cards with off-thread link facets
  size: medium
  depends_on:
  - record-registry
  description: 'record-relations: build the Relations card (session, clan, parent,
    workflow parent, retry_of/retried_as as copyable references; interactive jumps
    stay in the Jump panel, no jump behavior changes) and the Links card
    (incoming and outgoing artifact links grouped by relation) in a new
    widgets/decks/record_relations.py module, reusing the cached
    load_artifact_links_snapshot (ace/tui/relations/artifact_links.py) off-thread.
    Carry over the facts the retiring pane holds (widgets/artifacts/
    agents_detail_panel.py and agents_detail.py) without touching the pane
    itself. Test card content for an agent with links and an agent with none.'
- id: record-proof
  title: Both-view integration, PNG goldens, and land evidence
  size: medium
  depends_on:
  - record-chrome
  - record-facts
  - record-relations
  description: 'record-proof: integrate RECORD into the deck refresh seam
    (widgets/_agent_detail_deck_refresh.py) for inbox and archived agents with
    the existing 150 ms debounce and read-only guards; add PNG goldens for the
    Record deck on an inbox agent and an archived agent (slate #87AFAF chrome);
    complete deck-persistence and picker-key tests; run the scoped check and
    inspect every golden change. Record verification evidence for the child epic
    land agent, without closing sase-1jm.4 or any ancestor.'
proposed_by: sase-1jm.4
parent_bead: sase-1jm.4
create_time: 2026-10-11 05:00:00
status: wip
bead_id: sase-1jm.4.1
---

- **PARENT PHASE:** `sase-1jm.4` (Record deck for every agent)
- **PARENT EPIC:** [202610/agents_archive_view.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_view.md)
- **BEAD:** `sase-1jm.4.1`

# Record deck for every agent, built in five bounded phases

This is a child epic of the assigned phase **sase-1jm.4**, not a replacement for the
parent epic **sase-1jm**. It implements only the parent's **record-deck** phase. The
child epic land agent is responsible for verifying this complete scope and closing
`sase-1jm.4`; child phase workers close only their own assigned beads. The child land
agent may finish its own child epic and this enclosing assignment, but may never close
`sase-1jm` or any other ancestor plan bead.

## Why a child epic

Three worker attempts cycled on `sase-1jm.4` (notes #1–#4) without landing any code:
verified 2026-10-11, the tree contains no `DeckId.RECORD`, no spec entry, no Record
card module, and no Record goldens, and the working tree is clean. A prior note's
completion claim (Record goldens, a `record/document.py` mypy fix) matches nothing in
the tree; child workers must not treat it as landed. The remaining work crosses the
deck registry, panel chrome, two independent card families, and golden proof, and does
not fit one single-turn tale. The five child phases above each implement one bounded
seam. `record-facts` and `record-relations` touch disjoint new modules and may run in
parallel once `record-registry` lands; `record-proof` integrates and proves the whole.

The parent has separate phases for the Archive view (`sase-1jm.3`, whose child epic
`sase-1jm.3.1` phase `archive-reader` explicitly consumes "the independently supplied
Record deck"), restore/fork/copy, arrival routes, shelf/bridge/Node Finder, and
retiring the Artifacts Agent pane (`sase-1jm.8`, which absorbs the pane's Details
panel). Integrate their seams; do not duplicate their work here. Do not retire the
pane, create the sunset flag, or remove old routes in this child epic.

## Accepted constraints and scope

The governing design is `plan:202610/agents_archive_view.md`, read with
`sase artifact read`. Its final decisions apply without another review:

- `view_name = archive`: the Lifecycle card's location reads **Inbox** / **Archive**.
- `toggle_filter = restore` and `jump_opens_archive = yes`: out of scope here; the
  Relations card offers copyable references only and changes no Jump behavior.
- `glossary_archive_term = no` and `glossary_strand_updates = no`: edit no memory
  files. The skipped glossary follow-ups are already recorded as `PROPOSED
  FOLLOW-UP:` notes #1–#2 on `sase-1jm.4` for the parent land agent; do not re-record
  them and do not touch any other memory note.

## Existing seams verified during planning

These paths are relative to the primary checkout and are entry points, not a requirement
to grow the existing large modules. Resolve any moved filenames with `rg`. All TUI
paths below are under `src/sase/ace/tui/` unless the complete path is shown.

| Concern                        | Existing implementation                                                                                                                                 |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Deck identity and cycle         | `widgets/decks/model.py`: `DeckId` (`MAIN`, `FILES`, `TOOLS`, `FINAL`), `DECK_CYCLE`, `cycle_deck_id`                                                    |
| Deck registry                   | `widgets/decks/spec.py`: `DECK_SPECS`, `deck_spec`, `active_deck_cycle`, `coerce_known_deck` (FINAL pattern: name, glyph, picker key, blurb, accent class) |
| Picker keys and titles          | `widgets/decks/titles.py`: `DECK_PICKER_KEYS` derived from the specs; `widgets/decks/picker.py`; `modals/deck_picker_modal.py`                           |
| Availability probes             | `widgets/decks/availability.py`: `probe_files_deck`, `probe_tools_deck`, `probe_final_deck`                                                              |
| Panel compose and layout        | `widgets/decks/panel.py` (`DeckPanel`); `widgets/_agent_detail_deck_layout.py` (iterates `active_deck_cycle`); `widgets/decks/card_documents.py`        |
| Refresh seam                    | `widgets/_agent_detail_deck_refresh.py`, `widgets/_agent_detail_deck_source.py`, `widgets/_agent_detail_files.py`                                        |
| Persisted deck layout           | `models/agent_deck_persistence.py` (`ace_agents_deck_state.json`, tolerant unknown-deck fallback)                                                        |
| Archive derivation              | `src/sase/core/agent_archive_facade.py` (`AgentArchiveCorpus` summary/lookup/count, `capabilities_from_bundle`); `agent_archive_corpus_cache.py`          |
| Artifact link facets            | `ace/tui/relations/artifact_links.py:99` `load_artifact_links_snapshot`, cached by aggregate signature                                                  |
| Retiring pane facts             | `widgets/artifacts/agents_detail_panel.py`, `widgets/artifacts/agents_detail.py` (Details panel and relation rail source material; read-only here)       |
| Bindings and documentation      | `src/sase/default_config.yml` (`cycle_deck_view`), `keymaps/`, `actions/agent_workflow/_leader_mode.py`, `modals/help_modal/agents_main_sections.py`     |
| Deck picker tests               | `tests/ace/tui/modals/test_deck_picker_modal.py`; visual fixtures under `tests/ace/tui/visual/`                                                          |

## Shared contracts

Small modules implement this epic: one new `widgets/decks/record_facts.py` for the
Lifecycle and Provenance cards, one new `widgets/decks/record_relations.py` for the
Relations and Links cards. Presentation state belongs in Python; outcome
classification, last-activity derivation, container grouping, ordering, filter
evaluation, group counts, and exact lookup stay in the existing Rust-backed corpus.
Never implement a second status table or container algorithm in Python.

Choose the Record card builder names and the probe name once in `record-registry` (or
the first phase that needs them) and record them on that phase's bead; subsequent
phases use the same contracts.

If an actual missing shared-domain contract is found, open `sase-core` through the
repo skill, change the Rust contract and binding there, follow its instructions, and
update the primary revision pin only after its host commit exists. First try the
supplied phase-2 contract; do not expand this epic into a core redesign.

### Two implementation traps

1. `DECK_PICKER_KEYS` and the help modal's deck line derive from `DECK_SPECS` and
   `active_deck_cycle`, so registry changes propagate automatically — but the picker
   modal rows, panel compose branches, availability map, and persistence codecs each
   enumerate decks explicitly. Audit every `DeckId.FINAL` call site for a Record arm;
   a fallthrough that silently renders FINAL content under a RECORD header is the
   failure to fear.
2. Never call the corpus catalog-sized facet path on ticks or query previews. Build
   the Links card from the cached link snapshot off-thread, revalidating only on
   signature drift, the way the Archive controller reuses its warm handle.

## Public presentation seams for parent phases

- The Archive `archive-reader` phase (`sase-1jm.3.1.5`) renders RECORD through the
  same refresh seam with static read-only guards and titled missing-content cards
  (for example _Archive entry unreadable — Record only_). Keep one code path for
  both views; the read-only context decides behavior, never a second renderer.
- The retire-pane phase (`sase-1jm.8`) absorbs the pane's Details panel and relation
  rail into these cards. Keep card titles stable once `record-proof` lands so the
  pane migration can point at them.
