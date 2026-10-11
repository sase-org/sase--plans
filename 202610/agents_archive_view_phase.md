---
tier: epic
title: Implement the read-only Agents Archive view for sase-1jm.3
goal: The Agents tab can browse this machine's dismissed runs through a beta-flagged
  Archive view, preserve each view's state, and render retained decks without restoring
  runs or disturbing Inbox state.
phases:
- id: view-foundation
  title: Archive view state, flag, and selection ownership
  size: medium
  depends_on: []
  description: 'view-foundation: create the agents_archive_view beta flag through
    sase flag new. Add typed Archive presentation state, a view-aware selection API,
    read-only detail context, and centralized detail-ownership guards while leaving
    the Inbox collection and existing live selection semantics intact. Define the
    entry and selection seams for the remaining child phases and parent epic consumers.
    Test both flag states and refresh ownership.'
- id: corpus-controller
  title: Off-thread Archive loading, prewarm, and revalidation
  size: medium
  depends_on:
  - view-foundation
  description: 'corpus-controller: add a coalesced Archive controller around the existing
    core corpus cache and summary, rows, lookup, and count methods. Build and query
    off-thread, publish only current generations, gate prewarm on the first complete
    Inbox load and idle UI, and revalidate index and link signatures without archive
    scans on idle ticks. Preserve selection and page requests across rebuilds. Test
    cold, warm, missing, rebuilding, and stale-result paths.'
- id: archive-column
  title: Paged Archive list and live Inbox pulse
  size: medium
  depends_on:
  - corpus-controller
  description: 'archive-column: lazily mount an Archive column beside the mounted
    Inbox column, containing a selectable Inbox pulse and an OptionList-style Archive
    list. Render core light rows, date banners, containers, and banner-local pages
    of 100 rows; add j/k/g/G/h/l and J/K navigation, echo guards, responsive columns,
    and truthful past-tense outcomes and time chips. Keep all Archive navigation state
    separate from Inbox state.'
- id: query-parking
  title: View toggles, scoped filters, parking, and Archive chrome
  size: medium
  depends_on:
  - archive-column
  description: 'query-parking: implement the pure ,a toggle, Open Archive and Back
    to Inbox palette actions, complete per-view parking, scoped FilterBar completion
    and preview, query-history and saved-slot routing, Inbox-only last-query persistence,
    and the persisted Archive grouping picker. Add the Archive info row and Agents
    marker, hide the Inbox header in Archive, and test startup, toggle, edit cancellation,
    scope errors, history, slots, and both agents_unified_query states.'
- id: archive-reader
  title: Retained read-only decks and archived identity header
  size: medium
  depends_on:
  - query-parking
  description: 'archive-reader: hydrate a selected individual bundle off-thread through
    the existing 150 ms detail debouncer and recheck view, identity, and generation
    after awaits. Paint the new archived identity and placeholder immediately; reuse
    Main, Files, Tools, FINAL, and the independently supplied Record deck with static
    read-only guards and titled missing-content cards. Render container summaries
    from light rows only and prove selecting archived RUNNING entries never tails
    logs, refreshes workspaces, clears unread, writes archive state, or revives agents.'
- id: action-boundary
  title: Archive action isolation, caller audit, footer, and help
  size: medium
  depends_on:
  - archive-reader
  description: 'action-boundary: audit every Agents selected-agent caller and route
    read operations through the typed view-aware selection while live actions use
    Inbox-only selection. Add explanatory guards for live-only keys and tab or tribe
    moves, independent Archive marks, a view-specific conditional footer, and the
    Archive help box. Preserve deck keys and the parent''s later restore, route, bridge,
    and Record extension seams. Verify Archive rows do not enter Inbox accounting
    or bulk actions.'
- id: archive-proof
  title: Archive workflow regressions, performance evidence, and PNGs
  size: medium
  depends_on:
  - action-boundary
  description: 'archive-proof: complete cross-phase regression coverage and deterministic
    Archive PNG fixtures, benchmark warm opening and j/k at archive scale, and capture
    targeted goldens through sase monitor. Inspect all golden changes and resolve
    implementation regressions. Verify both flag states, parking, detail ownership,
    no-write reading, isolation, query parity, and unaffected Inbox first paint; record
    evidence for the child epic land agent, without closing sase-1jm.3 or any ancestor.'
