---
tier: epic
title: Pinned decision cards for the Plan Review gate panel
goal: 'Plan and epic review modals render decisions as readable cards that never clip:
  every decision stays on screen, the current card shows its full question, options,
  and rationale, the rail grows with the terminal, a z sheet gives a full-width reading
  view, and the inbox gate card matches, with no changed keys and no sase-core change.

  '
decisions:
  rail_cap:
    ask: How wide may the decisions rail grow on very wide terminals?
    choices:
      cap84: Cap at 84 cells; prose lines stay readable and the plan pane keeps room
        for code
      cap100: Cap at 100 cells; the rail fills more of a ~265-column terminal, the
        plan pane less
    default: cap84
    why: past ~76 inner cells prose lines get too long to read comfortably
    answer: cap84
  narrow_sheet:
    ask: Below 100 columns, should plans with 3+ decisions open straight into the
      z sheet?
    default: false
    why: starting docked keeps the plan visible on open, and z is one key away
    answer: false
  focus_color:
    ask: Which colour should mark the current decision card?
    choices:
      cyan: The modal's cyan (#5FD7FF), as prototyped; matches the Plan Review title
      accent: The theme's $accent; follows theme changes but was never prototyped
    default: cyan
    why: it is the prototyped, screenshot-verified colour and matches the modal title
    answer: cyan
phases:
- id: cards
  title: Decision cards replace the Button accordion
  depends_on: []
  size: medium
  description: 'cards: replace each decision Button with a wrapping DecisionCard built
    from pure Rich card builders, make the decisions rail a plain Vertical whose list
    fills to the Verdict, add the focus bar, answer chips, new header copy, mouse
    selection, once-per-keypress sheet computation, the syncthing and max-copy fixtures,
    the no-clip pilot invariant, and the regenerated decision goldens.'
- id: rail
  title: Responsive rail, one-line Actions, and pinned density
  depends_on:
  - cards
  size: medium
  description: 'rail: size the decisions rail at 40% (min 50, capped per rail_cap),
    dock it at 62% below 100 columns, collapse Actions to one line, drop reverse-video
    focus from one-line rail controls, and implement the density and pinning rule
    so every decision stays visible and the current card scrolls inside its own slot.'
- id: sheet
  title: z decision sheet, keymap, and footer
  depends_on:
  - rail
  size: medium
  description: 'sheet: add the decision_zoom keymap (z) end to end, the full-width
    decision sheet with every card expanded and Esc-leaves-sheet, the narrow_sheet
    auto-open rule, the merged and focus-conditional footer hints, docs, and the final
    large-terminal acceptance renders.'
- id: inbox
  title: Inbox gate card reuses the collapsed card
  depends_on:
  - cards
  size: small
  description: 'inbox: render the notification inbox plan-gate Decisions block with
    the read-only collapsed card builder (name left, answer chip right, dim question
    line, width-1 toggle glyphs) and regenerate its two goldens.'
- id: nudge
  title: Planner guidance against restating choice keys
  depends_on: []
  size: xsmall
  description: 'nudge: tell planners not to restate a choice key at the start of its
    label in the sase_plan skill source, the plan validate explain text, and the decision
    grammar docs; preview the generated skill without deploying it.'
proposed_by: bbugyi200.athena.research.4a.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-12 09:11:06
status: wip
bead_id: sase-1ka
---

- **PROMPT:** [prompts/202610/gate_panel_decision_cards.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/gate_panel_decision_cards.md)
- **BEAD:** [sase-1ka](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ka/README.md)

# Plan: Pinned decision cards for the Plan Review gate panel

## Context

Decisions in the Plan Review / Epic Review modal (`PlanApprovalModal`) look bad and clip
text. A five-report research consolidation diagnosed it and recommended a design that
the user has accepted in full. Read it before starting any phase:

```bash
sase artifact read file:explicit:22a22e31803043d4424dddfc "Accepted design spec for gate-panel decision cards"
```

