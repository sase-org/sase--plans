---
tier: epic
title: Agents tab Archive view retires Artifacts ▸ Agent
goal: "Every agent that ran on this machine can be found and read on the Agents tab. The
  Inbox stays exactly as it is today. A new Archive view (`,a`) lists dismissed runs by
  day and opens any of them read-only in the real decks. Restoring is one deliberate
  `⏎`. Every `agent:` route lands on the Agents tab. The Artifacts ▸ Agent sub-tab is
  retired behind a sunset flag.

  "
decisions:
  view_name:
    ask: What should the Agents tab's view of dismissed runs be called?
    choices:
      archive:
        Archive, in:archive; matches the agent_archive store and dismiss → archive
      history:
        History, in:history; matches the research wording, collides with ,y full-history
    default: archive
    why:
      Every agent sits in exactly one view, so Archive is exact; history already names
      inbox reloads
    answer: archive
  toggle_filter:
    ask: How should ,a treat the committed Inbox filter when it opens the Archive?
    choices:
      restore: Pure toggle; each view returns as left, shelf/bridge ⏎ carries a filter
      carry:
        Carry the Inbox filter when one is committed, else restore the parked Archive
    default: restore
    why:
      A toggle that always returns exactly where you were is predictable in both
      directions
    answer: restore
  jump_opens_archive:
    ask:
      Should Jump-panel digits open a dismissed target read-only in the Archive instead
      of reviving it?
    default: true
    why: Reading should never restore; restoring stays one deliberate ⏎ away
    answer: true
  glossary_archive_term:
    ask:
      Add a glossary strand defining the Archive view, the in:archive scope, and Restore
      to inbox?
    memory:
      - glossary:agent-archive
    default: false
    answer: false
  glossary_strand_updates:
    ask:
      Update the Nav Section, Nav Item, Agent Data Deck, and Agent Relation Jump Target
      glossary strands?
    memory:
      - glossary:nav-section
      - glossary:nav-item
      - glossary:agent-data-deck
      - glossary:agent-relation-jump-target
    default: false
    answer: false
phases:
  - id: archive-index
    title: Archive index v3 and honest timestamps
    depends_on: []
    size: medium
    description:
      "archive-index: bump the dismissed-bundle summary index to v3. Add the session,
      clan, agent-tab, and tribe columns the Archive groups and filters on. Record a
      real dismissed_at on every new bundle. Rebuild newest shard first, off the startup
      path, with cheap progress. Make the catalog accept every supported artifact-index
      schema version instead of one exact match."
  - id: archive-projection
    title: Core archive corpus, in scope token, and CLI parity
    depends_on:
      - archive-index
    size: large
    description:
      "archive-projection: build a cached sase-core archive corpus over the v3 index. It
      owns outcome, last activity, container, and restorable derivation, query
      evaluation, group summaries, windowed pages, and exact lookup. Add the
      agents-archive query profile and the host-owned in: token, then give sase agent
      search the same scope so the CLI is the parity oracle."
  - id: archive-view
    title: Archive view on the Agents tab
    depends_on:
      - archive-projection
    size: large
    description:
      "archive-view: add the beta-flagged ,a toggle and per-view parking. In the
      Archive, the node column becomes an Inbox pulse plus a day-grouped Archive list,
      with an archive info row, a filter bar driven by in:archive, a grouping picker,
      and read-only deck hydration from the bundle. Add the detail-ownership guard,
      live-only key toasts, the tab-label marker, the footer, and the help box."
  - id: record-deck
    title: Record deck for every agent
    depends_on:
      - archive-projection
    size: medium
    description:
      "record-deck: add a fifth agent data deck (picker key r) with Lifecycle,
      Provenance, Relations, and Links cards for inbox and archived agents alike. It
      absorbs the retiring pane's Details panel and relation rail."
  - id: archive-actions
    title: Restore, fork, and copy from the Archive
    depends_on:
      - archive-view
    size: medium
    description:
      "archive-actions: add the ⏎ chooser's ARCHIVE section (Restore to inbox by
      default, Restore and show, Fork, Open chat, Copy). Support container and marked
      bulk restore with skip reasons, and keep restored rows in place. Add y to copy the
      @agent reference on the Agents tab, plus the % copy-palette targets ported from
      the pane."
  - id: archive-arrivals
    title: Every agent route lands on the Agents tab
    depends_on:
      - archive-view
    size: medium
    description:
      "archive-arrivals: retarget agent: link follow, the !R custom search row, Files a,
      Jump-panel dismissed targets, the palette, and ,a from other tabs to the Inbox or
      the Archive. Each arrival gets a toast explaining why and one Ctrl+O hop back.
      Delete the Artifacts fallback route when the flag is on."
  - id: archive-bridges
    title: Inbox shelf, zero-result bridge, and Node Finder matches
    depends_on:
      - archive-view
    size: medium
    description:
      "archive-bridges: add the one-line Archive shelf docked under the Inbox node
      column, the zero-result bridge in the empty-state card, the dismiss toast's
      Archive pointer, and the Node Finder's In Archive section. All counts are computed
      off-thread on commit from the warm corpus."
  - id: retire-pane
    title: Retire Artifacts ▸ Agent and make the Archive unconditional
    depends_on:
      - record-deck
      - archive-actions
      - archive-arrivals
      - archive-bridges
    size: medium
    description:
      "retire-pane: delete the agents_archive_view beta flag's Off branches. Hide the
      Artifacts Agent sub-tab behind a new sunset flag, renumbering Stitch to 1, with a
      legacy mapping, a one-time toast, and the explicit-command redirect. Migrate pane
      saved queries, then update docs, help, goldens, and (if accepted) the glossary."
proposed_by: bbugyi200.athena.research.45.linker.w0
decided_by: auto
create_time: 2026-10-10 06:41:05
status: wip
---

# Plan: Agents tab Archive view retires Artifacts ▸ Agent

## Background

The user asked to move the Artifacts tab's **Agent** sub-tab into the top-level
**Agents** tab, and to lead the design so the result is intuitive, reliable, and
beautiful. Two research reports precede this plan. Read them with
`sase artifact read <ref> "<why>"` for the full evidence and the mock images:

- `research:202609/agent_history_in_agents_tab/agent_history_in_agents_tab.md`. It
  settled **whether** to retire the pane: yes, by giving the Agents tab a query-driven
  scope instead of turning the inbox into an archive.
- `research:202610/agents_tab_inbox_and_history_views/agents_tab_inbox_and_history_views.md`.
  It settled **how it should look**: two views of one tab, the same decks for both, read
  without restoring, and a deliberate restore.

