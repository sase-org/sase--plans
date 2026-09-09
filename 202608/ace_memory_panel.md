---
status: done
tier: epic
title: ACE Memory panel for browsing and editing SASE memory notes
goal: "A prompt-launched Memory panel lets a user browse, add, modify, and delete SASE
  memory notes for any memory-bearing scope, defaulting to the current project, with
  parent/child link travel and a publish step that keeps AGENTS.md and provider shims in
  sync.

  "
phases:
  - id: memory-catalog
    title: Memory scope ring and snapshot service
    depends_on: []
    size: medium
    description: "memory-catalog: build the scope ring, cached per-scope note snapshots,
      the note tree, and the generated-note contract the panel reads.

      "
  - id: memory-writes
    title: Shared memory-note mutation engine
    depends_on: []
    size: medium
    description: "memory-writes: add a CLI-free create/update/delete engine for memory
      notes with pure validation, atomic writes, a stale-write guard, and delete
      backups.

      "
  - id: panel-keymaps
    title: ace.keymaps.memory binding scope
    depends_on: []
    size: small
    description: "panel-keymaps: register the panel-scoped keymap dataclass, defaults,
      config schema, binding builders, and app help sections.

      "
  - id: panel-shell
    title: Memory panel shell, note tree, filter, and scope switching
    depends_on:
      - memory-catalog
      - panel-keymaps
    size: medium
    description: "panel-shell: build the modal, the nested note rail, the note card,
      filtering, scope cycling and picking, empty/error states, and passive open/copy
      actions.

      "
  - id: panel-links
    title: Parent and child link travel
    depends_on:
      - panel-shell
    size: small
    description: "panel-links: add the numbered PARENT/CHILDREN chips, chip cursor,
      numbered follow, and the bounded breadcrumb trail with back travel.

      "
  - id: panel-mutations
    title: Add, edit, delete, and publish surfaces
    depends_on:
      - panel-shell
      - memory-writes
    size: medium
    description: "panel-mutations: wire the add/edit forms, the delete confirmation, the
      tracked-proc writes, and the sase memory init publish flow with its unpublished
      state.

      "
  - id: prompt-entry
    title: Prompt gm and Ctrl+G m entry point
    depends_on:
      - panel-shell
    size: small
    description: "prompt-entry: claim the prompt g-prefix m continuation, post the
      request message, open the panel seeded from the memory reference under the cursor,
      and restore focus.

      "
  - id: memory-panel-verification
    title: Documentation, visual snapshots, and full verification
    depends_on:
      - panel-links
      - panel-mutations
      - prompt-entry
    size: small
    description:
      "memory-panel-verification: document the panel and its keymap scope, add PNG
      snapshot goldens, and run the exhaustive verification lanes."
proposed_by: bbugyi200.athena.07j
bead_id: sase-qt
create_time: 2026-09-09 19:49:37
---

- **PROMPT:**
  [prompts/202608/ace_memory_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ace_memory_panel.md)
- **BEAD:**
  [sase-qt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-qt/README.md)

# Plan: ACE Memory panel for browsing and editing SASE memory notes

## Context and verified current behavior

SASE memory notes are flat Markdown files under a content root's `sase/memory/`
(canonical) or `memory/` (legacy) directory. `sase.memory.notes.discover_memory_notes()`
parses each non-`README.md` `*.md` file into a `MemoryNote` carrying `type` (`short` |
`long`), `parent` (default `AGENTS.md`), `description`, body, and the frontmatter
mapping, with `type_source` / `parent_source` recording whether each field was present,
missing, or invalid. `sase.memory.paths` resolves canonical/legacy roots through the
shared `sase.content_layout` contract, and
`sase.content_layout.resolve_memory_file_sources()` is the Rust-owned, ordered
project-then-home source contract the xprompt memory loader already consumes.