Working prototypes (real Textual renders of `PlanApprovalModal` with a card list swapped
in) live in the research sidecar under `202610/gate_panel_decision_cards/`
(`plan_gate_decision_cards__cld_prototype.py`,
`gate_panel_decision_cards__final_pinned_prototype.py`, and their PNGs). Open that
sidecar with `/sase_repo` (`sase repo open research`) to consult them; they are
references, not code to copy verbatim.

Root causes, all confirmed in code:

1. Each decision is a Textual `Button` whose label is the whole multi-line card.
   `Button.get_content_width()` is the longest line + 2 and the CSS is `width: auto`, so
   the focused row is ~103 cells wide in a 44-cell viewport and is clipped sideways. The
   Python `[:77] + "..."` truncation never shows (`plan_decision_rows.py`).
2. Button `:focus` applies `b reverse` span by span, so `dim` spans turn grey: the
   ragged staircase.
3. `.plan-decision-rows { height: auto; max-height: 16 }` hides the fourth decision
   behind a scrollbar above ~28 empty rows (`styles.tcss`).
4. A `VerticalScroll` list inside a `VerticalScroll` rail (`scrollbar-gutter: stable`)
   gives two 2-cell gutters with text under the scrollbar.
5. The rail is a fixed 50 cells at every terminal width; Actions take 5 rows; the footer
   wraps even at ~265 columns; `why` reads like a third option; a redundant
   `default: db` line; `🧠 0` shows when zero; `◉` on every collapsed row.

Real-world sizes (28 plans, 47 decisions): p50 is **one** decision per plan (max 5);
asks p90 106 chars (grammar max 120); labels p90 86 (max 100); `why` p90 92 (max 100);
ids p90 23 (max 32); choice keys max 24; choices per decision 2–5 by grammar. 19 of 47
are memory toggles. The design must look calm with one decision and hold up at five.

Everything here is presentation-only Textual code in this repo. The Decision Sheet,
defaults, provenance, and summary sentence already come from `sase-core` through
`sase.sdd.plan_decisions`; the Rust boundary is untouched.

## Non-goals

- No `sase-core` change, and no change to the durable gate spec (the edit action's
  `description` in `src/sase/_plan_gate_metadata.py` stays as is for Telegram, CLI, and
  custom gates).
- No key changes besides the new `z`. Enter keeps "approve as shown"; digits keep
  submitting numbered branches with the current values; Enter never edits a value.
- The PLAN lane and `sase plan show` (shared `src/sase/sdd/_plan_display_decisions.py`
  builders), Telegram, and the CLI keep their glyphs (`☑️`/`⬜`, `◉ ○ ★ ● 🧠`).
- Custom and sudo gate rails keep their look (42 cells, `VerticalScroll`); they must not
  copy any Button-accordion pattern.
- The document pane is unchanged: fold, callout tint, and focus-follow (focusing a
  decision scrolls to its callout) keep working. The optional "answering <id>" caption
  is not built.
- No `RadioSet` / `SelectionList` / `OptionList`: they capture arrows, Space, and Enter.
- No feature flag: this replaces presentation with no old branch to keep reachable, and
  each phase leaves a fully working modal.

## Shared design spec

Every phase implements against this spec. Mocks show the current card with a `▌` bar.

### Layout (wide, 100 columns or more, plan with decisions)

```text
┌ rail: 40% · min 50 · max 84 ─────────────┐ ┌ plan.md (1fr) ──────────────┐
│ e ✏️ Edit plan · validated on save        │ │ …                           │
│ Decisions · 4 · 1 changed                │ │ > [!decision] root = legacy │
│▌root                         ‹ legacy ● ›│ │                             │
│▌Where should the relocated legacy tree   │ │                             │
│▌(formerly ~/Sync) live?                  │ │                             │
│▌ ○ db ★                                  │ │                             │
│▌   ~/db: matches the existing $DB/$DBB/  │ │                             │
│▌   $DBH variables in ~/.profile          │ │                             │
│▌ ◉ legacy                                │ │                             │
│▌   ~/legacy: names it as the pre-chezmoi │ │                             │
│▌   legacy tree; $DB is repointed at it   │ │                             │
│▌ ★ why: ~/.profile already calls this    │ │                             │
│▌   root $DB, and a ~/ sibling makes link │ │                             │
│▌   retargets one-component swaps         │ │                             │
│  dotfiles                      repoint ★ │ │                             │
│  Should the ~70 stow-style dotfile sy…   │ │                             │
│  conflicts                      delete ★ │ │                             │
│  old_tmp                          nest ★ │ │                             │
│        (list = 1fr, fills to Verdict)    │ │                             │
│ Verdict                      (docked)    │ │                             │
│ ☑ 🚀 Launch coder  ☑ 💾 Commit plan       │ │                             │
│ 1 ✅ Tale  2 ❌ Reject  3 💬 Feedback      │ │                             │
│ → coder + commit · root=legacy · …       │ │                             │
└──────────────────────────────────────────┘ └─────────────────────────────┘
 j/k=Navigate  h/l=Choice  Space=Cycle  r/R=Reset  Enter=Tale · 1 change  z=Sheet  …
```

