---
tier: epic
title: Bare `@` opens one reference menu for artifact kinds and local files
goal: 'Typing `@` in the ACE prompt input or in an LSP-backed editor immediately opens
  a single reference menu whose artifact-kind rows and local file-path rows are visibly
  grouped, with menu policy shared by Rust core so both frontends agree, and with
  no filesystem work on the keystroke path.

  '
phases:
- id: core
  title: Shared `@` reference menu core
  depends_on: []
  size: medium
  description: 'core: add the `sase_core::editor::at_reference` module — cursor context
    detection that accepts a bare `@` and path-shaped tokens, plus a pure I/O-free
    grouped menu builder whose inventory is supplied by the caller.

    '
- id: binding
  title: PyO3 bindings for the reference menu
  depends_on:
  - core
  size: small
  description: 'binding: export `at_reference_context` and `at_reference_menu` from
    `sase_core_py` as JSON-dict functions in the style of `placeholder_completion`,
    with binding-level tests.

    '
- id: lsp
  title: Editor LSP reference completion
  depends_on:
  - core
  size: medium
  description: 'lsp: route `sase lsp` artifact completion through the shared module,
    enumerate local path inventory from the client root, and shape items so any client
    filters and orders the two groups correctly.

    '
- id: panel_rows
  title: Completion panel row budget
  depends_on: []
  size: small
  description: 'panel_rows: derive the visible-row budget from the panel''s real content
    capacity so the highlighted row and the overflow indicator are never clipped,
    and reserve a line for a group rule when one is rendered.

    '
- id: tui_paths
  title: Warm local path inventory for the prompt
  depends_on: []
  size: medium
  description: 'tui_paths: add a mtime-validated directory-listing snapshot cache
    with a background loader, a loading row, and warm-on-focus, so prompt path rows
    never touch disk from a keystroke.

    '
- id: tui_menu
  title: TUI reference menu behavior
  depends_on:
  - binding
  - tui_paths
  size: medium
  description: 'tui_menu: rewire the TUI provider onto the shared binding, open on
    a bare `@`, merge the warm path inventory, and define accept, dismissal, and Enter-ownership
    rules.

    '
- id: render
  title: Grouped menu rendering
  depends_on:
  - tui_menu
  - panel_rows
  size: medium
  description: 'render: give the merged menu its aligned row anatomy, group rule line,
    adaptive border title, and a PNG snapshot golden.

    '
- id: docs
  title: Documentation and help sync
  depends_on:
  - render
  - lsp
  size: small
  description: 'docs: update the ACE completion docs, configuration reference, editor
    reference, and the `?` help popup entry for the new bare-`@` behavior.

    '
create_time: 2026-07-29 18:22:57
status: wip
bead_id: sase-ay
---

- **BEAD:** [sase-ay](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ay/README.md)

# Plan: Bare `@` opens one reference menu for artifact kinds and local files

## Problem

`@` completion is gated behind two typed characters, and it only ever offers artifact kinds:

- `src/sase/ace/tui/widgets/_file_completion_open.py:238-246` refuses to open the menu for a bare `@` and requires
  `len(context.partial_kind) >= 2` for an automatic open.
- `src/sase/ace/tui/widgets/artifact_ref_completion.py:191` abandons the artifact context as soon as the token looks
  path-shaped, and no automatic provider picks up the token afterwards, so `@src/` opens nothing.
- The LSP path (`crates/sase_core/src/editor/completion.rs`) returns kind candidates whose `label`, `insertion`, and
  `filter_text` all omit the `@` sigil, so client-side filtering of the typed `@…` word behaves inconsistently across
  clients and local files never appear.

The user wants `@` itself to open the menu, and wants local (working-directory-relative) file paths in that menu,
clearly separated from artifact prefixes.

## Design

**D1 — One menu, two groups.** A single menu owns the whole `@…` token before a `:` appears. It has two groups in fixed
order: `artifacts` (kind rows such as `@plan:`, `@chat:`) then `files` (local path rows such as `@src/`, `@Justfile`).
Artifacts lead because they are a small closed vocabulary; files are open-ended. Each group filters independently
against the same typed query, so `@pl` can show both `@plan:` and `plans/`, and `@src/` shows only file rows because no
kind can contain `/`. The payload stage (`@kind:…`) is unchanged and stays single-group.

