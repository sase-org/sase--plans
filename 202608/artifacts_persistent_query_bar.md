---
tier: epic
status: done
title: Always-on query bar across every Artifacts sub-tab
goal: "Every Artifacts sub-tab that can be queried shows its query bar at all times, in
  one shared visual grammar, so pressing the query key never shifts the layout and the
  active query is always readable from the pane it filters.

  "
phases:
  - id: chrome
    title: Shared persistent query-bar chrome
    depends_on: []
    size: medium
    description: "chrome: give FilterBar a real idle presentation -- a profile-driven,
      syntax-highlighted closed display with an empty-query placeholder, accent-driven
      border/sigil/completion colors, and click-to-edit -- and adopt it on the two bars
      that are already persistent.

      "
  - id: panes
    title: Persistent query bar on Bead and File
    depends_on:
      - chrome
    size: medium
    description: "panes: make BeadFilterBar and FileFilterBar persistent, add a single
      `_sync_query_bar` funnel that keeps each idle bar's text and status truthful, and
      stop echoing the active query as chips in those panes' identity headers.

      "
  - id: documents
    title: Persistent query bar on Plan and every document provider
    depends_on:
      - panes
    size: medium
    description: "documents: make PlanFilterBar persistent for the Plan pane and every
      `ref:<kind>` provider pane, surface deep-archive coverage in the idle status lane,
      and delete the now-unused filter-token echo from the shared scope renderer.

      "
  - id: grammar
    title: Always-on invariant, conformance, and visual grammar docs
    depends_on:
      - documents
    size: small
    description:
      "grammar: add the cross-pane regression that every non-degraded resolved sub-tab
      mounts a visible, read-only query bar in its own accent, and rewrite the visual
      grammar doc's filter slot, state table, and Patch-asymmetry sections."
proposed_by: bbugyi200.athena.07r
bead_id: sase-qy
create_time: 2026-09-09 19:49:58
---

- **PROMPT:**
  [prompts/202608/artifacts_persistent_query_bar.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_persistent_query_bar.md)
- **BEAD:**
  [sase-qy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qy/README.md)

# Plan: Always-on query bar across every Artifacts sub-tab

## Why

The Artifacts tab has five or more sub-tabs and two different query experiences.

- **Stitch** and **Patch** show their query bar all the time. The bar is part of the
  pane; pressing `/` starts editing in place.
- **Bead**, **File**, **Plan**, and every installed `ref:<kind>` document provider hide
  their bar until `/` is pressed. Opening it pushes three rows of content down and
  closing it pulls them back up, so the list you were reading jumps twice per query.
  While the bar is hidden, the active query is echoed as dim chips at the end of the
  identity/scope header instead.

That split costs three things at once:

1. **Intuitiveness.** The same key produces a different physical experience per sub-tab,
   and on the hidden-bar panes the query lives in a header lane that looks like
   metadata, not like an editable control.
2. **Reliability.** Layout shift on `/` moves the selected row under the user's eyes,
   and the query is rendered twice (chips + bar) from two code paths that can disagree.
3. **Beauty.** The two already-persistent bars do not even agree with each other: Patch
   renders its idle query as a syntax-highlighted `Static` with a saved-slot chip, while
   Stitch renders a read-only vim text area that advertises a stray `[NORMAL]` border
   title. Provider document panes inherit `PlanFilterBar`'s pinned purple CSS, so a
   `ref:research` pane draws a Plans-colored bar.

This epic converges all of that on one grammar: **a query bar is a permanent part of
every Artifacts pane that can be queried**, it renders the committed query the same way
everywhere, and it is the single place the active query is displayed.

## Scope

**In scope.** The `Bead`, `File`, and `Plan` panes, every `ref:<kind>` document provider
pane, and the shared `FilterBar` chrome that `Stitch` and `Patch` already use.

**Out of scope, deliberately.**

- **Degraded provider panes get no query bar.** A pane whose contract failed to compile
  has `PaneCapability.FILTER_SESSION` off (`_rule_filter_session` in
  `src/sase/ace/tui/_artifact_tab_contract_rules.py`) and no inventory to query. An
  inert bar there would be a lie; those panes keep the existing failure card. The same
  applies to any future provider whose spec declares no fields.
