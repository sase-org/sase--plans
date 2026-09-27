---
tier: epic
title: "Dynamic Agents sub-tabs: %tab, machine tabs, and the o/O layout ladder"
goal: "The Agents tab gains dynamic, launch-assigned sub-tabs. `%tab:<name>` places an
  agent's whole session, clan, or workflow on a named tab on every machine. Agents
  without a tab land on `main`, or on derived machine tabs (`⌨ local`, `⌨ <alias>`) when
  remotes are configured. The strip stays invisible until two tabs have agents, never
  hides something that needs you, and switches instantly. The `o`/`O` modal walks a
  Split → Merged → All tabs ladder. The whole feature is intuitive, reliable across
  machines and versions, and beautiful.

  "
phases:
  - id: card-block-keys
    title: Free the brackets and delete the dead Focus/Fleet state
    depends_on: []
    size: medium
    description:
      "card-block-keys: move card-block stepping from [/] to (/) across config, keymap
      types, availability, collision allowances, help, footer, rail hints, docs,
      goldens, and the card-block glossary strand; warn on legacy bracket overrides;
      delete the dead AgentsSubTab state."
  - id: core-tab-model
    title: sase-core agent tab model, directive contract, and typed units
    depends_on: []
    size: medium
    description:
      "core-tab-model: add sase_core agent_tab.rs (name canonicalization, reserved
      names, tab keys, effective-tab resolution, catalog ordering), the `tab` directive
      contract entry with a Tab value role and completion, and typed-unit parsing,
      validation, re-emission, and digest coverage, plus Python bindings."
  - id: core-tab-wires
    title: sase-core scan wire and fleet contract carry agent_tab
    depends_on:
      - core-tab-model
    size: medium
    description:
      "core-tab-wires: add agent_tab/agent_tab_source to the scan wire (schema 11) and
      agent_tab to the fleet owner and summary wires (contract 7) with root inheritance
      in fleet_catalog, gateway owner facts, stable revisions, validation, and a
      breaking-change marker."
  - id: tab-directive
    title: "%tab launch path, storage, query field, and completion"
    depends_on:
      - core-tab-wires
    size: large
    description:
      "tab-directive: bump the core pin; parse and validate %tab in Python; write
      agent_tab to meta, clan records, and session follow-ups with root-mismatch errors;
      preserve it across retry, revive, and fork; load it into the Agent model from
      meta, index, and fleet rows; add the tab: query field and %tab completion; update
      docs and the xprompts memory table."
  - id: tab-lineage-dispatch
    title: Lineage inheritance and dispatch preflight
    depends_on:
      - tab-directive
    size: medium
    description:
      "tab-lineage-dispatch: export SASE_AGENT_TAB from agent, gate, and monitor turns
      and insert an inherited %tab into agent-initiated launches (sase
      run/LaunchApproval, sase bead work, epic approval); add the %tab+%dispatch version
      preflight; update remote dispatch docs and the dispatch memory note."
  - id: tab-scope
    title: Tab index, active-tab scope, keys, and cross-tab navigation
    depends_on:
      - card-block-keys
      - tab-directive
    size: large
    description:
      "tab-scope: create the agent_tabs beta flag and the ace.agent_tabs config block;
      compute the per-root tab index and catalog; add the active-tab scope stage after
      the query; key panel index, folds, sticky panels, and selection memory by tab;
      persist the active tab; bind [/] to tab cycling; make every cross-tab jump switch
      tabs; scope bulk confirmations; add a minimal strip and perf metric."
  - id: tab-strip
    title: The beautiful tab strip
    depends_on:
      - tab-scope
    size: large
    description:
      "tab-strip: build AgentTabStrip in #agents-header with accent labels, the active
      pill, count and attention badges, tiers, an overflow window with a tab picker,
      tooltips, jump hints, arrival dots, the emptied-tab latch, three-cause empty
      states, and golden coverage."
  - id: layout-ladder
    title: The o/O layout ladder
    depends_on:
      - tab-strip
    size: medium
    description:
      "layout-ladder: replace the merged boolean with a Split/Merged/All-tabs level;
      turn the grouping modal's layout row into a segmented control with o (next) and O
      (previous); add titles, info-row chip, all-tabs strip state, row tab chips,
      tribe-roster tab chips, and anchor-preserving transitions."
  - id: machine-tabs
    title: Machine tabs
    depends_on:
      - tab-strip
    size: medium
    description:
      "machine-tabs: render machine tabs with the ⌨ glyph and health colors; adopt
      `local` vocabulary everywhere with machine:local; suppress redundant machine chips
      and banners on machine tabs; make Admin Center Enter select the machine tab; add
      off-tab tooltips and alias-rename and unenrolled-machine handling."
  - id: launch-view-ux
    title: Launch-from-view inheritance and launch UX
    depends_on:
      - tab-lineage-dispatch
      - machine-tabs
    size: medium
    description:
      "launch-view-ux: add the prompt-bar tab chip and remote-tab hint, the gb Launch
      Tab picker, %tab insertion on submit from a named tab, landing toasts with arrival
      marks, and the LaunchApproval tab field."
  - id: tab-moves
    title: Move agents between tabs
    depends_on:
      - tab-scope
    size: medium
    description:
      "tab-moves: add persist-directive agent_tab support (meta, prompt, clan record),
      the sase agent tab list/set/unset CLI, and a Tribe & Tab N modal with optimistic
      moves; disable moves on remote rows."
  - id: finish
    title: Unflag, document, measure, and record memory
    depends_on:
      - layout-ladder
      - machine-tabs
      - launch-view-ux
      - tab-moves
    size: medium
    description:
      "finish: delete the agent_tabs flag's Off branches and close its bead; finish the
      docs; add the tab-switch bench; do the full golden pass; add the agent-tab and
      machine-tab glossary strands and update the node-panel strand."
proposed_by: bbugyi200.athena.0t4
create_time: 2026-09-27 10:56:57
status: wip
---

- **PROMPT:**
  [prompts/202609/agents_dynamic_tabs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_dynamic_tabs.md)

# Plan: Dynamic Agents sub-tabs

## Context and sources

This epic implements the user's request for dynamic Agents sub-tabs. It adopts every
recommendation in `research:202609/agents_dynamic_tabs/agents_dynamic_tabs.md` (the user
agreed with all of them), including its answers to §11's open questions:

- View inheritance is on by default.
- `local` replaces `here` everywhere.
- Move-to-tab ships in this epic.
- Project-default tabs are a later follow-up.
- Named tabs sort alphabetically, with an optional configured `order`.

The earlier
`research:202609/agent_machine_tabs_and_cluster_semantics/agent_machine_tabs_and_cluster_semantics.md`
report is superseded on the points where it conflicts: no `All` tab, no Agent Cluster
term, and no always-visible empty machine tabs. Read both with `sase artifact read` when
a phase needs the reasoning behind a rule.

This plan refines the research in five places, verified against the current code:

1. **Tab keys are `default`, `machine(<installation_id>)`, and `named(<name>)`.** The
   research uses `Main` versus `Machine(Local)`. Here the default tab keeps one key
   whether it is labeled `main` or `⌨ local`, so enrolling a first remote relabels it
   without losing selection, folds, or memory.
