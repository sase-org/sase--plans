---
tier: tale
size: medium
title: Core and reference memory vocabulary
goal:
  Rename short-term/long-term memory to core/reference across the Rust tier wire, note
  frontmatter, AGENTS.md anchors, templates, skills, docs, and prose; accept the old
  spelling forever; and migrate this repo's existing notes in place.
proposed_by: bbugyi200.athena.sase-sq.1
bead: sase-sq.1
create_time: 2026-08-24 09:45:37
status: wip
---

- **PARENT:** [202608/memory_webs.md](memory_webs.md)
- **BEAD:**
  [sase-sq.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sq/sase-sq.1.md)

# Plan: Core and reference memory vocabulary

Implements phase `tiers` of epic bead `sase-sq` (phase bead `sase-sq.1`,
`plan:202608/memory_webs.md`). This is the first phase and has no dependencies.

## Goal

"Short-term memory" becomes **core memory** and "long-term memory" becomes **reference
memory** everywhere: the Rust tier wire, `type:` frontmatter, the `AGENTS.md` H2
anchors, templates, the `/sase_memory_read` skill, CLI help, docstrings, and docs.

Two properties are non-negotiable:

1. **`short` and `long` stay accepted forever on read.** Frontmatter, the Rust wire, and
   the draft validator all keep parsing them.
2. **Already-generated documents keep parsing.** `parse_amd_agents_document` reads
   documents that other, not-yet-reinitialized projects have already emitted, so the
   Tier 1 / Tier 2 anchor regexes **add** the new spelling and never replace the old.

The H2 anchors stay `## Tier 1 (...) Memory` and `## Tier 2 (...) Memory`; only the
parenthetical changes.

## Design decisions

### Normalize at parse, not at every comparison

`parse_memory_note_text` maps `short` → `core` and `long` → `reference` before it builds
the `MemoryNote`. Every downstream comparison (`note.type == "long"` in twenty-odd
modules) then becomes a plain value swap instead of a two-value test, and no module
outside `notes.py` has to know a legacy spelling exists. `type_source` stays
`"frontmatter"` for a legacy spelling: the note declared a valid tier.

### The Rust change lands in `sase-core`; the dependency window does not move here

`MemoryTierWire` lives in `../sase-core`. Land the parse/`as_str`/message change there
as its own commit. Do **not** hand-edit `[workspace.package].version`, crate versions,
or path-dependency pins — release-plz owns them (`sase-core/AGENTS.md`) — and do **not**
bump `sase-core-rs` in `pyproject.toml`: `just _setup` already prints that a checkout
ahead of the published window is normal, and the release-branch reconciler ratchets the
window at release time.

`just install` builds `sase_core_rs` from `sase/repos/linked/sase-core` when cargo is
present, and CI's `build-core` job builds it from `sase-core` master, so local and CI
verification both see the new binding.

### A published-floor compatibility shim in Python

The published floor (`sase-core-rs` 0.31.x) predates the rename, so its
`memory_note_issue` rejects `type: core`. Users who `pip install sase` without a
`sase-core` checkout get that floor, and CI's `release-core-floor-smoke` lane pins it
explicitly. Once this repo's notes say `type: core`, an unshimmed call would report
every memory note as an invalid xprompt memory.

So `sase.content_layout.memory_note_issue` translates the note type to the legacy wire
spelling before calling the binding. This is exactly the "compatibility adapters stay in
Python" rule from the epic's Rust-boundary section. The translation only ever affects a
_valid_ tier, so a genuinely invalid `type:` still reaches the binding verbatim and
still produces the binding's own message. The shim is inert once the floor ratchets past
the release carrying the rename; removing it is a recorded follow-up, not part of this
phase.

One bounded gap this does **not** close: the Rust-side editor catalog
(`load_editor_xprompt_catalog`, used by `sase_xprompt_lsp`) reads `type:` from disk
itself. An editor running an LSP binary older than the rename will stop offering
`#memory/<stem>` completions for a migrated note until it updates. Nothing in this
repo's Python path depends on that call.