- **Moving Patch's bar.** Patch's bar sits at the top of its _detail_ column, not the
  top of the pane, because `ArtifactsPatchesPane` is a `Horizontal`. Relocating it would
  shift a very large number of PNG goldens across otherwise unrelated suites (agents,
  axe, config center all screenshot the default Patch view) for no behavioral gain — the
  same reason `docs/artifacts_pane_visual_grammar.md` already records for not giving
  Patch a shared identity-header row. Patch stays where it is; the phase `grammar`
  records the exception in the doc. Relocating it is a good separate task.
- **Wiring saved-query slots and query history to the non-Patch panes.** Today only
  Patches applies a loaded slot (see the docstring on `_load_saved_query` in
  `src/sase/ace/tui/actions/patch/_query.py`). The saved-slot chip in the idle display
  therefore stays a Patch-only decoration. Extending slots to the other panes is
  separate work.

**Rejected alternative.** Folding each pane's identity/scope lane into the query bar's
border title would make the net cost roughly +1 row instead of +3, but it breaks the
documented layout order, collides with the vim mode indicator that already owns
`border_title` while editing, and would redesign all five panes at once. Rejected in
favor of the uniform, boring bar.

**Accepted trade-off.** A permanently visible bar costs three rows on Bead, File, Plan,
and provider panes. At 80x24 the Bead list drops from roughly eleven visible rows to
roughly eight. That cost is already paid transiently every time the user presses `/`
today, Stitch already pays it permanently at the same terminal size, and it buys the
elimination of layout shift. Do not add an adaptive-height mode to claw the rows back;
that would reintroduce the shift this epic exists to remove.

## Design

### One idle presentation for every persistent bar

`FilterBar` (`src/sase/ace/tui/widgets/filter_bar.py`) already has both halves of the
mechanism:

- `PERSISTENT: ClassVar[bool]` keeps the widget in layout and the editor read-only and
  unfocusable when the bar is not being edited.
- `DISPLAY_ID: ClassVar[str | None]` plus `_closed_display_text` / `_closed_display` /
  `_update_closed_display` / `_set_closed_display_visible` swap the editor for a Rich
  `Static` while idle. `open()` swaps back, `close()` swaps forward.

Only `PatchFilterBar` sets `DISPLAY_ID` today. The design is to make the closed display
the **universal** idle rendering for persistent bars, and to make its content
profile-driven rather than pane-specific:

- **Idle** (`_editing is False`): the editor is hidden and unfocusable; a `Static`
  renders the committed query, syntax-highlighted from the pane's
  `CompiledQueryProfile`. An empty query renders the profile's `free_text_hint` as dim
  italic placeholder text. No vim mode chrome is visible, because the editor that owns
  `border_title` is not displayed — this is what removes Stitch's stray `[NORMAL]`.
- **Editing** (after `open()`): the `Static` hides, the editor shows, focuses, enters
  INSERT, and the completion overlay behaves exactly as it does now.

### Highlighting reuses the existing lexer, it does not add a grammar

Two query dialects exist. `CompiledQueryProfile.boolean` distinguishes them
(`src/sase/ace/query_profile/profiles.py`): `patches` is the boolean dialect with
`AND`/`OR`/parens; `stitches`, `beads`, `files`, `ref:plan`, and every provider kind are
flat token dialects where a leading `-` negates.

- Boolean dialect: keep `tokenize_query_for_display` from
  `src/sase/ace/query/highlighting.py` and its `QUERY_TOKEN_STYLES`.
- Flat dialects: classify tokens with the **same lexer the flat parser uses** —
  `sase.filter_tokens.tokenize`, whose `FilterToken` already carries `negated`,
  `quoted`, `wholly_quoted`, and source offsets, together with `unquoted_index` /
  `split_unquoted`. `src/sase/ace/query/profile_reference_flat.py::_flat_clauses` is the
  reference for how a token becomes a predicate, a `key:value` clause, or free text; the
  highlighter must mirror that classification and must not re-lex the string with a
  private regex. One lexer, two consumers, so the highlight can never disagree with the
  parse.

Token classes to style, all resolved from the compiled profile rather than a hardcoded
table: known filterable key, **unknown key** (a distinct warning style — this is a real
usability win, an unknown field is visibly wrong before you submit), value, negation
marker, sigil-prefixed token (`profile.sigils`), macro token (`profile.macros`),
zero-argument predicate (`profile.predicates`, plus `*` when `profile.any_special`),
quoted string, and bare free-text term. Reuse `QUERY_TOKEN_STYLES` so both dialects
share one palette.