- One scroll region per card slot; the decisions rail itself is a plain `Vertical`.
  Scrollbars are 1 cell (`scrollbar-size-vertical: 1`, `scrollbar-gutter: auto`, muted
  colours). Wrap widths are measured inside any gutter, so text never runs under a
  scrollbar.
- Decision-free plans keep the 44-cell rail and generic gates keep 42.

### Card anatomy

Every card shares one header row: the decision id on the left (bold; the focus colour on
the current card; `🧠 ` prefix for memory decisions) and the answer chip right-aligned.
The id may ellipsize; the chip never does.

- Chip at the default: `value ★` (★ muted). Changed: `value ●` with value and `●` gold
  (`#FFD700`). Toggles show `☑ yes` / `☐ no` (U+2611 / U+2610, no VS16, width 1).
- The current, editable card wraps the chip in muted `‹ … ›` ("h/l changes this").

Collapsed card, two lines (when there is room), and one line (when space is short):

```text
  dotfiles                                  repoint ★
  Should the ~70 stow-style dotfile symlinks into t…
```

The preview line is the ask, muted, one line, ellipsized. A memory row whose quote or
claim was not found previews `⚠ quote not found · off` / `⚠ claim not found · off` in
yellow instead.

Current card, choice, inner width below 60 (stacked options):

```text
▌ root                                     ‹ db ★ ›
▌ Where should the relocated legacy tree
▌ (formerly ~/Sync) live?
▌ ◉ db ★
▌   ~/db: matches the existing $DB/$DBB/$DBH
▌   variables in ~/.profile
▌ ○ legacy
▌   ~/legacy: names it as the pre-chezmoi
▌   legacy tree; $DB is repointed at it
▌ ★ why: ~/.profile already calls this root $DB,
▌   and a ~/ sibling makes link retargets
▌   one-component swaps
```

Current card, choice, inner width 60 or more (two columns, hanging indent):

```text
▌ root                                                        ‹ db ★ ›
▌ Where should the relocated legacy tree (formerly ~/Sync) live?
▌
▌ ◉ db ★    ~/db: matches the existing $DB/$DBB/$DBH variables in ~/.profile
▌ ○ legacy  ~/legacy: names it as the pre-chezmoi legacy tree; $DB is
▌           repointed at it
▌
▌   why ★   ~/.profile already calls this root $DB, and a ~/ sibling makes
▌           link retargets one-component swaps
```

Current card, memory toggle (the most common card):

```text
▌ 🧠 human_only_record                     ‹ ☐ no ★ ›
▌ Add a decisions record that remote repository
▌ creation (sase project create) is human-only?
▌   edits    decisions:project-create-human-only
▌            web strand · project · new
▌   default  off ★ — you did not ask for this edit
```

The `default` line by provenance (always visible, long quotes wrap below it):

| Provenance                 | Default line                                                        |
| -------------------------- | ------------------------------------------------------------------- |
| `not_asked`                | `off ★ — you did not ask for this edit`                             |
| `asked`                    | `on ★ — you asked: "…"`                                             |
| `quote_not_found` (yellow) | `⚠ off — "…" is not in your messages; turn on to allow`             |
| `claim_not_found` (yellow) | `⚠ off — "…" is not in the note; turn on to allow`                  |
| `correction`               | `on ★ — corrects a statement this plan makes false`, then the quote |
| `inherited`                | `on ★ — approved in the epic`                                       |