### Migration rides the existing frontmatter-update channel

`plan_amd_memory_sync` already rewrites reference-note frontmatter through
`apply_memory_frontmatter` and surfaces it as `stale_operation="update"` expected files,
so `--check` and `--diff` report it like any other drift. Reference notes therefore
migrate for free the moment that call site emits `reference`. Core notes have no such
channel today, so the same pass gains one: a core note whose on-disk `type:` uses a
legacy spelling gets an update that rewrites its frontmatter. Its `description` is
passed through so a core note that carries one does not silently lose it.

### Regenerate this repo only

`sase memory init` is run for the `sase` repo. Home memory (`~/sase/memory/`,
`~/AGENTS.md`, `~/CLAUDE.md`) is chezmoi-managed and global to every workspace, and
`bob-cli` is a separate repo; regenerating either from an unlanded tree deploys content
that exists in no sase commit. Both are recorded as follow-ups for the epic's land
agent. Neither is broken by waiting: `type: short` stays valid forever.

### Three green stages

The stages below are ordered so the tree is green at the end of each one. A worker that
must hand off with `/sase_pipe` should do it on a stage boundary, never inside one.

- Stage 1 changes only what is **read**. Nothing on disk changes, so generated-content
  test fixtures are untouched.
- Stage 2 changes what is **written** (anchors, templates, frontmatter) and migrates.
- Stage 3 is prose, the skill, docs, and the vocabulary lock.

## Stage 1 — wire and data model

### `sase-core`

Open it with `/sase_repo` (`sase repo open sase-core -r "..."`) and use only the printed
path. In `crates/sase_core/src/content_layout.rs`:

- Rename `MemoryTierWire::{Short, Long}` to `{Core, Reference}`. The enum already
  carries `#[serde(rename_all = "snake_case")]`, so the editor wire starts emitting
  `core`/`reference`.
- `parse` accepts `"core"` and `"short"` as `Core`, `"reference"` and `"long"` as
  `Reference`, still trimming and still returning `None` for anything else.
- `as_str()` returns `"core"` / `"reference"`.
- `memory_note_issue`'s message says
  ``a SASE memory note must declare `type: core` or `type: reference` `` — keep naming
  the declared value the same way.
- Update the doc comment on the enum so it describes the rendering axis in the new
  vocabulary.

Update every construction and match site: `lib.rs` re-exports, `xprompt_catalog.rs`,
`editor/hover.rs`, `editor/definition.rs`, `editor/completion.rs`, `host_bridge.rs`.

Extend the `content_layout.rs` unit test that currently asserts
`parse("short") == Some(Short)`: all four spellings parse to the right variant, an
unknown value is still `None`, and `as_str()` emits `core` / `reference`.

Verify with `just check` from the `sase-core` root — never `cargo test -p sase_core`
alone, which skips the `sase_core_py` binding tests. Commit with a conventional-commit
subject (`feat(memory): accept core and reference memory tier spellings`); leave
versions to release-plz.

Then, back in this repo, `just install` so the venv picks up the rebuilt binding.

### Python data model

`src/sase/memory/notes.py`:

- `MemoryNoteType = Literal["core", "reference"]`.
- `_VALID_NOTE_TYPES` accepts all four spellings; add a legacy map
  (`{"short": "core", "long": "reference"}`) and normalize inside
  `parse_memory_note_text`. `type_source` stays `"frontmatter"` for a legacy spelling.
- `_children_of`, `_render_memory_note_references`, and `render_long_memory_sections`
  compare `"reference"`. Rename their local `long_notes` bindings and the
  `render_long_memory_sections` docstring accordingly; keep the public function name
  (renaming it is churn this phase does not need).

`src/sase/content_layout.py`: `memory_note_issue` translates `core` → `short` and
`reference` → `long` before calling the binding, with a comment naming the published
floor and the removal condition.

