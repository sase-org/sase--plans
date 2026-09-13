---
tier: epic
title: Line-addressed links in the pager
goal: 'Every pager link that carries a line location (plain paths, Markdown destinations,
  and typed artifact refs, in colon `:12` / `:12:5` / `:12-40` or GitHub `#L12` /
  `#L12-L40` / `#L12C5` form) follows without an error and lands on that line. The
  referenced line or range is marked with an accent rail in the gutter, and copy (`y`)
  and edit (`E`) carry the same location.

  '
phases:
- id: core-location
  title: Rust link-location grammar
  depends_on: []
  size: medium
  description: 'core-location: in sase-core, add the one link-location grammar (split_link_location
    plus its binding), accept GitHub-style line fragments in parse_fragment, and extend
    the document file-path scanner so a location suffix stays inside the link span.'
- id: landing-mark
  title: Pager landing placement and range rail
  depends_on: []
  size: medium
  description: 'landing-mark: generalize the goto mark into a line/range mark rendered
    as an accent gutter rail, place landed lines at a reading position shared with
    `;`, clamp past-EOF lines with a gentle toast, carry the mark through back/forward
    history and the breadcrumb.'
- id: location-resolution
  title: Location-first link resolution, copy, and the corpus
  depends_on:
  - core-location
  - landing-mark
  size: medium
  description: 'location-resolution: ratchet sase-core, add the Python adapter, split
    locations before resolving every pager ref (including the ACE LinkIndex fast path),
    retire the duplicate Python line grammar, make copy include the location, and
    pin every form in the rendered-link corpus and a pilot navigation test.'
proposed_by: bbugyi200.athena.1o
create_time: 2026-09-13 10:09:09
status: done
bead_id: sase-108
---

