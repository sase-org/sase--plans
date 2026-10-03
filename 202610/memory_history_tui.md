---
tier: epic
title: 'Memory history in the TUI: a time-aware Memory pane'
goal: 'The ACE Memory pane knows about time. Every memory note, web, strand, and agent
  instruction file can be stepped through, diffed, and reviewed in place, in the pager''s
  exact visual language. Cross-file memory changes can be reviewed without leaving
  ACE, agents show which memory version they actually read, and every hand-off to
  the pager lands on the exact version that was on screen. No key blocks, nothing
  fails silently, and there is no second history engine.

  '
phases:
- id: front-door
  title: Repair the H and C front door
  depends_on: []
  size: small
  description: 'front-door: make H and C actually open the pager from the Admin Center-hosted
    Memory pane by removing the call_from_thread misuse inside their async workers.
    Add failure toasts and stale-open guards, an AST guard test against call_from_thread
    inside async def under src/sase/ace/tui, and headless key-press tests that fail
    on the old code.'
- id: history-service
  title: App-scoped history service and the public history kit
  depends_on:
  - front-door
  size: medium
  description: 'history-service: add one app-scoped AceMemoryHistory over a process-wide
    shared HistoryService, which the pager provider factory also uses. It provides
    a stale-while-revalidate timeline memo, content-addressed body and comparison
    LRUs, single-flight queries, stat-only change tokens, explicit invalidation, and
    a quiet-time warm-up. Also publish sase.pager.history_kit as the only door ACE
    uses for history presentation, and migrate the History row onto the service with
    an honest unavailable state.'
- id: time-strip
  title: Pinned card head with the two-row time strip
  depends_on:
  - history-service
  size: medium
  description: 'time-strip: restructure the Memory card into a pinned head (title,
    path line with the pager pill, a two-row time strip) above a scrolling body. Render
    every honest state at now from the kit for notes, web descriptors, and strands.
    Replace the History property row and its goldens.'
- id: card-stepping
  title: Step through versions on the card
  depends_on:
  - time-strip
  size: medium
  description: 'card-stepping: add ( ) { } stepping on the card using the pager''s
    moment model, with prefetched bodies and atomic pill/body swaps. Includes the
    past read view with past frontmatter, the violet past frame, H at the exact pin,
    mutation and link guards, unaudited strand pasts, arrival rules, the first Esc
    ladder rung, footer destination verbs, and the help Time group.'
- id: card-diff
  title: Word-diff view on the card
  depends_on:
  - card-stepping
  size: medium
  description: 'card-diff: = toggles a sticky read/diff view rendered with build_diff_body.
    It covers past versions, latest change at clean now, pending edits at dirty now,
    creations, and tombstones. Includes cached committed comparisons, prefetch, H
    carrying the view, and the publish-loop test.'
- id: timeline-lens
  title: Lens framework and the Timeline lens
  depends_on:
  - card-diff
  size: medium
  description: 'timeline-lens: add the Notes/Timeline/Changes lens framework, which
    owns the Notes snapshot and restore, per-lens header, footer, filter, and key
    routing, and the full Esc ladder. @ turns the rail into the subject''s timeline:
    kit picker rows, debounced preview, b compare base, hidden toggle, and pager hand-off
    at the cursor pin.'
- id: changes-lens
  title: Changes lens over a shared feed view-model
  depends_on:
  - timeline-lens
  size: medium
  description: 'changes-lens: extract a pure feed_model shared with the pager feed
    document. C turns the rail into day-grouped changesets for the current scope or
    All scopes, with a bounded window, provenance chips, progressive per-subject diff
    sections, and pager hand-off in diff view.'
- id: rail-glance
  title: Rail recency glance and deleted subjects
  depends_on:
  - changes-lens
  size: medium
  description: 'rail-glance: add a right-aligned newest-change glyph and age on every
    Notes rail row from one subjects() plus feed() per scope. A D toggle lists tombstoned
    subjects in a DELETED group, each with a read-only tombstone card. Introduces
    the history-only rail node kind.'
- id: instructions-group
  title: Instructions group and instruction cards
  depends_on:
  - rail-glance
  size: medium
  description: 'instructions-group: add a collapsed INSTRUCTIONS rail group for AGENTS.md
    subjects with shim alias and diverged chips and the TEMPLATE chip for home. Their
    read-only cards show cause rows and the rendered body at now, and stepping, diff,
    the Timeline lens, and H all work.'
- id: agents-bridge
  title: Memory as seen by the agent in the Agents tab
  depends_on:
  - history-service
  size: medium
  description: 'agents-bridge: add a sase-core blob:OID version selector. The Agents-tab
    MEMORY lane gets version chips per read (aggregate for batch reads) and an AGENTS.md
    as launched row resolved from launch evidence, including a not-in-git snapshot
    pseudo-version. Hints open the pager pinned to the version read.'
- id: watermark-core
  title: Core review watermark and the CLI feed header
  depends_on:
  - agents-bridge
  size: medium
  description: 'watermark-core: add a sase-core per-scope review watermark that is
    shared across workspace clones and only marked explicitly, plus its bindings and
    N-new semantics. sase memory history feed output shows N new since you last reviewed,
    and -m/--mark-reviewed advances it.'
- id: watermark-tui
  title: Review watermark in the Changes lens
  depends_on:
  - instructions-group
  - watermark-core
  size: small
  description: 'watermark-tui: show the Changes lens header N-new chip and unreviewed
    row dots, add m to mark reviewed (optimistic, persisted off-thread), and add the
    quiet-time MEMORY sub-tab badge.'
- id: launch
  title: Document, measure, and review end to end
  depends_on:
  - watermark-tui
  size: small
  description: 'launch: finish the TUI history docs and run the full visual golden
    review. Measure and record every performance budget, do a live end-to-end walkthrough,
    and record the absorbed-bead bookkeeping and follow-ups for the land agent.'
proposed_by: bbugyi200.athena.0vj
create_time: 2026-10-02 14:43:00
status: done
bead_id: sase-1ev
---

- **PROMPT:** [prompts/202610/memory_history_tui.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/memory_history_tui.md)
- **BEAD:** [sase-1ev](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ev/README.md)

# Plan: Memory history in the TUI, a time-aware Memory pane

## 1. Summary

Epic `sase-1dr` gave SASE memory a time axis. It added a Rust index in sase-core, a
thread-safe `HistoryService`, `sase memory history`, and a modeless pager time axis:
pill, time band, scrubber, word diff, `@` picker, changes feed, and honest states. The
TUI got only a front door, and that front door is broken: `H` and `C` fail silently on
every press (§2.1). Today the pager is the only working history surface.

This epic makes the **Memory pane time-aware without building a second pager**:

- **The rail chooses _what_; the card shows it _at a moment in time_.** The card gets a
  pinned head with the pager's own pill and time strip. `( ) { } =` step through and
  diff versions in place, and a violet frame says "you are in the past".
- **Two rail lenses.** `@` turns the rail into the selected subject's **Timeline**, and
  `C` turns it into a **Changes** review of changesets across subjects. `Esc` peels back
  one layer at a time.
- **A complete catalog.** A recency glance on every rail row, an **Instructions** group
  for `AGENTS.md` and its shims, and an opt-in list of **deleted** subjects.
- **A bridge from the Agents tab.** Each memory read shows whether the version the agent
  saw is still current, and `AGENTS.md as launched` opens the exact launch version.
- **An explicit review watermark** in sase-core: `● N new since you last reviewed`, in
  the TUI and the CLI.
- **The pager stays the deep reader.** Every hand-off (`H`, `⏎`) carries the exact
  version pin, view, and compare base, so you never land on "now" after looking at the
  past.

Everything is built on the pager's pure renderers, published as one public kit
(`sase.pager.history_kit`). It is backed by one app-scoped history service with tip- and
content-keyed caches, so no keystroke ever waits on git.

The design follows the consolidated research report
`research:202610/memory_history_tui_support/memory_history_tui_support.md`; the user
agreed with all of its recommendations. Section 3 records every place this plan refines
or departs from it.

## 2. Context (verified on master)

### 2.1 `H` and `C` never open anything

`action_open_history` and `action_open_changes` in
`src/sase/ace/tui/modals/memory_pane_history.py` pass a coroutine to `run_worker(...)`
without `thread=True`, so it runs on the app's event loop. After
`await asyncio.to_thread(...)`, the coroutine calls `self.app.call_from_thread(...)`.
Textual raises `RuntimeError` when that method runs on the app thread. The worker uses
`exit_on_error=False`, so the error is swallowed. The failure toast goes through the
same call, so it never appears either. These are the only four
`call_from_thread`-inside-`async def` sites under `src/sase`. No test presses `H` or
`C`: `tests/ace/tui/modals/test_memory_panel_history.py` covers bindings, formatting,
and the footer only.

### 2.2 Other facts the design relies on

- **One production host.** The Memory pane (`MemoryPane`,
  `src/sase/ace/tui/modals/memory_pane.py`) is hosted in production only by the Admin
  Center Config hub (`config_hub_catalog._memory_factory`, inside `ConfigCenterModal`).
  `gm` / `Ctrl+G m` open it. The standalone `MemoryPanel` modal is used only by tests.
- **Card layout today.** `#memory-panel-detail` is a bordered `VerticalScroll` holding
  the title (`build_rail_node_card_title`: kind badge, name, scope, then the path line),
  description, a body `Markdown`, and a meta `Static`. The meta holds the type banner
  and the property grid with the History row
  (`build_rail_node_card_meta(..., history=...)`).
- **History row today.** It is loaded per selection by `fetch_history_summary`, which
  calls `service.sync()` and then `service.timeline()`. Every core query already runs
  the freshness sync internally (`query_timeline` calls `sync_scope`), so the panel pays
  that sync twice.
  - A successful summary is never revalidated.
  - A failed load shows a dim `…` forever.
  - Web descriptors get no row.
  - ACE imports `render_sparkline` from the private `sase.pager._time_band`. Its
    eight-cell sparkline renders as a single blob.