**D2 — Rust owns policy, frontends own inventory.** The shared module is pure: it decides context, filtering, grouping,
ordering, caps, and shared-prefix extension, and performs no I/O. Each frontend passes in the row inventory it already
has (TUI: warm catalogs; LSP: bounded `read_dir`). This satisfies the Rust-core backend boundary without migrating the
TUI's catalog loaders, keeps the shared code trivially testable, and makes it safe to call from a keystroke handler.

**D3 — Warm-only file rows in the TUI.** Per `sase/memory/tui_perf.md` rules 8 and 11, the keystroke path must not
`stat`, `scandir`, or block. File rows therefore come from an in-memory directory-listing snapshot cache; misses
schedule a background worker and refresh the open menu when it lands, exactly as `_schedule_vcs_repo_completion_fetch` /
`_apply_vcs_repo_completion_result` already do for repository rows. This applies to manual `Ctrl+T` as well, so there is
exactly one code path and no ordering race between a synchronous and an asynchronous source.

**D4 — File rows insert the `@` sigil.** Accepting a file row inserts `@src/foo.py`, not `src/foo.py`. `@`-prefixed
paths are the established file-reference spelling: `sase.history.file_references.extract_recordable_file_refs` records
them (stripping the `@`) while bare relative paths are deliberately ignored, and `is_path_like_token` already strips a
leading `@`. Keeping the sigil means the menu produces exactly the token the rest of SASE treats as an intentional file
reference.

**D5 — An un-narrowed menu does not own `Enter`.** `Enter` currently accepts whenever a completion menu is open
(`_prompt_text_area_key_handling.py:205-206`). A bare `@` is common in prose ("@mention", `@decorator`), so an
always-open menu would let `Enter` insert `@commit:` instead of submitting. Rule: while the menu's query is empty
**and** the user has not moved the selection, `Enter` submits and dismisses the menu; `Ctrl+L` still accepts. After the
first query character or the first `Ctrl+N`/`Ctrl+P`/`Up`/`Down`, the menu behaves like every other completion menu.
Nothing is ever inserted without an explicit accept.

**D6 — Group separators are rendering-only.** The group rule is drawn by the panel, never inserted into
`_file_completion_candidates`, so selection indexing, `_move_file_completion`, accept, and refresh keep working
unchanged.

**D7 — The panel must stop clipping its highlight.** `#prompt-completion` has `max-height: 10` with a 2-row border, so
only 8 content lines are visible, while `_update_file_completion_panel` windows `MAX_VISIBLE = 10` rows and may add an
overflow line. With 15 candidates and the last row selected, `scroll_offset` clamps to `total - 10` and the highlight
lands on the tenth line — invisible. `docs/ace.md` already claims the panel "scrolls to keep the highlight visible", so
this is a real defect that the new, denser menu would hit constantly. Fix it before adding a group rule line.

**D8 — Naming.** The internal completion kind stays `"artifact_ref"` to avoid churning every dispatch site; the
user-facing name in titles, docs, and help is "`@` reference".

## Cross-repo sequencing

`core`, `binding`, and `lsp` change the sibling Rust repo; open it with `sase repo open sase-core -r "<reason>"` and use
the printed path for every read and write. They must land and be pushed before `tui_menu` starts: CI builds the
`sase-core-rs` wheel from that repo's default-branch HEAD, and a phase agent's `sase repo open` checkout is refreshed to
`origin/master`, so an unlanded binding is invisible to the Python phases. `panel_rows` and `tui_paths` touch only this
repo and can run immediately, in parallel with the Rust work.

## Shared `@` reference menu core

New module `crates/sase_core/src/editor/at_reference.rs`, re-exported from `crates/sase_core/src/lib.rs` alongside the
existing `editor_*` surface. Keep the existing `scan_artifact_refs` scanner as the source of truth for well-formed
references; this module owns the _incomplete_ cursor context and the merged menu.

### Wire types