2. **A tab switch re-scopes a cached, tab-independent query result.** The pipeline gets
   one new stage after the committed query. A switch never re-projects the roster.
3. **Fold and panel scope become `(tab_scope, merged)`.** `tab_scope` is the active key
   or `ALL`. The All-tabs level is simply "merged over ALL".
4. **`tab:` is a known-fallback query field, like `tribe`.** There is no new
   artifact-index column and no index migration. Agents carry a tab only when they are
   launched after this ships, so no backfill exists.
5. **Lineage uses `SASE_AGENT_TAB`, and gate and monitor turns carry it.** Epics then
   stay with their planner. `sase bead work` launches every wave of an epic in one
   batch, so the approve command inheriting the tab covers every phase agent.

## Design contract

Every phase implements against this contract. When code and contract disagree, stop and
record a `PROPOSED FOLLOW-UP:` note on your phase bead rather than silently diverging.

### Vocabulary

| Term             | Meaning                                                                                                                                                                                                                                            |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **agent tab**    | An exclusive, root-level, presentation-only placement of top-level sase agents on the Agents tab. The field is always named `agent_tab`, never a bare `tab`, because "tab" already means main TUI tabs, Artifacts sub-tabs, and notification tabs. |
| **default tab**  | Where agents with no stored tab appear. It is labeled `main`, or `⌨ local` in machine mode.                                                                                                                                                        |
| **machine tab**  | In machine mode, the derived default placement for agents a remote machine owns (`⌨ <alias>`). The default tab renders as the `⌨ local` machine tab.                                                                                               |
| **named tab**    | A tab authored with `%tab:<name>`. It wins on every machine.                                                                                                                                                                                       |
| **layout level** | `Split by tribe` → `Merged` → `All tabs`, walked by `o`/`O` in the grouping modal.                                                                                                                                                                 |

A tab never changes where an agent runs, its identity, clan, session, or tribe, or what
`%wait`, `%hold`, fork, or `%dispatch` can target. It only says where an agent is shown.

### The `%tab` directive

| Form                                             | Behavior                                                                                                                                     |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `%tab:<name>`, `%tab(<name>)`                    | Places this launch's presentation root on the named tab. The input is trimmed and lowercased, then must match `^[a-z0-9][a-z0-9_.-]{0,31}$`. |
| `%tab:main`                                      | The explicit default. It is stored as absent and opts out of view and lineage inheritance.                                                   |
| `%tab:local`, `%tab:all`                         | Error with guidance: "machine tabs are derived; omit `%tab`" and "use `o` in the grouping picker to see all tabs".                           |
| Empty, keyword args, or more than one positional | Error.                                                                                                                                       |
| Two `%tab` in one launch unit                    | Error, even when they agree (single-valued, like `%model`). Use fan-out `%{%tab:a \| %tab:b}` to place branches on different tabs.           |
| On a stand-alone `%proc` unit                    | Error in v1.                                                                                                                                 |
| With a `%clan` declarer                          | Sets the generation's `clan_tab` on the clan record. Joiners inherit it. An explicit joiner `%tab` that differs is an error.                 |
| With a session attach `%id(<x>, session=<p>)`    | The follow-up inherits the session root's tab. An explicit differing `%tab` errors, naming the root's tab and `sase agent tab set`.          |
| With `%dispatch:<alias>`                         | Allowed. The target stores the tab. See the preflight below.                                                                                 |
| Inside fenced or disabled regions                | Ignored, through the existing literal-zone pipeline.                                                                                         |
| Alias                                            | None. `%t` keeps raising the retired-tribe migration error.                                                                                  |

`%tab` is stripped before the model sees the prompt. Machine aliases are **not**
reserved: `%tab:apollo` is a named tab distinct from `⌨ apollo`, and the glyph tells
them apart.

### Inheritance

1. **Root rule.** Membership is decided by the presentation root:
   - a session container takes the tab of the session's first turn;
   - a clan container takes the generation's `clan_tab`;
   - a workflow takes its workflow root's tab.

   Session follow-ups, clan joiners, and workflow children copy the root's tab into
   their own meta at launch. Every display surface still resolves through the root, so a
   container never splits across tabs.

2. **View inheritance (R1).** It applies to a TUI prompt-bar launch submitted while
   Agents is the current main tab and a _named_ tab is active.
   - The launch gets `%tab:<active>`, shown beforehand as a prompt-bar chip.
   - It is inserted at submit unless the prompt already has any `%tab` (including
     `%tab:main`) or is a session attach.
   - It never applies from the default tab, a machine tab, or the All-tabs level.
   - It is controlled by `ace.agent_tabs.launch_from_view` (default `true`).
3. **Lineage inheritance (R2).**
   - An agent, gate, or monitor turn whose root has a named tab runs with
     `SASE_AGENT_TAB=<name>`.
   - Agent-initiated launches insert `%tab:<name>` into each launched prompt or segment,
     using the same skip rules. That covers `sase run` through LaunchApproval,
     `sase bead work`, and gate option commands such as EpicApproval.
   - Spawn-time scrubbing of `SASE_AGENT_*` is correct here, because each child
     re-derives the variable from its own resolved tab.
4. **Replay.** R1 and R2 write the tab into the prompt text, so LaunchApproval review,
   retry, fork, `%dispatch`, and `%repeat` all replay deterministically.
   - Retry and revive preserve `agent_tab`.
   - Fork prefills `%tab:<source>`, which the user can edit.
   - Dismissed bundles keep the field.

### Effective tab resolution (viewer-relative, pure, no I/O)

```text
effective_tab(root, viewer):
  root.agent_tab set                   -> named(root.agent_tab)
  not viewer.machine_mode              -> default        (label "main")
  root owned by this machine           -> default        (label "⌨ local")
  otherwise                            -> machine(owner installation id)
                                          (label "⌨ <this viewer's alias>")
```

- **Machine mode** comes from `ace.agent_tabs.machine_tabs`: `auto` (the default), `on`,
  or `off`.
  - `auto` is on iff this machine's config has at least one `dispatch.machines` record,
    quarantined records included.
  - It is computed from config through a `current_config_token()`-keyed cached accessor
    (the `tribe_display.py` `lru_cache` pattern). It is never computed from fleet
    refresh results, so labels never flip after startup.
- **Remote identity.** The key is the owner's installation id
  (`fleet_origin_installation_id`); the alias is only the label. Renaming an alias
  relabels the tab and keeps its key, selection, and memory.
- **Origins with no configured record** are labeled with `fleet_origin_alias`. If the
  installation id is unknown, a session-only key is used and selection is never
  persisted against it.
- **Provisional dispatch rows** use their target installation id, falling back to
  `dispatch.machines.<alias>.pinned_installation_id`. They then land on the same tab as
  the authoritative row, with no hop.
- **A remote alias literally named `local`** renders as `⌨ local·remote`, so it cannot
  be mistaken for this machine.

### Tab catalog

- **Existence.** A tab exists iff at least one _visible_ root resolves to it.
  - Visible means after dismiss and hide rules (honoring the `I` show-hidden toggle),
    before the committed query, and including `STARTING` roots.
  - Typing a query changes counts, never which tabs exist.
- **Order**, identical on every machine:
  1. the default tab;
  2. remote machine tabs in `load_dispatch_config().machines` order, then unconfigured
     origins by alias;
  3. a `┊` kind divider;
  4. named tabs by configured `order`, then natural case-insensitive name.