- **The pager's pure kit already exists**, spread over private modules, and is
  Textual-free:
  - `sase.pager._chrome_history`: `history_badge`, `pill_forms`, `history_context`,
    `honest_chip`, `time_verbs_for_moment`.
  - The time band: `build_time_band_data`, `render_time_band`, `meaning_row`,
    `cause_row`, `render_scrubber`, `render_sparkline`, `format_age`,
    `chrome_row_budget`.
  - `sase.pager.history.moment`: `build_moment`, `moment_for_state`, `step_target`,
    `boundary_notice`, `canonical_ordinal`.
  - `sase.pager.history.models`: `VersionPin`, `SectionTimeState`, pin helpers.
  - `sase.pager.history.diff`: `build_diff_body`, `diff_endpoints`.
  - `sase.pager.history.styles`: `history_styles_for_theme`.
  - `sase.memory.history.timeline_picker`: `build_picker_rows` and its filters.
  - `sase.memory.history.pager_provider`: `visible_ordinals_for_timeline` and its
    sibling helpers.
  - The picker's column fitting (`_picker_columns`, `_format_picker_row`) is pure but
    lives in the Textual module `sase.pager._timeline_picker`.
- **Pager hand-off.**
  `build_history_document(scope, subject, initial_revision, view, compare_base, service, title)`
  already opens any version in either view. It cannot yet mark an explicit compare base
  (`VersionPin.explicit_base`), which a `now` target needs before it can compare against
  a committed base.
- **Service cost.** `memory_history_provider_factory()` builds a new `HistoryService`
  per provider. `HistoryService` memoizes scopes only.
- **Measurements** (warm, in-process, on the sase repo: 139 subjects, 489 feed
  changesets):

  | Call                    | Time        |
  | ----------------------- | ----------- |
  | `sync`                  | 33–62 ms    |
  | `subjects`              | 31–36 ms    |
  | `timeline`              | 57–60 ms    |
  | `version(include_body)` | 39–41 ms    |
  | `feed(limit=None)`      | 42–44 ms    |
  | First sync in a session | up to 3.8 s |

  A key that waits on the service would miss the 16 ms ACE j/k budget and the 30 ms step
  budget.

- **Geometry** (Admin Center at 120×40): the rail is 32–52 columns, the card is about
  68–87 columns, and more than 20 body rows are visible. At 80 columns the card is about
  46 columns wide.
- **Agents tab.**
  - `MemoryReadEvent` (`src/sase/memory/_read_log_models.py`) carries `blob_oid` and
    `included_blob_oids`.
  - Launch evidence (`src/sase/axe/launch_evidence.py`) writes `workspace_head` and
    `instruction_snapshot` (path, repo, `blob_oid`, tracked) into
    `<artifacts_dir>/agent_meta.json`. Bytes that are not in git go to
    `~/.sase/instruction_snapshots/<oid>`.
  - Nothing in `src/sase/ace` reads any of it. The MEMORY lane renders in
    `src/sase/ace/tui/widgets/prompt_panel/_agent_memory_reads.py`, loaded off-thread by
    `_agent_display_header_summary.py` (lane batch 2).
- **Core version selectors** (`select_committed` in sase-core
  `crates/sase_core/src/memory_history/query.rs`) accept `~N`, an ordinal, and a commit
  SHA prefix. There is no blob selector.
- **Scope keys** are `project:<name>` and `home`. They are clone-independent, which is
  what a cross-clone review watermark needs.
- **Free keys in `ace.keymaps.memory`:** `( ) { } = @ b m D`.
  - `.` is the fixed `.1`–`.9` chip prefix (`NUMBERED_LINK_BINDING`).
  - `[`/`]` cycle Config hub sub-tabs.
  - The pager's own action names are `history_older`, `history_newer`, `history_first`,
    `history_now`, `history_toggle_diff`, and `history_timeline`.

## 3. Design decisions

| #   | Decision                                                                                                                                                                                                                                                                                                                                                                                                                                    | Why                                                                                                                                                                                                                                                |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | **The Memory pane becomes time-aware, and the pager stays the deep reader.** No `PagerView` embedding: it is deferred, not rejected, until the pager-body rewrite (`sase-1es`) and the PaneGrid work (`sase-1eu`) land. Revisit it only if the card plus `H` proves too shallow in use.                                                                                                                                                     | The pure kit already gives the card the pager's pixels. Embedding a widget whose body and host are mid-rewrite is a merge trap with no ACE precedent.                                                                                              |
| D2  | **Same verbs, same pixels.** `( ) { } = @` mean exactly what they mean in the pager. Every history visual (pill, band, meaning and cause rows, scrubber, diff body, picker rows, glyphs, palette) comes from the public `sase.pager.history_kit`. ACE never formats `vK`, pills, or glyphs itself, and a guard test enforces the import door.                                                                                               | Nothing is re-implemented, so the pager and the pane cannot drift.                                                                                                                                                                                 |
| D3  | **Three lenses on one pane: Notes (default), Timeline (`@`), and Changes (`C`).** Lenses are entered from Notes only, and the other lens key is inert inside a lens. **`Esc` peels the innermost layer**: filter, then lens, then past pin, then the host's close. Leaving a lens restores the Notes snapshot (cursor, filter, expansion, scroll, focus, trail). Leaving the Timeline lens keeps the card on the version its cursor showed. | A lens reuses the pane's list/detail grammar, scope ring, filter, footer, and help, with no new screen or keymap scope. _Refines the research:_ the card never changes content as a lens closes, and each `Esc` visibly removes exactly one layer. |
| D4  | **The time strip lives in a pinned card head**, outside the scrolling body. It reserves its rows in every state and replaces the History property row. The pill sits on the path line, as it sits on the pager's title row.                                                                                                                                                                                                                 | Loading, failure, and stepping never move the body. The strip never scrolls away, and the card head mirrors the pager's head row for row.                                                                                                          |
| D5  | **The past is read-only and never audited.** `o` always edits _now_, and mutations refuse while the card is pinned. Relation chips are computed from now, so following one from a past card refuses and points to `H`, which resolves links at the past revision. Viewing a past strand writes no audited read (`sase-1dr` D10).                                                                                                            | You must always know when you are in the past, and the TUI must never silently open today's target from yesterday's text.                                                                                                                          |
| D6  | **One app-scoped `AceMemoryHistory` over one process-wide `HistoryService`**, which the pager provider factory also uses. Queries never call `sync()` first. Bodies and committed comparisons are cached by blob OID, so those caches can never go stale. Timelines use stale-while-revalidate plus explicit invalidation plus stat-only change tokens.                                                                                     | This removes per-selection double syncs and per-lookup service rebuilds without adding any IO to keystroke paths.                                                                                                                                  |
| D7  | **Arrival rules.** Selecting another subject, switching scope, or opening the pane always shows now. Pins never carry across subjects. The read/diff choice (`=`) is sticky for the pane session, and opening the pane resets it to read.                                                                                                                                                                                                   | This matches the pager (§4.5 of `plan:202609/memory_history.md`). A sticky diff view lets `j`/`k` sweep "what changed lately" across notes.                                                                                                        |
| D8  | **Changes defaults to the current scope.** `All scopes` is an entry on the existing `p`/`P`/`Ctrl+P` ring, in this lens only. Fetch the whole feed once (about 43 ms), render a window of 100 rows, and extend it on demand. Never fake a continuation cursor.                                                                                                                                                                              | The header names one scope. Render cost, not query cost, is the bound.                                                                                                                                                                             |
| D9  | **Core owns the new semantics**: the `blob:<oid>` version selector and the review watermark (per scope key, shared across clones, explicit marking only, stored as state, never in the disposable cache directory).                                                                                                                                                                                                                         | A CLI, web, or editor frontend would need identical answers (the `rust_core_backend_boundary` litmus test).                                                                                                                                        |
| D10 | **Batch memory reads get one aggregate chip** (`≡ now` or `⟲ K of N changed`). Per-file versions appear in the memory read report that the row's hint already opens.                                                                                                                                                                                                                                                                        | _Refines the research_ ("one chip per included blob"): per-file chips would explode a five-row lane. The report has room for detail.                                                                                                               |
| D11 | **Lenses keep two regions at every width.** At narrow widths the lens rail pins to its minimum width, and the card sheds and folds.                                                                                                                                                                                                                                                                                                         | _Refines the research_ ("one region at a time below about 50 columns"). A 46-column preview is still readable, and the deep read is one key away (`⏎`). A mode switch adds a concept for little gain.                                              |
| D12 | **No feature flag.** Every phase ships a complete increment, and no landed phase exposes half of a lens. If a worker must split a phase so that part of an unfinished lens would reach users, they create a `beta` flag with `sase flag new` (per `sase_flags`), and `launch` removes it.                                                                                                                                                   | Per the flags convention, a flag exists only to hide unfinished user-reaching behavior.                                                                                                                                                            |
| D13 | **Goldens are captured in the Admin Center host**, the only production host, in dark and light themes.                                                                                                                                                                                                                                                                                                                                      | The standalone-modal goldens never show what users see.                                                                                                                                                                                            |

## 4. UX specification

The mockups are illustrative: names, dates, and counts are placeholders. A heavy frame
(`┏━┓`) stands for the violet past accent, `[-…-]` for struck red deletions, and `{+…+}`
for bold green insertions.

### 4.1 Principles

1. **Time is an axis of the card, not a mode.** If the selected subject has history, the
   time verbs appear in the availability-driven footer. If it does not, they do not.
2. **You always know when you are in the past.** The pill, the violet frame, and the
   footer's `o edit now` all say so, and amber means only uncommitted or unpublished.
3. **You keep your place.** Small motions push no trail entries. Leaving a lens restores
   Notes exactly. Closing the pager returns to the same lens and row.
4. **Meaning before mechanics.** Show
   `⇧ promoted reference → core · § Default Keymap Config · +31w −4w`, not a path and a
   SHA.
5. **Never block, never lie, never fail silently.** No service call runs on a keystroke,
   and no `…` placeholder lasts forever. Every honest state is shown, and a pressed key
   that fails always toasts.

### 4.2 The Notes lens card

The card head is pinned. The title row and the path line are today's title; the pill
moves onto the path line. The two strip rows follow, then the scrolling body.

At now (clean):