```rust
pub enum AtReferenceStage { Kind, Payload }
pub enum AtReferenceGroup { Artifact, File, Payload }

pub struct AtReferencePathQueryWire {
    pub directory: String,  // token text through the last '/', "" for the base directory
    pub partial: String,    // trailing segment used to filter entries
    pub show_hidden: bool,  // partial starts with '.'
}

pub struct AtReferenceContextWire {
    pub stage: AtReferenceStage,
    pub candidate_span: (usize, usize),    // byte offsets of the whole `@…` token
    pub replacement_span: (usize, usize),  // bytes an accepted row replaces
    pub query_span: (usize, usize),
    pub query: String,                     // stage-local filter text
    pub kind: Option<String>,              // canonical kind, Payload stage only
    pub path_query: Option<AtReferencePathQueryWire>,  // Kind stage only
}

pub struct AtReferenceKindRowWire { pub kind: String, pub builtin: bool, pub detail: String }
pub struct AtReferencePathRowWire { pub name: String, pub is_dir: bool }
pub struct AtReferencePayloadRowWire { pub payload: String, pub label: String, pub detail: String, pub age: String }

pub struct AtReferenceInventoryWire {
    pub kinds: Vec<AtReferenceKindRowWire>,
    pub paths: Vec<AtReferencePathRowWire>,      // already scoped to path_query.directory
    pub payloads: Vec<AtReferencePayloadRowWire>,
}

pub struct AtReferenceRowWire {
    pub group: AtReferenceGroup,
    pub label: String,      // "plan", "src/", "Justfile", or the payload text
    pub insertion: String,  // "@plan:", "@src/", "@Justfile", "@kind:payload"
    pub is_dir: bool,
    pub detail: String,
    pub builtin: bool,
}

pub struct AtReferenceMenuWire {
    pub rows: Vec<AtReferenceRowWire>,
    pub shared_extension: String,
    pub artifact_count: usize,
    pub file_count: usize,
}
```

### Detection

`pub fn detect_at_reference_context(document: &DocumentSnapshot, position: EditorPosition, known_kinds: &[String]) -> Option<AtReferenceContextWire>`

- Preserve today's left-context rule from `incomplete_artifact_kind_candidate`: the `@` must start the text, follow
  whitespace, or follow `"`, `'`, or a backtick. Preserve the token-end scan and the `ARTIFACT_REF_PREFIX_LOOKBACK`
  bound. Preserve exclusion of prompt literal zones.
- A well-formed reference found by `scan_artifact_refs` keeps its current Kind/Payload split around the separator.
- **Change 1 — a bare `@` is a context.** Cursor at `@`'s right edge yields `stage: Kind`, `query: ""`,
  `path_query: Some({directory: "", partial: "", show_hidden: false})`. The current requirement that some known kind
  prefix-match the query is dropped: an empty query matches every kind anyway, and a query that matches no kind is now
  legitimate because it can still match files.
- **Change 2 — path-shaped tokens stay in this context.** `@src/fo`, `@~/dev/`, `@./x`, `@../x`, `@/etc/h`, and
  `@.sase/` all produce `stage: Kind` with the query split into `path_query.directory` (through the last `/`) and
  `path_query.partial`. Kind rows simply cannot match such a query, so the group empties by construction rather than by
  a special case. `show_hidden` is `true` when `partial` starts with `.`.
- A query containing a character that is neither a kind character (`[A-Za-z0-9_-]`) nor part of a path (`/`, `.`, `~`)
  yields `None`, so prose such as `@foo!` stops claiming the cursor.
- Payload-stage detection, fragment handling (`#lines`, `bug` exemption), and `replacement_span` semantics are
  unchanged; assert this with the existing tests.

### Menu assembly

`pub fn build_at_reference_menu(context: &AtReferenceContextWire, inventory: &AtReferenceInventoryWire) -> AtReferenceMenuWire`
— pure, allocation-only, no filesystem access.

- Kind stage, `artifacts` group first: builtin kinds in `BUILTIN_ARTIFACT_REF_KINDS` order, then remaining kinds sorted
  case-insensitively; keep only kinds whose lowercase form starts with the lowercase query; `insertion` is `@{kind}:`.