`parent` is a real edge, not decoration. `type: short` notes are inlined into
`AGENTS.md` Tier 1; `type: long` notes with `parent: AGENTS.md` render as Tier 2
sections; a `long` note whose parent is another `long` note becomes that note's child
and is surfaced only through the parent's generated `## Children` section (today
`sase/memory/sase_sizes.md` parents to `sase/memory/sase_beads.md`).
`sase.memory.inventory_reachability` already owns the legality rules that
`sase memory init` enforces: `memory_parent_blockers_for_init()` rejects self-parents,
missing parents, short-note parents, non-long parents, and parent cycles, and
`unreferenced_memory_files_for_init()` rejects notes nothing reaches.

Today there is no interactive surface for any of this. `sase memory list` prints a
read-only Rich dashboard, `sase memory show` / `read` print one note (and `read` appends
an audit row through `sase.memory.read_log`), and `sase memory write` / `review` handle
_agent-proposed_ memory only. Creating, editing, or deleting a note is a hand edit
followed by `sase memory init`.

The **Glossary panel** is the established precedent for this shape and the model this
plan follows: `GlossaryPanel` is a `ModalScreen` split into a shell plus state, view,
navigation, travel, and actions mixins; `sase/ace/tui/glossary_panel_catalog.py` owns a
project ring and an mtime-keyed snapshot cache loaded only on worker threads;
`glossary_panel_load.py` seeds the starting project from the launch workspace, then the
current project (honoring `ace.current_project.seed_filters`, the same setting the
Artifacts sub-tabs use in `_resolve_artifacts_scope_seed()`), then the first ring entry;
writes go through a shared engine (`sase/glossary/mutation.py`) on the app's session
worker queue; and the panel is opened from the prompt bar's `g` prefix table via a
presentation-only message the app handles.

Two facts constrain the design and are easy to get wrong:

1. **Home memory is chezmoi-managed here.** `use_chezmoi: true` is set, so the canonical
   home memory source is `~/.local/share/chezmoi/home/sase/memory/` and `~/sase/memory/`
   is only the applied copy. `sase memory init` already resolves this through
   `_home_root_path(use_chezmoi)` / `CHEZMOI_HOME`. Writing to the applied copy would be
   silently clobbered by the next `chezmoi apply`.
2. **Four notes are generated.** `sase memory init` regenerates `sase/memory/sase.md`
   and `sase/memory/task_types.md` in every root, plus project-only
   `sase/memory/sase_beads.md` and `sase/memory/sase_sizes.md`, from templates in
   `src/sase/main/init_memory/templates/`. Hand edits to these are overwritten. Their
   relative paths are currently only reachable through private helpers in
   `src/sase/main/init_memory/root_rendering.py`.

## Design decisions

1. **Scope, not just project.** The panel browses one _memory scope_ at a time. A scope
   is one content root with a memory directory: every enabled project that has one, the
   project the panel was opened from even when it has none (so `a` can bootstrap it),
   and one `Home` scope. Scope order is by display name with `Home` last. The starting
   scope follows the Glossary/Artifacts precedence: the prompt's launch-workspace
   project, then the current project when `ace.current_project.seed_filters` is on and
   it is in the ring, then the first ring entry. This is what "respects the current
   project" means here, and it is the same seed helper shape both existing surfaces use.
2. **The Home scope resolves through the active config mode.** Its content root is
   `CHEZMOI_HOME` when `get_use_chezmoi()` is true, otherwise `Path.home()`, and its
   display name says so (`Home (chezmoi)`). Reads and writes both use that root. Never
   browse one root and write another.
3. **The rail is a tree, not a flat list.** Tier 1 (`short`) notes sort first, then Tier
   2 (`long`) root notes alphabetically, each immediately followed by its children
   indented one level. That makes the `parent` edge legible before the user follows any
   link, and it mirrors what `AGENTS.md` actually renders. Selection identity is the
   note's root-relative path, so it survives filtering, reloads, and writes.
4. **Link travel mirrors the Glossary relation chips exactly.** The note card carries a
   numbered `PARENT` chip (only when the parent is a memory note, not `AGENTS.md`) and
   numbered `CHILDREN` chips, numbered continuously so `1`–`9` is unambiguous. `Tab` /
   `Shift+Tab` move a chip cursor, `l` follows the focused chip (or chip ① when none is
   focused), `1`–`9` jump directly, `h` / `Backspace` walk a trail bounded at 32
   entries, and a non-empty trail renders as `TRAIL  a › b › c`. Reuse the Glossary
   panel's proven semantics rather than inventing a second travel model.
