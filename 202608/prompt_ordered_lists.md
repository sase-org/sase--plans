---
tier: epic
title: Ordered-list auto-numbering in the prompt input widget
goal: "The prompt input widget grows and maintains `<N>. ` ordered lists through every
  keymap that already serves `- ` bullets (Ctrl+J, o, O, J, Tab, Shift+Tab), renumbering
  the surrounding list so its numbering is always correct, and colors ordered markers
  like bullet dashes.

  "
phases:
  - id: core
    title: Shared list-marker model and the ordered renumber engine
    depends_on: []
    size: medium
    description: "core: add the pure marker/boundary/ownership primitives shared by both
      marker families, the ordered run scanner, the Prettier-compatible renumber engine,
      and the single-TextEdit list-edit planner; refactor the hyphen helpers onto the
      shared primitives with no behavior change.

      "
  - id: newline
    title: INSERT-mode Ctrl+J for ordered items
    depends_on:
      - core
    size: medium
    description: "newline: route Ctrl+J through the list-edit planner so ordered items
      continue, split, grow a first sibling, exit from an empty marker, and de-list at
      the content column, each as one undo checkpoint with the run renumbered.

      "
  - id: openline
    title: NORMAL-mode o and O for ordered items
    depends_on:
      - newline
    size: medium
    description: "openline: add an optional planned-edit hook to the vim open-line
      commands so prompt panes open a correctly numbered ordered sibling below or above,
      renumber the run, and keep dot-repeat recomputing at the destination.

      "
  - id: shift
    title: INSERT-mode Tab and Shift+Tab nesting for ordered items
    depends_on:
      - core
    size: medium
    description: "shift: make Tab nest an ordered item under its parent at that parent's
      content column (starting a nested list at 1) and Shift+Tab unnest it into the
      enclosing run, renumbering both the source and destination runs and moving the
      item's owned block with it.

      "
  - id: join
    title: NORMAL-mode J for ordered items
    depends_on:
      - openline
    size: small
    description: "join: drop a pulled-up ordered marker when folding the next line up,
      and renumber the run the removed item left behind.

      "
  - id: highlight
    title: Ordered-marker highlighting
    depends_on:
      - core
    size: small
    description: "highlight: give ordered markers the same theme-aware accent the
      leading bullet dash already gets, with unit coverage and a PNG snapshot fixture.

      "
  - id: docs
    title: Documentation, help modal, and full verification
    depends_on:
      - newline
      - openline
      - shift
      - join
      - highlight
    size: small
    description:
      "docs: document the ordered-list behavior in the ace reference and help modal, and
      run the exhaustive verification lane over the combined tree."
proposed_by: bbugyi200.athena.ub
status: done
bead_id: sase-gi
create_time: 2026-09-09 19:51:14
---

- **PROMPT:**
  [prompts/202608/prompt_ordered_lists.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/prompt_ordered_lists.md)
- **BEAD:**
  [sase-gi](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gi/README.md)

# Plan: Ordered-list auto-numbering in the prompt input widget

## Goal

`PromptTextArea` already grows `- ` bullets automatically through `Ctrl+J`, `o`, `O`,
`J`, `Tab`, and `Shift+Tab`. This epic gives `<N>. ` ordered lists the same treatment,
plus the thing unordered lists never need: the rest of the list is renumbered so the
numbering stays correct after every structural edit.

## Current state

The hyphen feature is a small pure-helper module plus thin widget integrations:

- `src/sase/ace/tui/widgets/_prompt_bullet_editing.py` — pure helpers: marker regex,
  boundary detection, ownership scan (`prompt_bullet_sibling_prefix`), marker-only /
  content-column predicates, `strip_prompt_bullet_marker`, `plan_prompt_bullet_shift`,
  and dot-repeat replay normalization.
- `src/sase/ace/tui/widgets/_prompt_text_area_actions.py` — `action_insert_newline`
  (`Ctrl+J`), the `o` / `O` / `J` hook overrides, and the dot-replay normalizers.
- `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` — INSERT `Tab` /
  `Shift+Tab`, and `_apply_planned_text_edit`.
- `src/sase/ace/tui/widgets/_vim_normal_editing.py` (`o` / `O`),
  `src/sase/ace/tui/widgets/_vim_normal_operator_exec.py` (`J`), and
  `src/sase/ace/tui/widgets/vim_text_area.py` (the default no-op hooks).
- `src/sase/ace/tui/widgets/_bullet_highlight.py` — the `bullet.dash` overlay span and
  theme.