- Kind stage, `files` group second: keep entries whose lowercase name starts with `path_query.partial` lowercased; drop
  names starting with `.` unless `show_hidden`; order directories before files, then case-insensitively by name; `label`
  is `{name}/` for directories and `name` for files; `insertion` is `@{path_query.directory}{label}`.
- Payload stage: one `Payload` group built from `inventory.payloads`, matching payload or label prefixes, preserving the
  caller's order (callers already sort by recency).
- Cap each group at `AT_REFERENCE_MAX_GROUP_ROWS = 200`; record the pre-cap counts in `artifact_count` / `file_count`.
- `shared_extension` is computed from the **leading non-empty group only**, so `Ctrl+T` prefix extension never mixes a
  kind prefix with a filename prefix. Empty when that group has fewer than two rows or the common prefix does not extend
  the query.

### Tests

Add to the module's `#[cfg(test)]` block in `crates/sase_core/src/editor/at_reference.rs`:

- Context at every cursor position of `@`, `@p`, `@plan`, `@plan:`, `@plan:a/b.md`, `@src/`, `@src/fo`, `@~/dev/`,
  `@../x`, `@.sase/`, and rejection of `mail@example.com`, `word@`, `@foo!`, and an `@` inside a fenced literal zone.
- Menu assembly: bare `@` returns all builtin kinds then all visible entries; `@pl` narrows both groups; `@src/` returns
  files only; dotfile visibility flips with a leading `.`; directories precede files; group caps hold; shared extension
  comes from the leading group only; payload stage is untouched.
- Keep every existing artifact-completion test in `crates/sase_core/src/editor/completion.rs` green, re-pointing the
  internal callers at the new module and leaving the public `detect_artifact_ref_context_at_position` /
  `build_artifact_ref_*_completion_candidates` names in place as thin adapters if other callers rely on them.

Run `just rust-check` (fmt, clippy, tests) from the sase repo workspace before committing.

## PyO3 bindings for the reference menu

In `crates/sase_core_py/src/lib.rs`, follow `py_placeholder_completion` exactly: JSON-serializable dicts in and out, a
module-doc bullet, registration in the module init, and inline Rust tests that exercise the Python-facing signature.

- `at_reference_context(text: str, line: int, character: int, known_kinds: list[str]) -> dict | None` — LSP UTF-16
  position in, `AtReferenceContextWire` as a dict out.
- `at_reference_menu(context: dict, inventory: dict) -> dict` — `AtReferenceMenuWire` as a dict.

Both must be cheap enough for a keystroke: no I/O, no catalog loading, no global locks. Bump the `sase_core_py` and
`sase_core` crate versions per that repo's release convention so the sase-side `sase-core-rs` window can be widened when
needed, and note in the phase commit whether `pyproject.toml`'s `sase-core-rs>=0.12.12,<0.13.0` window needs a bump.

## Editor LSP reference completion

In `crates/sase_xprompt_lsp/src/server.rs`:

- Replace the `detect_artifact_ref_context_at_position` call in `completion_for_text` with
  `detect_at_reference_context`, keeping the existing early-return shape that answers artifact contexts before the
  xprompt catalog is awaited.
- Build the inventory:
  - `kinds` from `known_artifact_ref_kinds(artifact_context)`, with `detail` "builtin artifact kind" or
    `document artifact · {root}` as today.
  - `paths` by resolving `path_query.directory` against `config.root_dir` — expand a leading `~`, honor absolute
    directories, reject anything that fails to canonicalize — then a single bounded `read_dir` (cap 1000 entries, no
    recursion, no symlink following beyond `file_type()`). A missing or unreadable directory yields an empty group, not
    an error.
  - `payloads` from the existing chat/document/artifact-index loaders.