5. **Generated notes are read-only in the panel.** They render with a `GENERATED` badge
   and a one-line explanation, and the mutation engine refuses to write or delete them.
   The four paths come from a new _public_ helper in `root_rendering.py` so the panel
   never hardcodes filenames.
6. **Body edits go to `$EDITOR`; frontmatter edits stay in the panel.** A memory note
   body is prose; a TextArea in a modal is a worse editor than the user's own. `o`
   suspends ACE and opens `$EDITOR` through the existing `SourceFileActionsMixin`. `e`
   opens a small form for the three fields the graph actually depends on — `type`,
   `parent`, `description` — with live validation. `a` combines both: collect the
   fields, write a stub, then offer to open `$EDITOR` on the new note.
7. **Deletes never leave the tree broken.** Deleting a note that has children would
   create `invalid memory parent … (parent target does not exist)` blockers on the next
   `sase memory init`. The panel refuses that delete and says which children must be
   reparented first. Every delete writes a timestamped backup copy under the scope's
   SASE state directory and the success toast names it; deleting a `short` note
   additionally warns that always-loaded agent context is being removed.
8. **Writes are guarded against stale state.** Each snapshot records every note's
   `(mtime_ns, size, sha256)`. Update and delete pass the expected digest; a mismatch
   raises `MemoryConflictError`, which the panel surfaces as a toast plus an automatic
   scope reload — the same conflict path the Glossary panel already uses.
9. **Publishing is explicit and honest.** A memory write is not visible to agents until
   `sase memory init` regenerates `AGENTS.md`, the provider shims, and the memory
   README. The panel tracks which scopes it has written since the last publish, shows an
   `UNPUBLISHED` badge in the header, offers a publish confirmation right after a
   successful write, and binds `I` for publishing on demand. Publish runs the real
   command as a captured, non-interactive subprocess on the session worker queue — never
   in process, never on the event loop.
10. **Publish must pass a commit decision.** `sase memory init` prompts for a fold
    commit subject on a TTY and hard-fails without one on a non-TTY when memory or
    source files are dirty. The publish modal therefore always chooses explicitly:
    **Publish & commit** runs `sase memory init --message "<subject>"` with a prefilled,
    editable subject, and **Publish only** runs `sase memory init --no-commit`.
11. **Panel keys are panel-scoped.** A new `ace.keymaps.memory` scope, deliberately kept
    out of `AppKeymaps`, so `j` / `k` / `p` / `a` / `d` never become globally active —
    the same rule the Glossary, gate, statistics, and projects scopes follow.
12. **The panel is a user surface, not an agent surface.** It is the human editing their
    own memory in their own TUI. It does not touch the agent-facing `sase memory write`
    / `review` proposal path, and it never edits `AGENTS.md` or the provider shims
    directly — only `sase memory init` writes those.

### Non-goals

- No new `sase memory` CLI subcommands. The mutation engine is a Python module the panel
  calls; exposing `sase memory new` / `rm` is separate work.
- No note rename or move (project ↔ home). Renaming changes a note's `#memory/<stem>`
  reference and every child's `parent`, and deserves its own design.
- No changes to `sase memory write` / `review`, the audited-read ledger's write path, or
  the `#memory/<stem>` xprompt loader.
- No editing of `AGENTS.md`, provider shims, or `sase/memory/README.md` from the panel.

## Phase 1: Memory scope ring and snapshot service

Add `src/sase/ace/tui/memory_panel_catalog.py`, modeled directly on
`glossary_panel_catalog.py`. Every function here does disk work and must run only on a
worker thread; state this in the module docstring the way the glossary module does, and
follow `sase/memory/tui_perf.md` (read it with `/sase_memory_read` first).

