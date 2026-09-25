---
tier: epic
title: Agents-tab Node Finder on the " key
goal: 'Pressing `"` on the Agents tab opens a large Node Finder modal. It lists every
  reachable sase node as a tree, including nodes hidden by folds, collapsed grouping
  banners, collapsed or isolated tribe panels, the Agents query, and (last phase)
  the `I` hide-non-run toggle. Every jumpable row always carries a jump hint. A query
  bar that starts unfocused (Tab toggles it) fuzzy-filters by node name. Enter and
  Ctrl+N/Ctrl+P navigate. A fast two-tier preview shows the highlighted node. Every
  jump lands reliably through the existing identity-based reveal primitive, and every
  change it makes to view state is announced and can be undone.

  '
phases:
- id: finder-model
  title: Node Finder row model, snapshot, filtering, and hints
  depends_on: []
  size: medium
  description: 'finder-model: add the pure row model and the owner-aware snapshot
    builder, which projects every fold, banner, and panel open and classifies why
    each row is hidden. Add token-AND fuzzy filtering on name and title, with a contiguous
    pass, a relaxed fallback, and incremental narrowing. Add prefix-free hint allocation,
    wrapping cursor math, and the reason and action text, all unit-tested.

    '
- id: jump-ladder
  title: Identity jump ladder with an announced query clear
  depends_on: []
  size: small
  description: 'jump-ladder: add a non-notifying reveal variant, a query-clear helper
    that records history and is correct on both agents_unified_query branches, and
    _jump_to_node_identity. It reveals by identity, clears the Agents query only when
    the query hides the target, retries, and toasts. Test through a real AcePage.

    '
- id: finder-preview
  title: Two-tier Node Finder preview
  depends_on:
  - finder-model
  size: medium
  description: 'finder-preview: add the Tier 0 renderers, which do no I/O and paint
    on every highlight: kind chip, compact identity, breadcrumb, why-hidden and Enter-action
    lines, and kind-specific in-memory sections. Add the Tier 1 prompt-head and reply-tail
    loader. It is thread-only, never mutates live rows, and sits behind a modal-local
    LRU.

    '
- id: finder-modal
  title: NodeFinderModal screen, modes, keys, and layout
  depends_on:
  - finder-model
  - finder-preview
  size: medium
  description: 'finder-modal: build the large responsive ModalScreen. It has a tree
    list with a hint gutter and glyphs for why each row is hidden, a HINTS/SEARCH
    mode pill, a scope strip, and a preview pane. Add the full key model (hints, Tab
    and /, Enter, wrapping Ctrl+N/P and arrows, "" back, Esc, invalid-key flash with
    no leak to the app), debounced off-pump Tier 1 wiring, styles, exports, and modal
    key tests.

    '
- id: finder-wiring
  title: Keymap, action, docs, visual goldens, and bench
  depends_on:
  - jump-ladder
  - finder-modal
  size: medium
  description: 'finder-wiring: bind quotation_mark to jump_to_node on the Agents tab
    across the keymap validation, dataclass, default config, metadata, bindings, availability,
    command palette, help, docs, and module exports. Connect the modal''s result to
    the jump ladder and drop the epic-symbol whitelist. Add end-to-end tests, PNG
    goldens, and the 2,000-node perf bench.

    '
- id: hidden-by-i
  title: Rows hidden by I, with a flip-and-reveal rung
  depends_on:
  - finder-wiring
  size: medium
  description: 'hidden-by-i: list the running and failed rows that the I hide-non-run
    toggle hides as ◌ rows, in tree position, from an in-memory pre-hide roster. Add
    the ladder rung that turns I off, waits for the reload, and then reveals, with
    a toast. Update the scope strip, tests, and goldens.'
proposed_by: bbugyi200.athena.0s5
create_time: 2026-09-25 13:06:03
status: wip
bead_id: sase-19i
---

- **PROMPT:** [prompts/202609/agents_node_finder.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_node_finder.md)
- **BEAD:** [sase-19i](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19i/README.md)

# Plan: Agents-tab Node Finder on the `"` key

## Context

Every jump surface on the Agents tab today reaches only rows that are already rendered,
or only the relations of the selected container:

- `'` entry jump;
- `` ` `` `JumpAllModal`, which lists the already folded and filtered `_agents` and
  dismisses on any unrecognized key;
- `0-9` roster digits and the `.` footer;
- `$` link follow.

Reaching a hidden clan member or a folded `--mon-N` shell today means drilling through
clan, then session, then shell. The hard part already exists:
`MemberJumpNavigationMixin._reveal_agent_row(identity, *, subject=...)`
(`src/sase/ace/tui/actions/navigation/_member_jump.py`). It sits on
`prepare_agent_navigation_target` and `reveal_agent_navigation_target`
(`src/sase/ace/tui/actions/navigation/_agent_reveal.py`). It validates ancestry, expands
exactly the needed folds (`FULLY_EXPANDED` for hidden steps), grouping banners, and the
tribe panel, then selects the row by stable `Agent.identity` and saves a `ctrl+o`
anchor. It fails with a typed `AgentRevealFailure`.

This epic builds a well-designed modal on top of that primitive. It follows the
consolidated research report `agents_tab_node_finder.md` in the research sidecar. The
user has accepted all of that report's requirement adjustments:

- **A1. It is a modal, not a docked panel.** A large transient `ModalScreen`, titled
  `✦ Jump to Node ✦`. It is never called a "jump panel", which is already the `.`
  footer.
- **A2. Scope of "any node".** In scope: every loaded, non-dismissed sase node.
  - Listed: clans, agent nodes, session shells including monitors and gates, workflow
    roots, workflow `agent` steps, and stand-alone proc shells.
  - Never a jump target: any non-`agent` workflow step (bash, python, parallel,
    pre-prompt). Such a step appears only as a dim context row, and only when it has
    jumpable descendants.
  - Not listed: `STARTING` rows, structurally unreachable rows, and dismissed rows.
- **A3. Hidden-ness is tiered.**
  - The first five phases cover fold, banner, and panel hiding, plus loaded rows the
    query hides.
  - The `hidden-by-i` phase adds rows the `I` toggle hides.
  - Rows a windowed query never loaded cannot be listed. The finder says so
    (`history partial`) instead of implying they don't exist.
- **A4.** The finder may clear the Agents query. This is its only persistent view-state
  mutation besides `I` in the last phase. It is recorded in query history and announced
  with a toast.
- **A5. Keys beyond the request.**
  - `/` also focuses the query, and `↑`/`↓` also cycle.
  - `""` jumps back, and Esc in search mode returns to hints.
  - Backspace cancels a pending hint prefix.
  - Invalid keys flash; they never dismiss.
- **A6.** The modal opens on "you are here" (`◆`).
- **A7. A fast preview means two tiers.** An instant, I/O-free card, plus a debounced
  prompt head and reply tail loaded off the pump.
- **A8.** Filtering is by name first: the canonical node name (primary) plus the
  displayed title (secondary). It is _not_ the structured Agents query language.

**Rust-core boundary.** The snapshot depends on TUI fold, banner, panel, and query
state, which is presentation state. So the node set, classification, and rendering stay
in this repo. The one shared primitive, fuzzy matching, is already in `sase_core`
(`sase.core.fuzzy_facade.fuzzy_match`). No `sase-core` change is needed.

**No feature flag.** The key binding lands only in `finder-wiring`, after the model,
preview, modal, and ladder exist. So no landed phase exposes an unfinished feature. The
`hidden-by-i` phase extends a complete, honest v1, which shows an `I hides K` chip until
then. Nothing old has to stay reachable.

**Symvision.** Phases before `finder-wiring` add public symbols whose product consumer
lands later. Add `--epic-symbol '<epic-bead>(<symbol>)'` entries to the Justfile
symvision invocation for them; the epic bead id is on each phase bead. The phase that
adds the real consumer removes them. Never whitelist a symbol that nothing will consume:
make it private instead.

**Module size.** Keep every new or edited module under the repo's 500-line cap.
`_member_jump.py` is already at about 490 lines.

## Design

### Interaction model

| Key                            | HINTS mode (default: `OptionList` focused)                                                                     | SEARCH mode (query `Input` focused)                                  |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `0-9a-zA-Z`                    | Complete a hint and jump, or set a pending prefix                                                              | Edit the query; live refilter; hints re-allocated over the survivors |
| `Tab` / `Shift+Tab`            | Focus the query                                                                                                | Back to HINTS, query kept (flushes any pending refilter first)       |
| `/`                            | Focus the query                                                                                                | Types `/`                                                            |
| `Enter`                        | Jump to the highlighted node                                                                                   | Jump to the highlighted node (best match right after typing)         |
| `Ctrl+N` / `Ctrl+P`, `↓` / `↑` | Next/previous jumpable row, **wrapping**, skipping context and header rows                                     | Same                                                                 |
| `"`                            | Jump back (dismiss with `back`; same as `ctrl+o`). If there is no Agents jump history, flash `no jump history` | Types `"`                                                            |
| `Backspace`                    | Cancel a pending prefix; otherwise swallow                                                                     | Delete a character                                                   |
| `Esc`                          | Cancel a pending prefix; otherwise close                                                                       | Back to HINTS, query kept                                            |
| `Ctrl+U`                       | — (swallow)                                                                                                    | Clear the query (native `Input`)                                     |
| `PgUp` / `PgDn`                | Scroll the preview                                                                                             | Scroll the preview                                                   |
| Any other key                  | Swallow; the footer flashes `no hint ‹x›` for about 1.2 s                                                      | Types                                                                |

Hard rules:

- **Keys must never change meaning with list size.** An unrecognized key never
  auto-focuses the query or seeds it, because at 62 or more jumpable rows every
  alphanumeric is a hint.
- **Stop every key.** The app-level `on_key` (`actions/_event_keyboard.py`) has no modal
  guard, and its `_custom_mode_prefixes` branch would arm a leader or custom mode
  underneath the modal. So in HINTS mode the screen's `on_key` must `prevent_default()`
  and `stop()` every key it sees.
- **Tab bindings.** `Screen` binds `tab` and `shift+tab` to focus traversal. Bind them
  as screen `Binding(..., priority=True)` (precedent: `modals/revive_agent_modal.py`).
- **Cursor bindings.** Bind `ctrl+n`, `ctrl+p`, `up`, and `down` as priority screen
  bindings that call the modal's own wrapping cursor. Do not reuse
  `OptionListNavigationMixin.NAVIGATION_BINDINGS`: it binds `j`, `k`, and `q`, which
  must stay hint characters.
- **The hint alphabet needs no exclusions.** None of the command keys (`/`, `"`, Tab,
  Esc, Enter, Ctrl+\*) are in `JUMP_HINT_CHARS`.
- **Clicks.** A mouse click on a row highlights it and jumps (native `OptionSelected`).

### Visual design

The finder should read as a _map of the Agents tab_. Reuse the house style instead of
inventing a new one. Container: about 94% × 90%, `border: double $primary`,
`background: $surface`, a visual sibling of `JumpAllModal`. Layout at 140 columns or
more, in HINTS mode:

```
╔════════════════════════════════════ ✦ Jump to Node ✦ ════════════════════════════════════╗
║ ❯ Tab or / to search nodes…                        HINTS   142 nodes · 97 hidden · 3 ⊘  ║
╟───────────────────────────────────────────┬──────────────────────────────────────────────╢
║ @default ──────────────────── 9 · 4 hidden │  AGENT SHELL  research.8.cdx                 ║
║ [0] ◆ sase                     RUNNING 12s │ codex · gpt-6 @ high · RUNNING · 16m42s      ║
║ [1]   sase                     RUNNING  8m │ @research ▸ research.8 ▸ research.8.cdx      ║
║ [2]   bob-cli                     DONE 18m │ ▸ Inside collapsed clan research.8           ║
║ [3] ▸ └ bob-cli--plan             DONE 19m │ ⏎ expands 1 fold, then selects it            ║
║ @research ──────────────────── 7 · 6 hidden│                                              ║
║ [4] ▸ research.8          clan · 4 RUNNING │ PROMPT ────────────────────────────────────  ║
║ [5] ▸ ├ research.8.cdx         RUNNING 16m │ You are researcher cdx in a 4-researcher     ║
║ [6] ▸ ├ research.8.cld         RUNNING 16m │ swarm. The other researchers …               ║
║ [7] ▸ └ research.8.mus         RUNNING 16m │ REPLY · tail ──────────────────────────────  ║
║ @epic ▭ ───────────────────── 14 · 13 hidden│ ⋯ loading                                   ║
║ [8] ▭ sase-169            clan · 5    DONE │                                              ║
║ [9] ▭ ├ sase-169.3                    DONE │                                              ║
║ [a] ▭ │ ├ sase-169.3--plan            DONE │                                              ║
║ [b] ▭ │ └ sase-169.3--mon-1      COMPLETED │                                              ║
╟───────────────────────────────────────────┴──────────────────────────────────────────────╢
║ 0-Z jump · tab / search · ^n ^p move · ⏎ jump · "" back · esc close              a…     ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝
```

- **Top bar.**
  - A `FilterInput` query bar. Its border is muted when unfocused and `$accent` when
    focused.
  - A mode pill: `HINTS` in bold black on `#FFFF00` (the hint yellow), or `SEARCH` in
    bold black on `#87D7FF` (the Agents accent).
  - A dim scope strip.
    - HINTS: `N nodes · H hidden · F ⊘`, where `⊘` appears only while a query is active.
      It adds `· I hides K` (from `_hidden_count`, until `hidden-by-i`),
      `· history partial` (when `_agent_load_state.query_incomplete` is set), and
      `· type to narrow` (when more than 3,844 rows are jumpable).
    - SEARCH: `M of N`, plus `· relaxed` when the subsequence fallback is in use.
  - The finder never implies a completeness it doesn't have.
- **Hint gutter.** Fixed width, equal to the longest live hint. It shows `[h]`, with
  brackets dim and the hint in `bold #FFFF00`, the same as `apply_jump_hint_prefix`.
  - Rows with no hint (context, header, overflow) get blank padding.
  - In SEARCH mode, hints render dim-yellow: readable, but visibly inactive.
  - With a pending prefix, hints that don't start with it dim out, and matching hints
    emphasize their remaining character. The footer shows `a…`. There is no timeout.
- **Why-hidden cell (one glyph; most global hider wins).**

  | Glyph | Meaning                                         |
  | ----- | ----------------------------------------------- |
  | blank | visible                                         |
  | `◆`   | you are here                                    |
  | `⊘`   | hidden by the Agents query                      |
  | `◌`   | hidden by `I` (`hidden-by-i` phase)             |
  | `▭`   | inside a collapsed or isolated-away tribe panel |
  | `≡`   | inside a collapsed grouping banner              |
  | `▸`   | inside a collapsed fold                         |

  Each glyph gets its own stable color. The preview's why-hidden line repeats the glyph
  in the same color, which teaches the legend in context.

- **Tree rows.**
  - Tree guides (`├ └ │`) are colored by depth from `_TREE_DEPTH_COLORS`.
  - Type and step glyphs and colors are the Agents row's (clan, session, monitor, gate,
    proc `⚙`, workflow `≡`). Promote a private constant to public rather than importing
    a `_name` across modules (symvision).
  - Names show matched runs via `append_highlighted` with
    `match_style="bold underline <row color>"`, never the default `bold #FFD700`, which
    would compete with the hints.
  - A right-aligned status plus compact age uses the Agents status color mapping.
  - Hidden rows keep their hue at reduced intensity. Apply `dim` once and never stack
    dim on dim, so they stay readable.
- **Headers.**
  - Tribe-panel headers read `@tribe ──── N · H hidden`, in the tribe identity color
    (`models/tribe_display.py`), with `▭` when the panel is collapsed. There is one
    `All agents` header in merged-panel mode.
  - Grouping banners of the active grouping mode render as dim, non-selectable
    sub-headers.
- **Footer.** A mode-specific legend on the left, and on the right a flash/pending slot:
  `a…` in bold yellow, and `no hint ‹x›` in `#FF8700`.
- **Responsive layout.**
  - 140 columns or more: list 44%, preview 56%.
  - 100–139 columns: 50/50, and the list drops its status column.
  - Under 100 columns: the preview stacks below the list at 40% height.

  Toggle this with container CSS classes from `on_resize`. Re-render rows only when the
  class actually changes.

### Reliability rules

1. **Snapshot on open.** The row set, order, and hints are fixed for the modal's
   lifetime. Background refreshes never reshuffle rows or hints. Hints change only when
   the finder's own query changes.
2. **Only reachable rows get hints.** The ladder re-validates identity at jump time.
3. **Identity crosses the modal boundary, never an index.** The dismiss result is
   `NodeFinderResult(identity, name)` or `NodeFinderResult(back=True)`.
4. **The ladder is authoritative.** The snapshot's classification is display-only; the
   jump works even if the classification was wrong.
5. **Programmatic `OptionList.highlighted` writes are guarded.** Use a flag cleared
   synchronously in `finally:` (TUI perf rule 12).
6. **The finder mutates committed view state only through the ladder**, and always
   announces it.

### Performance budgets (2,000 synthetic nodes, enforced by a bench)

| Path                                        | Budget                                                            |
| ------------------------------------------- | ----------------------------------------------------------------- |
| `"` to first paint (snapshot, list, Tier 0) | < 50 ms p95                                                       |
| Keystroke to refiltered list                | < 16 ms p95                                                       |
| `ctrl+n`/`ctrl+p` to highlight plus Tier 0  | < 16 ms p95                                                       |
| Tier 1                                      | Off the pump, debounced 150 ms, LRU-cached, stale results dropped |

- Emit `tui_trace` spans (`src/sase/ace/tui/util/trace.py`): `node_finder.open`,
  `node_finder.filter`, `node_finder.preview`.
- Start with `OptionList`: highlighting, scroll-into-view, and disabled context rows
  come for free.