- Shape items in `crates/sase_xprompt_lsp/src/lsp_convert.rs` with a dedicated `at_reference_completion_item`, modeled
  on `vcs_project_completion_item`:
  - `label` and `filter_text` are the sigil-inclusive `insertion` (`@plan:`, `@src/`), so a client filtering the typed
    `@…` word matches regardless of its keyword pattern. This is the concrete fix for the "needs two characters"
    behavior in editors.
  - `text_edit` covers `candidate_span`, i.e. it includes the `@`.
  - `kind` is `ENUM_MEMBER` for artifact rows, `FOLDER` for directories, `FILE` for files.
  - `label_details.description` is `artifact kind` or `file` / `directory`.
  - `sort_text` is `"{group}:{index:04}"` with `group` 0 for artifacts and 1 for files, so client-side sorting cannot
    interleave the groups.
- `@` is already an advertised trigger character; add a test asserting that and asserting a bare `@` at cursor 1 returns
  both groups with artifact rows first.
- Tests: extend the server tests with a fixture root containing a directory and a file, and assert bare `@`, `@p`,
  `@src/` (files only), and `@plan:` (payload only, unchanged) responses, including `filter_text`, `sort_text`, and
  `text_edit` ranges.

## Completion panel row budget

Fix D7 in `src/sase/ace/tui/widgets/file_completion.py`,
`src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`, and
`src/sase/ace/tui/widgets/_file_completion_base.py`.

- Derive the row budget from the panel's real content capacity instead of the current `MAX_VISIBLE = 10`: content
  capacity is `_COMPLETION_PANEL_MAX_HEIGHT - _PANEL_BORDER_ROWS` (8). Reserve one line for the `↓ N more…` indicator
  whenever rows overflow, and one more line when the caller says a group rule will be drawn.
- Keep `MAX_VISIBLE` exported for compatibility, but make `_update_file_completion_panel`'s window and the panel's
  slicing agree on one shared helper, so the selected row is always inside the rendered window.
- Tests: extend `tests/ace/tui/widgets/test_prompt_completion_height.py` with a case that selects the last of 15
  candidates and asserts the highlighted row is among the rendered lines and that `panel.region.height` still respects
  the CSS cap.
- Expect PNG goldens for menus that currently render more than eight rows to change. Run `just test-visual`, inspect
  `.pytest_cache/sase-visual/` diffs, confirm each change is only the clipping fix, then accept with
  `--sase-update-visual-snapshots`.

## Warm local path inventory for the prompt

New module `src/sase/ace/tui/widgets/prompt_path_inventory.py`, plus a mixin hook next to the existing warmers in
`src/sase/ace/tui/widgets/_file_completion_base.py`.

- `PromptPathSnapshot(directory_key: str, rows: tuple[PromptPathRow, ...], token: tuple[int, int] | None)` where
  `PromptPathRow` is `(name, is_dir)` and `token` is `(st_mtime_ns, st_ino)` of the directory.
- `load_prompt_path_snapshot(abs_dir: str) -> PromptPathSnapshot` — worker-only. Stats the directory, `os.scandir`s it,
  caps at `MAX_PROMPT_PATH_ROWS = 1000` entries, resolves `is_dir` with `follow_symlinks=True`, sorts directories first
  then case-insensitively, and returns an empty snapshot on `OSError`. Mirror the mtime-token shape already used by
  `_read_cached_artifact_index` in `artifact_ref_completion.py`.
- Widget state: `_prompt_path_snapshots: dict[str, PromptPathSnapshot]`, `_prompt_path_inflight: set[str]`, worker group
  `"prompt-path-inventory"`. Reuse the `on_worker_state_changed` pattern from `_file_completion_base.py:272-295`: on
  success store the snapshot, discard the in-flight key, and refresh the menu only when it is still open on the same
  directory key.
- Keystroke path is a pure dict read. A miss schedules the loader and returns `None`.
- Revalidation happens on menu _open_ (and on directory drill-down), not per keystroke: schedule a coalesced worker that
  stats the directory and rescans only when the token changed. Never stat from the key handler.
- Warm-on-focus: warm the resolved base directory alongside `_warm_vcs_project_completion_catalog` and
  `_warm_model_completion_catalog`, gated the same way on the real app's `get_prompt_completion_settings`, so the first
  `@` in a session is already populated.
- Directory keys come from `resolve_prompt_completion_base_dir(self.text)` (falling back to the process CWD) joined with
  `path_query.directory`, with `~` and environment variables expanded exactly as `_lookup_path` does today, so the menu
  and the existing `Ctrl+T` file menu agree on what "local" means for a prompt that targets another project.