- Define `MemoryScopeRef` (scope kind `project` | `home`, key, display name, content
  root, resolved memory read root, `has_memory`) and `MemoryScopeSnapshot` (scope,
  ordered notes, the rail tree, per-note digests and stats, shadowed-stem set,
  generated-path set, audited-read summaries, diagnostics).
- `build_memory_scope_ring(launch_workspace)` enumerates enabled project records the way
  `build_glossary_project_ring()` does, keeps those whose content root resolves a memory
  read root, always keeps the launch project, and appends the Home scope with the
  chezmoi-aware root from decision 2. Display names must come from
  `effective_project_name()` / `project_display_names`, never a ProjectSpec key.
- `load_memory_scope_snapshot(ref)` resolves the read root through
  `resolve_memory_file_sources()` plus `memory_read_root()`, calls
  `discover_memory_notes()`, computes per-note stats with
  `sase.memory.inventory_references.stats_for_text`, marks generated notes, marks
  project notes that shadow a same-stem home note, and folds in that scope's
  `summarize_memory_reads_by_path()` results so the card can show the last audited read.
  A `LayoutCollisionError` or `OSError` becomes a diagnostics tuple, never an exception
  that sinks the panel — one broken scope must not shrink the ring for the others.
- Cache snapshots exactly like the glossary service: an `OrderedDict` keyed by scope,
  bounded at 8 entries, re-stat gated at 0.5 s, invalidated by directory
  `(mtime_ns, size)` plus each note's stat. Expose `invalidate_memory_scope(key)`.
- `memory_note_relations(snapshot, note)` returns the ordered `(parent, children)` notes
  the card and chips render, with `AGENTS.md` parents returning no parent chip.
- Add `src/sase/ace/tui/modals/memory_panel_load.py` with
  `load_memory_panel_initial_state(launch_workspace, initial_scope_key, seed_from_current_project)`
  implementing the precedence in decision 1.
- Promote the generated-note paths: add a public
  `generated_memory_note_relative_paths(*, include_project_memory: bool)` to
  `src/sase/main/init_memory/root_rendering.py` returning the canonical relative paths
  currently produced by the private `_generated_*_memory_relative_path()` helpers, and
  have those helpers feed it so the two can never drift.
- Add a pure `filter_memory_notes(notes, *, pattern, include_bodies)` next to the notes
  parser (mirroring `sase/glossary/text_filter.py`) matching stem and description by
  default and extending into the body when asked.
- Tests: ring composition and ordering (enabled + launch-only + Home, de-duplication,
  display names not keys), chezmoi and non-chezmoi Home roots, snapshot caching and
  invalidation, tree construction including a child note, shadowed-stem marking,
  generated marking, collision and unreadable-root diagnostics, the seed precedence
  table, and the filter predicate.

## Phase 2: Shared memory-note mutation engine

Add `src/sase/memory/mutation.py`, modeled on `src/sase/glossary/mutation.py`, with no
Textual import. This is the only code that writes memory notes.

- Outcome and error types: `MemoryMutationOutcome` (scope key, content root, relative
  path, stem, type, parent, description, backup path for deletes) and
  `MemoryMutationError` with `MemoryValidationError`, `MemoryConflictError`, and
  `MemoryGeneratedNoteError` subclasses.
- `validate_memory_note_draft(...)` is a pure function shared with the panel's forms:
  stem grammar (single flat segment, no traversal, no `README`, `.md` implied),
  collision with an existing note in the scope, `type` in `{short, long}`, `short` notes
  must parent to `AGENTS.md`, `long` notes require a non-empty description, and parent
  legality evaluated with the same rules `memory_parent_blockers_for_init()` enforces —
  no self-parent, parent must exist in the scope and be a `long` note, no cycles. Return
  diagnostics grouped by field so the form can render them inline.
- `create_memory_note(...)` renders frontmatter through
  `sase.memory.notes.apply_memory_frontmatter()` (so `description` wrapping stays
  Prettier-stable), writes to the scope's `memory_write_root()`, refuses to overwrite an
  existing file, and writes atomically with a temp file, `fsync`, replace, and a
  directory `fsync`, reusing the glossary engine's write shape.