- If the bench misses the keystroke budget, coalesce refilters latest-wins with a
  generation guard and thin synchronous callbacks. Tab must flush any pending refilter
  before HINTS mode accepts a hint.

## Phase `finder-model`: Node Finder row model, snapshot, filtering, and hints

**New `src/sase/ace/tui/models/node_finder.py`** (pure; no Textual imports):

- `NodeFinderReason` (`StrEnum`): `FOLDED`, `BANNER`, `PANEL`, `QUERY`, and `NON_RUN`
  (reserved for `hidden-by-i`).
  - It has a precedence order for the single row glyph:
    `QUERY > NON_RUN > PANEL > BANNER > FOLDED`.
  - A row with no reasons is visible.
- `NodeFinderRow`, a frozen slots dataclass:
  - `role`: `panel` header, `group` sub-header, or `node`;
  - `identity`, and a read-only `agent` reference (used by the preview);
  - `name`, `title`, `kind_label`, `kind_accent`, `depth`, `panel_key`;
  - `parent_row` index;
  - `jumpable`, `reasons: frozenset[NodeFinderReason]`, `unmet_fold_count`;
  - `nearest_collapsed`, a label such as `clan research.8` or `session sase-169.3`;
  - `group_label`, `is_here`;
  - for headers: jumpable and hidden counts.
- `NodeFinderSnapshot`:
  - the row tuple and the here-row index;
  - counts: nodes, hidden, and query-hidden;
  - the committed query text, `query_incomplete`, the `I` hidden count, and
    `hint_overflow`.
- **Naming.**
  - `node_finder_name(agent)` mirrors the name the Agents row shows: the clan name for
    clan containers, the proc label for proc shells, otherwise
    `presented_agent_name or agent_name or display_name or humanize_cl_name(cl_name)`.
  - `node_finder_title(agent)` returns the displayed title (`agent_tree_title`) when it
    differs from the name.
  - Unit-test both on clans, sessions, agent shells, monitors, gates, workflow roots,
    agent steps, and proc shells.
- **Kind.** `node_finder_kind(agent)` returns `("CLAN", "#D75FFF")` for clan containers
  and otherwise delegates to `identity_kind_for_agent`
  (`widgets/prompt_panel/_identity_header.py`), which covers SESSION, PROC SHELL, AGENT
  SHELL, GATE, MONITOR, STEP, WORKFLOW, and AGENT.
- **Jumpability.** `node_finder_jumpable(agent)` is false when
  `agent.is_pre_prompt_step` is set, or when `agent.is_workflow_step_child` is set and
  `step_type != "agent"`. A synthetic future step kind is therefore excluded too.
- **Filtering.**
  `filter_node_finder(snapshot, query, *, previous=None) -> NodeFinderView`:
  - Tokenize on whitespace; every token must match (AND). Per token, take the better of
    the name and the title haystacks via `fuzzy_match`, which is case-insensitive.
  - Evaluate each `(token, haystack)` once. A tier of 2 or less is contiguous, and any
    match (tier 3 included) counts for the relaxed pass.
  - Pass 1 keeps rows where every token is contiguous. If that is empty, the relaxed
    pass keeps rows where every token matches at all, and `relaxed` is set.
  - Stop evaluating a row at its first failing token.
  - **Incremental narrowing.** When the new token list refines `previous`, restrict
    candidates to `previous`'s any-match set. A refinement means earlier tokens are
    equal and the last token extends the old last token, or one token is appended. Both
    passes are monotone under refinement, so this is exact.
  - **Survivors keep tree order.** Their ancestor rows and their panel and group headers
    are kept as dim, non-jumpable context rows. Recompute the tree guides for the
    displayed subset so `└` stays correct.
  - **Best match.** It minimizes (worst token tier, name-before-title, −total score,
    display order). If the prior highlighted identity survives, keep it only when it is
    also the best match.
  - **Hints.**
    `build_jump_hint_maps(jumpable_identities_in_display_order, prefix_free=True)` over
    non-context jumpable rows only, mapping hint → identity. Rows past 3,844 get no hint
    and set `overflow`.
  - An empty query returns every row, with hints over all jumpable rows.
- **Cursor and text helpers.**
  - `next_jumpable_index(view, index, direction)` wraps and skips non-jumpable rows.
  - `node_finder_reason_text(row, query)` and `node_finder_action_text(row, query)`
    build the preview's two status lines. Wording is exact, so tests can assert it:
    - `◆ You are here`
    - `⏎ selects it`
    - `▸ Inside collapsed clan research.8` / `⏎ expands 2 folds, then selects it`
    - `≡ Inside collapsed group <label>` / `⏎ opens its group, then selects it`
    - `▭ Inside hidden panel @<tribe>` / `⏎ opens @<tribe>, then selects it`
    - `⊘ Hidden by the Agents query ‹<query>›` /
      `⏎ clears the Agents query, then selects it`
    - Combined actions join in order, for example
      `⏎ clears the Agents query, expands 1 fold, then selects it`.

**New `src/sase/ace/tui/actions/agents/_node_finder_snapshot.py`.** It is owner-aware
and mirrors `actions/agents/_prospective_clan.py`.
`build_node_finder_snapshot(owner) -> NodeFinderSnapshot`:

1. **Complete set.** `complete = owner._agents_with_children`: loaded, non-dismissed,
   clan-projected, and taken before folds and before the query.
2. **Universe.**
   - Set every fold key in `complete` (`agent_fold_key`) to `FULLY_EXPANDED` in a
     read-only fold projection, then run `filter_agents_by_fold_state`. This keeps the
     `hidden_only_parents` structural exclusion. Promote `_FoldStateProjection` from
     `_prospective_clan.py` to a public name rather than importing a private class
     across modules.
   - Build `AgentPanelGroup.from_agents(expanded, merge_tribe_panels=...)`.
   - For each `panel_key` in order, take `agents_for_panel(...)` (which already excludes
     STARTING rows) and
     `build_agent_tree(..., fold_registry=GroupFoldRegistry(), mode=owner._grouping_mode)`.
   - Walk the entries to emit the panel header, group sub-headers, and node rows in tree
     order. Deduplicate by identity.