Current card, plain toggle (the chip carries the value; no yes/no rows):

```text
▌ keep_shim                                ‹ ☑ yes ★ ›
▌ Keep the compatibility shim for one release?
▌   why ★  removing it now breaks pinned plugins
```

Wrapping rules: prose wraps to the card's real content width; unbreakable paths and ids
fold (`overflow="fold"`); ellipsis appears only on collapsed previews and the header id;
silent clipping anywhere is a bug. The `default: <value>` line is gone (the `★` on the
default option says it). `why` is a dim italic footnote labelled `why ★`, never indented
like a choice.

### Styling tokens

| Token        | Rule                                                                                                                                                                                                                                                                               |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Current card | `border-left: outer <focus colour>` plus ~10% focus-colour tint while focused; when focus is elsewhere (Actions/Verdict) the card stays expanded with a muted bar and ~6% tint. Other cards get `border-left: blank` so nothing shifts. **No reverse video anywhere in the rail.** |
| `◉` / `○`    | Only inside the expanded card. The selected option is bold in normal colour; unselected options and their labels are dimmed.                                                                                                                                                       |
| `★` muted    | The planner's default: on the chip, on the default option, and as `why ★`.                                                                                                                                                                                                         |
| `●` gold     | Changed from the default; always paired with the words `N changed`.                                                                                                                                                                                                                |
| `☐` / `☑`    | Toggle value, width-1 glyphs, wherever the TUI aligns columns.                                                                                                                                                                                                                     |
| `🧠`         | Memory decision, only at the start of a line.                                                                                                                                                                                                                                      |
| Header       | `Decisions · 4`, plus `· 1 changed` when nonzero and `· 🧠 2` only when memory decisions exist.                                                                                                                                                                                    |

Questions and selected options use the normal foreground; dim is only for metadata and
unselected options. One border per pane; no box per card.

> [!decision] focus_color = cyan The focus colour is `#5FD7FF` with a muted bar of about
> `#3A6F80`; keep both as named constants next to the card builders and in
> `styles.tcss`.