- Tests in `tests/ace/tui/widgets/`: snapshot loading (ordering, dotfiles retained in the snapshot and filtered later,
  cap, unreadable directory), token-based revalidation skipping an unchanged directory, and a keystroke-path guard test
  that monkeypatches `os.scandir` and `os.stat` to raise and asserts a `@` keypress still succeeds.

## TUI reference menu behavior

Rework `src/sase/ace/tui/widgets/artifact_ref_completion.py` into a mapping layer over the binding, in the spirit of
`placeholder_completion.py`'s "pure mapping from Rust payloads to prompt completion rows".

- Replace `detect_artifact_ref_completion_context` and `build_artifact_ref_completion_result`'s decision logic with
  calls to `at_reference_context` / `at_reference_menu` through a thin facade in `src/sase/artifact_refs.py` that uses
  `require_rust_binding`, converting between Python character offsets and LSP UTF-16 positions with the helpers already
  in `placeholder_completion.py` (extract them if they need sharing). Keep the existing catalog loaders
  (`_load_document_candidates`, `_load_artifact_file_candidates`, `_load_chat_candidates`, the commit/bug pane
  snapshots) as the inventory the facade is fed.
- Row metadata: keep `ArtifactRefKindCompletionMetadata` and `ArtifactRefPayloadCompletionMetadata`; add
  `AtReferenceFileCompletionMetadata(is_dir: bool, directory: str)` so the renderer and accept path can tell groups
  apart without string sniffing.
- `_file_completion_open.py:230-265`: delete the bare-`@` refusal and the two-character gate.
  `_try_artifact_ref_completion` now opens whenever a context exists and any row is available; when the artifacts group
  is non-empty but the path snapshot is cold, show artifact rows immediately and let the worker append file rows on
  refresh; when both are empty and the snapshot is cold, show a single dim `loading files…` row modeled on
  `build_loading_placeholder`.
- Precedence: keep the artifact branch where it is in `_try_auto_prompt_reference_completion`, after the placeholder,
  VCS, and directive branches, so `%model:@alias` and `#gh:owner/` keep winning. In
  `_structured_completion_claims_cursor` (`_file_completion_refresh.py:470-474`), treat _any_ `@` context as claiming
  the cursor — the current `partial_kind`-non-empty condition would let the prompt-word fallback fight a bare `@`.
- Accept (`_accept_artifact_ref_completion` in `_file_completion_accept.py`):
  - Artifact row → insert `@kind:` and immediately re-open the payload stage (unchanged).
  - File row, directory → insert `@dir/` and re-open the menu for the new directory, reusing the drill-down shape at
    `_file_completion_accept.py:376-381`; schedule the new directory's snapshot load if it is cold.
  - File row, file → insert `@path` and close the menu.
- `Enter` ownership (D5): add `_completion_selection_moved: bool` to the shared completion state, set it in
  `_move_file_completion`, and clear it in `_clear_file_completion` and on every menu open. In
  `_prompt_text_area_key_handling.py`, `Enter` falls through to submit when the active menu is the `@` menu with an
  empty query and `_completion_selection_moved` is false. `Ctrl+L` is unaffected.
- Manual `Ctrl+T` (`force=True`) keeps its exhaustive behavior — lone-match insertion and shared-prefix extension — but
  never performs synchronous filesystem work; a cold directory yields the loading row and a refresh.
- Tests in `tests/ace/tui/widgets/`: extend `test_artifact_ref_completion.py` for the new context/menu mapping and add
  integration coverage next to `test_prompt_at_prefix_completion.py` for: bare `@` opens with artifact rows first and
  file rows second; `@pl` narrows both groups; `@src/` shows files only; directory accept drills down; file accept
  inserts `@path`; kind accept re-opens the payload stage; `Enter` on an un-narrowed menu submits while `Ctrl+L`
  accepts; `Enter` accepts after `Ctrl+N`; a cold snapshot shows the loading row and the worker result refreshes the
  open menu. Add a Python parity table mirroring the Rust context/menu vectors, following the golden-vector convention
  in `tests/test_xprompt_vcs_project_completion.py`.