- **PROMPT:** [prompts/202609/pager_line_addressed_links.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pager_line_addressed_links.md)
- **BEAD:** [sase-108](https://github.com/sase-org/sase--beads/blob/main/pages/sase-108/README.md)

# Plan: Line-addressed links in the pager

## Context: what is broken today

The pager (`src/sase/pager/`) paints letter labels (`[b]`, `[c]`, …) over scanned links.
The Rust document scanner owns link spans. Python resolves the followed target string in
`pager/resolve.py::resolve_link`, then `_screen_actions.py::_apply_resolution`
navigates. Line locations are handled inconsistently across three grammars (a Rust scan
regex, the Python `_resolve_path_search._split_line_suffix`, and
`_resolve_fragments._LINE_FRAGMENT_RE`), and typed artifact refs never learned a colon
location at all. I probed the current code; the results:

| Link as written in a document               | Today                                                                                       |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `src/sase/pager/resolve.py:27`, `…:27:5`    | works                                                                                       |
| `plan:202609/foo.md:12`, `research:x.md:40` | **error**: `artifact reference filtered: kind=plan; payload=202609/foo.md:12; path_globs=…` |
| `memory:gotchas.md:3`                       | **error**: `could not be resolved.` (`:3` became part of the payload)                       |
| `[label](src/…/resolve.py:27-44)`           | **error**: `… not found (searched 2 locations)`                                             |
| `[label](src/…/resolve.py#L27C3)`           | **error**: `fragment #L27C3 is not a supported line fragment`                               |
| `plan:x.md#L12C5`                           | not a link at all (Rust `parse_fragment` rejects it → `well_formed=false`)                  |
| `src/…/resolve.py#L27`, `#L27-L44` in prose | span stops before `#`, file opens at the **top** (location silently lost)                   |
| `src/…/resolve.py:27-44` in prose           | span stops before `-44`, lands on 27 (range lost)                                           |
| any successful line landing                 | line is scrolled to the very top row with **no indication** of which line was referenced    |

The user's failing hints (`b`, `c`, `d`, `i`) are this class of link. The request says
"page numbers" but asks to "jump directly to that line number". This epic is about
**line** locations; PDF `#page=N` anchors are untouched.

## Design principles

1. **One grammar, one owner.** Link-location syntax is document-link grammar, which the
   Rust core already owns (`rust_core_backend_boundary`). The Rust split function is the
   only parser; Python stops carrying its own line regexes.
2. **A location is orthogonal to the thing it locates.** Split the location off _before_
   resolving, resolve the base ref exactly as today, then apply the location to the
   landing. Every ref kind — plain path, Markdown destination, typed artifact ref, ACE
   LinkIndex hit — gets line support from the same code path, and diagnostics name the
   real missing thing (never `…:12`).
3. **A location never causes an error.** A resolvable file with a past-EOF line still
   opens and lands on the last line, with an informational toast. A location on a
   non-file landing (bead card, commit, directory) is ignored silently.
4. **Show where you landed.** The referenced line or range gets an accent rail in the
   gutter, placed at a reading position with context above. Back/forward history
   remembers it.

## Supported forms (the contract every phase tests against)

| Written                                       | base               | line | column | end_line |
| --------------------------------------------- | ------------------ | ---- | ------ | -------- |
| `src/app.py:12`                               | `src/app.py`       | 12   | –      | –        |
| `src/app.py:12:5`                             | `src/app.py`       | 12   | 5      | –        |
| `src/app.py:12-40`                            | `src/app.py`       | 12   | –      | 40       |
| `src/app.py:12:5-40`                          | `src/app.py`       | 12   | 5      | 40       |
| `src/app.py#L12`                              | `src/app.py`       | 12   | –      | –        |
| `src/app.py#L12-L40`, `src/app.py#L12-40`     | `src/app.py`       | 12   | –      | 40       |
| `src/app.py#L12C5`                            | `src/app.py`       | 12   | 5      | –        |
| `src/app.py#L12C5-L40C2`                      | `src/app.py`       | 12   | 5      | 40       |
| `plan:202609/x.md:12`, `@plan:202609/x.md#L3` | `plan:202609/x.md` | 12/3 | –      | –        |

`L`/`C` letters are case-insensitive (parity with today's Python `IGNORECASE` fragment
regex).

Not a location (the target is returned whole, `location = None`):

- no suffix; `docs/guide.md#usage` (heading fragments stay on the base for the existing
  Python Markdown-heading lookup); `bead:sase-uk.1`; `commit:abc1234`.
- line or column `0` (`:0`, `#L0`).
- colon form whose base has no `/` or `.` (keeps `plan:12`, `bug:12345`, `plan:foo:12`
  whole).
- colon form whose base itself ends in `:digits` (`a/b.py:1:2:3` — preserves today's
  `_TRAILING_LINE_DIGITS_RE` guard).
- An end line smaller than the start (`:40-12`) keeps `line=40` and drops the end. An
  end equal to the start is reported as `end_line = None`.

Explicit non-goals (candidates for follow-up task beads, not this epic): ACE's numeric
`v` hints and prompt-panel jump/preview matchers; prose forms (`line 12`, `(line 12)`,
`File "x.py", line 12`, `x.py(12)`, en-dash ranges); a colon suffix after a quoted
typed-ref argument (`@plan:"a b.md":12` — `#L12` already works there);
`sase artifact read` accepting a colon suffix; bare `path.md#heading` in prose; PDF
`#page=N` landing.

## Phase core-location — Rust link-location grammar

Work in the sase-core repo (open it with `/sase_repo`; it is the linked Rust core,
`crates/sase_core` plus the pyo3 crate `crates/sase_core_py`).

1. **New module** `crates/sase_core/src/artifact_ref/location.rs` (re-exported from
   `artifact_ref/mod.rs`):
   - `pub fn split_link_location(target: &str) -> LinkLocationSplitWire`, with
     `LinkLocationSplitWire { schema_version, base: String, location: Option<LinkLocationWire> }`
     and `LinkLocationWire { line: u64, column: Option<u64>, end_line: Option<u64> }` in
     `artifact_ref/wire.rs` (serde, `skip_serializing_if = "Option::is_none"` for
     optional fields, matching neighboring wires). Add
     `LINK_LOCATION_WIRE_SCHEMA_VERSION = 1`.
   - Match anchored at the end of the target. Colon form:
     `:(\d+)(?::(\d+))?(?:-(\d+))?$`. Fragment form:
     `#[Ll](\d+)(?:[Cc](\d+))?(?:-[Ll]?(\d+)(?:[Cc]\d+)?)?$` (the end column is accepted
     and discarded). Apply every guard from the "Not a location" list. Pure, no I/O, no
     allocation beyond the result.
2. **`parse_fragment`** (`artifact_ref/mod.rs`): additionally accept `L12-40`, `L12C5`,
   and `L12C5-L40C2` as `Lines { start, end }`. Columns are accepted and dropped.
   `render_fragment` keeps its canonical `L12` / `L12-L40` output. That makes
   `plan:x.md#L12C5` well-formed, so the document scanner links it. Keep the
   end-before-start validation error.
3. **Document file-path scanner** (`artifact_ref/scanner.rs::scan_document_file_paths`):
   widen the optional suffix from `(?::\d+(?::\d+)?)?` to the colon-or-fragment location
   grammar above, so `src/foo.py:12-40`, `src/foo.py:12:5-40`, and `src/foo.py#L12-L40`
   are one span. Keep `trim_file_path_end` (sentence dots) and URL precedence unchanged;
   `src/foo.py:12-` still yields `src/foo.py:12`.
4. **Binding**: expose `artifact_ref_split_link_location(target: str) -> dict` and
   `artifact_ref_link_location_wire_schema_version() -> int` from
   `crates/sase_core_py/src/lib.rs`, following `artifact_ref_scan_document`'s JSON-value
   pattern.
5. **Tests** (Rust unit tests beside the code): a table test covering every row of both
   tables above; `parse_fragment` round-trips for the new forms; document-scan span
   tests for `see src/foo.py:12-40.`, `(see src/foo.py#L12)`, `src/foo.py#L12C5-L40C2`,
   a Markdown destination `[x](src/foo.py:27-44)`, and a URL containing `:12` staying
   one URL span. Run sase-core's own lint and test gates before finishing.

## Phase landing-mark — Pager landing placement and range rail

Python only, no Rust dependency, so it can run in parallel with `core-location`.
Everything here is UI-thread work over the already-composed body: no stat, read, or
subprocess (`tui_perf` rule 1).

1. **Line mark model.** Replace `_goto_mark: tuple[int, int] | None` (`_screen_goto.py`)
   with a frozen `LineMark(section_index: int, start_line: int, end_line: int)` (new
   small module, e.g. `src/sase/pager/_line_mark.py`). `;` goto sets a single-line mark
   (`start == end`). `compose_body(goto_mark=…)` (`_layout.py`) and
   `_screen_syntax.py`'s prepared compose take a `line_mark`.
   `apply_gutter(emphasis_line=…)` (`_gutter.py`) takes an inclusive
   `emphasis_range: tuple[int, int] | None`.
2. **The rail (the beautiful part).** For every visual row of every logical line inside
   the marked range, including wrapped continuation rows, the gutter separator `│ `
   becomes `┃ ` in the mark accent. The number on each logical line's first row renders
   `bold <accent>`, today's emphasis. Rows outside the range are unchanged. The gutter
   cell count does not change, so wrapping, label offsets, and search offsets are
   unaffected. Accent stays `section_accent(section.kind)` via `_goto_accent_for_mark`.
3. **Reading-position placement.** One helper, `_scroll_to_line_mark(mark)`, used by
   both `;` goto (replacing today's top-align in `_submit_goto`) and link landing:
   `context = clamp(viewport_height // 4, 2, 8)`; `y = start_row - context`. If the
   range fits in the viewport but its end row would fall below it,
   `y = max(end_row - viewport_height + 1, start_row - 1)`. Clamp `y` to
   `[0, max_scroll_y]`. Update `tests/pager/test_app_goto.py` expectations
   intentionally.
4. **Landing.** `LinkTarget` (`targets.py`) gains `scroll_end_line: int | None = None`,
   and `file_link_target(..., requested_end_line=None)` (`_resolve_common.py`) plumbs
   it. `_apply_resolution` passes `line`/`end_line` into
   `_navigate_to_document(document, line=…, end_line=…)`. That method sets the mark
   after `_clear_goto_state()`, then scrolls with the helper. If `line` exceeds the
   section's line count (from `body.section_line_counts`), mark and land on the last
   line and
   `notify("resolve.py has 98 lines — showing line 98.", severity="information")` (use
   the document title). An end line past EOF is clamped silently.
5. **History and breadcrumb.** `PagerTrailEntry` (`trail.py`) gains
   `line_mark: LineMark | None`. `_current_view_state` captures it;
   `_restore_view_state` restores it after `_clear_goto_state()`, so back/forward
   returns to the railed view. The breadcrumb label for an entry with a mark appends the
   location (`resolve.py:27`, or `resolve.py:27–44` for a range, en dash for display
   only). Derive it in the trail display model (`_trail_chrome_model.py`) from the
   entry's mark rather than mutating `PagerDocument.title`, so document identity and
   signatures stay stable.
6. **Help.** In the pager help (`src/sase/pager/_help.py`), add one line in its existing
   format: links with a line (`path:12`, `path:12-40`, `#L12`) land on and rail that
   line, and `E` opens the editor there.
7. **Tests.** `test_gutter.py` (rail cells over a range, wrapped rows, unchanged
   widths); `test_layout.py`; `test_app_goto.py` (placement); `test_app_navigation.py`
   (a synthetic `LinkTarget` with `scroll_line`/`scroll_end_line` sets the mark and
   placement, the past-EOF toast, back/forward restoring the mark);
   `test_trail_chrome.py` (crumb suffix). If `tests/pager/visual` snapshots cover the
   body gutter, add or refresh a railed-range snapshot and run `just test-visual`.

## Phase location-resolution — Location-first link resolution, copy, and the corpus

1. **Pick up the core.** Ratchet `sase-core-revision.txt` to the sase-core commit that
   landed `core-location`, and follow whatever the Justfile and
   `tools/validate_sase_core_rs` expect for a new binding. Add
   `artifact_ref_split_link_location` and
   `artifact_ref_link_location_wire_schema_version` to that tool's required-binding
   list.
2. **Adapter.** In `src/sase/artifact_ref_operations.py`, next to
   `scan_artifact_ref_document`, add
   `split_link_location(target: str) -> LinkLocationSplit` with a schema check mirroring
   `_require_artifact_ref_document_scan_schema`. Put the frozen
   `LinkLocation(line, column, end_line)` and `LinkLocationSplit(base, location)` models
   (`from_wire`) in the artifact-ref model module that fits under the file-size gate.
3. **Location-first resolution.** Add a pager helper module (e.g.
   `src/sase/pager/_resolve_location.py`) with
   `apply_link_location(target: LinkTarget | None, location) -> LinkTarget | None`. For
   a DOCUMENT target whose `edit_path` is a regular file, it sets `scroll_line`,
   `scroll_end_line`, `edit_line`, and `edit_column`. For a MEDIA target it sets
   `edit_line`/`edit_column`. Any other landing is returned unchanged. Use
   `dataclasses.replace`; no I/O. `resolve.py::resolve_link` becomes: strip →
   `split_link_location` → dispatch the **base** through today's exact branches (file
   path / bead / artifact ref) → apply the location to a returned target. Diagnostics
   from the base pass through unchanged. For a typed artifact ref whose base did not
   resolve, retry the whole unsplit ref once and use it if it resolves (protects any
   payload that legitimately ends in `:digits`). Plain file paths get no whole-ref
   retry: they already treated `:N` as a location, and a retry would double the dead-end
   search. `_resolve_ref_from_link_index` (`src/sase/ace/tui/actions/hints/_files.py`)
   splits first, looks the **base** up in the LinkIndex, and applies the location to the
   fast-path target with the same helper.
4. **Retire the duplicate Python grammar.** Delete `_LINE_COL_SUFFIX_RE`,
   `_LINE_SUFFIX_RE`, `_TRAILING_LINE_DIGITS_RE`, and `_split_line_suffix` from
   `_resolve_path_search.py`. `path_candidates` / `search_existing_path` stop returning
   line/column (keep trailing-dot and `a/`/`b/` diff-prefix variants and the `#fragment`
   split for headings). Drop the line branch and the "not a supported line fragment"
   error from `_resolve_fragments.py`, keeping Markdown heading lookup. Update
   `_resolve_file_paths.py` (`_owned_file_path_resolution`,
   `_link_resolution_for_existing_path`) and `_resolve_artifact_refs.py`
   (`_fragment_line` stays only as a harmless no-op path for any surviving `lines`
   fragment; locations now come from the split). Run symvision and fix any now-unused
   symbols properly.
5. **Copy and edit.** `copy_text_for_target` for a FILE_PATH that resolves to a file
   copies `"{abs_path}:{line}"`, or `"{abs_path}:{line}:{column}"` when a column is
   present (the range end is omitted: editors, grep, and agents all read
   `path:line[:col]`). Typed artifact refs keep copying their ref verbatim, since the
   durable ref round-trips. `E` already uses `edit_line`/`edit_column` through
   `build_jump_editor_argv`; verify it now lands for typed refs and `#L12C5`.
6. **Keep bounding in step.** Widen `_PAGER_FILE_PATH_PATTERN` in
   `src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py` (used only by
   `scan_bounded_links`' budget matcher) to the same suffix, with a comment naming the
   Rust grammar as the owner, so truncation never cuts a location in half. ACE's own
   `iter_file_path_matches` stays byte-identical.
7. **Contract corpus and pilot.** Add a "line-addressed links" block to
   `tests/pager/_rendered_link_corpus.py`, exercised by `test_rendered_link_contract.py`
   / `test_rendered_link_navigation.py`. Cover `src/…:12`, `:12:5`, `:12-40`, `#L12`,
   `#L12-L40`, `#L12C5`, `[label](path:27-44)`, `[label](path#L27C3)`, `plan:…md:12`,
   `@plan:…md:12-20`, `plan:…md#L3C2`, a configured sidecar kind with `:40`, and a
   past-EOF `:9999`. For each, assert: exactly one label span covering the whole suffix;
   following it produces a document landing with the expected
   `scroll_line`/`scroll_end_line`; no warning toast (only the past-EOF information
   toast); `y` copy text per step 5. Extend `test_resolve_paths.py`,
   `test_resolve_artifact_refs.py`, `test_resolve_path_search.py`, `test_copy_owned.py`,
   and `test_link_scan.py` for the moved grammar. Add one Textual pilot test
   (`_rendered_link_pilot.py` style): press the label on a `plan:…md:12-14` link and
   assert the body scrolled to the reading position and the gutter rail covers lines
   12–14.

## Verification (every Python phase)

Run `just install` if the workspace venv is stale, then `just check`. Hand
`just check-full` to `/sase_monitor` before the epic lands, since this touches the
shared pager resolver.