> [!decision] focus_color = accent The focus colour is the theme's `$accent` in CSS.
> Rich-styled text that uses it (the current card's id) resolves the accent from the
> running app's theme and passes it to the builders as a style argument; builders stay
> pure.

### States

| State                              | What changes                                                                                                                                                                                |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| At default                         | The chip reads `value ★`.                                                                                                                                                                   |
| Changed                            | The chip reads `value ●` in gold, the header gains `· N changed`; choosing the default again clears it. The Enter badge is unchanged.                                                       |
| Current card, focus on the Verdict | The card stays expanded with the muted bar.                                                                                                                                                 |
| Unverified memory                  | Yellow `⚠` default line; the value stays `☐ no` until the reviewer turns it on.                                                                                                             |
| Settled elsewhere                  | Cards become read-only (grey bar, no `‹ ›`); the Decisions header is prefixed with the settled text (for example `Decided via Telegram · Decisions · 4`); `h`/`l`/Space keep today's toast. |
| Esc, then reopen                   | Unchanged: `esc_drafts` restores the values.                                                                                                                                                |
| No decisions                       | No Decisions section; the rail stays 44.                                                                                                                                                    |

### Density and pinning

The current card is shown in full or scrolls inside itself; every other decision stays
visible as at least one line. With `H` = decision list content height, `n` decisions,
and `E` = the current card's rendered height:

1. If `E` plus every other card at two lines fits in `H`, use two-line collapsed cards.
2. Else, if `E + (n − 1)` fits, use one-line collapsed cards.
3. Else, use one-line collapsed cards and cap the current card's slot at
   `H − (n − 1) − 1` rows (at least 1). The slot scrolls with a 1-cell scrollbar, and a
   1-row muted hint under it reads `↓ more` (the `sheet` phase appends ` · z: sheet`).

The list itself never scrolls a decision out of view. Recompute on resize, focus change,
rebuild, sheet toggle, and an edit that changes a card's rendered line count; never per
render.

### Keys

Existing bindings are unchanged: `j`/`k` walk Actions → Decisions → Verdict; `h`/`l`/←/→
step a choice or set a toggle; Space cycles a choice or flips a toggle; `r`/`R` reset;
Enter approves as shown; digits and Ctrl+S submit with the current values. Mouse:
clicking a card makes it current; clicking an option line in the current choice card
selects that option. New: `z` toggles the decision sheet (`sheet` phase).

### Test fixtures

**Syncthing fixture** (the screenshot plan; use these decisions verbatim):

```yaml
decisions:
  root:
    ask: Where should the relocated legacy tree (formerly ~/Sync) live?
    choices:
      db: "~/db: matches the existing $DB/$DBB/$DBH variables in ~/.profile"
      legacy:
        "~/legacy: names it as the pre-chezmoi legacy tree; $DB is repointed at it"
    default: db
    why:
      ~/.profile already calls this root $DB, and a ~/ sibling makes link retargets
      one-component swaps
  dotfiles:
    ask:
      Should the ~70 stow-style dotfile symlinks into the tree become real files in ~?
    choices:
      repoint:
        "No: keep them as symlinks retargeted into the tree (pure rename, lowest risk)"
      materialize:
        "Yes: move each linked file into ~ and rewrite its references; more invasive"
    default: repoint
    why:
      a pure rename fixes everything with one verifiable substitution; de-stowing can
      come later
  conflicts:
    ask:
      Should all ~400 Syncthing *.sync-conflict-* files under ~ (2020-2022) be deleted?
    choices:
      delete:
        "Yes: delete after writing a manifest; copies remain in hercules monthly/yearly
        backups"
      keep:
        "No: keep them (links retargeted); only markers, config, and the package are
        removed"
    default: delete
    why:
      they are Syncthing artifacts by definition, years stale, and still recoverable
      from hercules
  old_tmp:
    ask:
      Should the old 33 GB ~/tmp contents (reachable only via ~/bar) move to
      ~/tmp/old_tmp?
    choices:
      nest:
        "Yes: move them to ~/tmp/old_tmp and delete ~/bar, as the Oct 9 history
        attempted"
      keep: "No: leave them inside the relocated tree and just delete the ~/bar link"
    default: nest
    why:
      zsh history shows `mv ~/old_tmp ~/tmp/` on Oct 9 and `ls ~/tmp/old_tmp` on Oct 12
```

Pair it with a short body that references every decision (one `> [!decision]` callout
per branch is enough) so `build_plan_approval_gate_spec` accepts it.

**Max-copy fixture** at the grammar limits: five decisions; one id of exactly 32
characters; every ask exactly 120 characters; every label and `why` exactly 100
characters; one choice decision with five choices and a 24-character key; one label
containing an unbreakable ~60-character path; one plain toggle with a `why`; one memory
toggle last (verified `requested:` quote, built with `verified_memory=True`). Assert the
lengths inside the fixture module so it cannot drift below the limits.

Both fixtures live beside the existing ones in
`tests/ace/tui/visual/_ace_plan_decisions_png_fixtures.py` (split the module if it nears
the 700-line `toobig` threshold) and are built through `decision_gate_bundle`.

## Phase: cards

Replace the Button accordion with wrapping cards and fix the list height.

1. **Pure builders** in a new module, `src/sase/ace/tui/modals/plan_decision_cards.py`:
   the header row with chip, collapsed (one or two lines), and the expanded variants
   (choice stacked below inner width 60, choice two-column at 60 or more, memory toggle,
   plain toggle) exactly as in [Card anatomy](#card-anatomy). Inputs are the sheet row,
   its definition, the content width, and flags (current, read-only, focus style);
   outputs are Rich renderables (`Group` / `Table.grid`). Use `memory_note_names` /
   `provenance_chip` / format helpers from `sase.sdd._plan_display_decisions` where they
   fit; keep the TUI-only glyphs (`☐`/`☑`) local. Delete `_collapsed_row_text`,
   `_expanded_row_text`, the 60/80-character slicing, and the dead
   `plan-decision-detail-*` Statics.
2. **Widget** in `src/sase/ace/tui/modals/plan_decision_rows.py`: a
   `DecisionCard(Static, can_focus=True)` per decision with id `plan-decision-{i}` (keep
   it so `focus_gate_control`, `_sync_focused_decision`, and `visible_control_ids` keep
   working), each inside a per-decision slot `VerticalScroll` with id
   `decision-slot-{i}` (avoid the `plan-decision-` prefix the controls parse). Slots are
   uncapped in this phase. Keep `PlanDecisionRows`' public interface (`focused_index`,
   `focused_id`, `update_rows`, `rebuild`, `set_focused_index`, `visible_control_ids`);
   `rebuild` still mounts/removes slots when the id list changes (the
   `test_plan_decision_ace_stale` cases must stay green). Mark the current card with a
   `-current` class and settled cards with `-readonly`. Stop swallowing mount and query
   errors with bare `except Exception: pass`.
3. **Mouse.** Focusing a card by click must run the same path as keyboard focus
   (`_sync_focused_decision`, document focus-follow). Clicking an option line in the
   current choice card selects that option through `_apply_draft_edit` with
   `PlanDecisionDraft.set_value`. Either map the click row to an option from line
   offsets the builder reports, or render option lines as child widgets; keep the
   builders pure and unit-tested either way.
4. **Layout and CSS** (`plan_approval_modal_view.py`, `src/sase/ace/tui/styles.tcss`):
   when the plan has decisions the rail is a plain `Vertical` (same classes); decision-
   free rails stay `VerticalScroll`. The list is `height: 1fr` with no `max-height`;
   1-cell muted scrollbars on the list and slots; card, bar, tint, `-current`,
   `-readonly`, and `:focus { text-style: none; }` rules per
   [Styling tokens](#styling-tokens). Scope every new rule to `PlanApprovalModal` so
   custom and sudo gates are untouched. The rail width stays 50 in this phase.
5. **Header copy** per the Header token, re-rendered on draft edits (the changed count
   moves); settled prefix per [States](#states).
6. **Keypress performance** (read the `tui_perf` memory): compute
   `PlanDecisionDraft.sheet()` once per keypress in `_refresh_verdict_summary` and pass
   it to the summary sentence, the Enter badge, the header, and the rows (`full_summary`
   / `short_summary` accept an optional precomputed sheet). On `h`/`l`/Space re-render
   only the edited card plus header, summary, and footer; on `j`/`k` only the old and
   new current cards. No `validate_plan`, re-lex, or span recache on keypress
   (`test_draft_edit_avoids_revalidate_relex` stays green).
7. **Tests.**
   - Builder unit tests at content widths 46, 76, and 100 for every card kind using the
     syncthing and max-copy rows: no rendered line exceeds the width; every word of each
     ask, label, and `why` appears (nothing clipped); ellipsis only in the header id and
     collapsed preview; no `default:` line; `why ★` present; `☐`/`☑` measure one cell
     (`rich.cells.cell_len`); `🧠` only at line starts. Rewrite the
     `test_plan_decision_ace_draft.py` row-text tests against the new builders.
   - A pilot invariant with the real stylesheet (follow
     `test_compact_verdict_stays_inside_rail_with_stylesheet`): syncthing and max-copy
     at 120×40 and 200×56, for every focus index, every card lies horizontally inside
     the list's content region and no list or slot can scroll horizontally. Run it
     against the Button implementation first and confirm it fails, then make it pass.
   - A test that one `h` keypress calls `sheet()` once and re-renders only the edited
     card.
   - Update `test_compact_verdict_stays_inside_rail_with_stylesheet` for the new rail
     container type.
8. **Goldens** (read the `tui_screenshot` memory first): regenerate the existing
   `plan_gate_*decisions*` goldens (tale, memory, unverified, stacked 90×40, epic) and
   `plan_toast_tale_decisions_120x40` if it moves, as one reviewed group; add
   `plan_gate_tale_decisions_syncthing_120x40`,
   `plan_gate_tale_decisions_syncthing_200x56`, and
   `plan_gate_tale_decisions_maxcopy_120x40`. Inspect every image.
9. **Docs:** update the Plan Decisions paragraph in `docs/ace.md` for cards, chips, and
   mouse selection.

## Phase: rail

Give decisions room and pin every decision on screen.

1. **Responsive width** in `styles.tcss`: the decisions rail is
   `width: 40%; min-width: 50; max-width: 84` (about 50 at 120 columns, 61 at 160, 77 at
   200, 84 from ~220). Below 100 columns (`-gate-review-narrow`) the decisions rail
   docks at the bottom at `height: 62%` and full width; reset `min-width`/`max-width`
   there, or the cap would shrink the docked rail. Decision-free narrow rails stay 48%.

   > [!decision] rail_cap = cap84 Use `max-width: 84`.

   > [!decision] rail_cap = cap100 Use `max-width: 100` (the rail reaches it from ~250
   > columns).

2. **One-line Actions.** Add a `compact` mode to `GateActionControls` and to
   `GateActionsMixin._compose_gate_actions` (no "Actions" title, borderless one-row
   controls, keep the `gate-action-{i}` ids and the draft banner), used only by the plan
   modal. Each action renders as `e ✏️ Edit plan · validated on save`: the plan modal
   passes a short presentation caption for its own edit operation
   (`PLAN_EDIT_OPERATION_ID`); any other operation falls back to its declared
   description and wraps rather than clips. Custom gates keep today's Actions.
3. **No reverse video in the rail.** The compact action and the `#plan-verdict` one-line
   controls replace `text-style: reverse` on focus with bold plus a focus-colour tint;
   check that the primary branch's badge stays distinguishable when focused.
4. **Density and pinning** in `PlanDecisionRows` per
   [Density and pinning](#density-and-pinning): a pure, unit-tested function decides
   collapse lines and the current slot's cap from the list height and rendered card
   heights (measure with the builders at the card content width; at most five cards);
   the widget applies it to slot `max-height`, toggles the `↓ more` hint row, resets the
   current slot's scroll on focus change, and keeps the list scroll at the top.
5. **Tests.** Unit tests for the density function (all three branches, tiny `H`). Pilot
   invariants with the real stylesheet for syncthing and max-copy at 90×40 and 120×40,
   for every focus index: every `decision-slot-{i}` intersects the list region by at
   least one row; the current card's last line is reachable by scrolling its slot; no
   horizontal overflow. Rail width assertions at 120, 160, 200, and 262 columns
   (rail_cap decides the last), the narrow docked rail at full width at 90 columns, and
   custom gates still 42.
6. **Goldens:** add `plan_gate_tale_decisions_syncthing_90x40`,
   `plan_gate_tale_decisions_maxcopy_90x40`, and
   `plan_gate_tale_decisions_syncthing_262x72` (the user's terminal size); regenerate
   the decision group (Actions and widths moved). Inspect every image; the 90×40 renders
   must show all ids.

## Phase: sheet

Add the full-width reading view and quiet the footer.

1. **Keymap `decision_zoom: "z"`** end to end: `ace.keymaps.gate` in
   `src/sase/default_config.yml`, `GateModalKeymaps`
   (`src/sase/ace/tui/keymaps/app_keymaps.py`), `_GATE_BINDING_META`
   (`keymaps/metadata.py`), `src/sase/config/sase.schema.json`, the gate table in
   `docs/configuration.md`, and the keymap example plus Plan Approval Keybindings table
   in `docs/ace.md`. `CustomGateModal` gets a no-op `action_decision_zoom` like its
   other decision actions. `gate_modal_taken_keys` picks `z` up automatically; test it.
2. **Fold-prefix collision.** The app-level `z` fold-mode prefix
   (`keymaps/mode_keymaps.py`) must not fire inside the modal; a pilot test presses `z`
   and asserts the sheet toggled and fold mode did not start.
3. **The sheet.** `action_decision_zoom` (plans with decisions only) toggles a
   `-decision-sheet` class on the modal: `#plan-approval-scroll` is hidden; the
   decisions rail is undocked and takes the full body width and height (overriding both
   the wide and narrow rules); card text is capped at about 100 cells; every card is
   expanded; pinning is off and the list scrolls, keeping the current card visible; the
   Verdict and outcome sentence stay docked; `j`/`k`/`h`/`l`/Space/`r`/`R` keep working.
   Esc leaves the sheet first and a second Esc cancels as today (the check lives in
   `action_cancel`, so `q` behaves the same). The pinned `↓ more` hint becomes
   `↓ more · z: sheet`.

   > [!decision] narrow_sheet On mount, when the terminal is narrower than 100 columns
   > and the plan has three or more decisions, open in the sheet. Resizing later never
   > toggles it automatically.

   > [!decision] narrow_sheet = no The modal always opens docked; the sheet opens only
   > on `z`.

4. **Footer** (`plan_approval_footer.py`): merge the pairs into `h/l=Choice` and
   `r/R=Reset` (first key of each binding; the docs keep the alternates); show
   `h/l=Choice`, `Space=Cycle`, and `r/R=Reset` only while a decision card is focused
   (otherwise `Space=Toggle`); show `z=Sheet` whenever the plan has decisions and
   `z=Back to plan` inside the sheet. The Enter badge is unchanged. Refresh the footer
   on every focus change (keyboard and mouse) without recomputing the sheet (reuse the
   badge from the last draft refresh). At 262 columns the decisions footer fits on one
   line.
5. **Tests:** keymap defaults/schema/taken-keys; `z` toggles the sheet; Esc leaves the
   sheet then cancels; the `narrow_sheet` behavior; footer content with and without a
   focused decision and inside the sheet.
6. **Goldens:** add `plan_gate_tale_decisions_sheet_120x40` and
   `plan_gate_tale_decisions_sheet_90x40`; regenerate the decision group (footer moved).
7. **Acceptance renders.** Render the syncthing fixture through the real modal at 262×72
   (docked and in the sheet) and at 90×40 with the canonical renderer and inspect them
   against the research mocks. If the host has a pending plan gate with decisions, also
   capture it live with `sase screenshot` (read the `tui_screenshot` memory).

## Phase: inbox

Restyle `_plan_decisions_block` in `src/sase/ace/tui/modals/notification_modal_gate.py`.

1. Build each row from the gate summary (id, kind, value, changed vs effective default,
   ask, memory) and render it with the read-only two-line collapsed card from
   `plan_decision_cards.py`: name left, chip right (`☐`/`☑` for toggles), question as
   the dim second line; terminal (answered) gates use the same chip without arrows. Only
   import the builders; if a read-only need forces a builder change, keep it additive so
   the `rail` and `sheet` phases are unaffected.
2. The pending golden rendered `◉ grouping  pane ★` letter-spaced; confirm the new
   rendering has no such spacing artifact.
3. Tests for pending and answered blocks; regenerate
   `notification_gate_plan_decisions_pending_120x40` and
   `notification_gate_plan_decisions_answered_120x40` and inspect them.

## Phase: nudge

Today's labels open with `~/db: …`, `No: …`, or `Yes: …`, which wastes 4–8 cells and
shows the key twice.

1. Add one sentence of guidance (do not restate the choice key, or a `Yes:`/`No:`
   prefix, at the start of its label; the label states the consequence) to the Plan
   Decisions instructions in `src/sase/macros/skills/sase_plan.md`, the decisions
   paragraph in `src/sase/main/plan_explain.py`, and the decision grammar section of
   `docs/sdd.md`. It is guidance, not a validation rule.
2. Read the `generated_skills` memory. Preview with `sase skill init --diff`; do not
   deploy from this phase. After the epic lands, deploy from the landed tree with
   `sase skill init --force`.

## Verification (every phase)

- Read the `lint_and_test` memory, then run `sase tool run check`.
- Run targeted `just fix-tui-screenshots -- <selectors>` for the goldens the phase
  touches (through `/sase_monitor` if it is long), inspect the report and every changed
  image, and prove host independence with a targeted `--check` run twice, once with
  `TZ=UTC`, as the `tui_screenshot` memory requires.
- Do not run `just check-full`.