- `update_memory_note(...)` rewrites only frontmatter and preserves the body
  byte-for-byte, requiring an `expected_digest` and raising `MemoryConflictError` on
  mismatch.
- `delete_memory_note(...)` requires an `expected_digest`, refuses when the note has
  children (naming them), copies the note to a timestamped backup under the scope's SASE
  state directory, then unlinks.
- All three refuse generated notes via the Phase 1 helper, and refuse any path that is
  not a flat note inside the resolved memory root.
- Tests under `tests/memory/`: every validation branch, canonical frontmatter output for
  short/long/child notes, body preservation across an update, digest conflict, generated
  refusal, child-blocked delete, backup creation, traversal refusal, and atomic-write
  behavior. Include a round-trip test proving a created note passes
  `memory_parent_blockers_for_init()` and `unreferenced_memory_files_for_init()`.

## Phase 3: ace.keymaps.memory binding scope

Register the panel's scope everywhere the `glossary` scope is registered, following that
scope's wiring exactly.

- `MemoryPanelKeymaps` in `keymaps/app_keymaps.py` with these defaults: `next_note: j`,
  `prev_note: k`, `first_note: g`, `last_note: G`, `scroll_body_down: ctrl+d`,
  `scroll_body_up: ctrl+u`, `filter_notes: slash`, `toggle_body_filter: full_stop`,
  `next_link: tab`, `prev_link: shift+tab`, `follow_link: enter,l`,
  `travel_back: backspace,h`, `next_scope: p`, `prev_scope: P`, `pick_scope: ctrl+p`,
  `add_note: a`, `edit_note: e`, `delete_note: d`, `publish: I`, `open_source: o`,
  `open_viewer: Z`, `copy_body: y`, `copy_source_path: Y`, `refresh: r`,
  `help: question_mark`.
- `_MEMORY_BINDING_META` in `keymaps/metadata.py`, `build_memory_bindings()` and
  `memory_help_bindings()` in `keymaps/bindings.py`, `load_builtin_memory_defaults()` in
  `keymaps/defaults.py`, `load_memory_keymaps()` in `keymaps/scopes.py`, the
  `KeymapRegistry.memory` field in `keymaps/types.py`, and the wiring in `registry.py`,
  `loader.py`, and `keymaps/__init__.py`.
- The `ace.keymaps.memory` block in `src/sase/default_config.yml` (the defaults loader
  reads its values from there, so this is required, not documentation) and the matching
  schema section in `src/sase/config/sase.schema.json`.
- A `memory_panel_section()` in `modals/help_modal/binding_common.py`, exported from
  `help_modal/bindings.py` and appended in `agents_bindings.py`, `axe_bindings.py`, and
  `patches_bindings.py` next to the existing glossary section.
- Tests: extend `tests/test_keymaps_defaults.py`,
  `tests/test_keymaps_registry_loading.py`, and `tests/test_keymaps_validation.py` for
  the new scope — defaults present for every dataclass field, unknown-action warning,
  invalid-key revert, duplicate-key revert.

## Phase 4: Memory panel shell, note tree, filter, and scope switching

Add `MemoryPanel` under `src/sase/ace/tui/modals/`, split the way the Glossary panel is
split so no module grows unbounded: `memory_panel.py` (widget tree, worker-backed loads,
passive actions), `memory_panel_state.py` (snapshot application, filtering, selection
identity), `memory_panel_view.py` (header, footer, card, trail widget updates),
`memory_panel_navigation.py` (cursor, body scrolling, filter box, scope cycling and
picking), `memory_panel_rendering.py` (pure renderables), `memory_panel_help_modal.py`,
and `memory_panel_scope_picker.py`.

- Widget tree: header `Static`, a `Horizontal` body holding the note rail `OptionList`
  and a `VerticalScroll` detail column (title `Static`, description `Static`, body
  `Markdown`, meta `Static`), the inline filter `Input`, the trail `Static`, and the
  footer `Static`. Mirror the Glossary panel's TCSS block in `styles.tcss`, including a
  runtime-fitted rail width (`note_rail_width()`) with `min-width` / `max-width`
  backstops kept in sync with the Python constants.