```text
 MEMORY · sase · 29 notes · scope 1/3
╭───────────────────────────────╮╭──────────────────────────────────────────────────────────────────╮
│ ● sase                  ⟳ 1h  ││ M MEMORY  gotchas                                           sase │
│ ● gotchas               ⇧ 8d  ││ sase/memory/gotchas.md   ● NOW · v25   8d                        │
│ ▸ glossary              ◆ 2d  ││ v1 ▁▂▁▃▅▁▇▁▂▃▁▅▇█ ● now   last changed Sep 24 · sase-1au.5       │
│ ○ cli_rules             ◆ 1mo ││ ⇧ promoted reference → core · § Default Keymap Config · +31w     │
│ ○ dispatch              ✚ 3mo ││ ──────────────────────────────────────────────────────────────── │
│ ○ lint_and_test         ◆ 1w  ││ Code conventions and gotchas.                                    │
│ ○ tui                   ◆ 3d  ││                                                                  │
│ ▸ INSTRUCTIONS · 4            ││ Default Keymap Config                                            │
│                               ││ When changing keymaps, leader mode keys, or any configuration…   │
╰───────────────────────────────╯╰──────────────────────────────────────────────────────────────────╯
 ( v24 · = diff · @ timeline · H pager · C changes · e edit · o source · ? keys
```

After pressing `(` once:

```text
 MEMORY · sase · 29 notes · scope 1/3
╭───────────────────────────────╮┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
│ ● sase                  ⟳ 1h  │┃ M MEMORY  gotchas                                           sase ┃
│ ● gotchas               ⇧ 8d  │┃ sase/memory/gotchas.md   ⟲ PAST · v24 of 25   10d                ┃
│ ▸ glossary              ◆ 2d  │┃ v1 ▁▂▁▃▅▁▇▁▂▃▁▅▇[█]▂ ● now   Tue Sep 22 · tend keymaps  1 newer  ┃
│ ○ cli_rules             ◆ 1mo │┃ ⇧ promoted reference → core · § Default Keymap Config · +31w −4w ┃
│ ○ dispatch              ✚ 3mo │┃ ──────────────────────────────────────────────────────────────── ┃
│ ○ lint_and_test         ◆ 1w  │┃ Code conventions and gotchas.                                    ┃
│ ○ tui                   ◆ 3d  │┃                                                                  ┃
│ ▸ INSTRUCTIONS · 4            │┃ Default Keymap Config                                            ┃
│                               │┃ When changing keymaps, leader mode keys, or any configuration…   ┃
╰───────────────────────────────╯┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
 ( v23 · ) v25 · } now · = diff · @ timeline · H pager · o edit now · esc now
```

The same past version after `=`:

```text
 MEMORY · sase · 29 notes · scope 1/3
╭───────────────────────────────╮┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
│ ● sase                  ⟳ 1h  │┃ M MEMORY  gotchas                                           sase ┃
│ ● gotchas               ⇧ 8d  │┃ sase/memory/gotchas.md   ⟲ PAST · v24 of 25   10d                ┃
│ ▸ glossary              ◆ 2d  │┃ v1 ▁▂▁▃▅▁▇▁▂▃▁▅▇[█]▂ ● now   Δ v23 → v24 · diff        1 newer   ┃
│ ○ cli_rules             ◆ 1mo │┃ ⇧ promoted reference → core · § Default Keymap Config · +31w −4w ┃
│ ○ dispatch              ✚ 3mo │┃ ──────────────────────────────────────────────────────────────── ┃
│ ○ lint_and_test         ◆ 1w  │┃ ⇧ type: reference → core — now loaded by every agent             ┃
│ ○ tui                   ◆ 3d  │┃  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄ 14 unchanged lines ┄┄┄┄┄┄┄┄┄┄┄┄┄┄                ┃
│ ▸ INSTRUCTIONS · 4            │┃ Update [-the default config-]{+src/sase/default_config.yml+} when┃
│                               │┃ changing keymaps, leader keys, or {+any configuration values+}.  ┃
╰───────────────────────────────╯┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
 ( v23 · ) v25 · } now · = read · @ timeline · H pager · o edit now · esc now
```

**Card head anatomy**

- **Path line**:
  - the path at the shown version (a rename shows the old path)
  - the pill from `history_badge`, using the longest form that fits
  - the pager's context text from `history_context` (the age, or `on top of v25`)
  - for strands viewed in the past, a dim `· not audited` chip
- **Strip row 1** is the band's first row from `render_time_band`:
  - the life strip at now in the read view
  - the timeline row in the past or in diff view: scrubber, playhead, date, commit
    subject, and `N newer`
  - the tombstone row for a deletion
- **Strip row 2**:
  - in the past: the `meaning_row`
  - for instruction subjects: the `cause_row`
  - at now: the newest version's meaning row, dimmed, which answers "what changed last?"
  - otherwise blank but reserved
- **Folding.** When the card's height is under 14 rows (`chrome_row_budget`), the strip
  folds to one row: the meaning row in the past, the life strip at now. The pill on the
  path line still carries the version.
- **Width shedding.**
  - The path line drops the age first, then ellipsizes the path from the left. The pill
    switches to shorter forms and is never cropped.
  - The band rows shed exactly as the pager's band does.

**Strip states**, all in the pager's words:

| State                       | Path line pill                                | Strip                                                                                                      |
| --------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Index building, no memo yet | none                                          | dim `indexing…`; row 2 reserved. Steps pressed meanwhile queue, and the last one wins.                     |
| Clean now                   | `● NOW · v25` + context                       | Life strip, then the newest meaning row (dim).                                                             |
| Uncommitted                 | amber `◌ NOW · uncommitted` + `on top of v25` | Life strip with `◌ edits not durable until committed`. `=` shows the pending edit against HEAD.            |
| Untracked or ignored        | none                                          | Amber `UNTRACKED · commit this file to start its history`.                                                 |
| Home without chezmoi        | none                                          | Dim `NO VCS · home memory is not in git`.                                                                  |
| Home with chezmoi           | pill + `TEMPLATE` chip                        | History follows the template source.                                                                       |
| Shallow clone               | pill                                          | `SHALLOW · history truncated at <date>`.                                                                   |
| Past                        | `⟲ PAST · v24 of 25`                          | Timeline row, then meaning row. Violet frame.                                                              |
| Deleted subject (tombstone) | `✖ DELETED · v12`                             | Tombstone row (`deleted <date> by <agent> · last content shown`). Deleted-style frame.                     |
| Failure                     | last good pill, if any                        | `history unavailable · <reason> · r retry`. The last good snapshot stays visible, with a dim `stale` mark. |

**The publish loop** is the moment the design is built around:

1. `e` edits a note, and the strip turns amber `◌ NOW · uncommitted`.
2. `=` shows the pending word diff.
3. `I` publishes.
4. On the next refresh, the strip settles to `● NOW · v26` and a new bar grows at the
   scrubber's right edge.

### 4.3 Stepping, views, and guards

- **Steps.**
  - `(` goes to the older version and `)` to the newer one. Hidden versions are skipped,
    and `)` from the newest goes to now.
  - `{` goes to the first version and `}` to now. For a deleted subject, `}` goes to its
    tombstone.
  - A boundary press toasts `boundary_notice` (for example
    `Already at v1, the oldest version.`).
- **Read view in the past.**
  - The historical body renders through the same `Markdown` widget as now, with
    frontmatter stripped by the parser the pane already uses.
  - The description and the type banner come from the past frontmatter, so a note that
    was `reference` then says so.
  - Relation chips (parent and children) are computed from today's graph. They render
    dimmed under a `links as of now` caption.
- **Diff view (`=`).**
  - A `Static` shows `build_diff_body(...)`: inline insertions, struck deletions, the
    frontmatter semantic block, and fixed folds labelled `H to expand`.
  - Endpoints come from the moment:

    | Card state       | Diff shown                     |
    | ---------------- | ------------------------------ |
    | Past             | Parent → pin                   |
    | Clean now (≡ vN) | v(N−1) → vN, the latest change |
    | Dirty now        | HEAD → worktree                |
    | First version    | Against empty                  |
    | Tombstone        | The deletion summary           |

- **`H` promotes the pin.** It opens `PagerScreen` at the card's exact subject, ordinal,
  view, and compare base.
- **Guards while pinned in the past:**
  - `a`, `e`, `d`, `I`, and strand-add refuse with one toast:
    `leave the past to edit · } now`.
  - `o` always opens _now_ in `$EDITOR`, and the footer says `o edit now`.
  - Following a link (`l`, `⏎`, `.N`) refuses with
    `links are as of now · H follows them at v24`.
  - `y` copies the body on screen and names its version in the toast.
  - `Y` and `Z` act on the current file, as today.
- **Strands.** Selecting a strand at now keeps today's audited read. Stepping into the
  past writes **no** `memory_reads.jsonl` event.
- **Footer.**
  - The time destinations come from `time_verbs_for_moment`
    (`( v23 · ) v25 · } now · = diff`), with the pager's `E edit now` mapped to
    `o edit now`.
  - While pinned, verbs that would refuse are hidden.
  - `?` gains a **Time** group with the pill legend and the glyph table.

### 4.4 Timeline lens (`@`)

```text
 MEMORY · sase › gotchas · timeline · 25 versions · 3 hidden
╭───────────────────────────────╮┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
│    now  ≡ v25  clean          │┃ M MEMORY  gotchas                                           sase ┃
│    v25  Sep 30  ◆ § Keys   +6w│┃ sase/memory/gotchas.md   ⟲ PAST · v24 of 25   10d                ┃
│ ▸  v24  Sep 22  ⇧ → core  +31w│┃ v1 ▁▂▁▃▅▁▇▁▂▃▁▅▇[█]▂ ● now   Compare v21 → v24 · b clear         ┃
│ ◇  v21  Sep 14  ◆ § Keys   +4w│┃ ⇧ promoted reference → core · § Default Keymap Config · +35w −4w ┃
│    v12  Aug 30  ◆ § Perf  +88w│┃ ──────────────────────────────────────────────────────────────── ┃
│     v1  Jul 02  ✚ created 210w│┃ ⇧ type: reference → core — now loaded by every agent             ┃
│  ·· 3 hidden · . show         │┃  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄ 14 unchanged lines ┄┄┄┄┄┄┄┄┄┄┄┄┄┄                ┃
│                               │┃ Update [-the default config-]{+src/sase/default_config.yml+} when┃
│                               │┃ changing keymaps, leader keys, or {+any configuration values+}.  ┃
╰───────────────────────────────╯┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
 j/k version · = read · b base · . hidden · / filter · ⏎ open in pager · esc notes
```