3. **Query set.** `_apply_active_agent_query(owner, expanded)` from
   `_prospective_clan.py`, promoted to a public name. It gives the identities that
   survive the committed query on both flag branches, reusing the cached facade. It is
   skipped when no query is active.
4. **Rendered set.** The identities of the `("agent", idx)` targets from
   `owner._jump_candidate_targets()`.
5. **Reasons per node row.**
   - `PANEL`: `panel_key in effective_panel_collapses(owner)`.
   - `BANNER`: any enclosing group key is collapsed in
     `panel_fold_registry(owner, panel_key)`.
   - `FOLDED`: there are unmet ancestor folds.
   - `QUERY`: a query is active and the identity is not in the query set.
   - A row in the rendered set is visible regardless of any computed reason. Log a
     contradiction at debug level.
   - A row not rendered that has no computed reason is **omitted**, because its hider is
     unknown and a hint must never lie.
   - A non-jumpable step row stays only as a context row, and only while it has a
     jumpable descendant; otherwise it is omitted. A panel or group header with no rows
     beneath it is omitted.
6. **Fold requirements.** Compute them in O(n · depth) with one `tree_parent_lookup`.
   - Refactor `_agent_reveal._ancestor_requirements` to accept a precomputed parent
     lookup.
   - Expose a public batch helper, for example
     `unmet_ancestor_folds(complete, fold_manager) -> dict[identity, tuple[str, ...]]`.
     It reuses the exact rule: clan folds `EXPANDED`, hidden steps `FULLY_EXPANDED`,
     judged by `_fold_requirement_is_met`.
   - `nearest_collapsed` names the nearest collapsed ancestor by kind word and name.
7. **Here row.** It is the selected agent identity when `current_tab == "agents"`,
   `_current_group_key is None`, and the index is valid. If a banner or panel is
   selected instead, there is no here row, and the initial highlight becomes the first
   jumpable row of the focused panel. If there is no such row, it is the first jumpable
   row.
8. **Remaining fields.** Record `query_incomplete` from
   `owner._agent_load_state.query_incomplete`, and the `I` hidden count from
   `owner._hidden_count` when `hide_non_run_agents` is set.

**Tests** (new `tests/ace/tui/test_node_finder_model.py` and
`tests/ace/tui/test_node_finder_snapshot.py`; reuse fixtures from
`tests/ace/tui/_member_jump_navigation_helpers.py` and the clan/session visual
fixtures):

- **One classification per class:**
  - visible;
  - a collapsed-clan member;
  - a collapsed-session shell;
  - a monitor and a gate gated by the session fold;
  - a collapsed banner;
  - a collapsed panel and an isolated panel;
  - query-filtered, on both `override_flags(agents_unified_query=...)` states;
  - folded and query-hidden at once;
  - a remote fleet row, which is listed.
- **Exclusions:**
  - bash, python, parallel, and pre-prompt steps are context rows only with jumpable
    descendants, and omitted otherwise;
  - a synthetic unknown step kind is excluded;
  - STARTING, `hidden_only_parents`, and dismissed rows are omitted.
- **Order and identity:** identities are unique, and with an empty query and everything
  expanded the order equals the tab's tree order.
- **Filtering:**
  - token AND;
  - contiguous before relaxed;
  - deterministic ranking and best match;
  - context ancestors kept;
  - incremental narrowing equals a full evaluation.
- **Hints:** prefix-free at 61, 62, 63, 3,844, and more than 3,844 jumpable rows, with
  overflow flagged. Hints map to identities.
- **Cursor:** wrapping skips context rows.
- **Text:** exact reason and action wording.

## Phase `jump-ladder`: Identity jump ladder with an announced query clear

1. **New `src/sase/ace/tui/actions/navigation/_row_reveal.py`.**
   - Move `_reveal_agent_row`, `_restore_member_jump_history`, and
     `_notify_member_reveal_failure` out of `_member_jump.py` into an
     `AgentRowRevealMixin` that `MemberJumpNavigationMixin` inherits. Keep method names
     and behavior, so the member-roster and Procs-tab callers are untouched.
   - Add `_try_reveal_agent_row(identity) -> AgentRevealFailure | None`, which does the
     same work without notifying. `_reveal_agent_row` becomes a thin notifying wrapper.
2. **Extract the history transition.** Pull
   `_record_agents_live_query_transition(old, new)` out of `_commit_agents_filter_query`
   (`actions/agents/_filter_bar_session.py`) and reuse it there.
3. **New `src/sase/ace/tui/actions/navigation/_node_jump.py`** with a
   `NodeJumpNavigationMixin`, mixed into `AdvancedNavigationMixin`
   (`actions/navigation/_advanced.py`).
   - `_clear_agents_query_for_navigation() -> str | None` returns the cleared query
     text, or `None` when no query is active.
     - **Unified branch** (`agents_unified_query_enabled()`): close an open filter-bar
       session first, record the `agents-live` transition, call
       `_record_explicit_agents_query_commit("")`, drop the committed live facade, call
       `_refilter_agents()`, then `_schedule_agents_async_refresh(source="filter")`.
     - **Legacy branch:** `_record_explicit_agents_query_commit("")`,
       `_refilter_agents()`, then `_schedule_agents_async_refresh(source="filter")`.
   - `_jump_to_node_identity(identity, *, name) -> bool`:
     1. Return early if `_guard_agent_navigation_for_artifact_file_viewer()` blocks.
     2. `failure = _try_reveal_agent_row(identity)`. On success, return `True` with no
        toast: the landed row speaks for itself.
     3. If the failure is `TARGET_FILTERED` and a query is active: clear it, retry once,
        and on success toast.
        - Unified:
          `Cleared Agents query ‹q› to reach <name> — <f> then <^> restores it`, with
          key displays read from the keymap registry.
        - Legacy: `Cleared Agents query ‹q› to reach <name>`.
        - Truncate `q` to about 40 characters.
     4. Otherwise, call `_notify_member_reveal_failure(failure, subject="Node")`. A
        vanished or ambiguous identity leaves navigation state and the `ctrl+o` stacks
        untouched.
   - Keep the display honest after a query clear. The final Agents list must not be
     stale, and it must not rebuild twice when the existing helpers allow once.
     `_refilter_agents(refresh_display=False)` followed by the reveal's own refresh is
     the expected shape; force one `_refresh_agents_display(list_changed=True)` only if
     the reveal did not rebuild.