- Header: `MEMORY  ·  <scope display name>  ·  N notes  ·  scope i/N`, plus a
  right-aligned `⚠ UNPUBLISHED` badge when this scope has unpublished panel writes.
- Rail rows: `●` for Tier 1, `○` for Tier 2, `└ ` indent for a child, `⚙` for generated,
  `⚠` for a note with an invalid `type` or `parent`, then the stem and a dim description
  snippet. Ordering per decision 3.
- Card: title (stem plus root-relative path), badge row (`TIER 1 · always loaded` /
  `TIER 2`, `GENERATED`, `SHADOWS HOME`, `ORPHANED`, `INVALID`), the description, the
  rendered Markdown body, then a property grid with type, parent, child count, size
  (lines and approximate tokens), last modified, last audited read (agent, reason,
  when), and the source path.
- Filter: `/` opens the inline box over stem and description; `.` extends the match into
  bodies; `Esc` closes the box and keeps the selection when it is still visible; an
  empty result reads `no notes matched: <pattern>`.
- Scope switching: `p` / `P` cycle the ring, clearing the filter and trail and restoring
  that scope's last-selected note for the life of the panel; `ctrl+p` opens
  `MemoryScopePicker`, a small filterable `OptionList` modal built from the panel's own
  ring showing each scope's display name and note count — thirty-odd registered projects
  make cycle-only navigation impractical.
- Empty and error states: a scope with a memory root but no notes shows a centered
  invitation naming the scope and pointing at `a`; a scope with no memory root says the
  directory will be created on first add; a scope whose read root collides or fails
  shows its diagnostics and the offending paths.
- Passive actions through `SourceFileActionsMixin` and `CopyModeForwardingMixin`: `o`
  editor, `Z` artifact viewer, `y` copy body, `Y` copy path, `r` refresh (invalidate and
  reload the scope), `?` help modal, `Esc` / `q` close. After `o` returns, re-stat the
  note and, when its digest changed, reload the scope and mark it unpublished.
- Footer lists only conditional keys, the way `build_panel_footer()` does: scope keys
  when the ring has more than one scope, link keys when chips exist, back when a trail
  exists, edit/delete when a writable note is selected, publish when the scope is
  unpublished.
- Export `MemoryPanel` from `modals/__init__.py` and `modals/__init__.pyi` using the
  existing lazy-import table.
- Tests under `tests/ace/tui/modals/`, with a `memory_panel_test_helpers.py` harness
  modeled on `glossary_panel_test_helpers.py` that installs a fixed load and asserts it
  ran off the main thread: mount and first selection, tree ordering with children, rail
  fitting, filter and definition-filter behavior, scope cycling and the picker,
  selection restoration across reloads, seed-filters plumbing, and each empty/error
  state.

## Phase 5: Parent and child link travel

Add `memory_panel_travel.py` implementing decision 4, reusing the Glossary panel's
travel semantics.

- `_refresh_links_for_current_note()` recomputes `(parent, children)` from
  `memory_note_relations()` whenever the selection changes and resets the chip cursor.
- Chip rows render as numbered `PARENT` and `CHILDREN` rows with continuous numbering,
  reusing the shared chip renderer the Glossary card uses so the two surfaces cannot
  drift visually.
- `next_link` / `prev_link` move the chip cursor, `follow_link` follows the focused chip
  or chip ①, `action_follow_link_number(1..9)` follows directly, and following pushes
  the previous note onto a trail bounded at 32 entries.
- Following a note hidden by the active filter clears the filter and notifies, exactly
  as `_land_on_term()` does; `travel_back` pops the trail, skipping entries no longer
  present.
- Document in the panel help and in Phase 8's docs that `follow_link` ships as `enter,l`
  but only `l` fires, because the focused `OptionList` consumes `Enter` — the same known
  behavior the Glossary panel documents.
- Tests: chip construction for a root note, a child note, and a note with both edges;
  continuous numbering; follow by cursor and by number; trail push, bound, and back;
  filter clearing on a hidden target; and no stale chips after a scope switch.

