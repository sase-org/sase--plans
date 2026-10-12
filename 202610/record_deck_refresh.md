---
tier: epic
title: Finish the Record deck and close sase-1jm.4
goal: 'Every agent the Agents tab can select (an inbox agent, a hydrated archived
  run, an unreadable archive entry, and an archive container row) gets a fifth RECORD
  agent data deck (glyph ≡, picker key r, slate #87AFAF). It has Lifecycle, Provenance,
  Relations, and Links cards, all built off the message pump and proven by unit tests
  and PNG goldens, so sase-1jm.4 closes and unblocks sase-1jm.8.

  '
parent_bead: sase-1jm.4
phases:
- id: shell
  title: RECORD registry, record package contract, and every explicit deck call site
  depends_on: []
  size: medium
  description: 'shell: close the stale sase-1jm.4.1 as superseded. Register RECORD
    last in the deck cycle. Create the record package contract with stub cards and
    an off-thread loader. Give every explicit deck call site a RECORD arm and keep
    new panels from auto-choosing RECORD. Update the unit tests and rebaseline the
    existing PNG goldens that the fifth deck changes.'
- id: facts
  title: Lifecycle and Provenance cards
  depends_on:
  - shell
  size: medium
  description: 'facts: fill record/facts.py with the Lifecycle and Provenance builders
    and their off-thread extras. Cover inbox agents, clan containers, thin agents,
    and hydrated archived agents; archived facts come from the Rust light row and
    the dismissed-bundle index.'
- id: relations
  title: Relations and Links cards
  depends_on:
  - shell
  size: medium
  description: 'relations: fill record/relations.py with the Relations card and the
    Links card. Relations shows resolved, copyable session, clan, parent, workflow,
    and retry references. Links shows artifact-link chips grouped by relation and
    direction, read off-thread from the shared cached link index.'
- id: archive-rows
  title: Light-row Record for unreadable archive entries and container rows
  depends_on:
  - facts
  - relations
  size: medium
  description: 'archive-rows: build a RecordSubject from an Archive light row so unreadable
    entries and container rows render a Record deck. Make the archived identity header''s
    time-basis label agree with the Lifecycle card. Rebaseline the affected Archive
    goldens.'
- id: proof
  title: Record goldens, docs, performance evidence, and land evidence
  depends_on:
  - archive-rows
  size: medium
  description: 'proof: add Record PNG goldens for inbox, archived, unreadable, and
    linked selections. Update docs/ace.md, confirm j/k performance with Record visible,
    and record the verification evidence on this epic''s bead for the land agent.'
proposed_by: bbugyi200.athena.0zs
create_time: 2026-10-12 06:48:31
status: done
bead_id: sase-1jm.4.2
---

- **PROMPT:** [prompts/202610/record_deck_refresh.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/record_deck_refresh.md)
- **BEAD:** [sase-1jm.4.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.4.2.md)

# Finish the Record deck and close sase-1jm.4

## Why this epic exists, and why it supersedes sase-1jm.4.1

`sase-1jm.4` is the **record-deck** phase of epic `sase-1jm`. Its governing design is
`plan:202610/agents_archive_view.md`, section _Record deck for every agent_. Read it
with `sase artifact read`. The phase adds a fifth agent data deck that becomes the one
home for the facts that the retiring Artifacts ▸ Agent pane's Details panel and relation
rail hold. It covers inbox and archived agents alike.

Nothing has landed yet:

- The first worker and its three retries built a Record deck in an uncommitted
  workspace. The scoped `just check` then failed with 11 tests: new panels auto-chose
  RECORD (four tests in `test_deck_layout.py`, plus `test_deck_panels.py` and
  `test_deck_split_pilot.py`) and the deck subtitle strings changed. The turn ended and
  the diff was lost. Note #3 on `sase-1jm.4` claims completion, but nothing in the tree
  matches it. Do not trust that note.
- Child epic `sase-1jm.4.1` was authored on 2026-10-11 but never launched. All five of
  its phases are open and unassigned. Its seam table predates the landing of
  `sase-1jm.3` (Archive view, including its own nested child epics), `sase-1jm.5`
  (restore, fork, copy), `sase-1jm.6` (arrivals), and the Archive module splits. It also
  misses the traps listed below. This epic replaces it, and phase `shell` closes it as
  superseded before writing any code.

`sase-1jm.8` (retire the pane) depends on `sase-1jm.4` and will point at these cards.
Keep the four card titles stable once `shell` lands: **Lifecycle**, **Provenance**,
**Relations**, **Links**.

## Final constraints

- The parent epic's decisions are final:
  - `view_name = archive`: the Lifecycle location reads **Inbox** or **Archive**.
  - `toggle_filter` and `jump_opens_archive` are out of scope. Relations offers copyable
    references only, and Jump behavior does not change.
  - `glossary_archive_term = no` and `glossary_strand_updates = no`: **edit no memory
    notes.** The skipped glossary follow-ups are already notes #1–#2 on `sase-1jm.4` for
    the `sase-1jm` land agent. Do not re-record them.
- Do not retire the pane, add a feature flag, add key bindings, or remove routes. Treat
  `src/sase/ace/tui/widgets/artifacts/agents_detail.py` and `agents_detail_panel.py` as
  read-only source material. Copy their patterns; do not import their private helpers.
- **Rust boundary.** Outcome class, time basis, last activity, and container grouping
  for archived runs come from the Rust-backed Archive light row (`AgentArchiveCorpus` in
  `src/sase/core/agent_archive_facade.py`). Never add a Python status→outcome table.
  Inbox agents show their stored status with no outcome line. No `sase-core` change is
  expected. If a phase proves a shared contract is genuinely missing, follow the
  `rust_core_backend_boundary` core memory, and keep the change minimal.
- **Performance (the `tui_perf.md` rules).**
  - No I/O in compose, render, keystroke, or message-pump paths. That includes
    `Agent.get_artifacts_dir()`, which stats the disk, and `reference_for_agent_name`,
    which reads identity state.
  - Record loads through `run_worker(..., thread=True)` the way FINAL does.
  - It refreshes only through the existing 150 ms detail debounce and the deck refresh
    seam, with no ticker, live tail, or idle-tick work.
  - Stale results are rejected by subject identity plus detail generation.
- Projects display by name (`project_label`, the display snapshot), never by directory
  key. This is the `project-labels-never-keys` decision.
- Every new module stays under the 700-line `toobig` gate. `panel_navigation.py` is
  already about 640 lines, so Record panel handling goes in a new mixin module.
- Each phase leaves `sase tool run check` green. Run long commands through
  `/sase_monitor`. Phase workers close only their own bead; the one exception is that
  `shell` closes `sase-1jm.4.1`. Workers record `PROPOSED FOLLOW-UP:` notes instead of
  creating beads.

## Shared contract (created by `shell`, consumed by every later phase)

Package `src/sase/ace/tui/widgets/decks/record/`, modeled on `widgets/decks/final/`:

| Module         | Owner                          | Contents                                                                                                                                                                                                                                                                                                            |
| -------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model.py`     | shell (archive-rows extends)   | `RecordSubject` and `RecordArchiveFacts` frozen dataclasses; `record_subject_from_agent(...)`; `live_related_names(agents, suffixes)`                                                                                                                                                                               |
| `facts.py`     | facts (shell writes stubs)     | `FactsExtras`; `load_facts_extras(subject) -> FactsExtras` (worker thread); `lifecycle_card(subject, extras)` and `provenance_card(subject, extras)` returning `CardPart`                                                                                                                                           |
| `relations.py` | relations (shell writes stubs) | `RelationsExtras`; `load_relations_extras(subject) -> RelationsExtras` (worker thread); `relations_card(subject, extras)` and `links_card(subject, extras)` returning `CardPart`                                                                                                                                    |
| `document.py`  | shell                          | `build_record_document(subject, facts, relations) -> MainDeckDocument` with exactly four cards in fixed order. Card ids are `lifecycle`, `provenance`, `relations`, `links`; titles are Lifecycle, Provenance, Relations, Links. Paged like FINAL, defaulting to Lifecycle, with Ctrl+J/K cycling.                  |
| `loader.py`    | shell                          | `RecordDeckLoadResult` (subject identity, generation, signature, document, default card). `load_record_deck(subject, *, generation)` is the worker body: it calls both extras loaders, builds the document, and keeps an LRU cache keyed by subject identity plus subject and extras equality, like `_FINAL_CACHE`. |
| `view.py`      | shell (archive-rows extends)   | `RecordDeckLoaded` message. `RecordDeckView(CardDocumentView)` provides `update_display(agent, *, attempt_number, generation, preferred_card, archive_light_row=None, bundle_path=None)`, `show_empty()`, and `get_record_text()`.                                                                                  |

`RecordSubject` is captured on the pump from plain attribute reads, wrapped defensively
(`getattr` with defaults). Verify each attribute name with `rg` against
`src/sase/ace/tui/models/_agent_state_*.py`. It carries:

- **Identity:** a hashable identity, `kind` (`agent` | `container` | `light_row`),
  `location` (`inbox` | `archive`), `agent_name`, and `raw_suffix`.
- **Lifecycle inputs:** `status`, `start_time`, `stop_time`, `attempt_number`,
  `historically_viewable`, `durably_revivable`, `restartable`, `missing_requirements`.
- **Provenance inputs:** project label and file, `workspace_num`, `workspace_dir`,
  `model`, `model_alias`, `llm_provider`, `reasoning_effort`, `agent_tab`, `tribe`,
  `clan_tribe`, `imported_source_owner`, `source_username`, `source_machine`,
  `fleet_origin_alias`, `source_run_id`, raw `artifacts_dir`, `response_path`, and
  `bundle_path`.
- **Relation inputs:** `agent_session`, `agent_session_role`, the presented session and
  clan reference names, `agent_clan`, `agent_clan_generation`, `parent_timestamp`,
  `parent_workflow`, `retry_of_timestamp`, `retried_as_timestamp`, and `retry_attempt`.
  `related_live_names` is a tuple of (suffix, agent name) pairs for the parent and retry
  suffixes, found in the app's in-memory agent list.
- **Container members:** `member_names`, capped.
- **Archive facts:** `archive: RecordArchiveFacts | None`, copied from the light row:
  outcome, time basis, last activity, start time, runtime seconds, restorable, member
  count, container key, and archive key.

If `facts` or `relations` needs a fact the contract lacks, compute it off-thread into
that phase's own extras dataclass. Never add it to `RecordSubject`, so the two parallel
phases never edit the same file.

## Phase `shell`: RECORD registry, contract, and call sites

1. **Supersede the stale epic first.** Run
   `sase bead close sase-1jm.4.1 --force -R superseded --reason "<names this epic's bead; the never-launched plan predates the landed Archive work>"`.
   Then confirm with `sase bead read` that it and its five phases are closed.
2. **Registry.**
   - Add `DeckId.RECORD = "record"` to `widgets/decks/model.py`, and add it to
     `DECK_CYCLE` last.
   - Add a `_DeckSpec` to `widgets/decks/spec.py`: name `RECORD`, glyph `≡`, picker key
     `r`, blurb "Lifecycle, provenance, relations, links", count noun card/cards,
     fallback accent `#87AFAF`, and classes `-deck-record`. It goes last, so Ctrl+P and
     the picker's back row from a fresh Main land on RECORD. Update the tests that
     assumed FINAL.
3. **Availability and layout.**
   - Add `probe_record_deck` to `widgets/decks/availability.py`. It returns
     `DeckAvailability(True, None)`, which means always available with no count, because
     the four-card set is fixed. The switcher therefore shows a bare `record` and the
     picker row shows no count.
   - In `widgets/decks/layout.py` `choose_new_panel`, never auto-choose RECORD for a new
     split. Record is reachable only explicitly (`p r`, `P R`, Ctrl+N/Ctrl+P, the
     palette), so the triage loop's second-panel choice is unchanged. This was the
     previous attempt's main failure.
4. **Call sites. Audit every `DeckId.FINAL` branch.** A fallthrough that renders Main or
   FINAL content under a RECORD header is the failure to fear. Four sites raise or crash
   on an unknown deck, and three `match` statements fail mypy without an arm. Wire
   RECORD through all of these:
   - **Card documents:** `widgets/decks/card_documents.py` (`CARD_DOCUMENT_DECKS`,
     `resolve_document_deck`, `document_view`, `document_for`), `empty_state.py`
     (`("record", …)` messages), and `search_corpus.py`.
   - **DeckPanel:**
     - `panel.py`: compose `agent-deck-panel-{i}-record-scroll` holding the view, add an
       arm to `_deck_is_empty`, and add a `@on(RecordDeckLoaded)` handler.
     - A new `panel_record.py` mixin holding `handle_record_deck_loaded`,
       `show_record_document`, and the card cycle used by `panel_navigation.cycle_card`.
     - `panel_interaction.py` (a `record_view` property), `panel_blocks.py` (all eight
       FINAL/TOOLS branches), and `panel_chrome.py` (`_record_tabs`,
       `_active_tab_index`, `refresh_chrome`).
   - **AgentDetail seams:**
     - `widgets/_agent_detail_deck_refresh.py`: the view arm, the preferred card, both
       availability maps, and the empty and tribe states. When
       `agent_is_archive_read_only(agent)` holds, pass the light row and bundle path
       from `archive_read_only_context` (in
       `actions/agents/_archive_view_selection.py`).
     - `_agent_detail_deck_show.py`: its else arm silently shows Main.
     - `_agent_detail_state.py`: `get_editor_file_info` for `E` export.
   - **Actions and styles:**
     - `actions/agents/_display_detail_footer.py`: add a RECORD arm in both the live and
       the archive paths, using the live path's `focused.` pattern. While there, fix the
       archive FINAL arm, which reads a nonexistent `agent_detail._final_document` and
       so always counts 0.
     - `actions/navigation/_basic.py`: release and pin the bottom.
     - `styles.tcss`: `.deck-panel.-deck-record`, `-unfocused` (#87AFAF 35%), `-zoomed`,
       and the picker's focused `-deck-record` row.
     - `modals/deck_picker_modal.py`: widen `f"{row.name:<5}"` to fit the six-letter
       RECORD.
   - Persistence (`models/agent_deck_persistence.py`), the help-modal capitals, picker
     keys, and palette commands derive from `DECK_SPECS`. Verify them; don't rewrite
     them.
5. **Stub cards.** Lifecycle shows the agent name, location, and status. Provenance
   shows the project label and model. Relations shows "—". Links shows _No artifact
   links_. They make the deck usable and testable until `facts` and `relations` land.
6. **Tests.**
   - Update the cycle, picker, help, subtitle, and layout assertions:
     `test_deck_spec.py`, `test_deck_model.py`, `test_deck_picker.py`,
     `test_deck_picker_modal.py`, `test_agents_deck_picker.py`,
     `test_keymaps_display_help_agents.py`, `test_deck_titles.py`,
     `test_deck_spread_pure.py`, `test_final_deck_shell.py`, and
     `test_card_document_view.py`. Keep `test_deck_layout.py`'s expectations intact.
   - Add `test_record_deck_shell.py`, modeled on `test_final_deck_shell.py`. It covers
     the spec, the picker key `r`, the probe, the tabs, loader caching, stale-result
     rejection, a `record` persistence round-trip (plus a state without it), never
     auto-choosing on split, and a RECORD header never showing other decks' content.
7. **Goldens.** The fifth deck adds `· record` to every deck switcher and a row to the
   picker, so many `agents_*` goldens change. Run `just fix-tui-screenshots` through
   `/sase_monitor`, read its WARNING block, and inspect a sample: the picker, a split
   layout, and an Archive golden.

## Phase `facts`: Lifecycle and Provenance cards

Fill `record/facts.py` only (plus its tests).

- **Extras, off-thread:**
  - The `@agent:` reference, from `reference_for_agent_name`.
  - The artifacts dir, derived like `Agent.get_artifacts_dir()`.
  - The chat path, from `<artifacts_dir>/agent_meta.json`. Follow the pattern in
    `load_agent_detail`, `widgets/artifacts/agents_detail.py`.
  - `dismissed_at`, `revived_at`, and `times_revived` (plus the bundle path when the
    subject lacks one), from
    `query_summaries(root, suffixes={raw_suffix}, visibility=None)` in
    `src/sase/ace/dismissed_bundle_index/_api.py`. Use the same index root the Archive
    reader uses. A missing index yields no values, never an error.
- **Lifecycle:**
  - The copyable `@agent:` reference, the location (Inbox or Archive), and the status.
  - For archived runs, the light-row outcome, styled with the Archive row's existing √ /
    ✗ / ○ WAS vocabulary and helpers from `widgets/agents_archive/_rows.py`.
  - Started, ended (or _end not recorded_), and dismissed times via `format_local` and
    `parse_archive_timestamp`.
  - The time basis, with a started basis in dim italics.
  - Revived ×N, shown only when N > 0.
  - Capabilities: viewable, restorable, and restartable, plus any missing requirements.
- **Provenance:**
  - Project label, workspace, model (via `append_model_field`), provider and effort.
  - Agent tab (`main` when unset), tribe, clan tribe, and owner (user@machine, or the
    imported owner).
  - Source run id, chat path, artifacts dir (_not recorded_ for thin agents), and the
    bundle path when archived.
- **Tests** cover card content for an inbox agent, a hydrated archived agent, an inbox
  clan container, and a thin agent with no artifacts dir. They also check that the
  extras loader does no work on the pump.

## Phase `relations`: Relations and Links cards

Fill `record/relations.py` only (plus its tests).

- **Relations:**
  - Session and role, clan and generation, parent (from `parent_timestamp`), workflow
    parent (`parent_workflow`), and the retry chain (`retry_of`, `retried_as`, attempt
    N). Each is rendered as the exact `@agent:<durable name>` text the existing copy
    action delivers.
  - Resolve suffixes through `related_live_names` first. Resolve the rest off-thread
    through `query_summaries(suffixes=..., visibility=None)`. Fall back to a dim raw
    suffix.
  - Interactive jumps stay in the Jump panel.
- **Links:**
  - Off-thread, call
    `link_index_for_snapshot(load_artifact_links_snapshot(None)).chips_for(reference)`
    (`ace/tui/relations/artifact_links.py`, `relations/link_index.py`). This shares the
    module caches the app's link-rail index already warms.
  - Group by relation: outgoing then incoming, with the direction-aware label and
    neighbor ref. Cap each group, with `+N more`. Show _No artifact links_ when empty
    and a dim note for snapshot errors.
  - Put the snapshot `source_key` in the extras, so only an aggregate change repaints.
    Never call catalog-sized facet paths.
- **Tests** cover an agent with links (both directions, two relations), an agent with
  none, a resolved and an unresolved retry chain, and a live versus archived parent.

## Phase `archive-rows`: light-row Record

- Add `record_subject_from_light_row(light_row, *, members=())` to `record/model.py`.
  Kind is `light_row` for unreadable entries and `container` for container rows; facts
  come from the light row only.
- Add a `RecordDeckView.show_subject(...)` entry point and an AgentDetail hook. In
  `actions/agents/_archive_reader.py`, an unreadable entry's `_apply_hydration_result`
  path and `_paint_container_summary` then make every visible RECORD panel render the
  light-row subject.
  - RECORD stays available. The other decks keep their titled cards (_Archive entry
    unreadable — Record only_, _Container summary_).
  - No bundle is hydrated per member.
- Make `archive_identity_header_from_light_row` (`models/agent_archive_reader.py`) label
  the time by its actual basis. Today a `dismissed` basis reads "Started", which would
  contradict the Lifecycle card.
- Tests cover an unreadable entry (Lifecycle from the light row, other decks titled) and
  a container (member count, worst outcome, newest activity, member references).
  Rebaseline the Archive goldens this changes.

## Phase `proof`: goldens, docs, performance, evidence

- **PNG goldens (120x40):**
  - Record Lifecycle on an inbox agent.
  - Record Links on an agent with links.
  - Record Lifecycle on a hydrated archived agent, showing slate chrome inside the amber
    Archive.
  - Record on an unreadable archive entry.
  - Templates: `show_final()` in
    `tests/ace/tui/visual/_ace_agents_final_png_snapshot_shared.py`, and the `_wire`,
    `light_row`, `_write_reply_bundle`, and `override_flags(agents_archive_view=True)`
    fixtures in `test_ace_png_snapshots_agents_archive.py`.
  - Pin time with `pin_agents_visual_now`.
- **Docs:** update the deck list in `docs/ace.md`, and fix its stale three-deck list.
  Confirm the `?` help line reads `p P/M/F/T/N/R` within the 57-character box rule.
- **Performance:** run the j/k bench (`tests/ace/tui/bench_tui_jk_final.py`, or a Record
  variant of it) with Record visible, and stay under its existing ceiling. Confirm that
  an idle tick starts no Record worker.
- **Evidence:** add a note on this epic's bead listing the commands run, their results,
  the new and rebaselined goldens, and each acceptance item below.

## Acceptance (verified by the land agent)

1. `p r`, `P R`, Ctrl+N/Ctrl+P (RECORD last), and the palette command
   `agents.show_deck.record` reach Record. The picker shows a fifth RECORD row (≡, `r`,
   slate accent).
2. New splits never auto-choose RECORD. Persisted states with and without `record`
   round-trip, and unknown deck names still fall back.
3. Record renders for inbox agents, clan containers, thin agents, hydrated archived
   agents, unreadable archive entries, and archive container rows. It is never blank and
   never shows another deck's content.
4. Archived outcome, time basis, and last activity come from the Rust light row.
   Dismissed time and revive count come from the index. No Python status table exists.
5. There is no pump I/O and no idle-tick Record work, and the j/k bench is within its
   ceiling.
6. The card titles are Lifecycle, Provenance, Relations, Links. No memory note, pane
   module, flag, or key binding changed.

## Landing

The land agent runs `sase tool run check` and the scoped visual check
(`just test-visual` on the deck, record, and archive modules, via `/sase_monitor`). It
walks the acceptance list and turns any `PROPOSED FOLLOW-UP:` notes into task beads
through `/sase_new_task`. One candidate found while planning:
`resolve_archive_bundle_path` checks a `bundle_path` key that light rows never carry, so
it always falls back to a run-id lookup. The land agent leaves the glossary follow-ups
on `sase-1jm.4` for the `sase-1jm` land agent. It closes this epic's bead, then closes
`sase-1jm.4` with a verification note, which unblocks `sase-1jm.8`. It never closes
`sase-1jm`.