- **Rows** come from `build_picker_rows`, laid out at the rail's width by the kit's
  picker column fitter, which sheds the SHA, then attribution, then age. These are the
  pager picker's exact cells.
  - The `now` row and its `≡ vN` alias are always listed.
  - A hidden summary row counts reflows and moves.
  - Markers: `▸` is the cursor, `●` the version the card showed when the lens opened,
    and `◇` the compare base.
- **Entering.** The cursor starts on the card's current version. `@` refuses with the
  honest state (untracked, `NO VCS`) when there is no history.
- **Motion.** The highlight moves at once. The card follows through the existing 150 ms
  detail debouncer. While a preview loads, the row shows a dim `…`, and the card keeps
  naming the content actually on screen.
- **Compare base (`b`).**
  - `b` marks the cursor row as the base, and pressing `b` on that row clears it.
  - The card shows `Compare vA → vB · b clear`, always ordered older to newer, or
    `Same version`.
  - `now` and `STAGED` are distinct endpoints, mirroring the pager picker's endpoint
    normalization.
  - The base clears when the subject or scope changes, and when the lens closes.
- **Other keys.**
  - `.` reveals hidden versions; chip shortcuts are inert in this lens.
  - `/` filters rows.
  - `=` toggles read and diff.
  - `⏎`/`l`/`H` open the pager at the cursor's pin, view, and base.
- **Leaving.** `Esc`/`@`/`h` return to Notes with the snapshot restored. The card stays
  on the cursor's version (D3), so a second `Esc` returns it to now.
- **Mouse.** Clicking the time strip opens the lens.

### 4.5 Changes lens (`C`)

The header and footer below show `● 3 new` and `m mark reviewed`. Those parts arrive
with `watermark-tui`.

```text
 MEMORY · sase · changes · last 100 of 489 · 9 regen-only folded · ● 3 new
╭───────────────────────────────╮╭──────────────────────────────────────────────────────────────────╮
│ ━ Today ━━━━━━━━━━━━━━━━━━━━━ ││ feat(goals): complete G1 acceptance                              │
│ ▸ 11:03 ✚ goal-ledger +2  180w││ Fri Oct 2 2026 11:03 · 2h ago · sase                             │
│   10:41 ◆ dispatch   +44w −10w││ ◈ sase-1bu.7   ⬡ athena.sase-1bu.7   ◉ 1a2b3c4                   │
│ ━ Yesterday ━━━━━━━━━━━━━━━━━ ││ 3 subjects · +200w −3w · ⟳ 3 regenerated                         │
│   18:29 ◆ README       +4w −4w││ ──────────────────────────────────────────────────────────────── │
│   14:19 ⇧ gotchas   ref → core││ .1 ✚ decisions:goal-ledger · new decision · 180w                 │
│ ━ Mon Sep 28 ━━━━━━━━━━━━━━━━ ││    A goal is its own Rust-owned domain, not a bead type…         │
│   ⋯ 5 regenerated-only        ││ .2 ◆ glossary:goal · § Definition · +20w −3w                     │
│   ··· 389 older · j loads more││    A goal is a [-durable-]{+host-bound+} unit of intent that…    │
│                               ││ ⟳ AGENTS.md §3.1 · README.md · shims regenerated                 │
╰───────────────────────────────╯╰──────────────────────────────────────────────────────────────────╯
 j/k changeset · ⏎ open in pager · .N subject · p/P scope · / filter · m mark reviewed · esc notes
```

- **The list.**
  - Each changeset row shows its time, its class glyph, the first authored subject plus
    `+N more`, a right-aligned word delta, and `⌂` for home.
  - Day rules (`Today`, `Yesterday`, `Mon Sep 28`) are disabled options, so `j`/`k` skip
    them.
  - Changesets that only regenerate files fold into one count row.
  - The last row, `··· N older · j loads more`, extends the window by 100.
- **The card.** It keeps the same pinned head grammar:
  - **Title row:** the commit subject.
  - **Path line:** absolute time, relative time, and scope.
  - **Strip row 1:** provenance chips using the Artifacts tab's icons and accents (`◈`
    bead, `⬡` agent, `◉` commit).
  - **Strip row 2:** totals.
  - **Body:** one titled section per authored subject (`.N glyph subject · meaning`)
    with its `build_diff_body` content. Sections fill progressively, and each reserves
    its line while loading. At most six sections show inline, followed by
    `+N more · .N or ⏎ to open`. A final `⟳` line folds the generated consequences.
  - **Frame:** normal, because Changes is not a pinned document.
- **Keys.**
  - `⏎`/`l`/`H` open the first authored subject at that changeset's version in the
    pager's diff view, and `.1`–`.9` open subject N.
  - `p`/`P`/`Ctrl+P` cycle the ring plus `All scopes`.
  - `/` filters every fetched changeset by commit subject, subject names, bead, and
    agent.
  - `r` refetches.
  - `Esc`/`C`/`h` return to Notes.
- **Scopes.** A scope that fails or is `NO VCS` shows as a header chip, never silently
  dropped.

### 4.6 Rail glance, deleted subjects, and the Instructions group

- **Recency column.** Each Notes rail row ends, right-aligned on its first line, with
  the newest change's class glyph and compact age (`⇧ 8d`, `◆ 3h`, `⟳ 1h`).
  - It comes from one `subjects()` plus one
    `feed(scope, limit=None, include_hidden=True)` per scope load or refresh,
    off-thread.
  - Promotions `⇧`/`⇩` keep their highlight.
  - Shed the age first, then the glyph. Never wrap the stem.
  - Rows paint first and gain the column when the map lands.
- **Deleted subjects (`D`).** `D` toggles a trailing `DELETED` group of subjects whose
  _latest_ entry is a deletion, newest deletion first.
  - Rows read `✖ name   deleted 3w` in the deleted style, and the header gains
    `· N deleted`.
  - The card is the tombstone: the `✖ DELETED · vN` pill, a deleted-style frame, the
    tombstone strip row, and the last content.
  - It is read-only. `(`/`{` step older, and `@` and `H` work.
- **Instructions group.** A collapsed `▸ INSTRUCTIONS · N` group sits at the rail's
  bottom, built from `subjects()` entries of kind `instructions`. `space` toggles it.
  - Rows name the file by directory (`AGENTS.md`, `src/sase/ace/AGENTS.md`, …).
  - Identical shims alias into the row as `≡ 3 shims`. Diverged history shows
    `⚠ diverged`, and home rows carry `TEMPLATE`.
  - The card's strip uses cause rows (`⟳ rendered · sources: gotchas.md, tui.md`). The
    body is the rendered file at now, loaded off-thread.
  - The meta grid says whether SASE renders the file (managed) or it is hand-written,
    and lists the shims.
  - The cards are read-only. Managed files refuse edits with
    `rendered from memory · edit its source notes`; a hand-written subdirectory
    `AGENTS.md` opens with `o`.

### 4.7 Agents tab: memory as seen by the agent

```text
┌─ Context ──────────────────────────────────────────────────────────────┐
│ ▾ MEMORY · 4 reads · 3 files · AGENTS.md as launched                   │
│   launch    ◇ AGENTS.md                v258 ⟲ 2 newer since launch     │
│   10:02:11  ◇ gotchas                  v25 ≡ now                       │
│             ↳ need keymap conventions                                  │
│   10:04:40  ◇ dispatch                 v12 ⟲ 2 newer                   │
│             ↳ remote dispatch rules                                    │
│   10:06:02  ◇ glossary:stitch goal     ⟲ 1 of 2 changed                │
└────────────────────────────────────────────────────────────────────────┘
```

- **Chips.** Each read row with a `blob_oid` gets a chip:
  - dim `≡ now`
  - past-accent `vK ⟲ N newer`
  - amber `◌ uncommitted at read`, when the blob matches no committed version
  - one aggregate chip for a batch read (D10)
  - nothing for reads that predate blob capture
- **Launch row.** A new first row, `AGENTS.md as launched`, resolves the
  `instruction_snapshot` entry for the workspace's root `AGENTS.md` and falls back to
  `workspace_head`.
  - If the bytes match no committed blob, the row reads `◌ as launched · not in git`.
  - If the stored bytes are gone, the row reads a dim `snapshot unavailable`.
  - Agents launched before evidence capture show no launch row.
  - Never substitute "the nearest commit before the timestamp".
- **Hints (`v`).**
  - A row whose chip resolved to a committed past version opens the pager pinned to that
    version in the read view. From there, `}` goes to now and `=` diffs.
  - The launch row opens `AGENTS.md` at the launch version, or the stored snapshot as a
    read-only document titled `AGENTS.md as launched · not in git`.
  - The memory read report for a batch read lists each target's version read.
- **Loading.** Chips load in a later off-thread pass than the lane's rows. They never
  delay or reflow the lane: chips are appended at the row end.

### 4.8 Review watermark

- **Model.** A per-scope watermark stores a commit, its committer time, and when it was
  marked.
  - It is keyed by scope key, so it is shared across every workspace clone of a project.
  - It is persisted as state by sase-core.
  - It changes only on an explicit mark.
- **"N new"** counts the changesets that would appear as rows by default (not hidden,
  not regen-only) that are strict first-parent descendants of the watermark commit. If
  this checkout does not know that commit, the count falls back to committer time.
- **No watermark yet.** The header shows a dim `not reviewed yet · m to mark` and no
  count. Nothing is ever marked automatically, and opening the lens marks nothing.
- **TUI.**
  - The Changes lens header shows `● N new`, and unreviewed rows carry a `●` dot in the
    accent.
  - `m` marks the shown scope reviewed through its newest changeset; in `All scopes` it
    marks each scope.
  - The Config hub `MEMORY` sub-tab shows `●N` when the launch project scope has
    unreviewed changesets. That count is computed at quiet time and on token drift, and
    the badge is hidden at zero or with no watermark.