## Phase 6: Add, edit, delete, and publish surfaces

Add `memory_panel_actions.py`, `memory_panel_add.py` (the shared add/edit form), and
`memory_panel_publish.py`.

- `MemoryNoteFormModal` serves both `a` and `e`: a stem `Input` (disabled in edit mode),
  a tier selector (`short` / `long`), a parent selector listing `AGENTS.md` plus every
  `long` note in the scope, and a description `TextArea`. Validation runs on a short
  debounce through `validate_memory_note_draft()` and renders per-field errors; `ctrl+s`
  submits and is refused while any blocking diagnostic stands; `Esc` cancels. Mirror
  `GlossaryTermAddModal`'s structure, including suppressing "required" errors until
  first submit.
- After a successful create, offer to open the new note in `$EDITOR` so the user can
  write the body immediately.
- `d` pushes a `ConfirmActionModal` with `ConfirmKind.DANGER` whose subject shows the
  note path, tier, description, child count, and — for a `short` note — an explicit
  warning that always-loaded agent context is being removed. When the note has children
  the panel instead shows a non-destructive explanation naming them and does not offer a
  delete.
- All three writes go through `self.app._submit_session_worker(...)` with `dedup_key` /
  `exclusive_scopes` of `memory-write:<scope key>`, calling the Phase 2 engine off the
  event loop and returning a `TrackedProcResult` carrying the reloaded snapshot, exactly
  like `_submit_glossary_write()`. On completion: toast, reselect the written note (or
  the neighbor after a delete), clear the filter when it hides the target, invalidate
  the scope, mark it unpublished, and refresh prompt catalogs that read memory xprompts.
  A `MemoryConflictError` toasts and forces a reload.
- Register the new producer in `src/sase/ace/tui/proc_producer_sites.py` as
  `memory.write`, following the `glossary.write` entry's fields (`session_worker`,
  concurrency keys, optimistic-UI note, restart-recovery note).
- Publish: after any successful write, push `MemoryPublishModal` — a small form with a
  prefilled, editable commit subject and two submit branches, **Publish & commit** and
  **Publish only**, plus cancel. Both submit a session worker that runs
  `sase.noninteractive_subprocess.run_noninteractive()` on
  `sase memory init --message "<subject>"` or `sase memory init --no-commit`, with `cwd`
  set to the scope's content root (`Path.home()` for the Home scope). Surface the
  captured stderr tail on failure and clear the scope's unpublished mark on success. `I`
  opens the same modal on demand.
- Tests: form validation for each branch, create/edit/delete happy paths and
  reselection, conflict handling, generated-note refusal surfacing, child-blocked
  delete, the publish argv for both branches and both scope kinds, publish failure
  surfacing, unpublished-state transitions, and a producer-site registry assertion.

## Phase 7: Prompt gm and Ctrl+G m entry point

Follow the Glossary panel's entry commit exactly.

- Add an `m` row to `_PROMPT_G_PREFIX_BINDINGS` in
  `widgets/_prompt_input_bar_g_prefix_actions.py` bound to `request_open_memory_panel`,
  with a `_g_prefix_label_memory` returning `memory…` and a `_g_prefix_available_memory`
  that is true in prompt mode. Because it is not `ctrl_g_only`, this claims both `gm` in
  NORMAL and `Ctrl+G m` in INSERT or NORMAL, and `gm` is unclaimed by the text area's
  vim `g` handling.
- Add `PromptInputBar.MemoryPanelRequested(note_reference, mode)` in
  `_prompt_input_bar_messages.py` and re-export it from `prompt_input_bar.py`. The bar
  captures the `#memory/<stem>` xprompt reference under the cursor when there is one —
  reusing the existing jump-target detection the way `_glossary_term_under_cursor()`
  reuses the glossary match — and otherwise sends `None`.
- Add `actions/agent_workflow/_prompt_bar_memory_panel.py` mirroring
  `_prompt_bar_glossary_panel.py`: capture the focused pane, vim mode, and cursor; open
  `MemoryPanel` with the launch workspace and the seed note; restore focus, vim mode,
  and cursor on dismiss. Mix it into the prompt-bar action set in `_prompt_bar.py`.