- **Visibility.** The strip shows iff the flag is on and the catalog has two or more
  tabs, or while the emptied-tab latch holds.
  - **Latch (R11):** if the active tab's last agent leaves, the tab stays selected,
    showing an empty state, until you navigate away.
  - With one tab, the Agents tab is pixel-identical to today.
- **Counts.**
  - The count is query-aware top-level sase agents, matching the tribe panel's `· N`.
  - Attention tokens `S` (stopped), `F` (failed), and `U` (unread) are also query-aware
    and use `format_agent_count_chip` colors.
  - Show `N+` whenever tribe panel titles would mark history incomplete.
- **Startup selection:**
  1. the persisted key (loaded off-thread), if that tab exists;
  2. otherwise the default tab, if it has agents;
  3. otherwise the tab holding the newest root that needs attention;
  4. otherwise the first tab.

  An active tab that disappears outside the latch (for example, an unenrolled machine)
  falls back to the default tab with a toast.

### Pipeline

```text
local ∪ fleet ∪ provisional dispatch rows   (post dismiss/hide = _agents_with_children)
  → tab index: per-root key + ordered catalog   (memo: roster identity, machine-mode token)
  → folds → committed query                     (tab-independent result, cached)
  → tab scope: active tab | ALL (All-tabs level) → _agents
  → tribe panels (Split) | one panel (Merged / All tabs) → grouping tree → rows
```

- **Tab switch** saves the current tab's memory, sets the active key, re-scopes the
  cached query result, invalidates the panel index, syncs panels, restores memory, and
  schedules persistence.
  - It does no disk, network, or roster re-projection work.
  - Panel widgets are reused by tribe key.
  - Re-read the active tab after every await (tui_perf rule 4).
- **Scope key** `(tab_scope, merged)` feeds several places:
  - the `_agent_panel_index()` memo;
  - `AgentPanelFoldScope`, which gains `tab_scope` (fold persistence schema v3 → v4; v3
    scopes migrate to the default-tab scope);
  - session-sticky tribe panels;
  - `_panel_selection_memory`;
  - the finalize stale token and `PreparedApplySnapshot`.

  Without this, switching tabs would garbage-collect other tabs' folds.

- **Per-tab memory:** the selected node identity, focused tribe panel, and scroll
  anchor. Restore by identity, falling back to the nearest surviving row.
- **Cross-tab navigation.** One helper switches to the target's tab first, or stays put
  at the All-tabs level, then reveals the row. Every jump uses it:
  - `_try_reveal_agent_row`;
  - Node Finder, which searches all tabs and chips rows that are off the active tab;
  - `,j`/`,J`, whose stopped candidates must come from the roster, not the visible
    `_agents`;
  - digit relation jumps;
  - notification `handle_jump_to_agent` / `navigate_to_agent_tab`, which today scan
    `_agents` directly;
  - link follow and link trail;
  - the Procs-pane monitor jump, the run-log modal jump, and Files "open agent".

  Jump-back anchors record the tab.

- **Scope honesty.** Bulk actions that act on "visible" rows become tab-scoped, and
  their confirmations name the scope: "on sase" or "across all tabs". Marks stay global,
  and a confirmation says when marked agents are on other tabs.

### The strip: placement, anatomy, style

It lives in the existing full-width `#agents-header` row, with tabs on the left. The
right side shows the active machine tab's health, falling back to today's fleet
diagnostic text. The row shows when the strip is visible or a diagnostic exists, and
stays hidden otherwise (the golden-preserving path).

```text
 ⌨ local 9 │ ⌨ apollo 14 S1 │ ⌨ mac 2  ┊  ▐ sase 12 U2 ▌ │ blog 3 •        apollo: stale · cached 2m ago
```

- **Active tab:** a pill with an accent background and bold near-black (`#1C1C1C`) text,
  capped by `▐`/`▌` half blocks in the accent. It is the only reversed segment, so it
  reads without color.
- **Inactive tabs:** the label in the tab's accent at normal weight, the count in
  `#AFAFAF`, then only the non-zero `S`/`F`/`U` tokens. Separators `│` and the `┊`
  divider are `#444444`.
- **Machine tabs** (including `local`): a `⌨` (U+2328) glyph in `#5FD7FF`.
  - A stale host's label turns amber, and an invalid or offline host's label turns red,
    reusing existing fleet and Machines-pane color constants.
  - The cause appears in the tooltip and on the right side of the row.
- **Named tabs:** no glyph by default; the accent-colored label carries identity.
  - The accent is `project_accent(name)` when the name is an enabled project key (so
    `sase` matches `+sase`), else `PROJECT_ACCENTS[hash_palette_index(name, ...)]`.
  - It can be overridden by `ace.agent_tabs.tabs.<name>.color`/`icon`.
- **`main`:** neutral `#AFAFAF`, no glyph.
- **Tiers:**
  - full;
  - compact (inactive tabs drop the count and `U`);
  - micro (glyph or first three letters, attention-colored);
  - an overflow window around the active tab with `‹3` / `2›` chips. A chip turns
    attention-colored when a hidden tab needs you, and clicking it opens the tab picker.
- **Tooltips:**
  - `local · this machine (athena)`
  - `apollo · stale 2m · +5 apollo agents on other tabs: sase 4, blog 1`
  - `sase · 12 agents on athena, apollo`, plus any configured `description`
- **Arrival dot.** A `•` in the tab's accent marks an inactive tab that received a new
  root since you last visited it. The cursor never moves.
- **Empty states** name one of three causes:
  - no agents on this tab;
  - the query hides them, with counts of matches elsewhere and a clear-filter hint;
  - the feed is unavailable, with a route to Admin Center Machines.
- **The All-tabs level** keeps the strip mounted, with every tab lit in its accent and
  no pill. There is never an "ALL" chip.

### The `o`/`O` layout ladder

| Level                    | Scope      | Panels        | Title                                                                              |
| ------------------------ | ---------- | ------------- | ---------------------------------------------------------------------------------- |
| Split by tribe (default) | active tab | one per tribe | `@tribe · N`, as today                                                             |
| Merged                   | active tab | one           | the tab's label and count (`⌨ apollo · 14`); `All agents` when the strip is hidden |
| All tabs                 | every tab  | one           | `All agents · every tab · 40`                                                      |

The grouping modal's single layout row becomes a segmented control:

```text
 Panel layout                                    o next · O back
   ◉ Split by tribe    ○ Merged    ○ All tabs
     One panel per tribe, showing sase only.
```

- `o` selects the next level and `O` the previous one (both wrap), then the modal
  dismisses. So `oo` is one step forward and `oO` one step back.
- `h`/`l` and left/right move the highlight while the layout row is focused. Enter or a
  click selects a segment directly.
- `on_key` gets an explicit `O` branch; today `O` is lowercased and swallowed.
- **R6.** With fewer than two tabs, only `Split by tribe` / `Merged` are offered.
  - A stored `All tabs` level renders as Merged, which is the same view.
  - The stored level returns when a second tab appears.
- The level is session-scoped, like today's merge flag. It is stored as an enum plus a
  remembered last per-tab level.