## Grouped menu rendering

Make the merged menu legible at a glance in `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows.py` and
`src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`.

- Artifact rows keep their current styling but widen the sigil cell to three columns (`"@  "`) so they align with the
  three-column file glyph cells, then the kind name in green (bold when selected), then a dim aligned detail column
  (`builtin`, `document · <root>`).
- File rows reuse the exact anatomy of the existing `Ctrl+T` file menu — `📁 ` with a `cyan` label for directories and
  `📄 ` for files — so the group reads as the file completion the user already knows. No extra columns; nothing is
  stat'd for display.
- One dim group rule is drawn between the groups when both are non-empty at the Kind stage: `── files · <base-dir>`
  padded with `─` to the panel's inner width (`panel.size.width - 2`), where `<base-dir>` is the resolved base directory
  shortened with `~`. It carries the base-directory context the file rows omit. It is not a candidate (D6), and the
  panel asks the row-budget helper from `panel_rows` to reserve its line.
- Adaptive border title at the Kind stage: `@ reference` when both groups have rows, `@ artifact kinds` when only
  artifact rows, and `@ <directory>` when only file rows — matching how the plain file menu titles itself with its
  directory. Payload-stage titles (`file: artifacts`, `chat: chats`, `commit: commits`, `bug: bugs`,
  `<kind>: documents`) are unchanged.
- Tests: unit tests over the row builders asserting group ordering, alignment widths, the rule's presence only when both
  groups are populated, and each adaptive title. Add a PNG golden
  `tests/ace/tui/visual/test_ace_png_snapshots_at_reference_completion.py` with a fixture workspace containing a couple
  of directories and files, capturing the Kind stage with both groups; follow the helper conventions in
  `_ace_prompt_png_snapshot_helpers.py` and register the golden under `tests/ace/tui/visual/snapshots/png/`.

## Documentation and help sync

- `docs/ace.md`: rewrite the artifact sentence in the automatic-completion paragraph (currently "Artifact-reference
  menus open after two kind characters … A bare `@` stays quiet.") to describe the merged menu, its two groups, group
  ordering, the `@`-prefixed insertion form, directory drill-down, the dotfile rule, and the `Enter`-ownership rule. Add
  the menu to the `Ctrl+T` completion-kind list. Confirm the "panel shows up to 10 candidates" sentence matches the new
  budget and correct it if not.
- `docs/configuration.md`: update the `auto_artifact_menu` row — it now controls automatic opening from a bare `@` and
  covers both groups — and update the following prose paragraph that states a bare `@` keeps ordinary file behavior.
- `docs/editor.md`: update the "Artifact references" feature row and the "File completion" row to state that `@`
  completes artifact kinds and local paths in one grouped response, and note the sigil-inclusive filter text.
- `src/sase/ace/tui/modals/help_modal/binding_common.py:30`: replace `("@kind:payload", "Complete artifact references")`
  with an entry covering both groups, e.g. `("@", "Artifact kinds + local files")`, respecting the 32-character
  description limit and 57-character box width from `src/sase/ace/CLAUDE.md`. Update
  `tests/test_keymaps_display_help.py:272-278` to match.
- Re-run `just check` and `just test-visual` after the doc and help edits.

## Risks and mitigations

- **`@` in prose opens a menu.** Mitigated by D5 (`Enter` is not hijacked), by dismissal on any token-terminating
  character, and by the existing `ace.prompt_completion.auto_artifact_menu: false` escape hatch, which the docs phase
  calls out.
- **Cold-cache flicker.** The first `@` in a session could show artifact rows and then grow file rows. Mitigated by
  warm-on-focus and by preserving the selected insertion across refreshes with
  `_replace_completion_candidates_preserving_selection`.
- **PNG golden churn from the row-budget fix.** Bounded by inspecting every diff before accepting, and by landing
  `panel_rows` as its own phase so the churn is reviewable separately from the new menu's golden.
- **Cross-language drift.** Bounded by D2 (one policy implementation) plus the Python parity table over the same
  vectors.
