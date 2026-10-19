---
tier: tale
title: Pager same-document anchor links
goal:
  Pressing a pager label on a same-document Markdown link such as [text](#heading) or
  [text](#L12) jumps to that heading or line in the open section, and Backspace returns.
size: medium
decisions:
  missing_paint:
    ask:
      Should same-document links with no matching heading show as (missing) as soon as
      the page opens?
    choices:
      eager:
        They show ⊘ (missing) right away, checked against the section text before any
        press
      on_press:
        They look like normal links until pressed, then turn ⊘ like other dead links
    default: eager
    why:
      The check is a pure, I/O-free lookup, so you see a broken anchor before you press
      it
    answer: eager
proposed_by: bbugyi200.athena.11p
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 08:11:41
status: wip
---

# Pager: follow same-document anchor links (`[text](#heading)`)

## Problem

A Markdown document opened in `sase pager` links to its own headings:
`[Requirement adjustments](#requirement-adjustments)`. The pager already paints these as
labels (hints `1`–`7` in the reported screenshot), but pressing one fails. The toast
says `#requirement-adjustments not found (searched 0 locations)`, and the label turns
`⊘ … (missing)`.

Root cause, verified by probing the current code:

- The Rust document scanner (`scan_artifact_ref_document`, called from
  `src/sase/pager/link_scan.py`) emits these as `file_path` spans. Their
  `markdown_destination` and `target` are `#requirement-adjustments`. Skill links such
  as `[plan](#skill/sase_plan)` come through as `macro_skill` spans, so a `file_path`
  span whose destination starts with `#` always means a same-document anchor.
- Pressing the label goes `_activate_label` → `_follow_target` → `_resolve_and_dispatch`
  → `resolve_link` → `resolve_file_path_link`. In `_resolve_path_search.py`,
  `path_candidates("#slug")` splits the text into an empty path plus a fragment, and
  then drops the candidate because the path is empty. The search probes 0 locations and
  returns "not found", and `_apply_resolution` caches that in `_dangling_refs`.
- Nothing in the pager resolves a fragment against the document that is already open.
  The only heading-fragment code is `src/sase/pager/_resolve_fragments.py`
  (`fragment_target_line`). It handles cross-file links (`docs/guide.md#target-heading`)
  by re-reading the file from disk, and its slugger is simplistic: it does not skip
  fenced code, it collapses hyphens unlike GitHub, it does not strip link syntax, and it
  removes a trailing `#` from headings such as `# C#`.
- Rust's `split_link_location("#L10-L12")` already returns `base=''` plus a line
  location. Same-document line links like `[see](#L42)` can share the new code path.

## Goal

Pressing a same-document anchor label jumps to its heading **inside the current view**,
the way `;` goto does: the heading line gets the accent rail and sits at the reading
position. The jump also records a back-trail entry, so `Backspace`/`Ctrl+O` returns to
the link list and `Ctrl+I` jumps forward again. The press never runs the async file
resolver and never re-reads the file.

Acceptance, using the screenshot's research document: hints `1`–`6` (and `7`, once its
heading exists) each land on their `##` heading. `Backspace` returns to the bottom-line
paragraph with the same scroll position. The trail band reads
`<file> › <file>:<heading line>`.

## Design

### 1. Pure Markdown anchor index: new `src/sase/pager/markdown_anchors.py`

This module has no Textual or Rich imports and does no I/O. It is pure text in, lines
out, and the pager cold path loads it lazily.

- `build_markdown_anchor_index(text: str) -> MarkdownAnchorIndex` scans the text once,
  splitting lines on `"\n"` **only** and stripping one trailing `"\r"` per line. That
  matches the body layout's `logical_line_starts` numbering (`_body_layout.py`) and
  section gutter line numbers. Do not use `str.splitlines()`: it also splits on `\x0b`,
  `\x1c`, ` ` and similar characters, which would desynchronize the line numbers.
- Recognized anchors, with 1-based line numbers:
  - **ATX headings**: up to 3 leading spaces, `#{1,6}`, then a space or end of line. A
    closing `#` run is removed only when whitespace precedes it, so `# C#` keeps `C#`.
    Headings inside blockquotes count: strip leading `>` markers (each with one optional
    space) before matching.
  - **Setext headings**: a non-blank paragraph line followed by a line of only `=` or
    `-` (up to 3 leading spaces). A `---` line after a blank line is a thematic break,
    not a heading.
  - **Explicit ids** share the exact namespace with heading slugs, first in document
    order wins:
    - a trailing `{#custom-id}` attribute on a heading line (also removed from the
      heading text before slugging);
    - `<a id="x">` and `<a name="x">` (also `id=` on any HTML tag on the line);
    - Obsidian block ids, meaning a line that ends with ` ^block-id` or is just
      `^block-id`. The fragment key is `^block-id`, as in `[x](#^block-id)`.
- Skipped regions:
  - YAML frontmatter: a `---` first line through the next `---`/`...` line.
  - Fenced code: ` ``` `/`~~~` fences with 3 or more fence characters, closed by the
    same character with at least the same length. An unclosed fence runs to the end of
    the text. The screenshot's ` ```md ` block contains `### Page 11`, which must not
    become an anchor.
- Heading text for slugging is an approximation of the rendered text:
  - `[text](url)` becomes `text` and `![alt](url)` becomes `alt`;
  - inline-code backticks are removed and their content kept;
  - `*`/`~~` emphasis markers and HTML tags are removed.
- **Exact slugs** follow GitHub (github-slugger):
  - lowercase;
  - remove every character that is not a Unicode letter, number, mark, `-`, `_`, or
    space;
  - replace **each** space with `-` (no collapsing).

  Duplicate base slugs get `-1`, `-2`, … in document order.

- **Loose keys**, a fallback for Obsidian- or hand-written fragments such as
  `#Requirement%20adjustments` or `#the-marker` (GitHub would write `#the--marker`):
  casefold, turn each run of non-alphanumeric characters into one `-`, and trim `-`. The
  first heading in document order wins each loose key.
- `MarkdownAnchorIndex.line_for(fragment: str) -> int | None` does the lookup:
  1. percent-decode the fragment;
  2. empty fragment → line 1;
  3. exact match against the original string;
  4. exact match against the lowercased string;
  5. loose match;
  6. `top` with no explicit anchor named `top` → line 1 (HTML behavior);
  7. otherwise `None`.

Refactor `_resolve_fragments.py` so `_heading_fragment_line` reads the file and calls
`build_markdown_anchor_index(text).line_for(fragment)`. Delete the old
`_HEADING_RE`/`_heading_slug`. Keep the public `fragment_target_line` signature and its
error messages. Cross-file `guide.md#heading` links then get the same fence-aware,
GitHub-faithful behavior for free.

**Rust boundary.** This stays in Python on purpose:

- the only existing heading-fragment resolver is this Python module;
- `sase-core` has no Markdown heading/slug code (checked);
- the behavior navigates text the pager has already loaded, and only the pager uses it.

No sase-core change and no `sase-core-revision.txt` bump. If a web or editor frontend
later needs identical anchor slugs, move this module into `sase_core` then.

### 2. Target detection and memoized per-section lookup in `src/sase/pager/document.py`

- `local_anchor_fragment(target: PagerTargetSpan) -> str | None` returns the raw
  fragment (without `#`, not yet decoded) when **all** of these hold:
  - `target.source == "scanned"`;
  - `target.kind == LinkSpanKind.FILE_PATH.value`;
  - the destination (`semantic_target.markdown_destination`, else the string
    `target.target`) starts with `#`.

  Otherwise it returns `None`. `./#a`, `other.md#a`, skill spans, URLs, and attached
  targets all return `None`, so their behavior does not change.

- `section_local_anchor_lines(section, fragment) -> tuple[int, int] | None` returns the
  landing `(start_line, end_line)`, memoized in the section's `_memo`. This follows the
  module rule that only `section_*` helpers touch `_memo`. Store the anchor index under
  one key and a per-fragment result dict under another, so many anchors do not copy the
  memo dict once per insert. The lookup:
  1. `split_link_location("#" + fragment)`, which is already imported in this module.
     When it yields `base == ""` with a location, return `(line, end_line or line)`
     unclamped; the view clamps it.
  2. Otherwise build the section's anchor index from `section.plain_text` with a lazy
     import of `sase.pager.markdown_anchors`, the same pattern as
     `section_content_digest`, and return `(line, line)` or `None`.

  Sections are immutable, so the memo never goes stale. A history version or a refresh
  builds a new section object.

### 3. Label marker: `src/sase/pager/_labels.py`

In `_target_marker`, a non-dangling target for which `local_anchor_fragment(...)` is not
`None` gets the icon `§`, a single cell wide, with the existing files accent. This tells
"jump within this page" apart from "open a file". Dangling targets keep `⊘`.

### 4. Press handling: new mixin `src/sase/pager/_screen_actions_anchor.py`

Add `PagerActionAnchorMixin` to the `PagerActionMixin` bases in `_screen_actions.py`.
Following the sibling-module rule, it never imports the other `_screen_actions_*`
modules and calls their methods through `self`.

In `_activate_label` (`_screen_actions_labels.py`), make these changes:

- Place the branch after the attached-handler check and the URL/`copy` branch.
- Before the `edit` branch, compute `fragment = local_anchor_fragment(target)`.
- When it is not `None`, call
  `self._activate_local_anchor(label.section_index, fragment, action)` and return.
- `copy` keeps the generic path, so `y<label>` still copies the authored `#slug`
  destination. That path probes 0 locations. Add a regression test for this.

`_activate_local_anchor(section_index, fragment, action)`:

1. Resolve `section_local_anchor_lines(section, fragment)` on the label's **own**
   section, never the scroll-current section.
   - Never search other sections: in a multi-file document each section is its own file.
   - The lookup always uses the section text as displayed. In the history diff view, a
     heading folded away is not found, and that is acceptable.
2. **Miss**:
   - Toast a warning:
     `No heading or anchor matches #<decoded fragment> in <section title>.`
   - If `action == "other"`, call `self.pager_host._disarm_other()`.
   - Do not add a trail entry and leave the document unchanged.

   > [!decision] missing_paint = on_press
   >
   > Also record the miss in `_dangling_refs` under
   > `self._dangling_ref_key(("local-anchor", section.identity, fragment), context)`,
   > and relabel (`_invalidate_body_layout(relabel=True)`). The "Missing paint"
   > predicate below reads that key.

3. **`follow`**: call `_land_local_anchor(section_index, start, end)`, a lightweight
   in-place jump. Do **not** use `_navigate_to_document`: it rebuilds the whole layout,
   resets search and syntax state, and restarts history discovery. The steps:
   - `self._resolve_generation += 1` cancels any in-flight async follow, which would
     otherwise still land because the document object is unchanged. Then
     `self._set_footer_status(None)`.
   - Build the clamped mark with
     `self._line_mark_for_landing(line=start, end_line=end, section_index=section_index)`
     (see §5). If it returns `None`, show an information toast and stop.
   - `self._push_trail_entry()`, then `self._forward_trail.clear()`.
   - `self._goto_mark = mark`, `self._invalidate_body_paint()`,
     `self._scroll_to_line_mark(mark)`, `self._after_scroll()`.
   - Call `self._update_trail()`, `self._update_footer()`, and `self._update_subject()`.
     The subject and crumb then show the `:<line>` suffix.
4. **`other`** (`Ctrl+W <label>`): call
   `self._show_document_in_other_view(self.document, start, end, section_index=section_index)`.
   With one pane this opens a split clone, which then lands on the heading. You keep
   reading the list while the target pane shows the hint.
5. **`edit`** (`E<label>`): open `$EDITOR` at the heading line of the section's backing
   file. This reuses resolution: find the live ref the same way
   `_dispatch_section_action("edit")` does (`_history_live_path_for_section`, else
   `section.subject_ref`), with the same "nothing to edit" and "deleted" toasts. Then
   call
   `self._resolve_and_dispatch(ref, intent="edit", context=self._link_context_for_section_index(section_index), edit_line=start)`.
   - Add an optional `edit_line: int | None = None` keyword to `_resolve_and_dispatch`
     and pass it through to `_apply_resolution`. When `intent == "edit"` and `edit_line`
     is not `None`, apply
     `dataclasses.replace(target, edit_line=edit_line, edit_column=None)` before
     `_launch_editor`.
   - Pass `edit_line=None` when the section shows a past version (pin ordinal > 0). Its
     line numbers belong to that version, not to the live file.

**Missing paint.** Extend `_is_target_dangling` (`_screen_actions_resolve.py`) so
local-anchor targets bypass the resolver-keyed cache:

> [!decision] missing_paint = eager
>
> Return `section_local_anchor_lines(section, fragment) is None` for the target's
> section. Return `False` when the section is in diff view
> (`getattr(section.version_pin, "view", "read") == "diff"`, the same check
> `_prepared_section_texts` uses), because folds hide headings there. Both the labeled
> body and the search base (`style_target_accents`) already route through this
> predicate.

> [!decision] missing_paint = on_press
>
> Return whether the section-scoped `("local-anchor", section.identity, fragment)` key
> is in `_dangling_refs`. `r` (refresh) already clears that cache.

### 5. Landing in a chosen section

- `_line_mark_for_landing` (`_screen_goto.py`) always builds `LineMark(0, …)`. Add a
  keyword `section_index: int = 0`:
  - read `body.section_line_counts[section_index]`;
  - return `None` for an out-of-range index;
  - build `LineMark(section_index, …)`.

  Keep the existing clamp-and-toast behavior when the line is past the end.

- Thread the same `section_index: int = 0` keyword through:
  - `_navigate_to_document` (`_screen_actions_resolve.py`);
  - `_show_document_in_other_view`;
  - the `_PagerViewHost.show_in_other_view` protocol (`view.py`);
  - `show_in_other_view` in `_screen_split_arm.py`, at its four `_navigate_to_document`
    call sites: armed target, existing split, no-room fallback, and new split clone.

  Every existing caller keeps the default.

### 6. Same-document back/forward keeps syntax styling

`_restore_view_state` (`_screen_trail.py`) currently always calls
`_reset_syntax_for_new_document()`. Going back from a hint to the list restores the
**same** document object, so the reset would drop the prepared Markdown styling and show
a frame of unstyled text until the worker re-prepares it.

Capture `same_document = state.document is self.document` before assigning, and skip the
reset when it is true. Prepared-syntax keys (`_syntax_key_for_section`) are already
version-safe, so keeping them is correct even if history pins differ. Apply the same
guard in `_navigate_to_document` when `document is self.document`, so a `Ctrl+W` landing
in a pane already showing this document does not flash either. Everything else in both
methods stays as it is.

### 7. Help sheet and docs

- `src/sase/pager/_trail_chrome_help.py`: directly after the
  `"path:12 / path:12-40 / #L12"` row, add
  `("[text](#heading)", "Jump to that heading in this section; backspace returns")`.
  Extend `tests/pager/test_help.py` to assert the row and its order (after `path:12`,
  before `backspace`).
- `docs/reference/tui/pager.md` (Reference tier, per the docs-routing rule):
  - In "Line-addressed links", replace the stale sentence that says `README.md#usage`
    "is followed as a plain target". Heading fragments already land on the heading.
  - Add a short "Same-document links" paragraph covering:
    - `[text](#heading)` and `#L12`/`#L12-L40` jump within the current section;
    - matching rules: GitHub slugs, then case- and punctuation-insensitive;
      `{#id}`/`<a id>`/`^block` anchors; fenced code ignored;
    - the trail and `Backspace` behavior;
    - `§` labels and the `⊘` missing paint;
    - `Ctrl+W`/`E`/`y` behavior.

  Keep it within the page budget and add no guide prose. No config, CLI, or keymap
  changes, so `default_config.yml` and generated CLI docs are untouched.

## Tests

New `tests/pager/test_markdown_anchors.py` (pure, fast):

- ATX levels 1–6:
  - a closing `#` run is stripped, but `# C#` → `c`;
  - 4-space indentation is not a heading;
  - a blockquote heading counts.
- Setext `===`/`---`; `---` after a blank line and frontmatter delimiters are not
  headings.
- Fenced code (backtick and tilde, longer closing fence, unclosed fence) hides headings
  inside.
- GitHub slugs:
  - ``# Continuation highlights in `bob ref`: the `<` marker`` →
    `continuation-highlights-in-bob-ref-the--marker` (exact);
  - `#continuation-highlights-in-bob-ref-the-marker` also resolves (loose);
  - `[Link](u)` text, inline code, emphasis;
  - Unicode `## Café` with fragment `caf%C3%A9`;
  - duplicates get `-1`/`-2`.
- Obsidian-style `Requirement%20adjustments` resolves loosely; the lookup is
  case-insensitive.
- Explicit `{#id}` (and its removal from slug text), `<a id>`/`<a name>`, `^block-id`
  standalone and trailing.
- Empty fragment and `top` resolve to line 1; an unknown fragment returns `None`.
- `\r\n` input; a ` ` or form-feed character inside a line does not shift line numbers.
  Compare with the `"\n"`-only split.

`tests/pager/test_resolve_paths.py`: the existing heading-fragment and missing-fragment
tests still pass. Add one cross-file case where a heading inside a fence is skipped in
favor of the real heading after it.

`tests/pager/test_document.py` (or `test_target_resolution_ref.py`):

- `local_anchor_fragment` returns `"slug"` for `[x](#slug)`, `""` for `[x](#)`, and
  `None` for `[p](#skill/sase_plan)`, `[x](other.md#a)`, `[x](./#a)`, URLs, and attached
  targets.
- `section_local_anchor_lines` handles `#L5-L7` → `(5, 7)` and memoizes. The second call
  does not rebuild the index; monkeypatch the builder to count calls.

`tests/pager/test_labels.py`: a resolvable local anchor paints `§`. Under `eager`, a
missing one paints `⊘ (missing)` with no press.

New Pilot file `tests/pager/test_app_local_anchors.py`. Follow `test_app_goto.py` and
`test_app_navigation.py`: build a long Markdown `PagerSection` body with a link list at
the top and headings far below.

- **Follow**:
  - press the label;
  - `screen.document` is the same object;
  - `_goto_mark == LineMark(0, heading_line, heading_line)`;
  - scroll equals `reading_scroll_y` for that row;
  - the back trail grew by 1 and the forward trail is empty;
  - monkeypatch `sase.pager.screen.resolve_ref` to fail if called, and assert it never
    is.
- `Backspace` restores the prior scroll and mark. `Ctrl+I` lands on the heading again.
  `_syntax_generation` does not change across the same-document back.
- **Multi-section**: the same slug exists in sections 0 and 1. A label in section 1
  lands on `LineMark(1, …)`.
- `#L5-L7` lands a range mark. `#L9999` clamps with the information toast.
- **Missing anchor**: shown per the decision; a press shows the warning toast, adds no
  trail entry, and leaves the document unchanged.
- `Ctrl+W <label>` in a single pane opens a split whose new pane lands on the heading
  with the correct section index. The source pane's scroll is unchanged.
- `E<label>` calls the patched `sase.pager.screen.subprocess.run` with the heading line,
  following the `test_e_then_label_opens_the_editor_at_its_line` pattern.
- `y<label>` copies `#slug`.

No pager visual golden contains `](#` links (checked), so PNG snapshots should not
change. Do not run `just fix-tui-screenshots` unless a visual test actually fails.

## Verification

- Run `just fmt`, then `sase tool run check`. Do not run `check-full`.
- `tests/pager/test_cold_path_import_cost.py` must still pass. The new module is
  imported lazily from `document.py` and must not pull in Textual or Rich.
- Manual check: open the screenshot's research Markdown in `sase pager`. Press `1`–`6`
  and confirm each lands on its heading with the rail. Press `Backspace` each time to
  return.

## Out of scope

- Obsidian wikilinks `[[#Heading]]` and `[[Note#Heading]]`: the Rust scanner does not
  emit them as spans today.
- Searching other sections when the label's own section has no match.
- Expanding diff-view folds to reach a hidden heading.
- Moving anchor slugging into `sase_core`. Revisit if another frontend needs identical
  slugs.