- App-level `O` stays unbound on Agents in v1.
- **Transitions preserve the selected node.**
  - Zooming out keeps it.
  - Zooming in from All tabs lands on the selected node's tab.
  - Choosing a tab (click, `[`/`]`, picker) at the All-tabs level drills into that tab
    at the remembered per-tab level.
- **At the All-tabs level:**
  - rows on named tabs get a tab chip (the name in its accent) before the tribe label;
  - the info row shows `panels: all tabs` (and `panels: merged` at the middle level);
  - the tribe summary's TRIBE MEMBERS roster chips members that are on other tabs.

### Keys

| Action                     | Key                                 | Notes                                                                                                                      |
| -------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| next / previous agents tab | `]` / `[`                           | New fields `next_agents_tab` / `prev_agents_tab`. They wrap, walk overflow tabs, and are a no-op when the strip is hidden. |
| next / previous card block | `)` / `(`                           | Moved from `]`/`[`.                                                                                                        |
| go to tab                  | unbound `pick_agents_tab`           | Also a palette entry "Agents: go to tab…" and the overflow chip click.                                                     |
| tab jump hints             | `'`                                 | `TabJumpTarget = ("tab", AgentTabKey)` hints on strip chips.                                                               |
| layout ladder              | `o` / `O` inside the grouping modal | See above.                                                                                                                 |
| launch tab picker          | `gb` / `Ctrl+G b` in the prompt bar | `b` is free in `_PROMPT_G_PREFIX_BINDINGS`.                                                                                |

- **Collision allowances.** Replace the card-block ↔ Artifacts-subtab pairs in
  `_CONTEXTUAL_APP_DUPLICATES` with:
  - `{next_agents_tab, cycle_artifacts_subtab}` and
    `{prev_agents_tab, cycle_artifacts_subtab_reverse}`;
  - `{next_card_block, files_next_version}` and `{prev_card_block, files_prev_version}`.
- **Legacy overrides.** If user config explicitly binds card blocks to `[`/`]`:
  - config load warns and names `(`/`)` as the new defaults;
  - the explicit binding is honored;
  - Agents tab cycling yields the brackets, so no key is double-bound.

### Storage, core boundary, and compatibility

- **Meta only.**
  - `agent_meta.json["agent_tab"]` is written only for an explicit named tab.
  - `["agent_tab_source"]` is `prompt` or `moved`.
  - The clan record gets `clan_tab`.
  - There is no `~/.sase/agent_tabs.json` side store, because the tribe store's
    split-brain is the defect to avoid.
- **sase-core `agent_tab.rs`** owns name canonicalization, reserved names, the key wire,
  effective resolution, and catalog ordering. Python calls it through `sase_core_rs` via
  a thin adapter, `src/sase/core/agent_tab.py`. Textual rendering, per-tab memory, the
  chip, and modals stay in Python.
- **Wires:**
  - the directive contract gets a `tab` entry;
  - typed units gain `AgentUnitWire.agent_tab`;
  - scan wire schema goes 10 → 11 (`agent_tab`, `agent_tab_source`);
  - fleet contract goes 6 → 7: `agent_tab` on `OwnerResolutionFactsWire` and
    `ResolvedAgentSummaryWire` with
    `#[serde(default, skip_serializing_if = "Option::is_none")]`, following the `tribe`
    precedent in commit `d0ec62c`.
- **Skew.**
  - An older controller reading a newer remote marks that host invalid (loud, not
    silent).
  - A newer controller reading an older remote gets rows without tabs, which fall back
    to machine tabs, and the header notes `apollo: tab data unavailable (upgrade sase)`.
  - Document the lockstep upgrade, controllers first.
- **Dispatch preflight.** A `%tab` + `%dispatch` launch reads the target's last-known
  contract version from an existing no-network source: a federation cache-only host
  response or the machine status cache.
  - If it is older than 7, refuse with an upgrade hint.
  - If it is unknown, warn and proceed.

### Configuration (`ace.agent_tabs`)

```yaml
ace:
  agent_tabs:
    machine_tabs: auto # auto | on | off — derive local/<alias> machine tabs
    launch_from_view: true # TUI launches from a named tab inherit it (%tab chip)
    tabs: {} # optional styling per tab name; never creates a tab
    # tabs:
    #   sase:
    #     color: "#AF87FF"
    #     icon: ""
    #     order: 10
    #     description: "Everything touching the sase repos, on any machine."
```

These are permanent user choices, so they are config fields, not flags. They need:

- a typed reader modeled on `src/sase/ace/tui/agent_decks_settings.py`;
- the JSON schema in `src/sase/config/sase.schema.json`;
- defaults and comments in `src/sase/default_config.yml`;
- documentation in `docs/configuration.md`.

### Feature flag scope

A beta flag, `agent_tabs`, is epic scaffolding: `tab-scope` creates it and `finish`
removes it.

- **Gated:**
  - the strip and tab scoping;
  - the `[`/`]` tab keys;
  - the All-tabs level;
  - machine-tab strip visuals and Admin Center Enter → tab;
  - the prompt chip and view inheritance;
  - the `N` modal's tab field.
- **Ungated** (safe and self-contained):
  - the directive, storage, and wires;
  - the `tab:` query field;
  - lineage inheritance and the dispatch preflight;
  - the card-block key move;
  - the `local` vocabulary;
  - the `sase agent tab` CLI.

Every gated behavior needs tests in both flag states.

### Memory and docs changes

The user asked for the machine-tabs glossary term and endorsed the research's §7 memory
updates. Every memory edit below goes through `/sase_memory_write`, then
`sase memory init`.

| Phase                  | Memory change                                                                                                                                                                                                                                                   |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `card-block-keys`      | `glossary:agent-data-card-block`: `[` / `]` becomes `(` / `)`.                                                                                                                                                                                                  |
| `tab-directive`        | `sase/memory/xprompts.md`: add a `%tab:<name>` row to the directive table.                                                                                                                                                                                      |
| `tab-lineage-dispatch` | `sase/memory/dispatch.md`: `%tab` travels with the prompt; note the preflight.                                                                                                                                                                                  |
| `finish`               | New strands `glossary:agent-tab` (aka agents sub-tab) and `glossary:machine-tab` (aka machine tabs), using the research §7 drafts adjusted to the final behavior and linked to each other; `glossary:node-panel` says tribe panels divide the active agent tab. |

Docs to update:

- `docs/xprompt.md`: the directive table and a `### Tab Directive` section.
- `docs/ace.md`:
  - a new Agent Tabs section near `### Machines`;
  - `### Grouping Modes` (the ladder);
  - `### Agent Search` (the `tab:` field);
  - the keybinding tables.
- `docs/remote_dispatch.md`: the `local` vocabulary, machine tabs, preflight, and
  lockstep upgrade.
- `docs/agent_sessions.md`: tabs belong to the root.
- `docs/configuration.md`, `docs/query_language.md`, and `docs/cli.md`.

### Verification gates (all phases)

- **In sase:** run `sase tool run check` (`just check`) and never `check-full` unless
  explicitly told.
- **In sase-core** (open with `sase repo open sase-core`): run `sase tool run check`.
- **Golden changes:** run `just fix-tui-screenshots` with targeted selectors through
  `/sase_monitor`. Inspect every created, removed, and updated golden before finalizing.