- **CLI.**
  - `sase memory history` (feed mode) prints `● N new since you last reviewed <date>`
    per scope in the text and pager formats. The JSON wire is unchanged.
  - `-m/--mark-reviewed` advances the shown scopes' watermarks to their newest
    changesets. It is an error with selectors.

### 4.9 The visual system

- **One palette.** `history_styles_for_theme(app.current_theme)` (memoized per theme)
  supplies the violet past accent, kept at least 60° in hue from amber; insert and
  strike styles; pill pairs; and the scrubber ramp.
  - Amber means only uncommitted or unpublished.
  - Switching themes repaints.
  - Colour always has a matching glyph or label.
- **The past frame.** While pinned in the past, the card border takes the past accent; a
  tombstone pin takes the deleted style. This costs zero rows.
- **One glyph table.** `✚ ◆ ⇧ ⇩ ▣ ⟳ ⚙ ≈ ↦ ✖ ◌ ⇡N` come only from
  `sase.memory.history.vocabulary`, through the kit.
- **Typography.**
  - Subject names are bold. Secondary text uses the theme's secondary foreground, not
    bare `dim`, wherever it carries state.
  - Word deltas are right-aligned.
  - Lists use relative day names; card heads use absolute timestamps.
  - Meaning comes first, then bead or agent, then SHA.
- **No layout jumps.** The strip and every lazily filled region (glance column, chips,
  changes sections) reserve or append space and never push content that is already on
  screen.

### 4.10 Keys

New bindings go in `ace.keymaps.memory`. All are free there today.

| Key       | Keymap entry                      | Action                                                   | Where                |
| --------- | --------------------------------- | -------------------------------------------------------- | -------------------- |
| `(` / `)` | `history_older` / `history_newer` | Older / newer version                                    | Notes card, Timeline |
| `{` / `}` | `history_first` / `history_now`   | First version / now (the tombstone for deleted subjects) | Notes card, Timeline |
| `=`       | `history_toggle_diff`             | Toggle read and diff (sticky)                            | Notes, Timeline      |
| `@`       | `history_timeline`                | Open or close the Timeline lens                          | Notes, Timeline      |
| `b`       | `history_compare_base`            | Set or clear the compare base                            | Timeline             |
| `.`       | `history_toggle_hidden`           | Reveal hidden versions (chips are inert there)           | Timeline             |
| `D`       | `toggle_deleted`                  | Show or hide deleted subjects                            | Notes                |
| `m`       | `mark_reviewed`                   | Mark the shown scope(s) reviewed                         | Changes              |
| `H`       | `open_history` (changed)          | Open the pager at the exact pin and view                 | All                  |
| `C`       | `open_changes` (changed)          | Open or close the Changes lens                           | Notes, Changes       |

**Plumbing checklist** for every key change:

- `MemoryPanelKeymaps` in `src/sase/ace/tui/keymaps/app_keymaps.py`
- `_MEMORY_BINDING_META` in `src/sase/ace/tui/keymaps/metadata.py`
- `build_memory_bindings` and `memory_help_bindings` in
  `src/sase/ace/tui/keymaps/bindings.py`
- `ace.keymaps.memory` in `src/sase/default_config.yml` (per the gotchas note)
- `src/sase/config/sase.schema.json`
- the keymap table in `docs/configuration.md`
- the conditional footer (`build_panel_footer` in `memory_panel_rendering.py`)
- the help modal (`memory_panel_help_modal.py`)
- the keymap, schema, and default tests (`tests/test_keymaps_defaults_panels.py` and the
  Memory panel tests)

## 5. Architecture

### 5.1 Boundary and module map

```text
sase-core (index, classes, prose diff, feed; new: blob:<oid> selector, review watermark)
        │  PyO3, GIL released
shared_history_service() ──► AceMemoryHistory (per app: memo, LRUs, single-flight, tokens, warm-up)
        │                          │
        │      ┌───────────────────┼──────────────────┬──────────────────────┐
        │   Notes card          Timeline lens       Changes lens          Agents MEMORY lane
        │   head + strip + steps  picker rows        feed_model rows       read/launch chips
        │      └──────── public pure kit: sase.pager.history_kit ──────────┘
        └──► pager provider factory (shared service) ──► PagerScreen ◄── exact pin from every door
```

| Piece                                                                  | Home                                                                                                   |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Blob selector, review watermark state and N-new                        | sase-core (`memory_history`), plus thin facade functions in `src/sase/core/memory_history_facade.py`   |
| `shared_history_service()`                                             | `src/sase/memory/history/service.py`                                                                   |
| `feed_model` (day grouping, row text, regen folding, provenance items) | `src/sase/memory/history/feed_model.py` (pure; shared by the pager feed document and the Changes lens) |
| The public kit                                                         | `src/sase/pager/history_kit.py` (pure re-export facade)                                                |
| Picker column fitting                                                  | `src/sase/pager/_timeline_picker_rows.py` (moved out of the Textual modal module)                      |
| `AceMemoryHistory`                                                     | `src/sase/ace/tui/memory_history.py`                                                                   |
| Card head and strip renderers                                          | `src/sase/ace/tui/modals/memory_pane_time_strip.py` (pure)                                             |
| Card time state, stepping, prefetch                                    | `src/sase/ace/tui/modals/memory_pane_time.py` (mixin)                                                  |
| Lens framework                                                         | `src/sase/ace/tui/modals/memory_pane_lens.py` (mixin)                                                  |
| Timeline and Changes lenses                                            | `memory_pane_timeline_lens.py` and `memory_pane_changes_lens.py` (mixins plus pure row builders)       |
| Rail glance, deleted subjects, Instructions group                      | `memory_pane_glance.py` plus rendering helpers                                                         |
| Agents chips                                                           | next to `_agent_memory_reads.py`, in a new module under `widgets/prompt_panel/`                        |

The module names are guidance. Keep every new module under about 700 lines (the `toobig`
warn threshold), and prefer a new mixin module to growing an existing one.

### 5.2 `AceMemoryHistory`

There is one instance per ACE app. It is created lazily and stored on the app, and tests
get a fresh instance per app. It wraps `shared_history_service()`. Every method blocks,
and the class docstring says that methods run **only off the event loop**: in thread
workers or pump-free tasks.

- **Queries:**
  - `scope_for_ref(ref)`: from the scope ring's `content_root`, or home.
  - `timeline(scope, selector)`: returns a snapshot of the wire, a fingerprint, and
    `validated_at`.
  - `version_body(scope, selector, version)`
  - `comparison(scope, selector, base, target)`
  - `feed(scopes)`
  - `subjects(scope)`
  - `version_for_blob(scope, selector, oid)` (from `agents-bridge`)
- **No pre-query sync.** Core queries sync internally. `sync()` is called only by the
  warm-up.
- **Caches** (bounded; sizes are guidance):
  - Timeline memo keyed by `(scope_key, core_selector, include_hidden)`.
  - Body LRU (128 entries) keyed by `(scope_key, blob_oid)`, for committed versions
    only.
  - Comparison LRU (64 entries) keyed by `(scope_key, base_blob, target_blob)`, for
    committed pairs only. Anything involving now or staged is never cached.
  - Subjects and feed memos keyed by the scope's change token.
- **Single-flight.** Concurrent requests for the same key share one in-flight call.
- **Freshness contract:**
  1. A memo hit renders immediately.
  2. If the entry is older than about 2 s, a background revalidation runs. The UI
     repaints only when the fingerprint changed.
  3. Explicit invalidation drops a subject's or a scope's entries and refetches the
     visible subject. It happens on `r`, publish, add, edit, delete, return from
     `$EDITOR` (the existing restat path), and a scope inventory change.
  4. While the pane is visible, a thin `set_interval` (about 5 s) spawns an off-thread
     probe of the stat-only change token. The token covers the scope repo's HEAD file,
     the ref it names, `packed-refs`, and the index (mtime_ns and size), plus the
     selected subject's worktree file. The git dir is resolved once per scope, and
     `.git` files are handled. The probe skips while `NavigationGate` reports activity
     and stops when the pane is hidden or unmounted (`tui_perf` rules 2, 10, 13, 14).
     Drift triggers rule 3.
  5. At now, a revalidation advances the strip. In the past, the pin holds: it
     re-resolves by commit and blob after an index rebuild, and `N newer` updates. If
     the pinned version vanished, the card returns to now with a toast.
- **Warm-up.** Once per app, after ACE's startup stopwatch ends, a thread worker syncs
  the launch project scope and home. It follows the `_startup_*.py` warm pattern, for
  example `src/sase/ace/tui/actions/_startup_misspellings.py`. First paint never waits
  on it, so the first `H` never pays the 2–4 s cold index cost.
- **Shared with the pager.** `memory_history_provider_factory()` uses
  `shared_history_service()` when no service is injected. `HistoryService` stays
  thread-safe; `forget_scopes()` on `r` drops memoized scope objects so a new
  `AGENTS.md` is picked up.

### 5.3 The public kit

`src/sase/pager/history_kit.py` is a thin, Textual-free re-export facade, so it can
never drift from the pager.

- **What it re-exports:**
  - the moment model, pins, and `SectionTimeState`
  - pill, context, honest-chip, and footer-verb helpers
  - time-band data and rendering, with the meaning and cause rows
  - scrubber, sparkline, `format_age`, and `chrome_row_budget`
  - `build_diff_body` and `diff_endpoints`
  - history styles
  - picker rows, filters, and header and footer text
  - the picker column fitter
  - `visible_ordinals_for_timeline` and its sibling timeline helpers
  - the vocabulary glyph and label functions
- **Promoted names.** The picker's `_PickerColumns`, `_picker_columns`, and
  `_format_picker_row` move into `src/sase/pager/_timeline_picker_rows.py` as the public
  `PickerColumns`, `picker_columns`, and `format_picker_row`. The modal imports them
  from there, and pager picker goldens stay byte-identical. Promote other private
  helpers (for example `_normalize_picker_compare`) the same way only when a phase needs
  them.
- **Import guard.** A test asserts that no module under `src/sase/ace/` imports
  `sase.pager._time_band*`, `sase.pager._chrome_history`,
  `sase.pager._timeline_picker*`, or
  `sase.pager.history.{diff,moment,models,styles,timeline}` directly.