4. **Tests** (new `tests/ace/tui/test_node_jump_ladder.py`, through a real `AcePage`):
   - a visible row lands with no structural rebuild;
   - clan, banner, and panel layers all expand in one reveal;
   - a monitor or gate shell behind a collapsed session lands;
   - a query-hidden target clears the query, records the history transition (unified),
     toasts, and lands, on **both** flag states;
   - a vanished identity toasts `Node roster changed; jump cancelled` and rolls back the
     anchor;
   - `ctrl+o` after a ladder jump returns to the prior row;
   - the existing member-jump and procs-jump tests still pass unchanged.

## Phase `finder-preview`: Two-tier Node Finder preview

**New `src/sase/ace/tui/modals/node_finder_preview.py`: Tier 0.** It is pure Rich
rendering with zero I/O. It paints synchronously with every highlight change, together
with the highlight, in under 16 ms.

`render_node_finder_preview(row, snapshot, query) -> Text` produces:

1. **Kind chip.** The kind label and name, styled like the Agents identity header's kind
   label in the row's accent.
2. **Compact identity.** Use `build_agent_compact_lines(agent=...)`. For clans, use
   `build_clan_compact_lines` with counts from `aggregate_clan_in_memory`. For workflow
   roots, use `build_workflow_compact_lines`. All come from
   `widgets/prompt_panel/_identity_header_compact.py` and
   `_agent_display_clan_identity.py`.
3. **Breadcrumb.** `@tribe ▸ clan ▸ session ▸ node`, built from snapshot parent rows:
   the tribe in its identity color, dim `▸` separators, and the last segment in bold.
4. **Status lines.** The why-hidden line (glyph-colored) and the `⏎ …` action line, both
   from `finder-model`.
5. **Kind section.**
   - Session: a SHELLS lane, one line per shell: glyph · name · kind · status · runtime,
     taken from the snapshot's child rows.
   - Clan: a MEMBERS roster with status counts and up to about 12 members.
   - Monitor or proc shell: the command and label, state, and exit code, plus OUTPUT ·
     tail from the in-memory `agent.proc_log_tail`, bounded with
     `tail_text_by_lines_and_chars` to about 12 lines.
   - Gate: the gate label and status.
   - Workflow root: step counts by status.
6. **Tier 1 slots.** For rows that have a Tier 1 source, add the fixed-height section
   headers PROMPT and REPLY · tail, with `⋯ loading` skeleton lines. The Tier 1 result
   replaces the skeleton lines in place, so the layout never jumps.

Section headers are uppercase in the accent color with a dim rule fill.

**New `src/sase/ace/tui/modals/node_finder_preview_loader.py`: Tier 1.**

- **Source.** `tier1_source(row, snapshot) -> Agent | None`.
  - Agent shells, agents, `agent` steps, and workflows with `appears_as_agent` use
    themselves.
  - A session container uses its newest agent-shell member. The section header is
    labelled `latest shell <name>`.
  - Clans, monitors, gates, and proc shells have no Tier 1 source.
- **Loader.** `load_node_finder_preview(agent) -> NodeFinderPreviewPayload`. It is
  **thread-only**.
  - Work on a shallow `copy(agent)`. If `projected_agent_waiting_for_hydration(copy)`,
    call `hydrate_projected_agent(copy)`. The live row is never mutated.
  - Read the prompt through `get_prompt_content(copy)`
    (`widgets/prompt_panel/_agent_display_content.py`), which uses the mtime-keyed
    artifact file cache.
  - Read the reply with the detail panel's precedence: timestamped chunks (joined), then
    live reply, then response, then chat response.
  - Bound the prompt head to about 8 lines or 2 KB. Bound the reply tail with
    `tail_text_by_lines_and_chars` to about 20 lines or 8 KB, and report the omitted
    counts as a dim `… N earlier lines` line.
  - Return a freshness token made of the source paths' mtime and size, stat-ed in the
    worker.
- **Cache.** `NodeFinderPreviewCache`: a modal-local LRU of 128 entries, keyed by
  identity and storing `(payload, token)`. A revisit paints the cached payload
  instantly, then the debounced loader revalidates it and repaints only if the token
  changed.
- **Rendering.** `render_tier1(payload) -> Text` renders plain wrapped text: the prompt
  in a soft color and the reply in the default color. Markdown is a non-goal.

**Tests** (new `tests/ace/tui/test_node_finder_preview.py`):

- **Tier 0 purity.** Monkeypatch `builtins.open`, `os.stat`, `os.scandir`, and
  `pathlib.Path.stat` to raise while rendering every kind. Warm config-cached styles
  once before patching.
- **Content per kind:** the session lane, the clan roster, and the monitor tail.
- **Loader:**
  - it respects the bounds;
  - it hydrates only a copy, leaving the original untouched;
  - it follows the reply precedence;
  - the LRU evicts and hits.

## Phase `finder-modal`: NodeFinderModal screen, modes, keys, and layout

**New `src/sase/ace/tui/modals/node_finder_modal.py`**:
`NodeFinderModal(ModalScreen[NodeFinderResult | None])` and the frozen
`NodeFinderResult(identity: AgentIdentity | None = None, name: str = "", back: bool = False)`.

- **Constructor.** It takes a prebuilt `NodeFinderSnapshot`, `has_back: bool`, and an
  optional `preview_loader` for tests. It never reads app state after construction.
- **Compose.** Title, top bar (`FilterInput`, mode pill, scope strip), body
  (`OptionList` plus a preview `VerticalScroll`/`Static`), and footer (legend, plus a
  flash/pending slot).
  - `on_mount` focuses the `OptionList` and highlights the here row (or the initial row)
    under the programmatic-highlight guard.
  - It then paints Tier 0 and schedules Tier 1.