- **Visible TUI changes:** confirm them with a live `sase screenshot` capture.
- **Performance:**
  - A tab switch does no filesystem or network I/O and mounts no inactive trees.
  - `j`/`k` p95 stays under 16 ms on every tab.
  - Tab-switch key-to-paint p95 stays under 50 ms at 500 roots.
  - A quiet refresh tick reloads no extra surfaces.
  - The strip repaints only when the catalog signature, counts, or active key change.

---

## Phase 1 — card-block-keys

Move card-block stepping off the brackets and delete dead state that collides with the
feature's name. The phase lands unflagged: after it, `[`/`]` do nothing on Agents until
`tab-scope`.

1. **Card-block keys become `(` / `)`**, in all of these places:
   - `src/sase/default_config.yml` (`next_card_block`/`prev_card_block`, plus the
     comment);
   - `keymaps/app_keymaps.py` and `keymaps/metadata.py`;
   - the fallback bindings in `bindings.py`;
   - `_app_action_availability.py` card-block availability;
   - the palette metadata aliases (`commands/_app_metadata_nav.py`);
   - the deck search key list (`_deck_search_host.py`);
   - the rail hints (`widgets/decks/document_rail.py` default key hint, `block_rail.py`,
     `panel_blocks.py`);
   - the help rows (`modals/help_modal/agents_bindings.py`) and footer
     (`widgets/_keybinding_bindings_agents.py`);
   - `docs/ace.md`.
2. **Collision allowances.** In `keymaps/registry.py` `_CONTEXTUAL_APP_DUPLICATES`, drop
   the card-block ↔ Artifacts-subtab pairs and add the card-block ↔ Files-version pairs.
   Artifacts Files `(`/`)` must keep working, because the two are on different main
   tabs.
3. **Legacy override warning.** Warn at config load when a user explicitly binds card
   blocks to the brackets, reusing the stale-key warning path near `registry.py`
   `:196-199`. Keep the explicit binding working.
4. **Delete the dead Focus/Fleet sub-tab state:**
   - `AgentsSubTab`, `current_agents_subtab`, and `validate_current_agents_subtab` in
     `app.py`;
   - `_AGENTS_SUBTABS` (`_fleet_common.py` and its re-export in `_fleet.py`);
   - `watch_current_agents_subtab` and `_set_agents_subtab` (`_fleet_projection.py`);
   - the no-op assignment in `_fleet_refresh.py`;
   - the TYPE_CHECKING annotations;
   - the tests that reference them: `test_agents_fleet_refresh_laziness.py`,
     `test_fleet_agents_projection_agent_session_tree.py`, and the fleet visual test's
     `"focus"` waits.

   Keep `test_command_availability_agents_fleet.py`'s assertion that no subtab-cycle
   command exists.

5. **Goldens.** Update the goldens whose block-rail hint text changes.
6. **Memory.** Update the `glossary:agent-data-card-block` wording to `(` / `)`.

**Done when:**

- `(`/`)` step card blocks on Agents, and Files `(`/`)` still steps versions.
- The legacy warning is covered by a test.
- No `AgentsSubTab` symbol remains.
- `just check` and the targeted goldens pass.

## Phase 2 — core-tab-model (sase-core)

Work in the linked `sase-core` repo. Read its `AGENTS.md` first and follow its recipes:
no `macro_rules!`, and the `mod.rs` facade holds only `mod`/`pub use` lines.

1. **New module `crates/sase_core/src/agent_tab.rs`**, with `thiserror` errors and
   tests.
   - `canonicalize_agent_tab_name(raw) -> Result<AgentTabNameWire, AgentTabError>`:
     trims, lowercases, and validates the grammar. It returns `Default` for `main`, and
     guidance errors for `local`, `all`, empty, and invalid input. The messages are
     user-facing, so write them carefully.
   - `AgentTabKeyWire`: `Default | Machine { installation_id } | Named { name }`, plus a
     session-only machine variant for an unknown installation id keyed by alias.
   - `resolve_effective_agent_tab(root: AgentTabRootWire, machine_mode: bool)`, where
     the root carries the stored tab and the owner: local, or remote with installation
     id and alias.
   - `build_agent_tab_catalog(roots, options)`:
     - `options` carries `machine_mode`, the configured machine order as
       `(installation_id, alias)` pairs, and a named-tab `order` map;
     - it returns index-aligned per-root keys plus ordered entries (key, kind, label,
       root count) following the catalog order rule;
     - it takes one batched call per roster, never one call per row.
2. **Python bindings** go in the domain that binds `agent_tribe`
   (`crates/sase_core_py/src/agent_identity/`) or a sibling `agent_tab` domain. Name
   them `canonicalize_agent_tab_name`, `resolve_effective_agent_tab`, and
   `build_agent_tab_catalog`.
3. **Directive contract.** Add the `tab` entry to `editor/directive/metadata.rs`
   `DIRECTIVES`:
   - `COLON_PAREN`, no alias, no keywords, single-valued;
   - positional role `DirectiveValueRole::Tab` (new in `editor/wire.rs`).

   Then update:
   - the explicit name lists in `wire.rs`: synopsis, examples, snippet recipes, and
     `directive_metadata_supports_colon`;
   - the contract tests (`editor/directive/tests.rs`,
     `sase_core_py/src/editor_completion/tests/surfaces.rs`);
   - the directive diagnostics tests.

4. **Completion.** The `DirectiveValueRole::Tab` candidates in
   `editor/completion/directive_candidates.rs` come from agent-inventory entries of kind
   `"tab"`, plus a `main` suggestion. Add `Tab` to the LSP's `needs_agent_entries` in
   `sase_xprompt_lsp/src/server/completion_items.rs`.
5. **Typed units** (`agent_launch/typed_units.rs`):
   - parse `tab` in the directive match (next to `dispatch`, before the `_ => {}`
     passthrough) into a new `AgentUnitWire.agent_tab: Option<String>` field
     (`wires.rs`, with serde default and skip-if-none);
   - reject duplicates (`duplicate-tab`) and `%tab` on `%proc` units (a specific code,
     not the residual-prompt error);
   - enforce clan-declarer and fan-out agreement where units share a generation;
   - include `agent_tab` in the unit content digest;
   - re-emit `%tab:<name>` in `agent_unit_dispatch_prompt[_with_flags]` (`admission.rs`)
     after `%hide`.

**Done when:** `sase tool run check` passes in sase-core. The commit subject is a
Conventional Commit (`feat:`). Do not edit versions or changelogs.

## Phase 3 — core-tab-wires (sase-core)

1. **Scan wire.**
   - Add `AgentMetaWire.agent_tab` and `agent_tab_source` (`agent_scan/wire.rs`), both
     serde default.
   - Read them in `scanner.rs` `agent_meta_from_object`, validating through
     `canonicalize_agent_tab_name` and dropping invalid values.
   - Bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` 10 → 11.
   - Add no artifact-index column. Make sure index-served records round-trip the new
     fields, and bump the index schema only if its stored record projection would
     otherwise drop them.
2. **Fleet contract.**
   - Add `agent_tab` to `OwnerResolutionFactsWire` and `ResolvedAgentSummaryWire`
     (`fleet_contract/resolution.rs`) with
     `#[serde(default, skip_serializing_if = "Option::is_none")]`.
   - Project and validate it next to `tribe` (`resolution.rs`, `projection.rs`).
   - Bump `FLEET_CONTRACT_SCHEMA_VERSION` 6 → 7 (`error.rs`). `FLEET_PROTOCOL_VERSION`
     is unchanged.