- **In-flight pager epics.** If one of them moves a re-exported name, update the kit's
  import in the same change. The kit isolates ACE from those moves.

### 5.4 Reliability rules (every phase)

1. **No git, service, or file IO on a keystroke or render path.** Steps and lens motion
   use prefetched data. Loads run in thread workers (or `spawn_pump_free_task`) with
   generation counters. Selection, scope, lens, and mount state are re-read after every
   await. Cancelling a worker is not enough (`tui_perf` rules 1, 2, 4, 7).
2. **One immutable "card moment" is applied atomically.** Path line, pill, strip, frame,
   body, and footer update together from one value. A pill is never painted over another
   version's body, and another subject's bytes never appear.
3. **The highlight moves at once; details are debounced** through
   `DetailPanelDebouncer`. Programmatic `OptionList.highlighted` changes use the
   established guard (`tui_perf` rule 12).
4. **Honest by construction.**
   - States come from the timeline wire through the kit.
   - A missing response becomes `history unavailable · <reason>`, never "no history".
   - Passive decorations (glance column, chips) are simply omitted when their lookup
     fails, never faked.
   - A key the user pressed that fails always toasts.
5. **Main-thread UI mutations only.** Async workers call `push_screen` and `notify`
   directly after their `await`. Thread workers marshal through `call_from_thread`.
   Never mix the two, and the guard test from `front-door` enforces it.
6. **Prefetch policy.**
   - After landing on a pin, and after the selection debounce at now, fetch the bodies
     of the nearest two visible older and newer versions. In diff view, also fetch their
     parent comparisons.
   - On a cache miss, strip row 1 says `loading v23…` while the previous version stays
     on screen. The last request wins.

### 5.5 Performance budgets

Each phase wraps its new paths in `SASE_TUI_TRACE` spans
(`src/sase/ace/tui/util/trace.py`; for example `memory.history.query`, `.step`,
`.strip_load`, `.lens_open`). Each phase records the measured numbers in its bead notes,
and `launch` re-measures all of them.

| Budget                                         | Target                                           |
| ---------------------------------------------- | ------------------------------------------------ |
| Pane first paint                               | Unchanged: no history work before first paint    |
| `j`/`k` key to paint in the pane and lenses    | p95 < 16 ms                                      |
| Warm `(`/`)`/`{`/`}`/`=` step to paint         | p95 ≤ 30 ms                                      |
| Cold note selection to populated strip         | about 200 ms (150 ms debounce plus one timeline) |
| Timeline lens open, warm (300-version subject) | ≤ 100 ms                                         |
| Changes lens open, warm                        | ≤ 150 ms populated                               |
| Rapid stepping or lens scrolling               | Zero rows in `~/.sase/logs/tui_stalls.jsonl`     |

Landing `sase-1ee` (about 9 git spawns per warm query in sase-core) helps every number,
but this epic does not depend on it.

## 6. Conventions for every phase

- **Memory notes to read first.** Before TUI changes, read `tui.md` and `tui_perf.md`.
  Before goldens or live captures, read `tui_screenshot.md` and `lint_and_test.md`. Read
  `symvision.md` when Symvision complains, and `cli_rules.md` before CLI option changes.
  Before core changes, read the `rust_core_backend_boundary` core memory and sase-core's
  `AGENTS.md`.
- **sase-core changes.** Open the linked checkout with `sase repo open sase-core`, work
  in the printed path, and run its `sase tool run check`. Commit both repos in the
  turn's declaration; the host moves `sase-core-revision.txt` automatically
  (`docs/rust_backend.md`, "The CI source revision pin").
- **Tests.** Write unit tests for pure renderers and view-models. Write headless key
  tests mounted in the production host (Admin Center, Config, Memory) with a
  deterministic stub service. Include a reliability test for every new worker path:
  stale results are dropped and failures toast.
- **Goldens.**
  - Capture in the Admin Center host (D13), in dark and light themes, at 120×40, plus
    the listed 80×24 states.
  - Run `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`.
  - Inspect every created, updated, and removed golden. Generation is not approval.
  - Name new goldens `memory_pane_<surface>_<state>_<theme>_<WxH>.png`.
- **Verification.** Run `sase tool run check`. Run `check-full` only when explicitly
  told to.
- **Symvision.** Land public symbols together with their consumer. Use an
  `--epic-symbol <phase-bead>(<symbol>)` Justfile entry only when a later phase of this
  epic is the consumer.
- **Docs travel with behavior.** Every phase that changes keys or visible behavior
  updates the "History in the Memory pane" subsection of `docs/ace.md` and the keymap
  table in `docs/configuration.md` in the same change.
- **Follow-ups.** Phase workers never create beads. They append
  `PROPOSED FOLLOW-UP: <summary — detail>` notes to their own phase bead.

## 7. Phase: Repair the H and C front door (`front-door`)

- **Fix both actions.** Rewrite `action_open_history` and `action_open_changes` in
  `src/sase/ace/tui/modals/memory_pane_history.py`.
  - Keep the async worker:
    `run_worker(coro, exclusive=True, group=..., exit_on_error=False)`.
  - Build with `await asyncio.to_thread(...)`.
  - Back on the loop, re-check `_closed`, `is_mounted`, `_host_visible`, the selected
    identity, and the scope key. Then call `self.app.push_screen(...)` and
    `self.notify(...)` directly.
  - No `call_from_thread` remains in either coroutine.
- **Honest failures.** Build failures toast with a reason:
  - `could not open history: <reason>`
  - `no history yet · commit this file to start its history`, when untracked
  - `home memory is not in git`, for `NO VCS`
- **Guard test** `tests/ace/tui/test_call_from_thread_guard.py`, following the pattern
  of `tests/test_timezone_display_guard.py`. It AST-scans `src/sase/ace/tui/**/*.py` and
  fails on any `call_from_thread` call whose nearest enclosing function is an
  `async def`. It must flag exactly the four current sites before the fix.
- **Key-press tests from the Admin Center host,** using a stub `HistoryService`:
  - `H` pushes `PagerScreen` for a note, a web descriptor, and a strand.
  - `C` pushes the feed pager.
  - A raising build shows an error toast.
  - Moving the selection, switching scope, or hiding the hub during a slow build pushes
    nothing.

  Write the tests first and confirm they fail on the old code. Record that in the bead
  notes.

- **Docs.**
  - Add a short "History in the Memory pane" subsection to `docs/ace.md` (`H`, `C`, and
    the History row).
  - Correct `docs/memory_history.md` wherever it implies that `Z` opens the pager's time
    axis. `Z` hands the file to the artifact viewer.
- **Done when:**
  - `H` and `C` open from the Admin Center.
  - The new tests fail on the old code and pass on the new code.
  - The guard passes.

## 8. Phase: App-scoped history service and the public history kit (`history-service`)

- **Shared service.** Add `shared_history_service()`: a process-wide, lazy, thread-safe
  `HistoryService` (§5.2). A construction failure is retried on the next call, not
  cached. `memory_history_provider_factory()` uses it when no service is injected.
- **`AceMemoryHistory`** implements every query, cache, single-flight, freshness rule,
  change token, and warm-up in §5.2. Add a `memory.history.query` trace span with cache
  hit or miss attributes.
- **Migrate the Memory pane onto it.**
  - `_history_service_or_none` and the per-pane `HistoryService` go away.
  - `H` and `C` use the shared service.
  - The History row loader uses `timeline()`; drop the redundant `sync()` in
    `fetch_history_summary`.
  - A failed load settles to a dim `history unavailable · r retry`. `r` clears the
    failure, calls `forget_scopes()` and invalidation, and refetches.
  - Publish, add, edit, delete, and return from the editor invalidate the subject.

  The History row is replaced in the next phase, so keep the row changes minimal.

- **The kit** (§5.3). Add `src/sase/pager/history_kit.py` and move the picker column
  fitter into `_timeline_picker_rows.py` with public names. Switch ACE's sparkline
  import to the kit, and add the import-guard test.
- **Tests.**
  - The memo, stale-while-revalidate, single-flight, and both LRUs, using a fake service
    and a fake clock. Assert that `now` bodies and comparisons are never cached.
  - Change-token drift, including `.git` files.
  - The invalidation hooks.
  - Factory sharing: two pager lookups construct one service.
  - The warm-up runs off-thread after the startup stopwatch and does not delay first
    paint.
  - Both guard tests.
  - No pager golden changes.
- **Done when:**
  - Repeated timeline calls are memo hits, and the History row never shows an eternal
    `…`.
  - The pager provider shares the service.
  - ACE imports history presentation only through the kit.

## 9. Phase: Pinned card head with the two-row time strip (`time-strip`)

- **Restructure the card.**
  - `#memory-panel-detail` becomes a bordered `Vertical`. It holds the head (title
    `Static` with the title row and path line, and a fixed-height
    `#memory-panel-time-strip` `Static`), then a `VerticalScroll`
    (`#memory-panel-card-scroll`, `1fr`) with the description, body, and meta.
  - Retarget `action_scroll_body_down`/`up` (`memory_panel_navigation.py`) and the
    travel `scroll_home` (`memory_panel_travel.py`) to the new scroll region.
  - Update `styles.tcss` (`MemoryPane #memory-panel-*` rules) and keep
    `note_rail_width`'s margin math correct.
- **Pure renderers** in `memory_pane_time_strip.py`:
  - `render_path_line(...)`: the path, the pill (`history_badge`), the context
    (`history_context`), and chips.
  - `render_time_strip(data, *, width, rows, styles)`: built from the kit's band
    builders and rows, covering every §4.2 state. At clean now, row 2 is the newest
    version's meaning row, dimmed.
- **Behavior.**
  - The strip shows for notes, web descriptors (new), and strands. It is hidden when no
    subject is selected and for diagnostics and empty-scope cards.
  - It reserves 2 rows and folds to 1 under 14 card rows.
  - A memo hit paints instantly. With no memo yet, it shows `indexing…`.
  - A failure keeps the last good snapshot with a `stale` mark and offers `r retry`.
  - The 150 ms debouncer drives loads, and the strip's rows never move the body.
- **Theme.** Styles come from `history_styles_for_theme(app.current_theme)`, memoized
  per theme, and a theme switch repaints.