- Update the `g`-prefix hint fixtures and tests (`prompt_g_prefix_hint_test_support.py`,
  the hint entry / lifecycle / routing tests) and add a
  `test_prompt_memory_panel_entry.py` plus an `actions/test_prompt_memory_panel_open.py`
  covering seeding, launch-workspace plumbing, and focus restoration.

## Phase 8: Documentation, visual snapshots, and full verification

- `docs/ace.md`: add a `#### Memory panel` section next to the Glossary panel section
  covering the entry keys, the header, the tree rail glyphs, both navigation axes, scope
  cycling and the picker with its seed precedence, add/edit/delete, the publish flow and
  the `UNPUBLISHED` badge, generated-note read-only behavior, the empty and error
  states, and the fixed (non-remappable) keys. Add the `gm` and `Ctrl+G m` rows to the
  two prompt keybinding tables.
- `docs/configuration.md`: add the `ace.keymaps.memory` table alongside the existing
  glossary scope table, and a "Remapping Memory Panel Keys" subsection.
- `docs/memory.md`: add a short pointer from the memory-management overview to the panel
  as the interactive surface, and state plainly that the panel does not replace the
  `sase memory write` / `review` proposal path for agents.
- Add PNG snapshot goldens under `tests/ace/tui/visual/snapshots/png/` for populated and
  empty states in light and dark themes, following
  `test_ace_png_snapshots_glossary_panel.py`. Inspect `.pytest_cache/sase-visual/`
  artifacts before accepting anything with `--sase-update-visual-snapshots`.
- Read `sase/memory/symvision.md` with `/sase_memory_read` before the final lint pass
  and resolve any unused-symbol or private-misuse findings the new modules introduce.
- No canonical `sase/memory/*.md` note needs to change for this feature; do not edit one
  without a fresh, explicit user request. If the user does request it, run
  `sase memory init` afterward as the rule requires.
- Run `just install`, then `just check-full` through `/sase_monitor` — this change
  touches the keymap registry, config schema, TCSS, the modal export table, and the
  producer-site registry, all in the broadening set — plus `just test-visual`. Finish
  with `sase doctor`, `sase validate`, and `sase memory init --check`.

## Acceptance criteria

- `gm` in NORMAL and `Ctrl+G m` in INSERT or NORMAL open the Memory panel from a prompt
  pane, seeded from the `#memory/<stem>` reference under the cursor when there is one,
  and `Esc` / `q` restore the originating pane, vim mode, and cursor.
- The panel opens on the current project's scope when `seed_filters` is on and that
  scope is in the ring, and `p` / `P` / `ctrl+p` reach every memory-bearing enabled
  project plus the chezmoi-correct Home scope, always labeled by display name and never
  by ProjectSpec key.
- The rail renders Tier 1 and Tier 2 notes with children nested under their parents; the
  card shows the note body plus its badges, properties, and last audited read; `/` and
  `.` filter as specified.
- Numbered `PARENT` and `CHILDREN` chips follow with `l` or `1`–`9`, the trail records
  travel and `h` / `Backspace` walks it back, and following a filtered-out note clears
  the filter.
- `a`, `e`, and `d` create, retype/reparent/redescribe, and delete notes through the
  shared engine; validation blocks illegal stems, tiers, parents, and cycles before any
  write; generated notes and notes with children are refused with an explanation;
  deletes leave a named backup; and a concurrent external edit produces a conflict toast
  and a reload rather than a lost update.
- Every write marks its scope unpublished, and publishing runs the real
  `sase memory init` non-interactively in the right working directory with an explicit
  commit decision, clearing the badge on success and surfacing captured errors on
  failure.
- All panel disk work runs on worker threads or the session-worker queue; no snapshot
  load, write, or publish runs on the Textual event loop.
- Documentation covers the panel and its keymap scope, PNG goldens exist for both
  themes, and `just check-full`, `just test-visual`, `sase doctor`, `sase validate`, and
  `sase memory init --check` all pass.