3. **Fleet catalog.**
   - Add `PresentationRecordFacts.agent_tab`, filled in
     `direct_presentation_facts_for_record`.
   - Inherit it from the root for members, exactly as `tribe` is inherited at
     `fleet_catalog.rs:167-178`.
   - Hash it in `stable_revision`.
4. **Gateway.** Fill the owner facts in `sase_gateway/src/fleet_reads/resolution.rs` and
   update the contract docs in `sase_gateway/src/contract.rs`.
5. **Tests.**
   - Old (v6) rows decode with no tab.
   - v7 rows round-trip.
   - A v6 reader rejects v7 rows. Assert today's behavior; do not change it.
   - Member inheritance works.
   - Stable revisions change when a tab changes.
6. **Breaking-change marker.** The commit uses `feat!:` or a `BREAKING CHANGE:` footer,
   because old readers reject contract-7 rows. The footer says: "upgrade controllers
   first, or the fleet in lockstep".

**Done when:** `sase tool run check` passes in sase-core.

## Phase 4 — tab-directive (sase)

1. **Pin and mirrors.**
   - Move `sase-core-revision.txt` to the pushed sase-core commit that contains phases 2
     and 3 (`just ratchet-core-revision`, or set it by hand).
   - Mirror the scan wire version and `AgentMetaWire` fields
     (`src/sase/core/agent_scan_wire_records.py`, `agent_scan_wire_markers.py`).
   - Mirror the `AgentUnitWire.agent_tab` field
     (`src/sase/core/agent_launch_wire_records.py`).
   - If the core release changes the published minor, follow `docs/rust_backend.md` "Who
     owns the published version window".
   - Add the adapter `src/sase/core/agent_tab.py`.
2. **Python directive.**
   - Add `tab` to `_KNOWN_DIRECTIVES` (`xprompt/_directive_types.py`).
   - Add the `PromptDirectives.agent_tab` field, plus an explicit-default marker for
     `%tab:main`.
   - Add paren validation modeled on `%dispatch` (`_directive_collect.py`).
   - Add `resolve_agent_tab` in `_directive_values.py`, calling the core canonicalizer.
   - Wire it in `_directive_extract.py`.
   - Duplicates are rejected by `_store_single_directive`.
   - Reject `%tab` together with `%proc`.
   - Update `tests/test_xprompt_directive_contract.py` (keywords, syntax forms, parity).
3. **Runner writes.**
   - `build_agent_meta` (`axe/run_agent_directive_metadata.py`) writes `agent_tab` and
     `agent_tab_source: prompt`.
   - `record_clan_attributes_at_launch` / `apply_clan_launch_defaults`
     (`axe/run_agent_directive_clans.py`) record `clan_tab`. Joiners copy the
     generation's `clan_tab`, and a differing explicit tab raises a `DirectiveError`.
   - `_add_agent_session_metadata` copies the session root's `agent_tab`. An explicit
     differing `%tab` raises a clear error next to the tribe/session guards in
     `run_agent_directives.py`.
   - Add `agent_tab` and `agent_tab_source` to `preserved_agent_metadata`.
   - Revive keeps the field (`_revive_artifacts.py` `_build_agent_meta_data`,
     `_restore_agent_meta`).
   - The TUI fork (`_fork_actions.py` `_complete_agent_fork_scope`) and the mobile fork
     (`integrations/_mobile_agent_lifecycle.py`) prefill `%tab:<source>`.
4. **Agent model.**
   - Add an `agent_tab` field next to `tribe` (`models/_agent_state_session.py`).
   - Fill it in both meta enrichers (`_meta_enrichment_filesystem.py`,
     `_meta_enrichment_wire.py`).
   - Fill it in fleet conversion (`models/_fleet_agents_rows.py` `_agent_from_summary`).
   - Synthesized remote session nodes inherit it (`_fleet_agents_nodes.py`).
   - Provisional dispatch rows take the prompt's `%tab` (`dispatch_launch_rows.py`).
5. **Query field `tab:`.**
   - It is an exact-match field: stored tab names, or `main` for rows with none.
   - Add it in `query_profile/profiles/_agents_live.py` and project it in
     `models/agent_live_query.py`.
   - Classify it in `KNOWN_FALLBACK_FIELDS` (`agent_live_query_pushdown.py`).
   - Add it to the legacy dialect tokenizer and evaluator (`ace/agent_query/`).
   - Update the profile tests and the query goldens.
6. **Completion.** Add `"tab"` kind entries, from the loaded roster's distinct
   `agent_tab` values and `main`, in `ace/tui/_agent_completion_candidates.py` and
   `integrations/_editor_helper_agents.py`, so `%tab:` completes in the TUI and editors.
7. **Docs and memory.**
   - Docs: the `docs/xprompt.md` table row and a `### Tab Directive` section; the `tab:`
     row in the `docs/ace.md` Agent Search table and `docs/query_language.md`.
   - Memory: the `xprompts.md` table row.
8. **Tests** cover:
   - `%tab` absent, valid, uppercase, invalid, duplicated, reserved, inside fences, and
     in xprompts, fan-out, and swarms;
   - `%tab` + `%dispatch`, and `%tab` + `%proc` (rejected);
   - clan and session inheritance and mismatch errors;
   - retry, revive, and fork;
   - wire and fleet loading;
   - the `tab:` query in both dialects.

**Done when:** `just check` passes. `%tab` is stored, loaded, queryable, and stripped
from the model prompt.

## Phase 5 — tab-lineage-dispatch (sase)

1. **Export.** The runner exports `SASE_AGENT_TAB` for a turn whose root has a named
   tab, next to where `SASE_AGENT_NAME` is set (`axe/run_agent_directive_identity.py`).
2. **One helper.** Add `apply_inherited_agent_tab(prompt, tab)` in `src/sase/xprompt/`.
   It inserts `%tab:<tab>` unless the prompt already has a `%tab` or is a session
   attach, and it handles each swarm segment. A sibling `set_agent_tab_directive` edits
   or replaces the directive for UI callers, mirroring `set_dispatch_directive`.
3. **Apply it at every agent-initiated launch path.** Read the tab from
   `SASE_AGENT_TAB`:
   - LaunchApproval request creation (`agent/launch_request.py`
     `create_launch_approval_request_from_prompt` / `create_launch_approval_request`),
     so the approver sees `%tab` in the reviewed prompt;
   - direct agent-context launchers;
   - `sase bead work` segment rendering (next to `epic_work_segment_env`).
4. **Gate and monitor turns.** They record the creator's `SASE_AGENT_TAB` when created
   and re-export it when running option commands and follow-up launches. An EpicApproval
   approve, which runs `sase bead work`, then places every phase and land agent on the
   planner's tab.
5. **Dispatch preflight** in `dispatch/launch.py` `preview_dispatch_launch`, for a
   `%tab` + `%dispatch` launch (see the contract). Surface it in both the CLI and the
   TUI dispatch preview.