**Rust-core boundary.** This stays in Python and does not belong in
`../sase-core/crates/sase_core`: it adds no grammar and no matching behavior. It is a
pure mapping from the existing shared lexer's output plus the already-compiled profile
onto Rich styles — presentation only, in the same layer as the existing
`highlighting.py`.

### Accent correctness

Border, sigil, and completion-highlight colors are pinned per bar class in
`src/sase/ace/tui/styles.tcss` (roughly lines 130-280). That is why a provider document
pane draws `PlanFilterBar`'s purple even though `FilterBar.__init__` already accepts an
`accent=` kwarg and `ArtifactsDocumentsPane.compose` already passes `contract.accent`.
Once the bar is on screen permanently this stops being a subtle mismatch and becomes the
loudest wrong color on the pane.

Every persistent bar must resolve those colors from its own `ACCENT` (which is the
pane's `ArtifactsPaneContract.accent`). Whether that is done by applying styles at mount
time or by keeping generic CSS plus runtime component-class overrides is the
implementer's call; the phase `documents` pins the outcome with a test that a synthetic
provider pane's bar renders in that provider's accent and in no built-in pane's accent.

### One sync funnel keeps the idle bar truthful

A hidden bar only had to be correct while it was open. A permanent one must be correct
whenever the pane's committed filters change — including paths that never open a
session: startup defaults, `set_project_scope`, the Files kind-cycle action
(`files_filter_session.py`), and `_clear_filter_for_entry_jump` on Bead and Plan.

Rather than patch each site, each pane gains one `_sync_query_bar()` called from its
`_refresh_options()` (Stitch already does exactly this inside `_set_result_status`,
guarded by `if not self._filter_session_open`). It:

- sets the bar's query text from the **committed** filters when no session is open, and
  leaves the text alone while the user is editing;
- always sets the status lane.

Every path that changes committed filters already routes through `_refresh_options()`;
the implementing phase must verify that and add the call anywhere it does not.

### What the status lane says

- Empty query: blank. The placeholder already says there is no filter; `48 matches` next
  to an empty box is noise.
- Non-empty query: `{N} matches  ·  {coverage}`, where coverage is `exact`, `preview`,
  or a pane-supplied label (`capped` on Stitch, the deep-archive label on Plan).
- Parse error: the error message, in the existing error style. Only reachable while
  editing — an idle bar always holds a committed, valid query.

`CommitFilterBar.set_status` already overrides the count away because the Stitches
legend owns position; that override stays. Callers become uniform even though renderers
stay pane-specific.

### The query is displayed once

Once a pane's bar is permanent, the filter-token echo in its identity/scope header is a
second rendering of the same string from a second code path. It goes away:

- `shell.build_shell_scope`'s `filter_tokens` parameter is removed outright (not left
  unused — the repo lints unused symbols), along with `build_beads_scope`'s and
  `build_plans_scope`'s pass-throughs and the `to_query_tokens(self.filters)` calls in
  `beads_options.py` and `plans_options.py`.
- `build_files_info` in `files_rendering.py` **keeps** its `filtered {visible}/{total}`
  count — that is a count, not a query echo — and drops only the trailing
  `" ".join(tokens)` fragment.

Removal is staged so the tree never breaks: the phase `panes` stops passing the argument
from Bead and File, and the phase `documents` stops passing it from Plan and deletes the
parameter.

### Click to edit

An always-visible control that cannot be clicked is a missed affordance, and
`ArtifactsView` already routes `PanelTabStrip.TabClicked` and `RelationPanel.Clicked`.
`FilterBar` gains a `Clicked` message posted when an idle persistent bar is clicked; the
host pane maps it to the same `show_filters()` entry point `/` uses. Clicking a bar that
is already being edited does nothing.

---

## Phase `chrome`: Shared persistent query-bar chrome

Build the shared mechanism and prove it on the two panes that are already persistent. No
pane changes from hidden to persistent in this phase.

**Work**

1. Add the profile-driven display highlighter as a new module beside
   `src/sase/ace/query/highlighting.py` (for example
   `src/sase/ace/query/profile_highlighting.py`), exposing one entry point that takes
   `(text, CompiledQueryProfile)` and returns a Rich `Text`. It dispatches on
   `profile.boolean`: boolean queries go through the existing
   `tokenize_query_for_display`; flat queries are classified from
   `sase.filter_tokens.tokenize` plus the profile's fields, sigils, macros, and
   predicates, mirroring `profile_reference_flat._flat_clauses`. It never raises on
   malformed input — a half-typed query must still render.