Today `_prompt_bullet_editing._UNSUPPORTED_MARKER_RE` matches `\d+[.)] ` and treats it
as a hard boundary, so ordered markers currently _stop_ hyphen ownership. That stays
true: the two families never own each other's lines.

## Design principles

These five rules decide every open question below; phase agents should resolve anything
unspecified by appealing to them in this order.

1. **Symmetry with hyphen bullets.** Every rule the hyphen feature already has (which
   lines a marker owns, what counts as a boundary, when a marker-only line exits the
   list, which cursor columns qualify) is mirrored exactly for ordered markers. Ordered
   lists only _add_ renumbering.
2. **Live numbering agrees with the formatter.** Prompt formatting (`gf` →
   `sase.file_references.format_agent_prompt_markdown`, Prettier) already renumbers
   ordered lists. Our live numbering reproduces Prettier's rules so formatting never
   contradicts what the editor just did. All rules in "Numbering" were verified
   empirically against that formatter.
3. **One keypress is one undo checkpoint.** Textual's `EditHistory` opens a new batch
   for any edit containing a newline or more than one character, so a structural edit
   plus its renumbering must reach the document as a _single_ `_replace_via_keyboard`
   call. This is what makes the apply-to-a-copy-then-diff planner (below) mandatory
   rather than stylistic.
4. **Never renumber what the edit did not restructure.** Renumbering fires only from the
   six list keymaps, and only over the one run the edit changed. Typing digits, `Ctrl+A`
   / `Ctrl+X`, paste, and every other edit path leave numbering alone.
5. **Keystroke paths stay bounded and pure.** Per `sase/memory/tui_perf.md` rule 11,
   planners do no I/O, take no locks, and scan a bounded number of lines; a run larger
   than the cap degrades to "insert the marker, skip the renumber" rather than doing
   unbounded work on a keypress.

## The model

**Ordered marker.** `^( *)(\d{1,9})([.)]) ` — leading spaces, a 1–9 digit number, a `.`
or `)` delimiter, then exactly one space. The _content column_ is
`indent + len(digits) + 2`. Mirroring the hyphen regex, extra spaces after the marker
are content, not part of the marker.

**Marker family.** `hyphen` (`- `) or `ordered`. Every scan is family-scoped: a marker
of the other family is a boundary, exactly as ordered markers are a boundary for hyphen
ownership today.