6. **Docs and memory.**
   - `docs/remote_dispatch.md`: `%tab` forwarding, the preflight, and the lockstep
     upgrade note.
   - The `dispatch.md` memory note.
7. **Tests** cover:
   - env export for root and follow-up turns;
   - insertion and skip rules (existing `%tab`, `%tab:main`, session attach, swarm);
   - a LaunchApproval prompt that carries the tab;
   - an epic approval whose segments carry the tab;
   - the preflight for an older, unknown, and current target.

**Done when:** `just check` passes.

## Phase 6 — tab-scope (sase, flagged)

1. **Flag.** Run `sase flag new agent_tabs -k beta` with the three authored sentences
   (enabled, disabled, remove-when: "the finish phase of this epic deletes the Off
   branch"). This approved plan authorizes this flag bead as epic scaffolding. Paste the
   registry entry. The check is `current_flags().enabled(FeatureFlag.agent_tabs)`,
   wrapped in one helper, as `widgets/decks/final/flag.py` does.
2. **Config block `ace.agent_tabs`** (the full contract block): a typed reader, schema,
   defaults, and docs, plus the token-cached machine-mode accessor over
   `load_dispatch_config()`. Pass the machine order and the named `order` map into the
   core catalog.
3. **Tab index.**
   - Build it from `_agents_with_children` using `presentation_anchor_lookup`
     (`models/_agent_tree_anchor.py`) and the root-stored-tab rule for session and clan
     containers.
   - Make one batched `build_agent_tab_catalog` call.
   - Memoize it by roster identity and the machine-mode/config token.
   - Compute it off the UI thread inside the worker finalize plan
     (`_loading_compute_finalize.py` `_compute_finalize_plan`), and inline on the
     refilter path.
4. **Scope stage.**
   - Cache the tab-independent query result.
   - Apply the active-tab scope in both the worker plan and the inline
     `finalize_agent_list` path.
   - Add the active key to `PreparedApplySnapshot` and the finalize stale token.
   - Add the tab scope to the `_agent_panel_index()` memo, `AgentPanelFoldScope` plus
     fold persistence v4 (with v3 migration), the session-sticky panel identity maps,
     and `_panel_selection_memory`.
5. **Active tab state.**
   - Add the per-tab memory.
   - Persist the active key off-thread to `ace_agents_tab_state.json` under
     `sase_home()`, modeled on `_deck_persistence.py`, and expose it on `ace_page.py`
     for tests.
   - Implement the startup selection, the latch, and the disappearance fallback.
6. **Keys.**
   - `next_agents_tab` / `prev_agents_tab` (`]`/`[`) go through the full keymap surface:
     config, types, metadata, fallback bindings, availability (Agents only, not while
     the prompt or a modal owns keys), `_CONTEXTUAL_APP_DUPLICATES`, help, footer, and
     palette.
   - Add the unbound `pick_agents_tab`.
   - When card blocks are explicitly bound to the brackets, tab keys yield and warn.
7. **Cross-tab navigation.** Add the switch-then-reveal helper and route every entry
   point listed in the contract through it.
8. **Scope honesty.** Make bulk and cleanup confirmations name their scope.
9. **Minimal strip.** Render a plain `PanelTabStrip` in `#agents-header`, labels only,
   so the scope is testable. `tab-strip` replaces it.
10. **Perf.** Add a tab-switch key-to-paint metric to `SASE_TUI_PERF`.
11. **Tests.** With the flag off, today's behavior is unchanged. With it on, cover:
    - zero, one, two, and many tabs;
    - a query that empties a tab without removing it;
    - `STARTING` roots;
    - provisional-to-authoritative rows staying on one tab;
    - selection and fold restore across switches;
    - cross-tab jumps from each entry point;
    - persistence round-trip;
    - a switch doing no I/O (assert on the stubs).

**Done when:**

- The flag-off goldens are unchanged, and `just check` passes.
- A live `sase screenshot` with the flag on shows tabs switching instantly.

## Phase 7 — tab-strip (sase, flagged)

1. **Widget.** Add `AgentTabStrip` (`widgets/agent_tab_strip.py`), a subclass of or
   companion to `PanelTabStrip` that reuses its click ranges, tier reflow, and tooltip
   plumbing. An `AgentTabDescriptor` carries the key, label, glyph, accent, count,
   incomplete flag, attention counts, health, arrival flag, description, and jump hint.
2. **Anatomy.** Implement the contract's anatomy exactly: the pill with half-block caps,
   accent labels, the `┊` divider, and the badges in `format_agent_count_chip`
   vocabulary. Keep the right-side health or diagnostic text.
3. **Tiers and overflow.**
   - Full, compact, and micro tiers, plus a windowed overflow with attention-colored
     `‹N` / `N›` chips.
   - A searchable tab picker modal lists the glyph, accent label, count, attention, and
     machines. It opens from the overflow chips, the palette entry, and
     `pick_agents_tab`.
4. **Jump hints.** Add `'` hints on strip chips (`TabJumpTarget` in
   `navigation/jump_hints.py` and `_entry_jump_mode.py`).
5. **Arrival dots**, cleared on visit.
6. **Empty states** with the three causes. The latch rendering keeps the emptied active
   tab's pill.
7. **Repaint only** when the catalog signature, counts, health, or active key change.
8. **Goldens:**
   - the hidden strip (pixel-identical to today);
   - `main` plus named tabs;
   - attention on inactive tabs;
   - overflow at narrow width;
   - each empty state;
   - a long name at 32 characters;
   - the picker modal.

**Done when:**

- The goldens are inspected and pass.
- A live screenshot at wide and narrow widths looks right.
- `just check` passes.

## Phase 8 — layout-ladder (sase, flagged)

1. **Level state.** Replace `_agent_panels_grouped` with an `AgentPanelLayout` level
   (`SPLIT | MERGED | ALL_TABS`) plus a remembered last per-tab level. Map it onto the
   `(tab_scope, merged)` scope key at every read site the survey lists (`_display*.py`,
   `_selection.py`, `_folding_*.py`, `_fold_scope.py`, `_loading_*`, navigation, and
   Node Finder).
2. **Modal.**
   - `modals/agent_grouping_modal.py`: the segmented layout row, with `o` next, `O`
     previous, `h`/`l`/arrows, Enter, and click.
   - An `AgentGroupingAction` result carrying the chosen level, handled in
     `actions/agents/_grouping.py`.
   - R6 two-segment mode.
   - A dynamic description line naming the active tab.
3. **Titles and chips.**
   - Titles per the ladder table (`_display_panel_titles.py`).
   - The info-row `panels:` chip (`widgets/agent_info_panel.py`).
   - The All-tabs strip state.
   - Row tab chips for named-tab rows at the All-tabs level.
   - TRIBE MEMBERS roster chips for off-tab members.
4. **Transitions.** Make them anchor-preserving, reusing the capture / generation /
   restore pattern from commit `12ae3014b9`, including drill-in on tab choice.
5. **Palette.** Update the palette command `agents.toggle_panel_grouping` wording to the
   ladder.
6. **Tests and goldens:**
   - the ladder with one and with several tabs;
   - `oo` and `oO`;
   - clicks;
   - anchor preservation;
   - the All-tabs view with row chips;
   - the modal in both modes.

**Done when:** the goldens are inspected and `just check` passes.

## Phase 9 — machine-tabs (sase; strip visuals flagged)

1. **Visuals.** Machine tabs get the `⌨` glyph in `#5FD7FF` and health-colored labels
   from `HostFeedIssue` and the per-row host fields.
   - When a host's contract version is below 7, the right side shows
     `apollo: tab data unavailable (upgrade sase)`.
   - Unconfigured origins, alias renames, and a remote alias literally named `local` are
     handled per the contract.
2. **`local` vocabulary (unflagged).** The display word for this machine becomes `local`
   in:
   - the BY_MACHINE banner (`models/agent_groups/_keys.py`, `_tree.py`);
   - the Launch Target picker (`_prompt_input_bar_dispatch.py`,
     `dispatch_target_modal.py`);
   - the Machines pane (`machines_pane*.py`);
   - the docs.

   `machine:local` is accepted alongside `machine:here` in both query dialects, with
   pushdown parity.

3. **Suppress redundant chips on machine tabs.** Hide the per-row remote alias chip
   (`_agent_list_render_agent_prefix.py` via `_agent_row_chrome_mode`) and the lone
   BY_MACHINE top banner, keeping the status subgroups. Keep both on named tabs and at
   the All-tabs level.
4. **Compact detail header.** Its machine chip changes from `⇄` to `⌨`
   (`prompt_panel/_identity_header_compact.py`).
5. **Admin Center Machines Enter** (`modals/machines_pane.py` `action_show_agents`)
   selects that machine's tab when the strip is visible. Otherwise it keeps the
   `machine:` filter behavior. A secondary key, chosen from the free Machines-pane keys,
   keeps the filter behavior in all cases. Update the keymaps, help, and hints.
6. **Tooltips** name the off-tab counts per machine.
7. **Goldens:**
   - machine mode (`local` / `apollo` / `mac` plus named tabs);
   - `%tab:apollo` beside `⌨ apollo`;
   - a stale host and an invalid host;
   - BY_MACHINE on a machine tab and on a named tab;
   - the renamed `local` surfaces.

**Done when:** the goldens are inspected and `just check` passes.

## Phase 10 — launch-view-ux (sase, flagged)

1. **Prompt-bar context line** (`#prompt-dispatch-context`,
   `widgets/_prompt_input_bar_dispatch.py`) shows one of:
   - `tab: blog` in the tab's accent, with `Ctrl+G b change · %tab:main for default`,
     when a named tab is active or the prompt has an explicit `%tab`;
   - `tab: main (default)` for `%tab:main`;
   - a subtle note when a named tab equals a machine alias;
   - `runs on ⌨ local · gD launch on apollo` on a remote machine tab.

   It refreshes on text change without I/O.

2. **`gb` / `Ctrl+G b` Launch Tab picker.** Add it to `_PROMPT_G_PREFIX_BINDINGS`. It
   lists existing tabs, the default, and "new tab…" with validated input. It edits the
   prompt through `set_agent_tab_directive`.
3. **Submit.** In `_launch_resolved_prompt` (`_launch_prompt_inputs.py`), apply
   `apply_inherited_agent_tab` with the active named tab, subject to R1's conditions and
   `launch_from_view`.
4. **Landing toast.** When a launch lands on a tab other than the active one, a toast
   names the destination (`blog-fix → blog`) and the destination tab gets its arrival
   dot.
5. **LaunchApproval.** Cards show a `Tab:` field when the prompt carries `%tab`.
6. **Tests:**
   - chip states;
   - picker insert and replace;
   - insertion and opt-out;
   - no insertion from default, machine, or All-tabs views, or with the config off;
   - swarms;
   - the toast.

   Add goldens for the chip and the picker.

**Done when:** the goldens are inspected and `just check` passes.

## Phase 11 — tab-moves (sase; modal field flagged)

1. **`persist-directive` support.**
   - Extend it for `meta_set {agent_tab, agent_tab_source: moved}` and `meta_remove`.
   - Add a prompt mutator `set_tab` that rewrites or removes `%tab` in stored prompts,
     next to `set_prompt_tribe` in `xprompt/_directive_edit_identity.py`.
   - Rewrite clan records' `clan_tab`, touching `ops/commands/_agent_directive.py` and
     `_directive_persistence.py`.
   - Keep the artifact index in sync.
2. **CLI `sase agent tab`** (`list` default, `set`, `unset`), following `cli_rules.md`:
   sorted, colored, with short aliases.
   - `list` shows the local catalog with counts, using the core catalog, and supports
     `-j/--json`.
   - `set <agent> <tab>` and `unset <agent>` move the agent's whole presentation root
     (session, clan generation, or workflow) and print what moved.
3. **`N` modal → "Tribe & Tab".**
   - Add a second input with tab completion (`modals/agent_tribe_modal.py`,
     `actions/agents/_tribe_assignment.py`), gated by the flag.
   - Bulk marks work as they do for tribes.
   - The update is optimistic with rollback, and invalidates the tab index.
   - A move off the active tab shows a toast (`moved 2 agents → blog`).
   - Remote rows are disabled, with the reason "tab moves run on the owning machine".
4. **Tests:**
   - CLI help and behavior;
   - moving a session, a clan, and a single agent;
   - prompt rewrite;
   - rollback;
   - remote refusal.

   Add a golden for the modal.

5. **Docs:** `docs/cli.md` and `docs/agent_sessions.md`.

**Done when:** `just check` passes.

## Phase 12 — finish

1. **Remove the flag.** Delete every `agent_tabs` Off branch and make the On branch
   unconditional. Remove the registry entry and close the flag bead in the same change,
   per `sase_flags.md`. Delete the flag-off tests and keep the behavior tests.
2. **Docs pass.** Finish `docs/ace.md`: the Agent Tabs section, the machine tabs, the
   ladder, the key tables, and a screenshot or two. Check `docs/configuration.md`,
   `docs/remote_dispatch.md`, `docs/agent_sessions.md`, and `docs/xprompt.md` for
   consistency with the final behavior.
3. **Perf.** Add a tab-switch case to `tests/ace/tui/bench_tui_jk.py` (or the trace
   bench). Record p50 and p95 at 500 roots against the contract targets. Confirm that
   `j`/`k` p95 is still under 16 ms.
4. **Goldens.** Run the full golden pass through `/sase_monitor` and inspect every
   change.
5. **Memory.** Through `/sase_memory_write`:
   - add `glossary:agent-tab` and `glossary:machine-tab`, linked to each other and to
     `glossary:agent-tribe`;
   - update `glossary:node-panel`;
   - run `sase memory init`.
6. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on this phase's bead for:
   - opt-in project-default tabs (`ace.agent_tabs.default: main | project`);
   - bead-remembered tabs for epics launched later from the Artifacts pane;
   - an app-level `O` if users ask for it.

**Done when:**

- No flag reference remains.
- The docs, glossary, and goldens are current.
- The bench numbers are recorded in the phase notes.
- `just check` passes.

## Out of scope

- The Agent Cluster umbrella term.
- An `ALL` tab.
- Saved-query or tribe-as-tab providers.
- Always-visible empty machine tabs.
- Project-default tabs.
- A `--tab` CLI flag on `sase run`.
- Negotiated fleet contract versions.
- Any change to clan, session, or tribe semantics.