2. In `FilterBar`:
   - Make `_closed_display_text` return the highlighted query when `self._profile` is
     set, falling back to plain `Text(text)` when it is not (small test apps mount bars
     without a profile).
   - Render `profile.free_text_hint` as dim italic placeholder text when the query is
     empty.
   - Resolve border, sigil, and completion-highlight colors from `self.ACCENT`.
   - Post a `Clicked` message when an idle persistent bar is clicked.
   - Keep `open()` / `close()` semantics exactly as they are; this phase changes what
     the idle state _looks like_, not when it is entered.
3. Declare `DISPLAY_ID` on `CommitFilterBar` so Stitch adopts the shared closed display
   and stops showing `[NORMAL]` on an unfocused, read-only bar. Rework
   `PatchFilterBar._closed_display_text` to call `super()` and append only its
   saved-slot chip, deleting its direct `build_query_text` call.
4. Collapse the per-class color blocks in `styles.tcss` into shared persistent-bar rules
   plus accent resolution. Persistent bars keep `min-width: 0` on the status lane so
   they stay legible at 80 columns.
5. Route `FilterBar.Clicked` to `show_filters()` for the Stitch and Patch panes.

**Tests**

- Unit tests for the highlighter over both dialects: known key, unknown key, negated
  key, quoted value, comma-repeated value, sigil, macro, predicate, `*`, bare term,
  empty string, and a half-typed token such as `status:`. Assert token spans and styles,
  and assert it never raises.
- A test that an idle persistent bar shows the display `Static` and hides the editor,
  that the editor is not focusable while idle, and that `open()` reverses both.
- A test that clicking an idle bar opens a session and that clicking an open bar does
  not close it.
- Regenerate the Stitch and Patch PNG goldens (`artifacts_stitches_*`,
  `patch_filter_bar_closed_120x40`, `patch_filter_bar_completion_120x40`) with
  `just test-visual --sase-update-visual-snapshots`, and inspect the diff artifacts
  under `.pytest_cache/sase-visual/` before accepting each one.

**Done when** Stitch's idle bar renders a highlighted query with no `[NORMAL]` title,
Patch's idle bar is unchanged apart from coming from the shared path, and no pane has
changed its hidden/visible behavior.

---

## Phase `panes`: Persistent query bar on Bead and File

**Work**

1. `BeadFilterBar` and `FileFilterBar`: set `PERSISTENT = True` and declare `DISPLAY_ID`
   (`bead-filter-display`, `file-filter-display`).
2. Add `_sync_query_bar()` to `BeadsFilterSessionMixin` and `FilesFilterSessionMixin`
   and call it from each pane's `_refresh_options()`, per the design above. Replace the
   existing `if self._filter_session_open: ... set_status(...)` blocks in
   `beads_options.py` and `files_options.py` with the funnel so the status is maintained
   when idle too.
3. Verify every committed-filter mutation reaches the funnel: startup defaults,
   `set_project_scope`, `_clear_filter_for_entry_jump` (Bead), the kind-cycle action
   (`files_filter_session.py`), and both submit and dismiss paths.
4. Stop echoing the query in the headers: drop `filter_tokens=to_query_tokens(...)` from
   `beads_options._scope_text` and its `build_beads_scope` parameter, and drop the
   trailing token join from `build_files_info` while keeping `filtered N/M`. Leave
   `build_shell_scope`'s parameter in place for the phase `documents` to delete.

**Tests**

- Update `tests/ace/tui/test_artifacts_beads_filtering.py` and
  `tests/ace/tui/test_artifacts_files_filtering.py`: the bar is visible before `/` is
  ever pressed, `bar.display` no longer toggles, `-status:closed` is asserted on the
  bar's display rather than in `#beads-info`, and submitting an empty query clears the
  bar to its placeholder.
- New tests: committing a query updates the idle bar; changing project scope keeps the
  idle bar in sync; the Files kind-cycle action is reflected in the bar; the pane's
  row-list geometry does not change between idle and editing (the no-layout-shift
  guarantee).
- Regenerate `artifacts_beads_*`, `artifacts_files_*`, `artifacts_split_*`,
  `notification_beads_*`, and `copy_as_over_artifact_files_modal_dark_120x40`, plus any
  other golden the visual run reports. Add one new golden for an idle Bead bar holding a
  non-empty query and one for an idle File bar with an empty query showing its
  placeholder.