This plan adopts most of that design. It changes the model in one place and simplifies
several others; [Departures from the research](#departures-from-the-research) lists each
one with its reason.

### What is true today (verified on master)

- **An agent has two homes, and the weaker one holds the past.**
  - The Agents tab has the good reader (Main / Files / Tools / FINAL decks), but it
    excludes dismissed agents by construction.
  - Artifacts ▸ Agent (`src/sase/ace/tui/widgets/artifacts/agents_*.py`) can see
    dismissed agents, but its detail panel is metadata only: the prompt is cut at 4,000
    characters, the chat is shown as a path, and there is no reply, diff, or tool
    content.
  - The pane is built on the Python name-registry catalog (`sase.agents.catalog`). On
    athena that catalog takes 11 s, and the pane's first rows take 12–90 s.
- **The code admits the split.**
  - Link follow falls back to the pane when the Agents tab can't show the target
    (`actions/link_follow.py:317-336`; toast at `_link_follow_toast.py:123`).
  - `!R` ▸ _Custom revival search…_ jumps to the pane (`_revive_flow.py:147`).
  - Files `a` and Jump-panel digits **revive** a dismissed agent just to show it.
- **The archive is large.** On athena there are 147 inbox agents, 10,917 archived
  top-level runs, and 36,807 workflow children. About 110 runs are archived per day.
  Every top-level row is `durably_revivable`. 15% of rows carry a non-terminal stored
  status, and 30% have only a start time.
- **The pane is barely used.** athena's `agents` query history has three entries
  (`limit:100`, `linked:true limit:100`, `linked:true limit:200`). The `linked:` facet
  is the part people use, so it must survive the move.
- **The data is already on disk.**
  - `~/.sase/dismissed_bundles/<YYYYMM>/<suffix>.json` holds the full serialized `Agent`
    (`Agent.from_bundle_dict` restores it).
  - Dismissal deletes only loader marker files (`done.json`, `workflow_state.json`,
    `prompt_step_*.json`). The chat, diff, and `tool_calls.jsonl` stay unless retention
    prunes them, so **read without restore is feasible**.
  - `dismissed_bundle_summaries` (Python-written, schema v2) indexes every bundle. It
    lacks session, clan, tab, and tribe columns. Its `dismissed_at` silently falls back
    to `stop_time`, because TUI dismissals never record one.
  - sase-core already has an unused paged `query_agent_archive` and an
    `agent_archive_facet_counts` (`crates/sase_core/src/agent_archive/mod.rs`).
- **Keys.** These are free on the Agents tab:
  - Leader `,a`.
  - Deck-picker `r` (reserved picker keys are `j k q p`).
  - `y`: it is globally bound to `artifacts_copy_reference`, which no-ops off Artifacts.

  These are taken: `w` (reword), `r` (refresh), `R` (retry), `g` (top), `[`/`]` (agent
  tabs), `<`/`>` (deck swap), and `H` (collapse). In leader mode, `,y` is "full history
  refresh", so the word _history_ already means something on this tab.

## Design principles

1. **The Inbox is home, and it does not change.** With no Archive interaction, the
   Inbox's rendering, loading, and keys stay byte-for-byte as they are today.
2. **Every local agent is in exactly one view.** Dismissing moves an agent to the
   Archive, and restoring moves it back. The Inbox is non-dismissed agents; the Archive
   is this machine's dismissed agents.
3. **Read without restoring.** Selecting an archived run opens the real decks,
   read-only. Reading never revives, never marks anything read, and never writes.
4. **The query is the truth.** The Archive's committed query really contains
   `in:archive`, shown as a chip, so `^` history, saved slots, and `sase agent search`
   all agree on what you're looking at.
5. **Every road leads to the Agents tab.** Links, `!R`, Files `a`, Jump digits, the
   palette, and old Artifacts commands land in the Inbox or the Archive, never in
   Artifacts.
6. **Nothing archive-sized runs on the paint path.** The archive corpus is built
   off-thread, cached by signature, and never touched by inbox first paint or idle
   ticks.

## The UX

### Two views, one tab

```text
 INBOX (startup, unchanged)                      ARCHIVE (,a)
 ┌───────────────────────────────────────┐       ┌──────────────────────────────────────────┐
 │ 147 [R10 W35 U1] · group: by status (o)│      │ ◷ ARCHIVE  this machine · 10,917 runs     │
 │ ⌂ athena 144 │ ⌨ apollo 3              │  ,a   │ in:archive · by day (o) · ✓ all searched  │
 │ ┌ @default ──────────────────────────┐ │ ────▶ │ ⌂ Inbox · 147 [R10 W35 U1] · 1 needs you  │
 │ │ ● RUNNING  research.45 ×6          │ │ ◀──── │ ▾ Today · Thu Oct 8 ─────────────── 33    │
 │ │ √ DONE     bob-cli-5p.4            │ │  ,a   │   19:52 ✗ FAILED      toobig-7e ×7  muse  │
 │ └────────────────────────────────────┘ │       │   19:18 ○ WAS RUNNING 0yn           opus  │
 │                                        │       │   12:06 √ DONE        bob-cli-5p.4  muse  │
 │ ◷ Archive · 10,917 runs · 42 today     │       │ ▸ Yesterday · Wed Oct 7 ─────────── 110   │
 └───────────────────────────────────────┘       │ ▸ Tue Oct 6 ─────────────────────── 98    │
   (shelf: archive-bridges phase)                 └──────────────────────────────────────────┘
```

- **Inbox** is today's Agents tab. Its only addition is a one-line **◷ Archive shelf**
  docked at the bottom of the node column. The shelf renders only once the archive has
  at least one run. It shows the archive's size or, under a committed filter, how many
  archived runs match.
- **Archive** replaces the agent-tab strip, the fleet status, and the tribe panels with
  two nav sections:
  - a one-line, always-live **⌂ Inbox pulse**, so an agent that needs you never goes
    unnoticed while you read old runs;
  - the **◷ Archive list**, with day banners and run rows.

  The identity header, the decks, and the Jump panel on the right are the same widgets
  the Inbox uses.

- **`,a` toggles** between the views. **Startup always opens the Inbox.** While the
  Agents tab is in the Archive, its top tab label reads **`Agents ◷`**, so you know
  where you'll land when you come back.

> [!decision] view_name = history
>
> Every user-facing "Archive" string reads "History" instead: the chip is `◷ HISTORY`,
> the shelf reads `◷ History · …`, the token is `in:history`, and the palette entry is
> _Agents: Open History_. The identity header keeps `◷ ARCHIVED · read-only`, because it
> describes the row's state. Code identifiers keep `archive`, matching the
> `agent_archive` store. The optional glossary strand keeps the `agent-archive` slug.

### Archive view anatomy

| Region          | Inbox                                   | Archive                                                                                                                                                                                                           |
| --------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Top tab label   | `Agents`                                | `Agents ◷` (also while the tab is parked in the Archive)                                                                                                                                                          |
| Info row        | counts · status strip · `group: … (o)`  | `◷ ARCHIVE` amber chip · `this machine · N runs` (or `N matches`) · filter chips with `in:archive` · `by day (o)` · completeness (`✓ all searched`, `rebuilding index · 4,210 so far`) · right-aligned `,a inbox` |
| Header row      | agent-tab strip, fleet status           | hidden (agent tabs and fleet are Inbox concepts)                                                                                                                                                                  |
| Node column     | tribe panels                            | **⌂ Inbox pulse** (one line), then the **◷ Archive list**                                                                                                                                                         |
| Identity header | live identity                           | same widget, titled **`◷ ARCHIVED · read-only`**, with a ribbon: `ended Oct 8 12:06 · ran 1h00 · dismissed 14:20 · reply + 6 diffs retained`                                                                      |
| Decks           | Main · Files · Tools · FINAL (+ Record) | the same decks, hydrated from the bundle                                                                                                                                                                          |
| Jump panel      | relations                               | the same, over the archived agent's relations                                                                                                                                                                     |
| Footer          | live conditional keys                   | Archive conditional keys (`⏎ restore…`, marks, …), following `src/sase/ace/CLAUDE.md` footer rules                                                                                                                |

The pulse shows `⌂ Inbox · 147 [R10 W35 U1] · 1 needs you`. It reuses
`format_agent_count_chip` and the existing attention count, and repaints on every inbox
refresh. The _needs you_ segment uses the attention color and is omitted at zero.
`J`/`K` move between the pulse and the list. `⏎`, `l`, or a click on the pulse returns
to the Inbox, the same as `,a`.

### Archive rows tell the truth

```text
  TIME   OUTCOME          NAME                         MODEL   RUNTIME  CHIPS
  19:52  ✗ FAILED         toobig-7e ×7                 muse    —
  19:18  ○ WAS RUNNING    0yn                          opus    —                 (dim italic time: start only)
  12:06  √ DONE           bob-cli-5p.4                 muse    1h00
  01:38  √ DONE           sase-1h7.land                opus    31h18    from Oct 6
  00:51  √ TALE DONE      sase-1hz.3                   opus    0h42     ⊘ can't restore
```

- **Time is last activity.** It is `stop_time` when recorded, else the bundle's real
  `dismissed_at`, else `start_time`.
  - A start-only time renders in dim italics, and the identity header then says
    `started 19:18 · end not recorded`. Never infer a time from file mtimes.
  - Rows sort newest last-activity first.
  - A run that started on an earlier day gets a `from Oct 6` chip.
- **Outcome is past tense.** The row keeps the stored status word; the glyph carries the
  class.

  | Class (`outcome:`) | Stored status                                           | Row              | Style                                             |
  | ------------------ | ------------------------------------------------------- | ---------------- | ------------------------------------------------- |
  | `done`             | DONE, TALE DONE, EPIC CREATED, TESTED, …                | `√ <status>`     | dim green                                         |
  | `failed`           | FAILED, PLAN FAILED, EPIC TIMED OUT, LAUNCH REJECTED, … | `✗ <status>`     | dim red                                           |
  | `interrupted`      | any non-terminal status on an archived row              | `○ WAS <status>` | the existing `STOPPED` violet, never a live color |

  The class comes from sase-core's existing agent status classification; do not add a
  parallel string table.

- **One row per presentation root.** Archived members of one clan generation, or of one
  session when there's no clan, collapse to a container row, `name ×N`.
  - The container's time is its newest member's last activity.
  - Its outcome is the worst member outcome (failed > interrupted > done).
  - `l`/`h` expand and collapse it, listing the members newest first.
  - Under a filter, `×N` counts only the matching members, and only those are listed.
  - Workflow children never appear as rows; an exact reference reveals their owner.
- **Chips mark exceptions only:** `⊘ can't restore` (not `durably_revivable`, or no
  readable bundle), `from <day>`, and `⌂ restored` (see
  [archive-actions](#restore-fork-and-copy-from-the-archive)). A "restorable" chip on
  every row would carry no information.
- **Grouping (`o` in the Archive):** day (the default), project, outcome, or model. The
  choice persists across sessions, separately from the Inbox's grouping.
- **Day banners** form one flat level: `Today · Thu Oct 8`, `Yesterday · Wed Oct 7`, one
  banner per remaining day of the last week (`Tue Oct 6`), one per week for the rest of
  the last two months (`Sep 21-27`), and one per month after that (`August 2026`).
  - Reuse the Inbox BY_DATE label helpers (`day_subgroup_label`, `week_subgroup_label`
    in `models/agent_groups/`) so the two views read alike.
  - Each banner shows its count from one core group summary.
  - Today and Yesterday start open and older banners start collapsed. When a query
    matches 50 runs or fewer, every banner opens.
  - A collapsed banner is a nav item, never a node.
- **Paging inside a banner.** An open banner loads 100 rows at a time and ends in
  `⋮ 129 more · l`. `l` or `⏎` on that row loads the next page. There is no global "load
  more" key, because `Ctrl+J`/`Ctrl+K` are deck-card keys on this tab.
- **Width.** Columns follow the inbox's responsive rules. Names truncate in the middle,
  and MODEL then RUNTIME drop first. No new compact mode is added in this epic.

### Reading archived runs

Selection moves the highlight immediately. The decks hydrate through the existing 150 ms
`DetailPanelDebouncer` by running `load_bundle_file` → `Agent.from_bundle_dict`
off-thread, and the selection is re-checked after every await. A new selection shows a
placeholder under the **new** identity header; it never shows the previous agent's text.

| Deck / card          | Archived row                                                                         |
| -------------------- | ------------------------------------------------------------------------------------ |
| Main ▸ Context       | the full prompt from the retained artifacts, never cut at 4,000 chars                |
| Main ▸ Reply         | the chat at `response_path`; a partial chat is marked _(partial)_                    |
| Main (container row) | the existing Summary card over the members' light rows, with no per-member hydration |
| Files                | `diff_path` and attached files, else _not retained_                                  |
| Tools                | `tool_calls.jsonl` if retained                                                       |
| FINAL                | the finalizer record carried in the bundle                                           |
| Record               | always present; see [record-deck](#record-deck-for-every-agent)                      |

- **Static, not live.** An archived `Agent` must never start log tailing, spinners, live
  timers, or workspace refreshes. That holds even when its stored status says `RUNNING`.
  Guard the Files deck's live `update_display` path the way `fleet_origin_alias` is
  guarded.
- **Missing content is a titled card, never a blank deck.** For example: _Reply not
  retained — the chat file was removed by retention_, or _Archive entry unreadable —
  Record only_.
- Reading never clears an unread flag, never touches the inbox's unread or attention
  sets, and never writes to the bundle, the index, or the artifact dir.

### Acting on archived runs

- **`⏎` opens the existing `AgentActionChooserModal`, with an ARCHIVE section first:**
  - `r` **Restore to inbox** — "shows it again · starts no new run". This is the
    default. If the row can't be restored, the default falls to _Open chat_.
  - `s` **Restore and show in inbox**.
  - `F` **Fork into a new agent**, offered only when the existing fork path accepts the
    hydrated bundle `Agent`.
  - READ section: `e` Open chat in `$EDITOR` (read-only), `y` Copy `@agent` reference,
    `%` More copy targets.
- **After a restore you stay in the Archive.**
  - The row stays in place, dimmed, with a `⌂ restored` chip. It leaves the list only
    when you change the query or leave the Archive, so the cursor never jumps.
  - The pulse counts the agent immediately.
  - Toast: `bob-cli-5p.4 is back in the inbox · no new run started`.
- **Restore and show** switches to the Inbox once the revive refresh has landed, and
  reveals the agent. If the parked inbox filter would hide it, the same filtered-target
  rule as link follow applies.
- **Containers and marks.**
  - On a container row, the action reads _Restore clan research.41 (6 members)_ and
    restores the members that can be restored.
  - `m`/`u` mark and clear Archive rows; these marks are separate from inbox marks. With
    marks, the default becomes _Restore N marked_.
  - The result toast reports restored, skipped, and failed counts with reasons, and
    capabilities are re-checked just before submitting.
- **Live-only keys explain themselves.** In the Archive, `x X s w W A R n N`, tab and
  tribe moves, and `[`/`]` act on nothing. Each raises one toast instead, for example
  `x acts on inbox agents — bob-cli-5p.4 is archived · ⏎ restore` or
  `Agent tabs live in the Inbox · ,a`.

### Getting in and out

```text
 Inbox ──,a──────────────────────▶ Archive (parked state, or fresh on first use)
 Inbox ──⏎ on shelf / bridge─────▶ Archive with  in:archive <inbox filter>
 Inbox ──commit "in:archive …"───▶ Archive (the inbox's prior query is parked)
 Archive ──,a / ⏎ on pulse───────▶ Inbox exactly as left (filter, selection, folds, tab, deck)
 Archive ──commit without in:────▶ Inbox with that query committed (old one goes to ^)
```

- **`in:` is a host-owned token**, extracted before the dialect parse just as `limit:`
  is (`src/sase/ace/query/limit_token.py`).
  - Values are `inbox` and `archive`, case-insensitive.
  - At most one `in:` term is allowed. It must be top-level: never negated, never inside
    `OR`, `NOT`, or parentheses.
  - It is rendered as an amber chip. The chip and the view can never disagree.
- **No implicit view switching.** An Archive-only field (`outcome`, `restorable`,
  `linked`, `relation`, `artifact`, `clan_tribe`) typed in the Inbox fails validation
  with a scope hint: `restorable: is an Archive field · add in:archive or press ,a`. An
  Inbox-only field (`unread`, `pinned`, `needs`, `source`, `cl`, `machine`, `text`)
  typed in the Archive fails the same way.
- **Parking.** Each view parks its own state on every switch: committed query, selection
  by stable identity, folds or expanded banners and loaded pages, scroll, and each deck
  panel's deck and card. The deck **layout** (splits) is shared.
  - The Inbox's widgets stay mounted and keep refreshing while hidden, so coming back
    costs nothing.
  - The persisted last query (`ace_agents_last_query.json`) is always the Inbox's query.
    The Archive's query lives only in `^` history.
- **History and slots** share the `agents-live` namespace, with `in:archive` stored in
  the query text. A saved slot like `#3 in:archive linked:true` therefore works from
  either view and switches to the Archive. A `,a` toggle records no history entry.

> [!decision] toggle_filter = carry `,a` from the Inbox opens the Archive with
> `in:archive <committed inbox filter>` when the Inbox has a committed filter and every
> term in it is Archive-answerable. With no Inbox filter, it restores the parked
> Archive. If the filter has Inbox-only terms, it restores the parked Archive and toasts
> which terms blocked the carry. `,a` from the Archive still restores the parked Inbox
> exactly. The shelf's and bridge's `⏎ search` stay as explicit aliases.

### Keyboard map

| Key                                             | Inbox                                                               | Archive                                                   |
| ----------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------- |
| `,a`                                            | open the Archive                                                    | back to the Inbox, exactly as left                        |
| `,a` on any other tab                           | open Agents ▸ Archive                                               | —                                                         |
| `⏎`                                             | act on agent (as today)                                             | chooser with the ARCHIVE section                          |
| `j k g G h l`                                   | as today                                                            | rows, banners, containers, `⋮ more` rows                  |
| `J K`                                           | panels (+ shelf)                                                    | pulse ⇄ list                                              |
| `o`                                             | grouping picker                                                     | Archive picker: day · project · outcome · model           |
| `/ ^ #N`                                        | query, history, slots                                               | the same, with the `in:archive` chip                      |
| `p` then `m f t n r`                            | deck picker (+ `r` Record)                                          | the same                                                  |
| `y`                                             | copy `@agent:` reference (new)                                      | the same                                                  |
| `%`                                             | copy palette (+ `!` handoff, `l` link, `j` json, `d` artifacts dir) | the same                                                  |
| `e` / `F` / `m` / `u`                           | as today                                                            | open chat (read-only) / fork / mark / clear Archive marks |
| `"`                                             | Node Finder (+ _In Archive_ section)                                | the same                                                  |
| `x X s w W A R n N`, `[ ]`, tab and tribe moves | live actions                                                        | explanatory toast                                         |
| `Ctrl+J` / `Ctrl+K`                             | next / previous card                                                | the same                                                  |

Every new or changed binding is added to `src/sase/default_config.yml`, the keymap
registries, the `?` help modal (with an **Archive** box on the Agents tab), and the
footer metadata.

### States and copy

| Situation                        | What you see                                                                                                   |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| First `,a`, corpus warm          | The list paints from the cached corpus; no spinner.                                                            |
| First `,a`, corpus cold          | The chrome paints at once, then a `◷ loading archive…` row; rows land off-thread.                              |
| Index rebuilding (v3 migration)  | `rebuilding index · 4,210 so far` in the info row. Indexed rows stay usable, newest first.                     |
| No index, or an unreadable index | One card: _Archive index unavailable — sase agent archive rebuild-index_. The Inbox is unaffected.             |
| Empty archive                    | `◷ Archive is empty — x moves finished agents here`. The Inbox shows no shelf.                                 |
| Query matches nothing            | `0 archived runs match · ^ previous filter · ,a inbox`, plus `N in the inbox` when the inbox count is non-zero |
| Bundle unreadable                | The row stays. The decks show one card: _Archive entry unreadable — Record only_.                              |
| Chat or diff pruned              | That card says _not retained (retention)_, and the other cards render.                                         |

### Visual language

- **Amber `#d7af5f` means Archive.** It is used for the `◷ ARCHIVE` chip, the
  `in:archive` chip, banner rules, the `◷ ARCHIVED` identity title, the shelf glyph, and
  the `◷` tab marker.
- Live yellow (`RUNNING`, unread) and the deck accents stay exactly as they are, so a
  glance tells you which view you're in.
- Every state also carries a glyph and a text label, so color is never the only cue.
- The Record deck gets its own accent, slate `#87AFAF`, so amber stays the Archive's
  signal.

## Architecture

```text
 dismiss → bundle JSON + summary row (Python writer, index v3)
                     │
 sase-core archive corpus (built off-thread, cached by index + link-facet signature)
   derive: outcome · last_activity (+ basis) · runtime · container · restorable
   evaluate: agents-archive profile (Python-owned schema, Rust query engine)
   serve:   group summaries (day / project / outcome / model) · row pages · exact lookup
                     │
   ┌─────────────────┼──────────────────────────┬─────────────────────────────┐
 Archive list    shelf / bridge counts    link follow / Node Finder     sase agent search in:archive
                     │
 selected row ─150 ms debounce─▶ load_bundle_file → Agent (read-only) ─▶ decks
```

- **Rust-core boundary.** Archive semantics live in sase-core: outcome mapping,
  last-activity derivation, container grouping, ordering, filter evaluation, group
  counts, and lookup. That is because `sase agent search 'in:archive Q'` must return
  exactly what the TUI lists. Presentation stays in Python: banners, labels, chips,
  widgets, and parking.
- **Why an in-memory corpus, not SQL windows.**
  - 11k top-level rows is small. One indexed read plus a compiled corpus answers the
    whole query language with no pushdown/fallback matrix.
  - Scrolling, banner expansion, and counts all come from the same cached handle.
  - The handle rebuilds only when the index signature or the link-facet signature
    changes, and every rebuild is off-thread and last-request-wins.
  - No new SQL time index is needed.
- **Link facets.** `linked:`, `relation:`, and `artifact:` reuse the existing
  `build_agent_catalog_link_facets` over `load_artifact_links_snapshot`. That runs
  off-thread, cached by aggregate signature, and its result is passed into the corpus
  build. The CLI uses the same helper, so parity holds. Porting it into core is
  [later work](#non-goals-and-later-work).
- **Detail ownership.** While the Archive is active, only the Archive selection may
  drive `AgentDetail`. Put the gate at the single render entry
  (`actions/agents/_display_detail_render.py`) so inbox refresh, finalize, and
  incremental paths cannot overwrite an archived agent's decks. Switching back
  re-renders the inbox selection.
- **Selection accessor.** Introduce one typed Agents-tab selection accessor (view,
  agent, read-only flag).
  - Read actions (decks, copy, open chat, fork) use the archived `Agent`.
  - Live-only actions see "no live selection" and raise the explanatory toast.
  - Audit every Agents-tab caller of the current selected-agent helpers.
- **Isolation (non-negotiable).** Archive rows never feed unread, attention, `load:`,
  runner slots, prospective clans, auto-dismiss, bare `%wait`, bulk `x`/`X`/`s`,
  agent-tab counts, or inbox marks.
- **Perf rules** (`tui_perf.md`), with the rule numbers:
  - no sync I/O on handlers or the pump (1, 2);
  - re-capture the selection after awaits (4);
  - debounce detail, never the highlight (7);
  - render paths never stat (8);
  - no archive work before the inbox's first complete load, and the corpus prewarms only
    once the UI is idle (9);
  - idle ticks compare a stat-only index signature (14);
  - honor `NavigationGate` (13);
  - guard programmatic highlight echoes in the new list widget (12).
- **`agents_unified_query`.** The Archive always uses the unified `FilterBar` path with
  the `agents-archive` profile. With that sunset flag Off, the Inbox keeps its legacy
  editor, and the Archive is reachable only through `,a` and the other entry points.
  Test both states.

### Flags

- **`agents_archive_view` (beta, epic scaffolding).** archive-view creates it with
  `sase flag new agents_archive_view -k beta --when-enabled … --when-disabled … --remove-when …`.
  - On: `,a`, the `in:` token on the Agents bar, the Archive palette entries, and every
    later phase's arrival, bridge, and chooser behavior.
  - Off: everything behaves as on master today.
  - Every phase tests both states. retire-pane deletes the Off branches and closes the
    flag bead, as `sase_flags.md` requires before the epic lands.
- **`artifacts_agent_pane_retired` (sunset).** retire-pane creates it with
  `sase flag new … -k sunset`.
  - On (the default): Artifacts has no Agent sub-tab.
  - Off: the pane returns as sub-tab `1`, unchanged.
  - The pane code is deleted later, when this flag is removed through its own flag bead,
    after the usual removal evidence. This epic doesn't delete it.

## Departures from the research

| Research proposal                                    | This plan                                                                                    | Why                                                                                                                                                                    |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| History ⊇ Inbox: live runs listed with `⌂ inbox`     | **Partition**: the Archive is dismissed runs only. The pulse and shelf cross-link the views. | One rule ("dismiss archives, restore un-archives"); no dedup or liveness overlay; no live rows mixed into a read-only list; matches the existing `agent_archive` store |
| Name "History"                                       | **Archive** (see `view_name`)                                                                | Exact under the partition; `,y` "full history" already means inbox reloads                                                                                             |
| `,a` carries the inbox filter                        | **Pure toggle**; the shelf and bridge carry (see `toggle_filter`)                            | The return gesture is predictable both ways; carrying stays an explicit `⏎`                                                                                            |
| Implicit widening on Archive-only fields             | **A scope-hint validation error**                                                            | Explicit beats magic, and `,a` is one keystroke                                                                                                                        |
| SQL windows plus a new time index                    | **A cached in-memory core corpus**                                                           | Full query language, no pushdown matrix, one handle for every consumer                                                                                                 |
| Source chips (This machine / Published / Combined)   | **None in v1**; the info row says `this machine`                                             | One source; dimmed future chips are noise                                                                                                                              |
| Shelf always visible                                 | **Only once the archive is non-empty**                                                       | No dead chrome on fresh installs; existing goldens stay stable                                                                                                         |
| One converged Agents schema                          | **A separate `agents-archive` profile** built from `_agents_shared.py`                       | Keeps the inbox hot path and its pushdown coverage untouched                                                                                                           |
| `g` restore-and-show in the toast; amber Record deck | **`s`, inside the chooser only; slate Record accent**                                        | `g` is go-to-top; amber is reserved for Archive chrome                                                                                                                 |
| Compact layout below 110 columns                     | **Not in this epic**                                                                         | The inbox's responsive rules apply; measure first                                                                                                                      |

## Archive index v3 and honest timestamps

The goal is to give the core everything it needs from the summary index, and to stop the
index from conflating "ended" with "dismissed".

- In `src/sase/ace/dismissed_bundle_index/`, bump `SCHEMA_VERSION` from 2 to 3. Add the
  columns the Archive groups and filters on that v2 lacks: at least `agent_session`,
  `agent_session_role`, `agent_clan`, `agent_clan_generation`, `agent_tab`, `tribe`, and
  `clan_tribe`, read from bundle fields that already exist.
- **Record `dismissed_at`.** Every newly saved bundle carries an offset-aware UTC
  ISO-8601 `dismissed_at`, written in `_prepare_archive_bundle`
  (`src/sase/ace/dismissed_agents_bundles.py`).
  - The summary stores `dismissed_at` only when the bundle carries one; drop the
    `stop_time` fallback. Audit the current readers of `summary.dismissed_at` and move
    any that relied on the fallback to an explicit `stop_time` read.
  - Synced bundles that already carry a real `dismissed_at` keep it.
- **Migration without blocking.**
  - The version bump drops the tables through the existing `_ensure_schema` path. Verify
    that the refill (the verify-then-rebuild path in
    `src/sase/core/agent_artifact_index_lifecycle_projection.py` and
    `sase agent archive rebuild-index`) runs off the TUI startup and first-paint path.
  - Rebuild **newest shard first**, so today's archive is queryable within seconds.
  - Expose a cheap progress probe (rebuilding flag plus rows indexed so far) that the
    Archive's info row can read without parsing bundles.
  - Measure the rebuild on a synthetic 47k-bundle fixture and record the timing in the
    phase notes.
- **Catalog schema check.** Make `src/sase/agents/catalog/_sources.py` and
  `src/sase/history/chat_catalog_provenance/artifacts.py` accept
  `SUPPORTED_AGENT_ARTIFACT_INDEX_SCHEMA_VERSIONS`, re-exported from
  `sase.core.agent_scan_wire`, instead of an exact match. Make degraded enrichment
  visible (a `sase agent search` stderr notice) instead of a silent `{}`.
- **Tests:**
  - v2 → v3 migration and rebuild;
  - newest-first refill order;
  - `dismissed_at` recorded on TUI dismiss, kill, and mark paths
    (`_dismiss_persistence`, `_kill_persistence`, `_marking`, `_dismiss_memory`);
  - no fallback to `stop_time`;
  - the progress probe;
  - the catalog accepts every supported version and reports degradation.

## Core archive corpus, in scope token, and CLI parity

The goal is one core-owned answer to "which archived runs match this query, in what
order, grouped how", shared by the TUI and the CLI. This phase is `large`; plan it
before implementing. Open the linked repo with `sase repo open sase-core` and follow its
`AGENTS.md` binding recipe. Move `sase-core-revision.txt` past the core commit (see
`docs/rust_backend.md`).

- **Corpus (sase-core `agent_archive`).** Build a compiled corpus handle from the v3
  index.
  - It holds the top-level dismissed rows (effective visibility `hidden`, joined with
    `archive_visibility_projection`) and a name map for workflow children, so lookups
    can return the child's owner.
  - Stable row identity is the archive key
    `(source_username, source_machine, source_run_id)`.
  - Accept optional link facets (keyed by agent name) as a build input.
  - Report index status (`missing`, `rebuilding` with a count, `ok`) rather than
    triggering a rebuild.
- **Derived fields (core):**
  - `outcome` (`done` / `failed` / `interrupted`), from the existing core status
    classification;
  - `last_activity_at` plus `time_basis` (`ended` / `dismissed` / `started`). Parse both
    the naive-local and the offset-aware timestamp forms;
  - `runtime_seconds`;
  - the container key and label (clan generation, else session, else self) with member
    counts and the worst outcome;
  - `restorable` (`durably_revivable` and the bundle present).
- **Operations (bindings plus Python facade in
  `src/sase/core/agent_archive_facade.py`):**
  - `summary(query, group_by)` → total, the groups (key, count, newest activity), and
    completeness. `group_by` is local day, project, outcome, or model; Python turns days
    into banners.
  - `rows(query, group_by, group_key, offset, limit)` → light rows: container rows,
    member rows on expand.
  - `lookup(name)` → exact match on the short name, canonical name, or global name,
    including children (which resolve to their owner).
  - `count(query)` → for the shelf, the bridge, and Node Finder.
  - Unknown fields or values return typed errors, never panics.
- **Python cache.** Key the corpus by (index signature, link-facet signature, profile
  digest). Build it off-thread, last-request-wins. One shared accessor serves the TUI
  and the CLI. Never build it at import time or on the event loop.
- **Profile `agents-archive`.** Add
  `src/sase/ace/query_profile/profiles/_agents_archive.py` from the `_agents_shared.py`
  builders, and register it in `pane_registry.py`.
  - Shared fields:
    `name session clan project role workflow model provider kind status since until after before min max attempt retry`.
  - Archive fields: `outcome`, `restorable`, `tab` (the bundle's `agent_tab`), `tribe`
    (the user tribe, as on the Agents tab), `clan_tribe`, and
    `linked relation artifact`.
  - Add profile golden cases to
    `tests/ace/tui/artifacts_contract/goldens/query/profile_cases.json`.
- **`in:` token.** Add `src/sase/ace/query/scope_token.py`, modeled on `limit_token.py`.
  - It handles extraction, validation (single, top-level, known value), completion
    items, and the scope-hint error texts, which need both profiles' field sets.
  - Agents-tab and CLI paths only. Other panes keep rejecting `in:` as an unknown term.
- **CLI.** `sase agent search 'in:archive Q'` evaluates through the corpus.
  - Output is the TUI's order and fields: a colored table (time, outcome, name, model,
    runtime) by default, and derived fields in `-j`.
  - `in:inbox` maps to the catalog's non-dismissed rows. No `in:` keeps today's
    catalog-wide behavior, which is documented.
  - Follow `cli_rules.md` for help text, and update `docs/cli.md` and
    `docs/query_language.md`.
- **Tests and benches:**
  - Core unit tests for the derivations: every status class, all three time bases,
    container aggregation, and mixed timestamp formats.
  - Binding round-trip tests.
  - A **parity oracle**: over one fixture, CLI `-j` order equals facade `rows()` order
    for several queries.
  - A bench (`tests/perf/`) over 11k top-level plus 37k child rows, reporting corpus
    build p95 and query + first-window p95. Targets: build < 300 ms, query + window < 30
    ms.

## Archive view on the Agents tab

This phase delivers the view itself, read-only. It is `large`; plan it before
implementing. Create the `agents_archive_view` beta flag first (see [Flags](#flags)).

- **View state.** An `ArchiveViewState` holds the committed query, grouping, expanded
  banners, loaded pages, selection identity, scroll, per-panel deck and card, and marks.
  Add the per-view parking and restore described in
  [Getting in and out](#getting-in-and-out), plus the `,a` leader action
  (`toggle_agents_archive: "a"` in `leader_mode`) and the palette entries _Agents: Open
  Archive_ and _Agents: Back to Inbox_.
- **Widgets.** Add an Archive column (pulse plus list) as a sibling of
  `#agent-list-container` in `_app_layout.py`. Mount it lazily on the first `,a` and
  toggle it with `display`. While the Archive is active, hide `#agents-header`.
  - The list is a lightweight OptionList-style widget over light rows, with banners,
    containers, and `⋮ more` rows.
  - Guard programmatic highlight echoes (perf rule 12).
  - Keep modules small (the `toobig` gate applies), for example a
    `src/sase/ace/tui/widgets/agents_archive/` package.
- **Info row, filter, and grouping.**
  - Build the Archive info-row text in `AgentInfoPanel`.
  - The FilterBar picks its profile from the edited text's `in:` value on each
    completion request.
  - On commit, `in:archive` switches to the Archive and its absence switches to the
    Inbox.
  - Add the Archive grouping picker on `o`, with persistence.
- **Reading.** Implement the deck hydration, the `◷ ARCHIVED · read-only` identity
  header and ribbon, the static guards, and the _not retained_ cards from
  [Reading archived runs](#reading-archived-runs).
- **Integration seams.** Add the detail-ownership gate, the typed selection accessor
  with its caller audit, the live-only key toasts, `[`/`]` in the Archive, the isolation
  rules, a marker API on `widgets/tab_bar.py` for `Agents ◷`, the footer conditional
  keys, and the help modal's Archive box.
- **Refresh.**
  - An idle tick compares the stat-only index signature. On drift, the corpus rebuilds
    off-thread, then the current pages re-query with the selection kept by identity.
    Defer the re-query during `NavigationGate` windows.
  - The pulse repaints from inbox refreshes with no extra loads.
  - Prewarm the corpus once the inbox's first complete load has applied and the UI is
    idle.
- **Tests:**
  - both flag states;
  - parking round-trips (query, selection, folds, tab, deck);
  - startup always opens the Inbox, and the persisted query is always the Inbox's;
  - selecting an archived row performs no writes and no revive, and leaves the index
    signature unchanged;
  - Archive rows never change inbox signals;
  - detail ownership under inbox refreshes;
  - scope-hint errors;
  - no corpus build before the first complete load.
- **Benches and goldens.**
  - `bench_tui_jk.py` j/k p95 < 16 ms in the Archive.
  - `,a` → first rows painted < 100 ms with a warm corpus at athena scale.
  - PNG goldens (flag on): Archive by day with open and collapsed banners, a container
    row, `○ WAS RUNNING`, an italic start-only time, the archived identity header with a
    Reply card, _not retained_ cards, the empty archive, an index rebuilding, and
    `Agents ◷` on the tab bar.

## Record deck for every agent

The goal is one home for the facts the pane's Details panel and relation rail held, for
inbox and archived agents alike.

- Add `DeckId.RECORD` to `widgets/decks/spec.py` (`name="RECORD"`, glyph `≡`, picker key
  `r`, accent `#87AFAF`, blurb "Lifecycle, provenance, relations, links"). Wire it
  through `DeckPanel` compose, titles, the picker, availability (always at least one
  card), `Ctrl+N`/`Ctrl+P` cycling, and deck persistence. Persisted states without
  Record must still round-trip.
- **Cards:**
  - **Lifecycle:** the `@agent:` reference; where the agent lives (Inbox / Archive);
    status and outcome; started, ended, and dismissed times with the time basis; revived
    ×N; and capabilities (viewable, restorable, restartable, plus missing requirements).
  - **Provenance:** project, workspace, model, provider and effort, agent tab, tribe,
    clan tribe, owner (user@machine), source run id, chat path, artifacts dir, and the
    bundle path when archived.
  - **Relations:** session, clan, parent, workflow parent, and retry chain (`retry_of`,
    `retried_as`), as copyable references. Interactive jumps stay in the Jump panel.
  - **Links:** incoming and outgoing artifact links grouped by relation, from the cached
    `load_artifact_links_snapshot`, loaded off-thread.
- Use the archive-projection derivation binding for the outcome and time basis of
  archived agents. Use existing `Agent` fields for inbox agents.
- This phase may run in parallel with archive-view. Both touch the deck refresh seam
  (`_agent_detail_deck_refresh.py`), so rebase carefully.
- **Tests:** card content for an inbox agent, an archived agent, a container, a thin
  agent with no artifacts dir, and an agent with links. Also deck persistence, the
  picker key, and PNG goldens for the Record deck (inbox agent and archived agent).
- If `glossary_strand_updates` is accepted, update `glossary:agent-data-deck` (add
  Record and `r`) through `/sase_memory_write`, then `sase memory init`.

## Restore, fork, and copy from the Archive

- In `modals/agent_action_chooser_modal.py`:
  - widen the section type with an `archive` section and a `read` section;
  - add target builders in `_agent_enter_builders.py`;
  - set the ordering in `_agent_enter_resolver.py`;
  - map and run them in `_agent_enter_action.py`.

  On an Archive row the chooser always opens, even with one choice, so the default is
  visible.

- **Restore** reuses the existing revive execution (`_do_revive_agent` /
  `_do_revive_agents` in `actions/agents/_revive_execution.py`), the same path `!R`
  uses. Then:
  - mark the row `⌂ restored` in place;
  - update the pulse;
  - show the toast.

  **Restore and show** waits for the revive delta refresh, then switches views and
  reveals the agent.

- **Containers and marks.** Implement container and marked bulk restore with per-member
  skip reasons, as described in [Acting on archived runs](#acting-on-archived-runs).
- **Fork** is offered when the existing fork path accepts the hydrated bundle `Agent` (a
  retained chat). **Open chat** is read-only.
- **`y` and `%`.**
  - Add an Agents-tab `agents_copy_reference` on `y` that copies the `@agent:` reference
    in both views, with an availability split from the global
    `artifacts_copy_reference`.
  - Add the `%` targets `!` handoff, `l` link, `j` json (the bundle JSON when archived),
    and `d` artifacts dir to the `agents` copy group (`default_config.yml` `copy_mode`,
    `_palette_registry.py`). Keep `%p` meaning prompt.
- **Tests:**
  - chooser composition and defaults (restorable, not restorable, container, marks);
  - restore stays in the Archive with the row kept until the query changes;
  - restore-and-show reveal, including the inbox-filter-hides-it case;
  - skip reasons;
  - `y` on both views and its no-regression on Artifacts;
  - `%` targets;
  - both flag states.
- **Goldens:** the chooser with and without marks, and the restored row with its toast.

## Every agent route lands on the Agents tab

With the flag on, every route below lands on the Agents tab. With it off, every route
keeps today's behavior.

- **`agent:` link follow** (`actions/link_follow.py`, `_link_follow_targets.py`,
  `_link_follow_toast.py`):
  1. Loaded and visible: reveal, as today.
  2. Loaded but hidden by the inbox filter: stay in the Inbox, commit
     `name:"<canonical>"` (the previous query goes to `^`), reveal, and toast
     `The inbox filter hid it — showing name:… · ^ restores your filter`.
  3. Not loaded: run an exact `lookup` off-thread. On a hit, open the Archive with
     `in:archive name:"<canonical>"`, select the row (a workflow child resolves to its
     owner, with a note), and toast
     `Not in your inbox — dismissed Aug 13 (56 days ago) · ⏎ restore · ctrl+o back`.
  4. Not found anywhere: toast `agent:… has no record on this machine` and keep the
     current context.

  Each arrival records exactly one `Ctrl+O` trail hop. Delete the `agents_tab_fallback`
  flag plumbing and toast line under the flag. Point `target_for_ref_kind("agent")` at
  the Agents tab.

- **Selection survives clearing the filter.** After a link arrival, clearing the `name:`
  filter keeps the same run selected, now inside its day banner. That generalizes the
  archive-view rule of keeping the selection by identity.
- **`!R` modal** (`modals/saved_agent_group_revival_modal.py`). The last row becomes
  _Browse the Archive (⏎ restores)…_, which opens the Archive with its parked state.
  Delete the test-only `_show_dismissed_agents_for_custom_search` and fix
  `docs/ace.md`'s stale description of this flow.
- **Files `a`** (`actions/artifacts_files.py:267-299`). A dismissed agent opens in the
  Archive, read-only, instead of being revived first.
- **Jump-panel digits.** A dismissed target opens in the Archive with the target
  selected, instead of reviving it. A live target behaves as today.
- **`,a` from any other tab** opens Agents ▸ Archive.
- **Tests:** the link-follow ladder for all four cases with each flag state, trail hops,
  `!R` routing, Files `a`, Jump digits for live and dismissed targets, and `,a` from
  another tab. Update the `saved_agent_group_revival_*` goldens.

> [!decision] jump_opens_archive = no Jump-panel digits keep reviving dismissed targets
> as they do today. Drop that bullet, its tests, and the `agent-relation-jump-target`
> strand edit in retire-pane. Every other route still lands in the Archive.

## Inbox shelf, zero-result bridge, and Node Finder matches

- **Shelf.** A one-line nav section docked at the bottom of the Inbox node column. It
  renders only when the archive has at least one run.
  - No filter: `◷ Archive · 10,917 runs · 42 today`.
  - Committed filter: `◷ Archive · 7 match this filter · ⏎ search`.
  - When the inbox result is empty: the same line, amber-lit.

  `J`/`K` reach it. `⏎`, `l`, or a click opens the Archive: with no filter it is the
  same as `,a`, and with a filter it carries the filter (`in:archive <filter>`). Counts
  come from `count()` off-thread, on commit and on index-signature drift, never per
  keystroke. The shelf is never counted in tab totals, unread, attention, `load:`,
  marks, or bulk actions.

- **Zero-result bridge.** When an inbox filter matches nothing, the empty-state card
  (`_agent_tab_strip_model.py` / `_agent_detail_state.py`) replaces today's "0 matches
  elsewhere" dead end with:
  - the Archive count and its top 5 rows (time, outcome, name);
  - `⏎ search the Archive for this filter · ^ previous filter`.

  If the filter uses Inbox-only fields, the card says the Archive can't answer it. It
  never claims "no matches" before the count completes.

- **Dismiss toast.** The `x`, `X`, and `s` toasts end with `→ ◷ Archive · ,a browse`.
- **Node Finder** (`actions/agents/_node_finder_builder.py`,
  `modals/node_finder_modal.py`). Add an _In Archive_ section of up to 5 name matches
  under the live matches, evaluated off-thread from the warm corpus only (show
  `Archive warming…` otherwise). `⏎` opens the Archive with that run selected.
- **Tests:** shelf text for all three states; its absence when the archive is empty; nav
  order; isolation from inbox signals; bridge states; no counting per keystroke; Node
  Finder section with a warm and a cold corpus; both flag states.
- **Goldens:** the shelf (plain and lit), the bridge, and the Node Finder section.

> [!decision] toggle_filter = carry The shelf's `⏎` and the bridge's `⏎` stay as
> explicit carries. The shelf's hint then reads `⏎ / ,a search`, because `,a` also
> carries.

## Retire Artifacts ▸ Agent and make the Archive unconditional

- **Remove the beta flag.** Delete the `agents_archive_view` Off branches across every
  phase's code and make the On branches unconditional. Remove the registry entry and
  close its flag bead in this change.
- **Create the `artifacts_agent_pane_retired` sunset flag** with `sase flag new`. With
  it on:
  - `FIXED_ARTIFACTS_SUBTAB_ORDER` and the descriptors drop `agents`, so Stitch becomes
    `1`. The digits come from visual position.
  - `LEGACY_ARTIFACTS_SUBTABS["agents"]` maps to `stitches`, following the Chats
    precedent.
  - A one-time marker toast shows:
    `Agent history moved to the Agents tab · ,a opens the Archive`. Follow the
    `_agent_decks_notice.py` precedent.
  - An explicit `show_artifacts_agents` action, palette entry, or user keymap opens
    Agents ▸ Archive.
  - `sase artifact pane show agents` explains the move.
  - The help modal drops its Agent Pane section.

  With it off, the pane is exactly as it is today. Test both states.

- **Saved-query migration** (one-time, idempotent, with a marker).
  - `agents`-namespace saved slots and history entries get `in:archive` prepended and
    are appended to free `agents-live` slots and to history.
  - `limit:`, `state:dismissed`, and `dismissed:true` are dropped, and `tribe:` becomes
    `clan_tribe:`.
  - Existing slots are never overwritten. Unconvertible entries stay in the old
    namespace.
- **Docs:**
  - `docs/ace.md`: a new Archive section replaces Agent Pane; fix the digits at
    `:150-155` and the other references;
  - `docs/query_language.md`;
  - `docs/cli.md`;
  - `docs/configuration.md` (keymaps);
  - `docs/artifacts_pane_contract.md`;
  - `docs/artifacts_pane_visual_grammar.md`.
- **Goldens.** Regenerate every Artifacts PNG whose strip showed `⬡ Agent`, plus any
  Inbox golden that now shows the shelf or footer changes. Add goldens for the
  renumbered strip and the one-time toast. Run `just fix-tui-screenshots` through
  `/sase_monitor`, and inspect the report before finalizing.
- **Parity gate.** Before hiding the pane, confirm that restore, copy targets, relations
  and links, the `linked:` facet, saved queries, and link conformance all have
  Agents-tab tests.

> [!decision] glossary_archive_term
>
> Add the `glossary:agent-archive` strand through `/sase_memory_write`, then run
> `sase memory init`. It should define the Archive view (this machine's dismissed
> agents, one view of the Agents tab), the `in:archive` scope, Archive rows (not agent
> nodes), and _Restore to inbox_. Keep the `agent-archive` slug either way. If
> `view_name = history`, title it _History (agent archive)_ and define `in:history`.

> [!decision] glossary_strand_updates Through `/sase_memory_write`, then
> `sase memory init`:
>
> - `nav-section`: drop the Artifacts Agent pane; add the Archive list, the Inbox pulse,
>   and the shelf.
> - `nav-item`: drop "an agent" from the Artifacts entries; add Archive rows, container
>   rows, banners, and `⋮ more` rows.
> - `agent-relation-jump-target`: dismissed targets open in the Archive. Only edit this
>   when `jump_opens_archive` is accepted.
>
> The `agent-data-deck` edit happens in record-deck.

## Acceptance gates (whole epic)

- With no Archive interaction, Inbox first paint, idle ticks, and j/k are unchanged. The
  archive corpus is never built before the inbox's first complete load.
- In the Archive, j/k p95 < 16 ms. `,a` paints the chrome in one frame, and rows arrive
  in < 100 ms with a warm corpus at athena scale.
- A dismissed agent's prompt, reply, diffs, and tool calls can be read without restoring
  it, and selecting it writes nothing.
- Archive rows never affect unread, attention, `load:`, runner slots, clans, bulk
  actions, or inbox marks.
- An `agent:` target that is not in the Inbox is revealed in the Archive, without
  opening Artifacts and without creating local agent state.
- `sase agent search 'in:archive Q' -j` and the Archive list return the same ordered
  identities for the same Q.
- A missing, rebuilding, or unreadable index degrades visibly and never affects the
  Inbox.
- `just check` passes after every phase (`sase tool run check`, plus
  `sase tool run check` in sase-core for the core phase). PNG goldens are captured with
  targeted `just fix-tui-screenshots` selectors through `/sase_monitor`.
- New and changed keys are in `src/sase/default_config.yml`, the help modal, and the
  footer metadata.

## Non-goals and later work

- **Published sidecar history (`in:published`).** A later epic, once the publisher stops
  freezing runs at `active` and stops republishing imported runs under a second owner.
  The corpus is keyed so that a second source is additive.
- **Transcript full-text `text:` in the Archive.** It needs a populated
  `dismissed_bundle_search_fts` or a real full-text index.
- **Deleting the pane code.** That happens when `artifacts_agent_pane_retired` is
  removed through its flag bead.
- **Retiring the catalog `agents` profile and moving link-facet derivation into
  sase-core.** This happens once `sase agent search` defaults to scoped queries.
- **Converting revive to a durable proc.** File it only if restores measurably stall the
  UI.
- **A compact layout below about 110 columns.**