- **Remove the History property row.** Remove it from `build_rail_node_card_meta` and
  `_build_note_property_grid`, delete its now-dead helpers in `memory_panel_history.py`
  and their tests, and delete the `memory_panel_history_*` goldens. This absorbs the
  panel half of `sase-1en`.
- **Goldens.**
  - At 120×40 in dark and light: now clean, now uncommitted, untracked, `NO VCS`,
    indexing, unavailable, a web descriptor, and a strand.
  - At 80×24 in dark and light: now clean and now uncommitted, showing the folded strip.
- **Done when:**
  - Every now state renders in the pager's words.
  - Load and failure never move the body.
  - The goldens are reviewed and the History row is gone.

## 10. Phase: Step through versions on the card (`card-stepping`)

- **Keys.** Add `history_older` `(`, `history_newer` `)`, `history_first` `{`, and
  `history_now` `}` through the full §4.10 checklist.
- **Card time state** (`memory_pane_time.py`).
  - Reuse the kit's `SectionTimeState`, `build_moment` and `moment_for_state`,
    `step_target` and `boundary_notice`, and `visible_ordinals_for_timeline`. Never
    re-derive numbering.
  - Produce one immutable card moment and apply it atomically (§5.4 rule 2).
- **Past read view** (§4.3).
  - Bodies come from `version_body`.
  - Frontmatter is stripped with the pane's existing note parser.
  - The past description, type banner, and path are shown.
  - Relation chips are dimmed under `links as of now`.
- **Prefetch and pending intents.** Follow §5.4 rule 6. A step pressed before the
  timeline lands is recorded as a pending intent (last wins) and applied when the
  timeline arrives.
- **Past frame.** Set the card border inline from the past accent, and from the deleted
  style for a tombstone pin. Clear the inline rule at now so the CSS border returns.
- **`H` at the pin.** It calls `build_history_document(initial_revision="v<N>", ...)`
  for the card's subject.
- **Guards, strands, and arrival rules** follow §4.3 and D7.
- **Esc ladder.** The filter closes first, then a past pin returns to now, then the host
  closes. `q` still closes directly.
- **Freshness while pinned.** Follow §5.2 rule 5.
- **Footer and help.** The footer uses `time_verbs_for_moment` destinations and hides
  refusing verbs while pinned. The help modal gains a **Time** group with the pill
  legend and the glyph table.
- **Tests.**
  - Every step intent and boundary, using a fake timeline with hidden versions, a dirty
    now, and a deleted-then-recreated subject.
  - Atomic swap: a slow body never shows under a new pill.
  - Rapid stepping: the last step wins.
  - Selection changes mid-load.
  - The guards.
  - `o` opens now.
  - A past strand view writes no `memory_reads.jsonl` event, while a strand selected at
    now still does.
  - The pin re-resolves after a timeline change.
- **Perf.** Add the `memory.history.step` span. Record the warm step p95 (≤ 30 ms), the
  `j`/`k` p95, and zero stalls during rapid stepping.
- **Goldens.**
  - At 120×40 in dark and light: the past read view, a strand in the past with the
    `not audited` chip, and a tombstone pin.
  - At 80×24 in dark and light: the past read view.
- **Done when:** stepping meets its budget, and the pill and body never disagree.

## 11. Phase: Word-diff view on the card (`card-diff`)

- **The key.** `history_toggle_diff` `=` through the checklist. The choice is sticky for
  the pane session; opening the pane resets it to read.
- **The diff widget.** Add a `#memory-panel-card-diff` `Static` that renders
  `build_diff_body(comparison, target_body, history_styles=...)`.
  - The `Markdown` body hides while the diff shows.
  - Folds are fixed, and their label reads `H to expand`.
  - Strip row 1 shows the band's `Δ vA → vB · diff`.
- **Endpoints** follow §4.3. Instead of re-deriving them, use the kit's `diff_endpoints`
  and the moment's `diff`.
- **Comparisons.** They come from `AceMemoryHistory.comparison`. Committed pairs are
  cached by blob pair. Pairs with now are never cached, and they refetch after an edit
  or a return from the editor. In diff view, prefetch the pin's parent comparison and
  its neighbours' comparisons.
- **`H` carries the view.** It passes `view="diff"` and the compare base.
- **Tests.**
  - Endpoint selection in every state.
  - The view stays sticky across selections and resets when the pane opens.
  - A comparison failure keeps the read view and toasts.
  - **The publish loop**, end to end with a fake service: edit, `◌ NOW · uncommitted`,
    `=` shows the pending diff, publish, then `● NOW · v26` with a new bar.
- **Goldens.**
  - At 120×40 in dark and light: past diff, dirty-now pending diff, clean-now latest
    change, and the first-version diff.
  - At 80×24 in dark and light: past diff.
- **Done when:** `=` works in every card state within the step budget.

## 12. Phase: Lens framework and the Timeline lens (`timeline-lens`)

- **Lens framework** (`memory_pane_lens.py`).
  - The lens state is Notes, Timeline, or Changes.
  - On entry, take a Notes snapshot: selected identity, filter text and the body-filter
    flag, expanded webs, rail and card scroll, focus, card pin and view, and the trail.
    Restore it on exit.
  - The header names the lens, the scope, and the subject.
  - Each lens has its own footer and key routing. Inert keys do nothing and are hidden.
  - The `/` filter routes to the lens's rows.
  - `.` routes per lens: it is the chip prefix in Notes and the hidden toggle in
    Timeline (`on_key` ordering with `handle_numbered_link_key`).
  - `travel_back` (`h`/backspace) exits a lens.
  - The full Esc ladder (D3): filter, then lens, then pin, then the host.
- **Timeline lens** (§4.4).
  - Keys: `history_timeline` `@`, `history_compare_base` `b`, and
    `history_toggle_hidden` `.`, through the checklist.
  - Rows use `build_picker_rows` with the kit's `picker_columns` and `format_picker_row`
    at the rail's width. The rail may widen to its maximum in this lens.
  - The rows carry the markers and the hidden-summary row, and `/` filters them with
    `filter_picker_rows`.
  - Previews follow the debounced card moment.
- **Compare base and hand-off.**
  - Extend `build_history_document` with `explicit_base: bool`, so a `now` target can
    compare against a committed base.
  - `⏎`/`l`/`H` push the pager at the cursor's pin, view, and base. Closing the pager
    returns to the same lens and row.
- **Exit and mouse.**
  - Exiting keeps the card on the cursor's version (D3).
  - Clicking the strip opens the lens.
  - The rail motion pushes no trail entries.
- **Tests.**
  - Snapshot and restore round trip: filter, expanded web, scroll, and trail survive.
  - The Esc ladder order.
  - Each lens's key routing, including `.` in both lenses.
  - Base ordering and `Same version`.
  - `now` and `STAGED` stay independently selectable.
  - The pager hand-off carries pin, view, and base.
  - A stale preview is dropped.
- **Perf.** A 300-version subject opens within 100 ms warm. Window the rows if it does
  not.
- **Goldens.**
  - At 120×40 in dark and light: the cursor on a past version, with a base, and with
    hidden versions revealed.
  - At 80×24: one state.
- **Done when:** any version of any subject is comparable without leaving ACE.

## 13. Phase: Changes lens over a shared feed view-model (`changes-lens`)

- **`feed_model`.** Extract a pure `src/sase/memory/history/feed_model.py` from
  `feed_document.py`. It holds the changeset view-model:
  - day key and header, and clock
  - authored subjects, the first-subject label and `+N more`
  - the word delta and the home tag
  - provenance items
  - the generated-consequence fold and regen-only folding

  `build_feed_document` consumes it, and the pager feed goldens stay byte-identical.

- **The `C` key.** `open_changes` now toggles the Changes lens instead of pushing the
  pager feed. Update the footer, help, and docs.
- **The lens** (§4.5).
  - **Data.** One `feed(scopes, limit=None, include_hidden=True)` per open, refresh, or
    scope change, off-thread.
  - **Window.** Render the newest 100 rows. Extend by 100 when the cursor reaches the
    end row, and dedupe by `(scope, commit, subject, ordinal)`.
  - **Filter.** It matches the whole fetched list.
  - **Scopes.** The ring plus the lens-only `All scopes` entry, with failed scopes shown
    as header chips.
- **The card.**
  - The head shows provenance chips, using the Artifacts tab's icons and accents.
  - Per-subject sections come from `version_body` plus `comparison` against the parent
    (against empty for `created`; a tombstone line for `deleted`). They fill
    progressively under the debouncer with a generation guard.
  - At most six sections show inline, followed by the `⟳` consequences line.
- **Opening.** `⏎`/`l`/`H` and `.1`–`.9` open the pager in the diff view at that
  changeset's version.
- **Tests.**
  - `feed_model` parity with the old document builder.
  - Day rules are skipped by `j`/`k`.
  - Window extension, and the filter over all rows.
  - `All scopes` merges scopes and tags home rows `⌂`.
  - A failed scope shows as a chip.
  - Section loads drop stale results.
  - The pager hand-off uses the diff view at the right version.
  - Leaving restores Notes.
- **Perf.** Warm open is populated within 150 ms.
- **Goldens.**
  - At 120×40 in dark and light: the lens with a multi-subject changeset, and
    `All scopes`.
  - At 80×24: one state.
- **Done when:** every changeset in the window is reviewable without leaving ACE.

## 14. Phase: Rail recency glance and deleted subjects (`rail-glance`)

- **Recency column** (§4.6).
  - Per scope load or refresh, build an identity → newest-entry map off-thread from
    `subjects()` and `feed()`.
  - Render it right-aligned on each row's first line, with the kit's `format_age`.
  - Extend `note_rail_width` to account for the column. Recompute the rail width once
    when the map lands.
  - Shed the age, then the glyph.
- **Deleted subjects.** Add `toggle_deleted` `D` through the checklist.
  - The `DELETED` group comes from the same feed. Keep a subject only when its latest
    entry is a deletion, because some subjects were deleted and later recreated.
  - Tombstone cards follow §4.6.
- **History-only rail nodes.** Introduce a synthetic node kind for subjects that have no
  live `MemoryNote`. Audit every `node.note` access (mutations, chips, copy, source,
  viewer, strand reads, filters, the session bookmark) so note-only paths skip these
  nodes or refuse with a toast.