**Done when** Bead and File show their bar at all times, the header no longer repeats
the query, and pressing `/` moves nothing on screen except the bar's own contents.

---

## Phase `documents`: Persistent query bar on Plan and every document provider

**Work**

1. `PlanFilterBar`: set `PERSISTENT = True` and declare `DISPLAY_ID`
   (`plan-filter-display`). This covers the built-in Plan pane and every `ref:<kind>`
   pane, since both are `ArtifactsDocumentsPane`.
2. Add `_sync_query_bar()` to `PlansFilterSessionMixin` and call it from
   `_refresh_options()`. The idle status lane carries the deep-archive coverage label
   that `plans_filter_session.py` already computes, so the user can see from the bar
   whether the shown result set covers the deep archive.
3. Verify the funnel covers `_clear_filter_for_entry_jump`, project scope changes, and
   the deep-archive result arriving asynchronously after a query is committed.
4. Drop `filter_tokens=to_query_tokens(...)` from `plans_options._scope_text`, remove
   the parameter from `build_plans_scope`, and remove it from `shell.build_shell_scope`
   entirely.
5. Confirm the accent work from the phase `chrome` reaches provider panes: a
   `ref:<kind>` bar's border, sigil, and completion highlight all come from that
   descriptor's accent.

**Tests**

- Update `tests/ace/tui/test_artifacts_plans_filtering.py` and
  `tests/ace/tui/widgets/test_plan_filter_bar.py` for the always-visible bar.
- A test mounting a synthetic provider pane (the fixtures under
  `tests/ace/tui/artifacts_contract/` already build one) asserting its bar renders in
  the provider's own accent and in no built-in pane's accent — the regression that this
  phase exists to prevent.
- A test that the deep-archive coverage label is visible in the idle status lane.
- Regenerate `artifacts_plans_*` goldens, including
  `artifacts_plans_narrowed_filter_chips_120x40` (whose subject is now the bar, not
  header chips — rename it if the name no longer describes the picture).

**Done when** Plan and every installed document provider show a permanently visible bar
in their own accent, and `build_shell_scope` no longer knows what a filter token is.

---

## Phase `grammar`: Always-on invariant, conformance, and visual grammar docs

**Work**

1. Add the cross-pane regression that could not be written until every pane was
   converted: walk `resolve_artifacts_subtabs()`, activate each sub-tab in a mounted
   app, and assert that every descriptor whose contract has
   `PaneCapability.FILTER_SESSION` mounts a `FilterBar` that is displayed, read-only,
   and not focusable while idle, and that every degraded descriptor mounts none. Put it
   with the other mounted Artifacts tests, not in
   `tests/ace/tui/artifacts_contract/harness.py`, whose checks are pure and never mount
   widgets.
2. Rewrite `docs/artifacts_pane_visual_grammar.md`:
   - Layout order item 1 becomes the **query bar** — always present on every pane whose
     contract has `FILTER_SESSION`, absent only on degraded panes and on providers that
     declare no fields. Delete "collapsed when that pane's current query UX does not
     expose one."
   - Add the bar's two states (idle: highlighted committed query or placeholder, no vim
     chrome, click to edit; editing: vim editor plus completion overlay) and the status
     lane's contents.
   - Add the highlighter to the accent rules section: one lexer shared with the parser,
     colors from `contract.accent`.
   - Update the Patch asymmetry section: Patch's bar living in its detail column is now
     the single documented exception to the layout order, with the golden-churn reason
     and a pointer that relocating it is separate work.
   - Extend the extension checklist so a new pane is told to declare `PERSISTENT` and
     `DISPLAY_ID` on its bar and to call `_sync_query_bar()` from its refresh funnel.
3. File a task bead through `/sase_new_task` for moving Patch's bar to the top of its
   pane, so the one remaining inconsistency is tracked rather than forgotten.

**Done when** `just check-full` is green through `/sase_monitor`, the doc describes the
shipped grammar, and the always-on invariant is enforced by a test rather than by
convention.

## Verification

Every phase runs `just install` first (workspaces are ephemeral), then `just check`
inline. Any phase that regenerates PNG goldens also runs `just test-visual` and inspects
`.pytest_cache/sase-visual/` actual/expected/diff artifacts before accepting a golden —
never blanket-accept a visual run. The phase `grammar` runs `just check-full` through
`/sase_monitor` with a `--next` action, since it routinely outruns a single agent turn.