**Boundary line** (stops a family's backward/forward ownership scan): a blank line, a
tab-indented line, a Markdown fence, a thematic break, a blockquote, a `*` / `+` bullet,
a marker of the other family, or a "tight" marker of this family (`-` or `<N>.` with no
following space).

**Ownership.** A marker owns its own line, and owns a later line only when every line in
between has indent ≥ that marker's content column and no boundary line intervenes — the
existing hyphen rule, generalized from "marker indent + 2" to "content column". This is
what makes continuation lines from Prettier wrapping work for `1. ` items exactly as
they do for `- ` items.

**Run.** The maximal sequence of ordered items sharing the same indent _and_ the same
delimiter, where consecutive items are joined only by lines that the earlier item owns
or by blank lines. Blank lines are transparent (a loose list is still one list —
Prettier renumbers across them); prose at a lower indent, a boundary construct, or a
delimiter change ends the run. Nested items at a deeper indent belong to their own run
and are skipped by the outer run's scan.

Known, deliberate divergence from CommonMark: CommonMark lets a sibling item be indented
up to three spaces more than its predecessor, while a run here requires the _same_
indent. Markers this feature generates always inherit the owner's exact indent, so
generated lists are always uniform, and formatting normalizes hand-written stragglers.

## Numbering

**Style detection (Prettier's rule).** Look at the first two items of the run. If the
second item's number equals the first's, the run is _repeat style_ and every item keeps
that number (this is the `1. / 1. / 1.` convention Prettier deliberately preserves).
Otherwise the run is _sequential style_ and item _i_ is `first_number + i`. A one-item
run is sequential. Verified: `1. 1. 1.` is preserved, `1. 3. 7.` becomes `1. 2. 3.`,
`5. 6. 7.` keeps its start, and `1. 1. 5.` becomes all `1.`.

**Inserted number.** One rule serves `Ctrl+J`, `o`, and `O`: the new item's number is
the number of the nearest preceding sibling marker in the post-edit text, plus one; if
it has no preceding sibling it takes the run's first number; under repeat style it
always takes the repeated number. The delimiter and indent are copied from the owning
item.

**Renumber anchor.** After the structural edit is applied, exactly one run is renumbered
— the maximal run containing the _anchor item_:

- insertion paths: the newly inserted item;
- removal paths (empty-marker exit, content-column de-list, `J`): the nearest preceding
  ordered item at the affected indent; if there is none, nothing is renumbered;
- `Tab` / `Shift+Tab`: two anchors — the moved item (destination run) and the item
  preceding the hole it left (source run).

This single rule produces the intuitive answers without special cases. Removing the
empty `3. ` from `1. / 2. / 3.(empty) / 4.` anchors at `2.`, whose run reaches `4.`
across the blanks, so `4.` becomes `3.`. De-listing an item at its content column leaves
prose that terminates the run, so the items below become a separate list and keep their
own start — which is exactly what the formatter does.

**Marker width changes.** When renumbering changes an item's marker width (`9.` →
`10.`), every line that item owns is shifted by the same delta so ownership and
Prettier's indentation both stay correct. A negative delta only removes spaces that
exist.

**Leading zeros.** `007. ` is recognized as a marker; renumbering emits plain decimal.

## The single-edit planner

Every list keymap goes through one shared planner shape, which is what guarantees
principle 3:

1. copy the document's lines;
2. apply the structural change (insert / remove / move a marker line, split a line, fold
   a line) to the copy;
3. renumber the anchored run(s) in the copy, shifting owned blocks for width changes;
4. diff the copy against the original and take the minimal contiguous changed row range;
5. emit one `TextEdit` (the existing `_paired_text_editing.TextEdit`) replacing that
   range, with the cursor offset computed from the rebuilt text — never by offset
   arithmetic on the original, because renumbering can change the width of lines _above_
   the insertion point.

Callers apply it with the existing `_apply_planned_text_edit`, which already routes
through one `_replace_via_keyboard` and clears transient completion state.

**Bounds.** If the anchored run exceeds the cap (a module constant; 400 items and 2000
scanned lines are sane starting values) the planner still performs the structural edit
and skips renumbering. No notification — silent, bounded degradation.

## Behavior per keymap

`Ctrl+J` (INSERT, collapsed selection unless noted) — each row mirrors the hyphen rule
of the same name, with renumbering added:

| Situation                                                                    | Result                                                                    |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Cursor in an item's content (or on a line it owns)                           | Split at the cursor; the tail moves to a new sibling item; run renumbered |
| Marker-only line with an ordered item above                                  | The marker is replaced by a bare newline (list exit); run renumbered      |
| Marker-only line with no ordered item above                                  | Grows one sibling below (exit happens on the next press); run renumbered  |
| Cursor exactly at the content column of a populated item, with an item above | Marker removed, content pushed down (de-list); anchored run renumbered    |
| Active selection                                                             | Selection replaced by newline + sibling marker; run renumbered            |

`o` / `O` (prompt NORMAL): open a sibling below / above with the inserted-number rule,
then renumber. `O` on an item's marker row takes that item's number; `O` on a line the
item owns takes the next number, because the new marker lands after that item's marker.
Dot-repeat recomputes the marker from the destination context, and a marker typed into
the recorded insert text is stripped on replay, both mirroring the hyphen normalizers.

`J` (prompt NORMAL): folding the next line onto nonblank content drops that line's
ordered marker (mirroring `strip_prompt_bullet_marker`) and renumbers the run the
removed item left.

`Tab` / `Shift+Tab` (INSERT, cursor from column zero through the marker's content
column):

- `Tab` moves the item to the content column of the nearest preceding marker line of
  either family at the same or a lower indent — the parent's content column, not a fixed
  two spaces. This is required, not cosmetic: a `2. ` item indented two spaces under a
  `1. ` parent is still a _sibling_ to CommonMark, and an ordered item can only
  interrupt a parent's paragraph when it is numbered `1`. So a Tab that starts a new
  nested list numbers the moved item `1`; a Tab that lands directly under an existing
  nested run continues that run instead. With no preceding marker line to nest under,
  `Tab` is a no-op.
- `Shift+Tab` moves the item out to the indent of its parent marker and gives it the
  next number in that outer run. At the outermost level it is a no-op.
- Both move the item's owned block with it and renumber the source and destination runs.

Hyphen `Tab` / `Shift+Tab` keep their current fixed two-space unit; for `- ` the content
column _is_ two, so the two rules already agree wherever they overlap.

## Non-goals

- `*` and `+` bullets stay unsupported for both families.
- `Ctrl+A` / `Ctrl+X` keep vanilla vim semantics (change one number, renumber nothing).
- No renumbering from ordinary typing, paste, or `p` / `P`.
- No new configuration or keymaps; ordered lists behave like hyphen bullets, always on.
- No checklist (`- [ ]`) or definition-list support.

## Rust core boundary

This stays Python in this repo, deliberately. The behavior is prompt-widget editing
built directly on Textual's `TextArea` document, cursor, and undo history, and its whole
point is to mirror the hyphen feature that already lives in `src/sase/ace/tui/widgets/`.
Splitting one list-editing feature across two repos — hyphen in Python, ordered in Rust
— would be worse than either whole choice. If the prompt editor's list engine is ever
ported to `sase_core`, both families move together.

## Phases

### Shared list-marker model and the ordered renumber engine

Add the pure layer, with no widget changes.

- New `src/sase/ace/tui/widgets/_prompt_list_markers.py`: the marker dataclass (row,
  indent, number, delimiter, content column), family-scoped marker matching, boundary
  detection, and the ownership scan. Everything cross-module must be public names —
  `_`-prefixed _modules_ are the norm here, but Symvision flags private _symbol_
  imports.
- New `src/sase/ace/tui/widgets/_prompt_ordered_editing.py`: the run scanner, the style
  detector, the inserted-number rule, the renumber pass with owned-block width shifting,
  and the shared apply-copy-renumber-diff planner that returns a `TextEdit`.
- Refactor `_prompt_bullet_editing.py` to build its boundary and ownership logic on the
  shared primitives. This must be **behavior-preserving**:
  `tests/ace/tui/widgets/test_prompt_bullet_*.py` and
  `tests/test_prompt_normal_mode_join.py` pass unchanged, and ordered markers remain
  hyphen boundaries.
- Tests: `tests/ace/tui/widgets/test_prompt_ordered_list_helpers.py` (markers,
  ownership, continuation lines, boundaries, family separation, tight markers, fences,
  tabs) and `tests/ace/tui/widgets/test_prompt_ordered_renumber.py` (style detection,
  sequential and repeat runs, loose runs across blank lines, prose-split runs, nested
  runs, delimiter-scoped runs, `9.` → `10.` width shifts of owned blocks, cap
  degradation).
- Add a formatter-agreement test: for a curated set of small prompts that are already
  Prettier-stable, assert `format_agent_prompt_markdown(after_edit) == after_edit`.
  Follow the existing prettier-dependent tests in `tests/test_format_with_prettier.py`
  for environment handling (`SASE_DISABLE_PRETTIER`), and keep fixtures short so prose
  rewrapping cannot muddy the assertion.

### INSERT-mode Ctrl+J for ordered items

Extend `action_insert_newline` in `_prompt_text_area_actions.py` with the ordered
branches from the `Ctrl+J` table, each built as one planned `TextEdit` and applied
through `_apply_planned_text_edit`.

- Try the ordered planner first; when it declines, the existing hyphen branches run
  **byte-identical** to today. Do not restructure the hyphen path.
- Verify the undo contract explicitly: one `u` reverts the insertion _and_ its
  renumbering together, and the two-press exit sequence stays two checkpoints (as
  documented for hyphen bullets today).
- Tests: `tests/ace/tui/widgets/test_prompt_ordered_insert_editing.py`, modeled on
  `test_prompt_bullet_insert_editing.py` — one case per table row plus renumbering of
  following siblings, repeat-style preservation, `)` delimiter preservation, mid-item
  splits, continuation-line presses, active selections, and cursor landing at the new
  content column when renumbering changed the width of lines above it.

### NORMAL-mode o and O for ordered items

Give the vim open-line commands an optional planned-edit hook so a prompt pane can
produce a wider single edit than the current "insert this string" hooks allow.

- Add `_normal_open_line_plan(row, *, above)` to `vim_text_area.py` returning `None` by
  default, and consult it in `_vim_normal_editing.py` before the existing string hooks.
  `VimTextArea` and `SingleLineVimTextArea` behavior must not change. Keep the current
  ordering — the pane enters INSERT mode before the edit, because
  `_replace_via_keyboard` is a no-op while read-only.
- Implement the override in `_prompt_text_area_actions.py` using the core planner; leave
  `_normal_open_below_insert_text` / `_normal_open_above_insert_text` in place for
  hyphen bullets.
- Extend the dot-replay normalizers so a manually typed ordered marker is not replayed
  on top of a structurally supplied one, mirroring
  `normalize_prompt_bullet_replay_text`.
- Tests: `tests/ace/tui/widgets/test_prompt_ordered_open_line_editing.py`, modeled on
  `test_prompt_bullet_open_line_editing.py` — `o` and `O` on marker rows and on owned
  continuation lines, renumbering of the rest of the run, undo as one checkpoint,
  dot-repeat rechecking the destination context, and a bare `VimTextArea` staying bare.

### INSERT-mode Tab and Shift+Tab nesting for ordered items

Extend the `tab` / `shift+tab` branch in `_prompt_text_area_key_handling.py`: try an
ordered nest/unnest planner before `plan_prompt_bullet_shift`, keeping the
queued-snippet-tabstop precedence and the existing selection guard exactly as they are.

- Implement the nesting rules from the keymap section, including the "new nested list
  starts at 1" rule and the no-op cases, moving the item's owned block and renumbering
  both runs in one edit.
- Preserve `remap_dot_capture=True` handling for the INSERT dot capture.
- Tests: `tests/ace/tui/widgets/test_prompt_ordered_shift_editing.py` — nesting under a
  hyphen parent and an ordered parent, continuing an existing nested run, source-run gap
  closing, unnesting into the outer run, owned blocks moving with the item, both no-op
  cases, cursor tracking, one undo checkpoint, and snippet-tabstop precedence.

### NORMAL-mode J for ordered items

Extend the prompt's `J` behavior so folding an ordered item up drops its marker and
renumbers the run it left.

- `_normal_join_next_line_text` already strips the pulled-up hyphen marker; add ordered
  markers to it, then renumber the affected run. `_join_lines` in
  `_vim_normal_operator_exec.py` already emits more than one recorded edit per join, so
  this phase must not make that granularity _worse_: fold the renumbering into the
  join's existing replacement span rather than adding another separate edit, and state
  the resulting undo behavior in the test names.
- Counts (`5J`) must renumber once against the final text, not once per fold.
- Tests: `tests/ace/tui/widgets/test_prompt_ordered_join.py`, alongside the existing
  `tests/test_prompt_normal_mode_join.py` cases — marker dropped on a nonblank fold,
  marker kept when folding onto a blank line, run renumbered, counts, and non-prompt
  editors keeping vanilla `J`.

### Ordered-marker highlighting

Extend `_bullet_highlight.py` so ordered markers get the same theme-aware accent as the
leading bullet dash: the digits and the delimiter are colored, the trailing space is
not, matching how the dash span excludes its space. Register the span under its own
style name sharing the dash's `Style`, so one list-marker color covers both families.
Keep the existing overlay ceilings and the cheap early-out before scanning.

- Tests: `tests/ace/tui/widgets/test_prompt_ordered_highlight.py`, modeled on
  `test_prompt_bullet_highlight.py` — span boundaries, indented markers, `)` delimiters,
  multi-digit numbers, no false positives on `2024. ` mid-prose or tab-indented markers,
  coexistence with search and yank highlights, UTF-8 byte columns, and theme-change
  re-registration.
- Add an ordered-list fixture next to `BULLET_HIGHLIGHT_SOLO` in
  `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py` plus a snapshot test in
  `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`, and generate its
  golden with `just test-visual --sase-update-visual-snapshots`. Inspect the rendered
  PNG before committing the golden — this is the phase where "beautiful" is actually
  verified.

### Documentation, help modal, and full verification

- `docs/ace.md`: update the INSERT-mode and NORMAL-mode keymap tables and the prose
  paragraphs that currently describe hyphen-only behavior (around the "Prompt Input
  Widget", `Ctrl+J`, `Tab` / `Shift+Tab`, and `o` / `O` / `J` sections), covering the
  numbering rules, the run definition, the nesting rules, and the formatter-agreement
  property.
- `src/sase/ace/tui/modals/help_modal/binding_common.py`: the `Tab / Shift+Tab` row
  currently reads "Indent / dedent bullet marker"; reword it for both families within
  the modal's width limits (`src/sase/ace/CLAUDE.md`: descriptions cap at 32
  characters).
- Run `just install` then `just check-full` on the combined tree, plus
  `just test-visual`.

## Verification

- `just install` before anything else — workspace virtualenvs go stale.
- Each phase: `just check` (whole-repo lint gates plus the diff-scoped test lane).
- Final phase: `just check-full` and `just test-visual`.
- Manual smoke in `sase ace`, since this feature is judged by feel: type `1. one`, press
  `Ctrl+J` twice, `o` in the middle of a five-item list, `Tab` on the third item,
  `Shift+Tab` back out, `J` over an item, then `gf` — the formatter must make no
  numbering changes at any point.