- **Tests.**
  - Glance mapping for notes, webs, and strands.
  - Width shedding.
  - A recreated subject is not listed as deleted.
  - Tombstone card behavior and refusals.
  - A failed glance load omits the column and keeps the rail.
- **Goldens.**
  - At 120×40 in dark and light: the rail with the glance column, the `DELETED` group,
    and a tombstone card.
  - At 80×24: the glance column.
- **Done when:** every live and deleted subject the index knows is reachable from the
  rail.

## 15. Phase: Instructions group and instruction cards (`instructions-group`)

- **The group** (§4.6). Build the `INSTRUCTIONS` rail group from `subjects()` entries of
  kind `instructions`, using the history-only node kind. It is collapsed by default, and
  `space` toggles it.
  - Rows carry the alias, `⚠ diverged`, and `TEMPLATE` chips.
  - The filter matches rows by path.
- **The cards.**
  - The title uses a kind badge of the same grammar.
  - The strip uses cause rows.
  - The body is the rendered file at now, loaded off-thread through
    `version_body(..., "now")`.
  - The meta grid shows whether SASE renders the file and lists the shims.
  - The cards are read-only, with the refusal toasts from §4.6.
  - Stepping, diff, the Timeline lens, and `H` all work.
- **Tests.**
  - Grouping and aliasing.
  - Diverged and template chips.
  - Cause rows in the strip.
  - Refusals, and `o` on a hand-written file.
  - Stepping on an instruction subject.
- **Goldens.**
  - At 120×40 in dark and light: the group collapsed and expanded, an instruction card
    at now with its cause row, and a diverged chip.
- **Done when:** "why did `AGENTS.md` change?" is answerable from the pane.

## 16. Phase: Memory as seen by the agent in the Agents tab (`agents-bridge`)

- **Core.**
  - Add `blob:<oid>` to `select_committed` in sase-core
    `crates/sase_core/src/memory_history/query.rs`.
  - It accepts a full OID or a unique prefix of at least 7 hex characters.
  - It returns the newest committed version whose `blob_oid` matches. A deleted row's
    tombstone blob does not count.
  - An ambiguous prefix and a missing blob are errors that name the blob.
  - Add core tests.
  - No Python facade change is needed: `version()` passes the selector through.
  - Update the `sase memory history -A` help to list `blob:OID`, refresh the CLI
    completion caption and `cli_spec.json` as the CLI tests require, and follow
    `cli_rules`.
- **Service.** Add `AceMemoryHistory.version_for_blob(scope, selector, oid)`. It returns
  the ordinal, the newest version, `newer_count`, and `now_matches`, using `version()`
  plus the memoized timeline and the kit's `build_moment`. Memoize it by
  `(scope_key, selector, oid, timeline fingerprint)`.
- **Agents MEMORY lane** (§4.7).
  - Chips are computed in a follow-up enrichment pass after lane batch 2 publishes
    (`_agent_display_header_summary.py` / `_agent_display_async*.py`), as a separate
    partial publish. They never delay the rows.
  - Only visible rows are resolved: at most five reads plus the launch row.
  - The scope comes from the read's project, using the same `content_root` resolution as
    the Memory pane's ring. An unresolvable scope omits the chips.
  - The launch row is read from `agent_meta.json` (`instruction_snapshot`, then
    `workspace_head`).
  - The not-in-git snapshot opens from `~/.sase/instruction_snapshots/<oid>`.
- **Hints and report.**
  - Extend `register_memory_read_report_hint` and the view processing
    (`actions/hints/_view_processing.py`) with a version-pinned memory target, which is
    built with `build_history_document`.
  - The batch memory read report (`src/sase/memory/memory_read_report.py`) lists each
    target's version read.
- **Tests.**
  - Core selector cases: full OID, prefix, ambiguous prefix, missing blob, tombstone,
    and a reverted blob resolving to its newest match.
  - Chip states.
  - The aggregate batch chip.
  - Launch-row resolution, the not-in-git and unavailable cases, and an agent with no
    evidence.
  - A hint opens the pager at the pinned version.
  - Chip loading never delays the lane's first publish.
- **Goldens.**
  - The Agents context card with chips and the launch row. Use
    `patch_startup_loaders(memory_reads=...)` and the existing `context_memory_reads()`
    fixture in `tests/ace/tui/visual/`, at 120×40 in dark and light.
- **Done when:** from any agent, you can open exactly what it read and see whether it
  has changed since.

## 17. Phase: Core review watermark and the CLI feed header (`watermark-core`)

- **Core** (sase-core `memory_history`).
  - Add a watermark store keyed by scope key. Each record holds the commit, the
    committer time, and the time it was marked.
  - Writes are atomic under a lock, and readers never wait on the lock.
  - The file lives under a state directory the caller passes in: SASE's state area,
    never the disposable `cache_dir`. A corrupt store reads as empty and is reported,
    not silently overwritten.
- **Bindings**, each releasing the GIL:
  - `memory_history_review_state(scopes, state_dir)` returns the watermark, `new_count`,
    and the newest commit for each scope, with N-new semantics as in §4.8.
  - `memory_history_mark_reviewed(scope, through_commit, state_dir)` records a
    watermark.
  - Add facade functions in `memory_history_facade.py` and `HistoryService` methods.
- **CLI.**
  - In feed mode, `sase memory history` prints the per-scope
    `● N new since you last reviewed <date>` header line in the text and pager formats,
    through `feed_model`.
  - Add `-m/--mark-reviewed` (feed mode only; an error with selectors), with help text
    per `cli_rules`. Refresh the completion and `cli_spec.json` tests.
  - The JSON wire is unchanged.
- **Tests.**
  - Core: descendant counting, the non-ancestor fallback, the first-use empty state, the
    atomic write, and a corrupt store.
  - CLI: the header, the flag, the error with selectors, and the help snapshot.
- **Done when:** the watermark survives restarts, is shared across workspace clones, and
  never advances on its own.

## 18. Phase: Review watermark in the Changes lens (`watermark-tui`)

- **Keys.** Add `mark_reviewed` `m` through the checklist. It is active in the Changes
  lens only.
- **The lens.**
  - The header shows `● N new` per §4.8, or `not reviewed yet · m to mark`.
  - Unreviewed rows carry a `●` dot.
  - `m` clears the dots optimistically, persists off-thread, and toasts
    `marked N changesets reviewed · <scope>`. A failure restores the dots and toasts.
- **Sub-tab badge.** The Config hub `MEMORY` sub-tab label shows `●N` for the launch
  project scope. It is computed after the warm-up and on token drift, off-thread, and
  never blocks the hub. It is hidden at zero or with no watermark.
- **Tests.**
  - The header states.
  - Optimistic marking and the failure rollback.
  - Badge visibility.
  - Opening the lens marks nothing.
- **Goldens.** At 120×40 in dark and light: the lens with new rows, and the hub sub-tab
  badge.
- **Done when:** unreviewed memory changes are visible and explicitly dismissible.

## 19. Phase: Document, measure, and review end to end (`launch`)

- **Docs.**
  - Finish the "History in the Memory pane" subsection of `docs/ace.md`: strip anatomy
    and states, keys, lenses, the Esc ladder, the Agents chips, and the watermark.
  - In `docs/memory_history.md`, replace the "front door" description with an "In the
    TUI" section, and document `blob:OID` and `-m/--mark-reviewed`.
  - Verify every new key in the `docs/configuration.md` keymap table.
- **Visual review.**
  - Run the full `just fix-tui-screenshots` through `/sase_monitor`.
  - Inspect every created, updated, and removed golden.
  - Confirm that no unrelated golden changed and that the standalone-modal History-row
    goldens are gone.
- **Performance.** Re-measure every §5.5 budget with `SASE_TUI_TRACE=1` and the stall
  log, including rapid `(`/`)` on `AGENTS.md`. Record a table in the bead notes.
- **Live walkthrough.** Use `sase screenshot` with `--keep` and `tmux send-keys` to
  capture the past card, the diff, the Timeline lens with a base, the Changes lens, the
  Instructions group, a tombstone, and the Agents chips. Inspect each PNG.
- **Bookkeeping for the land agent**, as notes on this phase's bead:
  - This epic fully implements `sase-1e6`.
  - It implements the TUI half and the core blob selector of `sase-1e5`. The
    `sase memory log` rows half remains.
  - It implements the panel half of `sase-1en`. The CLI text half remains.
  - Add `PROPOSED FOLLOW-UP:` notes for anything discovered.
- **Finally**, run `sase tool run check`.

## 20. Coordination with other work

- **In-flight pager epics.** `sase-1es` (pager body rewrite) and `sase-1eu` (PaneGrid
  splits) are in progress.
  - This epic never edits the pager body or screen layout.
  - Its pager-side edits are limited to the kit facade, the picker-row module move, the
    `feed_model` extraction, `build_history_document`'s `explicit_base`, and the shared
    service in the provider factory.
  - Rebase on master at each phase start. If those epics moved a re-exported name,
    update the kit in the same change.
- **Absorbed beads.**
  - `sase-1e6` (review watermark) is implemented by `watermark-core` and
    `watermark-tui`.
  - The TUI half of `sase-1e5` and its core lookup are implemented by `agents-bridge`.
  - The panel half of `sase-1en` is implemented by `time-strip`.
  - Do not launch `sase-1e6` separately while this epic is in flight.
- **Related, not blocking.** `sase-1ee` (per-query git spawns in sase-core) improves
  every budget, and `sase-1e4` and `sase-1ak` are known flakes in neighbouring tests.

## 21. Non-goals and follow-ups

- Embedding `PagerView` in the card (D1). Revisit after `sase-1es` and `sase-1eu` land,
  only if the card plus `H` proves too shallow.
- Restore as an unpublished draft (`sase-1e7`).
- The age lens (`sase-1e8`).
- A plain git-file history provider (`sase-1e9`).
- `path@rev`, side-by-side diffs, and per-host home rendering (`sase-1ea`).
- Fold expansion, hunk navigation, search, and splits inside the card. These are
  deep-read verbs and belong to the pager.
- `sase memory log` rows linked to the version read (the remaining half of `sase-1e5`),
  and the CLI text vocabulary from `sase-1en`.