`src/sase/xprompt/models.py`: `MemoryType = Literal["core", "reference"]`.
`src/sase/xprompt/loader_memory.py`: `_memory_type` maps all four spellings; the
defense-in-depth message names `type: core` / `type: reference`.

Value swaps, all `"short"` → `"core"` and `"long"` → `"reference"`:

- `src/sase/memory/read_log.py`, `render.py`, `mutation.py`,
  `inventory_reachability.py`, `proposals/review.py`
- `src/sase/memory/mutation_validate.py` — additionally accept all four spellings on
  input and normalize; the field error becomes "memory note type must be core or
  reference" and the child-guard error "cannot change type to core while children exist"
- `src/sase/amd/_memory.py`
- `src/sase/ace/tui/memory_panel_catalog.py` and
  `src/sase/ace/tui/modals/memory_panel_{rendering,delete,add,actions}.py`. The add
  form's `Select` options become `("core — Tier 1, always loaded", "core")` and
  `("reference — Tier 2", "reference")` with `"reference"` as the default; the delete
  confirmation reads `1 (core)` / `2 (reference)`.
- `src/sase/main/init_memory/root_rendering.py` — the README sort key and both note
  counts

Leave every `apply_memory_frontmatter(note_type=...)` call site emitting `short`/`long`
in this stage. Nothing on disk changes yet, so generated-content fixtures stay valid.

**Stage 1 tests**: a note declaring `type: short` parses to `type == "core"` with
`type_source == "frontmatter"`; the same for `long` → `reference`; an unknown value is
still `invalid`; `memory_note_issue` accepts a `type: core` note against the installed
binding. Update existing assertions that expect `"short"` / `"long"` off a parsed note.

Green check: `just check`.

## Stage 2 — anchors, emitted frontmatter, templates, migration

### Document anchors — add, never replace

`src/sase/amd/_agents_doc.py`:

- `_SHORT_SECTION_RE` → `_CORE_SECTION_RE`, matching
  `Tier 1 \((?:short-term|core)\) Memory`.
- `_LONG_SECTION_RE` → `_REFERENCE_SECTION_RE`, matching
  `Tier 2 \((?:long-term|reference)\) Memory`.
- Rename the tier-named identifiers that ride along
  (`_AmdAgentsDocument.has_short_section` / `has_long_section` / `short_memory_paths` /
  `long_memory_entries`, `_AmdLongMemoryEntry`, `_SHORT_MEMORY_*` / `_LONG_MEMORY_*`
  regex constants) and their users in `src/sase/amd/_memory.py`.
- **Do not touch the `memory/[A-Za-z0-9_.-]+\.md` path character classes.** They match
  paths, not tiers, and the epic depends on them staying flat.

`src/sase/amd/templates/AGENTS.template.md` emits `## Tier 1 (core) Memory` and
`## Tier 2 (reference) Memory`. The Tier 1 preamble already says "core (always loaded)
context" and stays as is.

`src/sase/amd/_memory.py`: the two structural-anchor assertion strings name the new
anchors; `_LONG_MEMORY_INTRO` / `_LONG_MEMORY_INTRO_FIRST_SENTENCE` become
`_REFERENCE_MEMORY_INTRO` / `_REFERENCE_MEMORY_INTRO_FIRST_SENTENCE`. Their **text is
unchanged** — it already says "detailed reference material" with no "long-term" in it.

### Emitted frontmatter

Flip every `apply_memory_frontmatter(note_type=...)` call site to `"core"` /
`"reference"`: `main/init_memory/root_rendering_notes.py` (four sites),
`root_rendering_task_types.py`, `root_rendering_artifact_relations.py`,
`amd/_memory.py`, `memory/mutation.py`, `memory/mutation_validate.py`,
`memory/proposals/review.py`.

### README template

`src/sase/main/init_memory/templates/memory-README.template.md`:

- The frontmatter-schema and how-it-is-used bullets describe `type: core` and
  `type: reference`.
- Statistics report "Core notes" / "Reference notes".
- Prose that says "long note" says "reference note"; "short note" says "core note".

`root_rendering.py` passes **both** `core_notes`/`reference_notes` and the legacy
`short_notes`/`long_notes` in the render context, while `_MEMORY_README_TEMPLATE_VARS`
requires only the new names. A user whose overridden `memory-README.template.md` still
references the old variable names keeps rendering for this release.

### Migration

In `src/sase/amd/_memory.py`, generalize `_long_memory_description_updates` into a
single frontmatter-update pass:

- Reference notes: unchanged behaviour, now emitting `type: reference`.
- Core notes: when the on-disk `type:` uses a legacy spelling, emit an update built with
  `apply_memory_frontmatter(text, note_type="core", parent=..., description=note.description)`.
  Emit nothing when the rewrite is a no-op, so `--check` stays quiet on an already
  migrated tree.

Rename `AmdLongMemoryDescriptionUpdate` (`src/sase/amd/_shared.py`) to
`AmdMemoryFrontmatterUpdate` and `AmdMemorySyncPlan.description_updates` to
`frontmatter_updates`; `root_rendering.py`'s expected-file `detail` becomes
`"memory note frontmatter"`. Keep `stale_operation="update"` so the change reports as
drift rather than an overwrite.

### Regenerate this repo

Run `sase memory init`. Expect: `sase/memory/*.md` legacy `type:` values rewritten
(`build_and_run.md`, `feature_flags.md`, `gotchas.md`, `rust_core_backend_boundary.md`
and every reference note), `AGENTS.md` plus the four provider shims re-anchored, and
`sase/memory/README.md` restated. Then `sase memory init --check` must be clean.

**Stage 2 tests**

- An `AGENTS.md` already generated with `## Tier 1 (short-term) Memory` /
  `## Tier 2 (long-term) Memory` (including the numbered `## 1. …` shim form) still
  round-trips through `parse_amd_agents_document`.
- A freshly rendered document carries the new anchors.
- `sase memory init` plans an `update` for a note declaring `type: short` and plans
  nothing for one already declaring `type: core`.
- The README template renders when a user override references only `short_notes` /
  `long_notes`.

Update the generated-content fixtures across `tests/main/test_init_memory_*.py`,
`tests/main/test_memory_agent_docs_list.py`, `tests/main/test_section_numbers.py`,
`tests/memory/`, `tests/test_memory_*.py`, `tests/xprompt/`, and `tests/ace/tui/`.

If ACE memory-panel rendering shifts (the delete confirmation and the add form's tier
labels both carry visible text), refresh the goldens with
`just test-visual --sase-update-visual-snapshots` and say so in the bead note.

Green check: `just check`.

## Stage 3 — prose, skill, docs, vocabulary lock

### Source prose

- `src/sase/main/parser_memory.py` (eleven help strings) and `src/sase/main/parser.py`
  (the `memory` group description).
- `src/sase/memory/read_log.py` — the user-facing "memory file is not a long-term memory
  note" error becomes "… is not a reference memory note".
- Docstrings and module headers: `memory/cli_write.py`, `memory/cli_show.py`,
  `memory/cli_review.py`, `memory/review_tui/app.py`, `memory/notes.py`,
  `memory/render.py`, `amd/_shared.py`, `amd/inline_memory.py`,
  `notifications/senders.py`.
- `src/sase/memory/assets/memory-directory-map.prompt.md` — update the prose. The
  checked-in `memory-directory-map.png` is **not** regenerated here; refreshing the
  diagram is a recorded follow-up.

### Skill

Rewrite `src/sase/xprompts/skills/sase_memory_read.md` for the new vocabulary and
introduce the two axes explicitly, because every later sentence in the epic is ambiguous
without them:

> What a memory **is** (note, web, strand) and how a memory **renders** (core,
> reference) are independent. A web is not "a core memory"; a web _renders as_ core
> memory.

State that core memory is always loaded and cannot be read with `sase memory read`, that
reference memory is read on demand, that `type: short` / `type: long` remain accepted,
and that web and strand addressing (`sase memory read <web>:<keyword>`) is **not
available yet** — it arrives with the memory-web substrate. Do not advertise a command
this phase does not ship.

Do **not** run `sase skill init --force`: per `sase/memory/generated_skills.md` the
template is committed and landed first, and deployment happens from a clean, merged
tree. Record the deploy as a follow-up for the land agent.

### Docs

`docs/memory.md`, `docs/init.md`, `docs/cli.md`, `docs/configuration.md`,
`docs/xprompt.md`, `docs/notifications.md`, `docs/index.md`, `docs/architecture.md`.
Where a doc quotes an anchor, quote the new one and note that the old spelling is still
parsed.

Leave `docs/blog/posts/**` alone. Those are dated, published posts; rewriting them would
be falsifying a record, and they read fine as history.

### Vocabulary lock

Last, so the whole phase produces one regeneration. Add five terms with
`sase glossary add`, each with the aliases below:

| Term             | Aliases                         |
| ---------------- | ------------------------------- |
| Core Memory      | `core`, `short-term memory`     |
| Reference Memory | `reference`, `long-term memory` |
| Memory Web       | `web`                           |
| Memory Strand    | `strand`                        |
| Strand Keyword   | `keyword`                       |

Core Memory and Reference Memory define the rendering axis. Memory Web, Memory Strand,
and Strand Keyword define the kind axis this epic introduces; write them as the shape a
web takes (one flat descriptor note plus a sibling directory of strand files, addressed
`<web>:<keyword>`) without claiming a command that does not exist yet. Definitions
should mention each other so `resolve_glossary_closure` links them.

Confirm `sase glossary add` regenerated cleanly and re-run `sase memory init --check`.

## Verification

1. `just install` — required after the `sase-core` change and because this workspace may
   be stale.
2. `just check` at the end of each stage.
3. `sase memory init --check` clean for this repo.
4. `just check-full` through `/sase_monitor` before landing — it routinely outruns a
   single turn, so never run it inline:

   ```bash
   sase monitor start --command 'just check-full' \
     --start-status TESTING --stop-status TESTED --next '...'
   ```

5. `just test-visual` only if ACE memory-panel goldens moved.

## Out of scope, recorded as `PROPOSED FOLLOW-UP:` notes on `sase-sq.1`

- Regenerate home memory (chezmoi-managed) and the `bob-cli` project after this lands.
- Deploy the rewritten `/sase_memory_read` skill with `sase skill init --force` from a
  clean, merged tree.
- Drop the `memory_note_issue` legacy-spelling translation once the `sase-core-rs` floor
  ratchets past the release carrying the rename.
- Refresh `sase/memory/assets/memory-directory-map.png` for the new tier vocabulary.

## Risks

| Risk                                                                | Severity | Mitigation                                                                                             |
| ------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| Anchor rename breaks already-emitted shims in unmigrated trees      | high     | Anchor regexes add the new spelling and keep the old; an explicit round-trip test on an old document   |
| Published-floor `sase_core_rs` rejects a migrated `type: core` note | high     | The Python translation shim in `memory_note_issue`, exercised by a test                                |
| A partial rename leaves the tree red mid-turn                       | medium   | Three stages, each ending green; hand off only on a stage boundary                                     |
| A core note silently loses its `description` during migration       | medium   | The migration passes `description=note.description` through `apply_memory_frontmatter`                 |
| Editors on an old LSP binary stop completing migrated notes         | low      | Bounded and self-healing on the next `sase-core` release; stated here so it is not discovered as a bug |
| A user's overridden README template breaks on renamed variables     | low      | Both old and new variable names are passed; only the new ones are required                             |