- **Rows.** Header, group, and context rows are disabled `Option`s. Row `Text` comes
  from a new `src/sase/ace/tui/modals/node_finder_rendering.py`, which also renders
  headers, the pill, the scope strip, and the legends. Keep per-row cached parts, so a
  pending-prefix change restyles only the hint gutters.
- **Keys.** Implement the interaction table exactly.
  - HINTS handling lives in the screen's `on_key`, via `normalize_jump_key` and
    `match_jump_hint`, and it stops every key.
  - SEARCH relies on the focused `Input` for printable keys.
  - `on_input_changed` refilters with `filter_node_finder(..., previous=...)`, rebuilds
    the options, and moves the highlight to the best match.
  - `Input.Submitted` and `OptionList.OptionSelected` both jump.
  - Enter with no rows flashes `No matching node`.
- **Cursor.** It is wrap-aware via `next_jumpable_index`. Guard programmatic highlight
  writes, and ignore echoed `OptionHighlighted` while the guard is set.
- **Preview.** Tier 0 runs synchronously on each highlight change. Tier 1 goes through a
  modal-owned `DetailPanelDebouncer(self.app, delay_s=0.15)`. Its callback is thin: it
  calls `spawn_pump_free_task(self, ..., registry_attr="_node_finder_preview_tasks")`,
  which runs `asyncio.to_thread(load_node_finder_preview, agent)`.
  - A generation counter drops stale results.
  - Re-capture the highlighted identity after the await (perf rule 4).
  - `on_unmount` cancels the debouncer and calls `cancel_pump_free_tasks(self)`.
- **Empty state.**
  - The list shows `No nodes match ‹q›`, and the preview shows
    `esc / tab back to hints · ^u clear`.
  - An empty snapshot is never opened; the wiring phase notifies instead.
- **Flash.** Use one `set_timer` handle, replaced on each new flash.
- **Styles.** A `styles.tcss` block next to `JumpAllModal`: container geometry, input
  focus colors, the divider between list and preview, the disabled-option override (so
  context rows keep their own dim styling), and the `-narrow`/`-medium` responsive
  classes.
- **Exports.** `NodeFinderModal` and `NodeFinderResult` go in `modals/_export_table.py`,
  `modals/__init__.py`, and `modals/__init__.pyi`.
- **Trace spans.** `node_finder.open` (compose to first paint), `node_finder.filter`,
  and `node_finder.preview`.

**Tests** (new `tests/ace/tui/modals/test_node_finder_modal.py`; use the
`_ModalHost(App)` pattern from `tests/ace/tui/modals/test_revive_agent_modal.py` with
synthetic snapshots):

- **Opening:** the `OptionList` has focus and the here row is highlighted.
- **Hints:**
  - a single-key hint dismisses with the identity;
  - a two-key hint goes pending, then completes;
  - Backspace and Esc cancel a pending prefix;
  - Esc with no prefix closes.
- **Invalid keys:** they flash, don't dismiss, and don't reach the host app. The host
  records keys; assert that nothing leaked.
- **Mode switching:**
  - Tab and Shift+Tab round-trip focus;
  - `/` focuses the query;
  - Esc in SEARCH returns to HINTS with the query kept;
  - Tab after typing re-allocates shorter hints.
- **Navigation:**
  - Enter works in both modes;
  - `ctrl+n`, `ctrl+p`, `↑`, and `↓` wrap and skip context rows in both modes;
  - clicking a row dismisses with its identity.
- **Back:** `"` returns `back=True` when `has_back`, and flashes otherwise.
- **Stability:**
  - mutating the source agents after open changes no hint;
  - an echoed `OptionHighlighted` never repaints a stale preview;
  - Tier 1 drops stale results, hits the LRU on revisit, and is cancelled on unmount.

## Phase `finder-wiring`: Keymap, action, docs, visual goldens, and bench

1. **Key validation.** In `keymaps/key_validation.py`, add `"quotation_mark": '"'` to
   `_KEY_DISPLAY` and the `'"' → "quotation_mark"` alias to `_KEY_ALIASES`.
2. **Keymap field.** In `keymaps/app_keymaps.py`, add `jump_to_node: str` next to
   `jump_to_all_entries`.
3. **Default config.** In `src/sase/default_config.yml` under `ace.keymaps.app`, add
   `jump_to_node: "quotation_mark"` with a comment:
   `# Agents only: Node Finder — jump to any node, including hidden ones`. The project
   gotchas require default-config updates for keymap changes, and the registry raises at
   startup if the entry is missing.
4. **Binding metadata.** Add `("jump_to_node", "Jump to Node", False)` to
   `keymaps/metadata.py`, and
   `Binding("quotation_mark", "jump_to_node", "Jump to Node", show=False)` to
   `tui/bindings.py`.
5. **Command palette.** In `commands/_app_metadata_nav.py`, add
   `("jump_to_node", "Find and jump to any node (even hidden)", "Navigation", AGENTS_ONLY, ("node", "finder", "find", "hidden", '"'))`.
6. **Availability.** In `_app_action_availability.py`, `jump_to_node` is available only
   when all of these hold:
   - `current_tab == "agents"`;
   - the prompt bar does not own keys;
   - no `ModalScreen` is active;
   - `_agents_first_load_done` is set;
   - no Agents filter-bar session is open.
7. **Action.** Add `action_jump_to_node()` in `actions/navigation/_node_jump.py`:
   1. Cancel any entry-jump or member-jump pending state (`_exit_entry_jump_mode` and
      `_cancel_member_jump_pending`).
   2. Run the artifact-file-viewer guard.
   3. Build the snapshot. If it has no jumpable rows, notify `No nodes to jump to`.
   4. Push `NodeFinderModal(snapshot, has_back=...)`. Compute `has_back` with the same
      predicate the `'` footer uses (`_entry_jump_footer_has_back()`), or'd with
      `bool(_link_trail)` (the `$` link-follow trail that `ctrl+o` walks first), so `""`
      is offered exactly when `ctrl+o` would move.
   5. On dismiss: `None` → nothing; `back` → `action_jump_to_entry_fast()`; otherwise
      `_jump_to_node_identity(result.identity, name=result.name)`.