proposed_by: bbugyi200.athena.sase-1jm.3
parent_bead: sase-1jm.3
create_time: 2026-10-10 20:21:17
status: wip
bead_id: sase-1jm.3.1
---

- **PROMPT:** [prompts/202610/agents_archive_view_phase.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agents_archive_view_phase.md)
- **PARENT:** [202610/agents_archive_view.md](https://github.com/sase-org/sase--plans/blob/main/202610/agents_archive_view.md)
- **BEAD:** [sase-1jm.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.3.1.md)

# Implement the Agents Archive view

This is a child epic of the assigned phase **sase-1jm.3**, not a replacement for the
parent epic **sase-1jm**. It implements only the parent's **Archive view on the Agents
tab** phase. `sase plan propose` supplies the managed `parent_bead` association from the
active bead. The child epic land agent is responsible for verifying this complete scope
and closing `sase-1jm.3`; child phase workers close only their own assigned beads. The
child land agent may finish its own child epic and this enclosing assignment, but may
never close `sase-1jm` or any other ancestor plan bead.

## Accepted constraints and scope

The governing design is `plan:202610/agents_archive_view.md`, read with
`sase artifact read`. Its final decisions apply without another review:

- `view_name = archive`: all scope and user-facing names use **Archive** and
  **in:archive**.
- `toggle_filter = restore`: `,a` restores the other view's parked state; it carries no
  Inbox filter and adds no query-history entry.
- `jump_opens_archive = yes`: provide the read-only exact-arrival seam needed by the
  parent routing phase. That phase implements Jump-digit routing.
- `glossary_archive_term = no` and `glossary_strand_updates = no`: edit no memory files.
  Record the skipped glossary work as `PROPOSED FOLLOW-UP:` notes on `sase-1jm.3` for
  the parent land agent.

Phase `sase-1jm.2` is closed and has already supplied the Rust corpus, Python
facade/cache, `agents-archive` query profile, host scope parser, and CLI parity. There
are no earlier work or remaining-work notes on `sase-1jm.3` at planning time. The
remaining TUI work crosses several independent lifecycles and is too large for a single
medium tale. The seven child phases above each implement one bounded seam; explicit
serial dependencies keep ownership changes from racing presentation changes in the same
files.

The parent has separate phases for Record decks, restore/fork/copy chooser behavior,
agent arrival routes, shelf/bridge/Node Finder, and retiring the Artifacts Agent pane.
Integrate their seams; do not duplicate their work here. Keep the beta flag until the
parent's retire-pane phase removes it. Do not create the sunset flag or remove the old
pane in this child epic.

## Existing seams verified during planning

These paths are relative to the primary checkout and are entry points, not a requirement
to grow the existing large modules:

| Concern                          | Existing implementation                                                                                                                                                          |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Archive corpus operations        | `src/sase/core/agent_archive_facade.py`: `AgentArchiveCorpus.summary`, `rows`, `lookup`, `count`, and `agents_archive_profile`                                                   |
| Shared lazy corpus cache         | `src/sase/core/agent_archive_corpus_cache.py`: `get_agent_archive_corpus`, cache key and invalidation                                                                            |
| Scope extraction and validation  | `src/sase/ace/query/scope_token.py`; `models/agent_live_query_engine.py` already canonicalizes scoped queries                                                                    |
| Filter editor and query commits  | `actions/agents/_filter_bar_session.py`, `actions/artifacts_query_history.py`, `widgets/agents_filter_bar.py`, and `widgets/_filter_bar_completion.py`                           |
| Persisted last query             | `actions/agents/_query_persistence.py` and `models/agent_query_persistence.py`                                                                                                   |
| Layout                           | `_app_layout.py`: `#agents-content`, `#agent-list-container`, `#agents-header`, and shared `#agent-detail-panel`                                                                 |
| Detail rendering and generations | `actions/agents/_display_detail_render.py`: immediate and debounced rendering, enriched-header and async-result handlers                                                         |
| Inbox refresh                    | `actions/agents/_display_refresh.py`, `_display_incremental.py`, `_loading_apply.py`, `_loading_finalize.py`, and `actions/event_refresh/`                                       |
| Selected Inbox agents            | `actions/agents/_selection.py`: `_get_selected_agent` and `_agents_in_focused_panel`                                                                                             |
| Deck content                     | `widgets/_agent_detail_deck_source.py`, `_agent_detail_deck_refresh.py`, `_agent_detail_files.py`, `widgets/prompt_panel/`, and `widgets/decks/`                                 |
| Tab marker and count pulse       | `widgets/tab_bar.py` and `agent_count_chip.py`                                                                                                                                   |
| Bindings and documentation       | `src/sase/default_config.yml`, `keymaps/`, `actions/agent_workflow/_leader_mode.py`, `modals/help_modal/`, and footer modules                                                    |
| Archive test data and scale      | `tests/_agent_archive_index_fixtures.py`, `tests/core/test_agent_archive_corpus_facade.py`, `tests/_agent_search_cli_archive.py`, and `tests/perf/bench_agent_archive_corpus.py` |

Resolve any moved filenames with `rg`. All TUI paths in the table are under
`src/sase/ace/tui/` unless the complete path is shown.

Two implementation traps deserve explicit handling:

1. The detail seam has several paths, not just `_apply_agent_detail_update`:
   `_apply_agent_detail_immediate`, deferred rendering, onboarding, clan or tribe
   summaries, enriched headers, and pending widget workers can all repaint the shared
   panel. A view-owned render token must guard each path and invalidate earlier work
   when the view changes.
2. `get_agent_archive_corpus` without explicit link facets builds the name-registry
   catalog to derive facets. Never call it on every tick or query preview. Keep a warm
   handle in the Archive controller, revalidate cheap signatures, and reuse the shared
   facet/cache path off-thread only when inputs drift. This phase must not reintroduce
   the old pane's catalog-sized first-row delay on warm opens.

## Shared contracts

Implement small modules under `models/` and `actions/agents/`, and a dedicated
`widgets/agents_archive/` package. Presentation state belongs in Python; outcome
classification, timestamps, root/container identity, membership, order, matching,
counts, and exact lookup stay in the existing Rust-backed corpus. Never implement a
second status table, SQL matcher, or container grouping algorithm in Python.

If an actual missing shared-domain contract is found, use `/sase_repo` to open
`sase-core`, change the Rust contract and binding there, follow its instructions, and
update the primary revision pin only after its host commit exists. First try the
supplied phase-2 contract; do not expand this epic into a core redesign.

### View and selection

- `ArchiveViewState` owns committed scoped query (initially `in:archive`), grouping
  (initially day), expanded banners and containers, loaded page windows, selected stable
  archive identity, scroll, focused pulse/list, per-panel deck and card, and Archive
  marks. Only grouping persists across launches.
- Park the Inbox's committed query, selected identity, group/panel folds, agent tab,
  panel focus, scroll, and deck/card choices independently. Reuse its existing tab/fold
  persistence rather than re-creating an Inbox state machine.
- Deck splits are shared between views; deck/card choices and scroll are parked.
- Keep `_agents`, `_agents_with_children`, indices, unread/attention/pin sets,
  prospective clans, tab counts, and marks as Inbox data. Never append a bundle agent or
  a core light row to those collections.
- Introduce a typed selected-view result carrying view, row kind, hydrated agent when
  available, read-only status, stable archive key, and light-row provenance. A banner,
  pulse, loading row, or page tail is not an agent. A container summary owns matching
  light members without hydrating every member.
- Keep the existing `_get_selected_agent` semantics explicitly Inbox-only. Read callers
  opt into the new accessor; live-only callers cannot accidentally act on a parked Inbox
  selection while the Archive is focused.
- Carry read-only context outside serialized bundle state. Do not rewrite stored status
  to DONE to suppress live behavior, and do not persist TUI state into a bundle.

### Public presentation seams for parent phases

Choose clear method names once in view-foundation and record them on that phase's bead;
subsequent phases use the same contracts:

- Open Agents Archive with parked state, or with an explicitly supplied scoped query and
  optional exact agent name. An explicit arrival is not the pure toggle. Return a typed
  found/owner/missing outcome so the parent routing phase can own its toasts and one
  trail hop.
- Return to the parked Inbox and expose the existing live reveal mechanism to
  restore-and-show callers. Do not implement revive execution here.
- Read the selected Archive row, matching container members, Archive marks, and hydrated
  agent without consulting Inbox selection.
- Expose warm-corpus state, status/count results, and generation-safe requests to the
  parent shelf/bridge/Node Finder phase. No consumer builds a second corpus.
- Expose a retained restored-row presentation overlay and its invalidation on query
  change or Archive exit; the parent's restore phase writes this overlay. It never
  changes the immutable corpus or counts restored rows as dismissed.
- Pass archive provenance and read-only context to deck renderers so the parallel Record
  phase can render Lifecycle/Provenance facts without creating another hydration worker.

## Phase details

### view-foundation

1. Read the flag and TUI memory through `/sase_memory_read`. Create
   `agents_archive_view` exactly once with `sase flag new`, then paste its generated
   registry member using the printed bead ID. This prescribed flag command is the
   exception to the prohibition on manually creating follow-up beads. Suggested authored
   fields:
   - enabled: the Agents tab offers a read-only Archive of local dismissed runs;
   - disabled: the Inbox and Artifacts Agent pane keep their existing behavior;
   - removal: the parent epic has verified all Archive entry, reading, restore, copy,
     migration, and pane-retirement paths. If another parent worker has already created
     this exact key, reuse it and its flag bead rather than creating another one.
2. Add the view and selection records, startup state, lifecycle initialization, teardown
   invalidation, and public contracts above. Startup is always Inbox.
3. Centralize detail ownership in `_display_detail_render.py` with a token that contains
   view plus selection generation/identity. Block Inbox-originated immediate, debounced,
   async-header, onboarding, and summary rendering while Archive owns the panel,
   including workers started before a switch.
4. With the beta off, preserve existing detail behavior and expose no new view. Test
   both states, no selected-agent leakage, pending callbacks after a view switch, and
   return-to-Inbox repaint. Guard before cancelling the Archive's pending debounce from
   a hidden Inbox refresh.

### corpus-controller

1. Wrap the supplied corpus accessor in a pump-free controller with immutable request
   snapshots and last-request-wins generations. All disk probing, facet loading, compile
   work, and corpus query calls run off-thread. UI callbacks only schedule tasks or
   apply still-current results; use `spawn_pump_free_task` and cancel at teardown.
2. Hold the warm corpus and resolved profile, summaries, loaded windows, status, and
   cheap revalidation tokens. Query/page requests use the handle; they never re-run
   name-registry discovery. Detect both index and artifact-link drift, profile changes,
   and rebuild progress changes. A rebuild from one overlapping request cannot replace a
   newer query or overwrite another view's selection.
3. Arm one idle prewarm only after the first complete Inbox load has actually been
   applied, not after compose, a bounded first paint, or an incomplete reconciliation.
   An early explicit `,a` paints loading chrome and queues work until this gate is
   satisfied. If the Inbox has not loaded yet (for example, startup on Artifacts),
   schedule its existing async load/reconciliation so the gate can actually settle.
   Defer background re-query during `NavigationGate` or prompt typing.
4. Add minimal hooks to the existing auto-refresh and Inbox-applied paths. Quiet ticks
   compare only cheap signatures/progress; no bundles or catalog scan. Inbox refresh
   supplies pulse data without an additional agent load. With beta off, do no Archive
   probing or prewarming.
5. On index failure, publish an honest missing/unreadable state with
   `Archive index unavailable — sase agent archive rebuild-index`. Do not build or
   migrate the index as a side effect of reading. A rebuilding index serves its indexed
   rows and row-count progress. Keep selection by stable identity after drift, and
   define an adjacent valid fallback when it no longer exists.
6. Tests use blocked workers and input drift to prove generation rejection,
   cancellation, warm-handle reuse, no pre-gate compile, no per-tick scan, and an
   unaffected Inbox when index loading fails.

### archive-column

1. Lazily mount the Archive column under `#agents-content` as a sibling of
   `#agent-list-container`. Toggle `display`; the Inbox widgets stay mounted and
   continue refreshing while hidden. The Archive contains an always-live one-line Inbox
   pulse followed by its list; both are nav sections.
2. The pulse reuses `format_agent_count_chip` and the existing attention count. It shows
   `⌂ Inbox · N [R… W… U…] · M needs you`, omitting attention at zero. Enter, `l`, or
   click on the pulse returns to the parked Inbox. `J`/`K` switches pulse/list focus
   without changing the parked Inbox cursor.
3. Render only core light windows: time, outcome, name, model, runtime, and exception
   chips. Use amber `#d7af5f` for Archive chrome, dim green/red for terminal outcomes,
   and existing STOPPED violet for `○ WAS <stored status>`. A start-only time is dim
   italic. Use core-derived time basis and runtime; never file mtimes. Middle-truncate
   names; drop MODEL and then RUNTIME when width requires it.
4. Flatten date banners as Today, Yesterday, remaining last-week days, weeks for the
   rest of two months, then months. Reuse existing BY_DATE label helpers. Core daily
   summaries remain authoritative for membership/count/order; Python combines
   consecutive day summaries into display banners only. Request windows across those
   component groups in order to implement a banner-local page, rather than reading all
   older runs or evaluating another query in Python.
5. Today/Yesterday default open; older banners default folded. When the complete query
   matches at most 50 runs, open all banners. Preserve user fold changes for the current
   view. Missing/rebuilding counts never claim a complete empty result.
6. Each open banner loads 100 visible rows and has a selectable `⋮ N more · l` tail.
   Enter/`l` requests its next page. Respect the corpus's expanded-container row and
   pagination contract, distinguishing root counts from visible member rows. `l`/`h`
   expands/collapses containers using only matching members. Workflow children remain
   absent from the list; exact lookup identifies their owner.
7. Add `j/k/g/G/h/l`, immediate highlight updates, scroll preservation, and programmatic
   highlight echo guards at `watch_highlighted`. Banners/tails are nav items but never
   agents. Never steal `Ctrl+J`/`Ctrl+K` from deck cards.
8. Test banner boundaries, time zones, folds, filtered container counts, page boundaries
   (including expanded members), click/key equivalence, widths, selection survival, and
   absence of synthetic rows in Inbox signals.

### query-parking

1. Wire `toggle_agents_archive: "a"` in `leader_mode`, the registry/metadata and leader
   dispatcher, plus palette entries **Agents: Open Archive** and **Agents: Back to
   Inbox**. `,a` from another main tab opens Agents Archive. An ordinary main-tab switch
   keeps `Agents ◷` when Archive is parked there.
2. Complete atomic per-view capture and restore of all state listed above, including
   per-panel deck/card and focused row/section. Reconcile a vanished Inbox identity
   through existing selection rules. Close/cancel a preview before changing view; late
   preview or persistence tasks cannot install the wrong query.
3. The Archive's committed query always contains `in:archive`. Use `extract_scope`,
   field validation, and the supplied canonicalizer. On successful commit only, Archive
   scope switches to Archive; absence or `in:inbox` switches to Inbox and commits that
   Inbox query. Invalid or uncommitted editing text never changes the active view.
   Entering Archive parks the prior Inbox query before replacing any active query
   variable.
4. Pick the FilterBar's profile from the editor's `in:` value on each completion
   request, not the currently visible view. Preview uses the matching worker and corpus,
   and ESC restores the committed view and query. Render the amber scope chip
   consistently. Keep saved slots and `^` in `agents-live`; use the selected profile's
   digest and canonical text. A pure toggle records no transition.
5. Persist only the Inbox query in `ace_agents_last_query.json`. Archive queries remain
   in history/slots and in-process parked state. Quit from Archive must not write its
   query over the Inbox's durable snapshot; startup still restores Inbox. Persist
   Archive grouping in its own settings, independently of Inbox grouping. Use `o` for
   day/project/outcome/model, with a default of day.
6. With `agents_unified_query` off, keep the legacy Inbox editor and its namespace
   rules. Archive always uses FilterBar and remains reachable through `,a` and explicit
   entry seams. Test the beta/unified flags as a two-by-two matrix; do not reinterpret
   legacy sigils as Archive predicates or silently carry them.
7. Build the Archive info row in `AgentInfoPanel`: amber Archive chip, this-machine
   total or matches, scope/filter chips, grouping hint, completeness/progress, and
   right-aligned `,a inbox`. Hide `#agents-header` and inactive Inbox load chrome. Add a
   marker API on `TabBar` that updates rendered ranges and cached cell width as well as
   the label; marker remains visible while another main tab is active.
8. Test query/selection/fold/tab/deck/scroll round trips, explicit carries versus pure
   toggle, history and saved slots from either view, persistence after quit, invalid
   scope hints, editor cancellation, and tab-marker hit ranges.

### archive-reader

1. On selection, immediately replace the identity header and clear earlier agent text
   into a titled loading placeholder. Use the existing 150 ms `DetailPanelDebouncer` to
   schedule a pump-free `load_bundle_file` then `Agent.from_bundle_dict` worker. Recheck
   view, selection key, query/corpus and detail generation after every await and before
   accepting widget output.
2. Reuse real deck renderers with explicit read-only context, including their generation
   checks. The archived header says `◷ ARCHIVED · read-only`; its ribbon uses core time
   basis, ended/started wording, runtime, real dismissed time when known, and
   retained-content facts probed off-thread. Do not cut prompts at 4,000 characters.
   Show partial replies as partial.
3. Ensure archived RUNNING/WAITING status cannot start live reply following, spinners,
   runtime timers, workspace diff refresh, named-proc synchronization, or live tool
   polling. Guard static Files dispatch ahead of `_ACTIVE_STATUSES` and the workspace
   fallback; use retained diffs and attached files only.
4. Show titled retained/missing cards for Main Context/Reply, Files, Tools, and FINAL
   instead of dropping whole decks into blank panels. Unreadable bundle:
   `Archive entry unreadable — Record only`, with light-row facts available to the
   Record phase. Do not create a parallel Record deck. If that parent phase is not yet
   merged, retain a usable titled metadata/error placeholder and the provenance seam;
   the child's land agent reconciles the final integration.
5. Container selection uses the existing Summary presentation over supplied light
   members, without hydrating every bundle or loading current live clan members. The
   shared Jump panel renders archived relations read-only; navigation to its targets
   belongs to the parent's routing phase.
6. Test a full prompt, retained reply/diffs/tools/finalizer data, partial reply, missing
   files, unreadable JSON, containers, stale hydration after switch or rapid j/k, and
   hidden Inbox finalize/enrichment events. Spy on revive, marker writes, unread
   clearing, live pollers, and workspace calls, and compare bundle, index, and
   artifacts-dir fingerprints before/after selection and deck reads.

### action-boundary

1. Audit all callers of `_get_selected_agent`, `_get_current_agent` equivalents, current
   indices, `_agents_in_focused_panel`, marks, and detail current-agent shortcuts.
   Classify callers explicitly as read, live mutation, view navigation, or Inbox
   accounting; migrate only read/navigation callers that need Archive. Record the audit
   and guard coverage in the phase note.
2. Deck selection, cycling, scrolling, prompt/chat read access, and applicable copy
   readers use the typed Archive selection. Provide the hydrated read target to the
   parent's fork/copy phase without implementing its new chooser or targets. Any editor
   read of Archive chat uses the existing read-only editor mechanism.
3. Guard live-only `x X s w W A R n N`, agent-tab and tribe moves, and `[`/`]` before
   availability checks silently swallow keys. Emit one explanatory toast, for example
   `Agent tabs live in the Inbox · ,a`, or
   `x acts on inbox agents — <name> is archived · ⏎ restore`. Guard direct
   palette/action invocation as well as key dispatch. Nothing may mutate the parked
   Inbox selection when Archive is active.
4. Implement independent Archive `m`/`u` marks by stable identities, with container
   membership access for the parent's bulk restore builder. Inbox marks remain parked
   and untouched. Banners/tails/pulse cannot be marked as runs.
5. Explicitly prove rows never affect unread, attention, `load:`, runner slots,
   prospective clans, auto-dismiss, bare `%wait`, live bulk actions, or agent-tab
   totals. Inbox accounting continues while Archive owns the screen; pulse counts
   reflect live refresh without replacing archived detail or resetting its debounce.
6. Add a view-specific conditional footer and the Agents help Archive box. Follow
   `src/sase/ace/AGENTS.md`: only conditionally available actions belong in the footer,
   global toggle/filter/navigation belong in help, keys sort alphabetically with symbols
   first, and help boxes remain 57 columns wide. Add metadata/default configuration for
   changed keys. Stage the conditional Enter restore hint with the parent chooser's
   actual capability; do not expose a nonfunctional restore.
7. Test each live-only key family and direct action, marks and bulk isolation,
   readable-deck key parity, beta-off regressions, pulse freshness, and footer/help
   content and formatting. Preserve existing parent action extension points rather than
   installing an Archive no-op that would shadow later restore handlers.

### archive-proof

1. Complete integration tests covering the two flag states and both unified-query
   states, startup, parking, query changes, scope errors, saved slots/history, delayed
   loading, selection reveal/identity stability, readable decks, no-write reading,
   action isolation, and Inbox refresh while Archive detail is visible. Use real core
   corpus fixtures for the list/CLI identity-order comparison.
2. Add synthetic archive-scale input (11k presentation roots and 37k children), reuse
   the existing benchmark fixture where practical, and extend `bench_tui_jk.py` for
   Archive. Record p95 j/k under 16 ms and warm `,a` to first rows under 100 ms. Measure
   UI scheduling/paint, not just facade query time. Separate cold corpus/facet cost from
   the warm-opening measurement, and verify Inbox startup/idle tracing contains no
   premature Archive loads.
3. Add deterministic PNG fixtures for day grouping with open/folded banners, containers,
   interrupted rows and italic start-only time, the archived identity plus Reply,
   missing-content cards, empty/missing/rebuilding state, and the tab marker while
   parked. Include pulse and applicable footer/help coverage. Use realistic widths and
   controlled dates/data.
4. Run targeted `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`.
   Inspect the retained manifest and every creation/removal and representative plus
   members of each update group. Resolve `partial` skips before claiming those goldens
   current. Inspect a live `sase screenshot` PNG for the toggle/read/return flow when
   real layout confidence needs it; never use live bytes as goldens.
5. Record the completed contracts and evidence on the assigned proof bead for the child
   land agent. This phase closes only its own assigned bead.

## Verification and child landing

Every implementation phase reads `lint_and_test.md` before finishing, runs `just fix` as
needed and `sase tool run check`, and closes only its own assigned bead with specific
verification evidence. Do not run `just check-full`; it is not authorized by this plan.
Use `/sase_monitor` for genuinely long commands and wait until the monitor-start command
itself exits before ending the turn. Mutating golden runs require a successor
inspection, not prepared completion that bypasses the inspection.

Use `/sase_final` for normal turn completion and host-owned commits; no manual git
commits, branches, or PRs. Changes to another repository require its own final
declaration decision and appropriate verification there.

Temporary unused symbols may be tied only to the still-open phase that will consume
them, or to the child epic. Before closing **each** assigned child phase, run
`sase bead epic-symbols <assigned-id>`, resolve every entry or re-key its Justfile line
to a still-open consuming phase/epic, then close it. There were no `--epic-symbol`
entries for `sase-1jm.3` at planning time; the child land agent must check again before
closing it.

The **child epic land agent**, once all child phases are closed:

1. Reconcile parent-phase integrations that have landed concurrently, especially
   Record's deck refresh and archive-action/routing consumers of the selection API.
   Resolve merge conflicts and verify the Archive stays behind the beta flag.
2. Confirm the entire assigned Archive-view scope above, all recorded test/perf/ PNG
   evidence, and run `sase tool run check` on the final integrated tree. The parent
   epic's final land agent owns removal of the beta flag and the wider pane-retirement
   parity gate.
3. Record discovered unrelated work only as
   `sase bead note sase-1jm.3 'PROPOSED FOLLOW-UP: <summary — detail>'`. Do not manually
   create task beads. A check failure reproduced identically on the clean base is
   follow-up evidence, not a reason to leave this phase open; cite any already-existing
   task that tracks it.
4. Run `sase bead epic-symbols sase-1jm.3`; resolve all entries or re-key them to the
   still-open parent epic or the actual later consumer. Resolve the child epic's entries
   as well before its own completion.
5. Close **only the assigned enclosing phase** using
   `sase bead close sase-1jm.3 --note "<implemented behavior and verified checks, timings, and inspected PNG evidence>"`.
   Follow the child epic land protocol for its own epic bookkeeping, and never close
   `sase-1jm` or any other ancestor. A child phase description suggesting parent closure
   supplies evidence to this land agent; it grants no child phase worker permission to
   close an ancestor.

This child epic is complete when the beta-on Archive is usable and read-only, per-view
state round-trips, beta-off Inbox behavior remains intact, no archive work appears
before the first complete Inbox load, and `sase-1jm.3` has the verification note and
closure that unblock the parent's dependent phases.