8. **Help.** In `modals/help_modal/agents_bindings.py`, add a Navigation row after the
   `` ` `` row: `(d(a.jump_to_node), 'Find any node, even hidden ("" back)')`.
9. **Docs** (`docs/ace.md`):
   - add a `"` row to the Agents Navigation table: "Find and jump to any node, including
     hidden ones (see Node Finder)";
   - add a `## Node Finder` section next to `## Jump All Modal`, covering the modes, the
     key table, the glyph legend, what "hidden" covers, query clearing and restoring,
     and the `'` / `` ` `` / digits / `"` one-line jobs.
10. **Symvision.** Remove every `--epic-symbol` entry this epic added that now has a
    real consumer.
11. **Tests.**
    - Extend the keymap and catalog tests: `tests/test_keymaps_validation.py`,
      `tests/test_keymaps_app_bindings.py`, `tests/test_keymaps_registry_loading.py`,
      `tests/test_command_catalog.py`, and `tests/test_command_availability_scope.py`.
      - `quotation_mark` validates, displays as `"`, and has the alias.
      - The default config, registry, metadata, catalog, and help agree.
      - The action is unavailable off the Agents tab and while an editor owns keys.
    - Add an end-to-end `AcePage` test in `tests/ace/tui/test_node_finder_e2e.py`:
      1. `"` opens the modal;
      2. a hint jumps to a collapsed clan member, which is revealed and selected;
      3. `"` then `"` jumps back;
      4. an invalid key leaves the leader and custom modes unarmed.
12. **PNG goldens.** Add a new
    `tests/ace/tui/visual/test_ace_png_snapshots_agents_node_finder.py`. Use the
    clan/session fixtures, pin time, and stub the Tier 1 loader to return deterministic
    text.
    - `node_finder_hints_160x48`: mixed reasons, `◆`, and a loaded preview.
    - `node_finder_search_160x48`: `Tab`, then `cdx`.
    - `node_finder_pending_prefix_160x48`: at least 62 rows, with a pending prefix.
    - `node_finder_query_hidden_160x48`: `⊘` rows and the clears-query action line.
    - `node_finder_narrow_100x40`: the stacked layout.
    - `node_finder_no_results_120x40`.

    Run the targeted `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`,
    then inspect every created golden and the report. Also take a live `sase screenshot`
    of the real flow as a sanity check.

13. **Bench.** Add `tests/ace/tui/bench_node_finder.py` (`pytest -m slow`). It uses a
    synthetic 2,000-node clan/session tree and asserts the budget table: open, keystroke
    refilter including the `OptionList` update, and highlight plus Tier 0. If a budget
    misses, apply the coalescing fallback from the design section before landing.

## Phase `hidden-by-i`: Rows hidden by I, with a flip-and-reveal rung

1. **Pre-hide roster** (in `_node_finder_snapshot.py`). When `hide_non_run_agents` is
   set and `owner._hideable_agents` is non-empty, build the universe from an in-memory
   roster:
   - union `owner._agents_local_with_children` with the `_hideable_agents` rows, deduped
     by identity;
   - drop identities in `_dismissed_agents`;
   - re-project through the same helpers `_refilter_agents` uses:
     `filter_explicitly_removed`, then `_agents_source_for_current_mode`.

   No loader or disk change. Rows present in this roster but absent from
   `_agents_with_children` gain `NodeFinderReason.NON_RUN` (`◌`). Their other reasons
   are computed as usual. A test must prove that the re-projection places hidden clan
   members under their clan and matches the tree the tab shows after pressing `I`.

2. **Scope strip.** Replace `I hides K` with the listed `◌` count. Add the why-hidden
   and action lines `◌ Hidden by I (hide non-run agents)` and
   `⏎ shows agents hidden by I, then selects it`.
3. **Ladder rung** in `_jump_to_node_identity`, placed before the query rung:
   - It applies when the failure is `TARGET_MISSING`, the identity is among
     `_hideable_agents`, and `hide_non_run_agents` is set.
   - Add a helper, `_show_hidden_agents_for_navigation(on_complete)`, sibling to
     `_toggle_hide_non_run_agents`. It sets `hide_non_run_agents = False`, refilters,
     and calls `_schedule_agents_async_refresh(source="filter", on_complete=...)`.
   - Toast: `Showing agents hidden by I to reach <name> — press I to hide them again`.
   - In `on_complete`, re-capture state first (perf rule 4). Continue only if the tab is
     still Agents and the selected identity equals the one captured at dismiss.
     Otherwise toast `Jump to <name> cancelled — you moved` and stop.
   - Then run the ladder again. It can still clear the query.
4. **Tests.**
   - Snapshot: `◌` rows sit in tree position, and dismissed or explicitly removed
     hideable rows stay out.
   - Ladder: a hidden running `%hide` agent is reachable, the toast is shown, and `I` is
     left off.
   - Moving away before the reload lands cancels the jump.
   - An `I`-hidden row that is also query-hidden takes both rungs.
   - Update the affected goldens: extend `node_finder_hints_160x48` or add
     `node_finder_hidden_by_i_160x48`, and inspect the report.

## Verification (every phase)

- Run `just install` first if the workspace's virtualenv is stale. Run `just fix` (or at
  least `just fmt`), then `sase tool run check`. Do not run `just check-full` unless a
  bead or the user explicitly asks for it.
- TUI PNG goldens are not part of `just check`. `finder-wiring` and `hidden-by-i` run
  targeted `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`. Every
  golden creation or update must be inspected; generation is not approval.
- Run the bench with `pytest -s -m slow tests/ace/tui/bench_node_finder.py` in
  `finder-wiring`, and again in `hidden-by-i`.

## Non-goals and follow-ups

- Out of scope, and driven by feedback later:
  - viewport-relative hints;
  - recalling the last finder query on reopen;
  - Markdown rendering of reply tails;
  - a `:jump <name>` command-line verb over the same snapshot.
- No SASE memory changes are part of this epic. A glossary strand for "Node Finder" can
  be proposed separately once the feature lands.
- Out of scope, because it is a revive (a mutation): dismissed rows. The revive modal
  covers them.
