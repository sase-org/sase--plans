---
status: done
tier: epic
title: Glossary panel with term-and-relation navigation, project cycling, and add/delete
goal: "A user drafting a prompt can press `gG` or `<ctrl+g>G` to open a Glossary panel
  that browses one project's terms alphabetically, travels through related terms in both
  directions with a back trail, cycles the visible project with `p`/`P`, and adds or
  deletes terms through the same engine that backs the new `sase glossary add` and `sase
  glossary del` commands.

  "
phases:
  - id: mutation
    title: Shared glossary add/delete engine
    depends_on: []
    size: medium
    description:
      "mutation: build the project-scoped glossary write engine that resolves a target
      project's config, validates a candidate entry set through the Rust glossary
      validator, applies a source-preserving YAML insert or removal with an atomic
      stale-write guard, and returns a typed outcome carrying diagnostics and a restore
      command; add the shared reverse-reference index alongside it."
  - id: cli
    title: sase glossary add and del commands
    depends_on:
      - mutation
    size: medium
    description:
      "cli: register `add` and `del` in the alphabetically ordered glossary subcommand
      group, render rich and json outcomes, resolve `del` targets through the existing
      alias/prefix lookup, print the restore command, regenerate the project's agent
      instruction files in-process unless suppressed, and extend shell completion and
      docs."
  - id: catalog
    title: Multi-project glossary catalog service for the TUI
    depends_on:
      - mutation
    size: medium
    description:
      "catalog: add the ACE-side service that builds the ordered project ring for
      `p`/`P`, loads and compiles each project's glossary off the event loop behind an
      mtime-keyed snapshot cache, exposes outbound and inbound relation lists per entry,
      and invalidates cleanly after a write."
  - id: panel
    title: Glossary panel shell, term list, filter, and project ring
    depends_on:
      - catalog
    size: medium
    description:
      "panel: factor the duplicated focused-scope keymap loaders into one generic
      loader, declare the `glossary` keymap scope and every panel action up front, then
      build the modal shell with its header, filterable term list, definition card,
      footer, and `p`/`P` project cycling over the ring."
  - id: travel
    title: Related-term travel, relation chips, and the back trail
    depends_on:
      - panel
    size: medium
    description:
      "travel: render numbered SEE ALSO and REFERENCED BY chip rows on the definition
      card, add the chip cursor, digit shortcuts, follow and back keys, and the bounded
      breadcrumb trail that keeps the term list selection synchronized with every jump."
  - id: actions
    title: Panel add and delete surfaces
    depends_on:
      - mutation
      - panel
    size: medium
    description:
      "actions: add the term-add form with live validation, the delete confirmation that
      shows the inbound blast radius, tracked-proc writes through the shared engine, the
      post-write commit offer, and the optimistic reselect-and-refresh behavior."
  - id: entry
    title: Prompt keymap entry point and focus handoff
    depends_on:
      - travel
      - actions
    size: small
    description:
      "entry: claim `G` on the prompt `g` prefix table so `gG` and `<ctrl+g>G` both open
      the panel with a hint row, post the open request as a message the app handles,
      seed the selection from the glossary term under the cursor, and restore prompt
      focus and vim mode on close."
  - id: polish
    title: Help, docs, and visual snapshots
    depends_on:
      - entry
    size: medium
    description:
      "polish: document the panel in the help modal and the ace guide, document the new
      commands in the CLI and memory docs, record PNG goldens for the panel's light and
      dark themes, and drop the epic symbol whitelist entries this epic added."
proposed_by: bbugyi200.athena.056
bead_id: sase-p1
create_time: 2026-09-09 19:50:39
---

- **PROMPT:**
  [prompts/202608/glossary_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_panel.md)
- **BEAD:**
  [sase-p1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p1/README.md)

# Plan: Glossary panel

## Why

SASE already has a rich project glossary. Terms live under `memory.glossary` in a
project's `sase/sase.yml`, the Rust core validates and compiles them, the prompt input
underlines them, `sase glossary list|show|read|log` reads them, and `AGENTS.md`
publishes the term list to agents. Two things are missing.

**There is no way to write.** Every glossary entry is hand-edited YAML. There is no
`sase glossary add`, no `sase glossary del`, and nothing in the TUI. Adding a term means
leaving whatever you were doing, finding the right `sase.yml`, guessing whether your new
alias collides with an existing term, and remembering to re-run `sase memory init` so
`AGENTS.md` reflects it.

**There is no way to browse.** `GlossaryPreviewModal` shows one entry beautifully, but
it only opens from a term already highlighted in a prompt. You cannot open the glossary
cold, you cannot walk it alphabetically, and you cannot see what _points at_ a term —
only what it points to. The relation graph is half-visible.

This epic closes both gaps with one surface: a Glossary panel opened from the prompt
input, backed by a shared add/delete engine that the new CLI commands use too.

## What the user sees

### Opening

From the prompt input widget, in either vim mode:

| Keys        | Where it works                                       |
| ----------- | ---------------------------------------------------- |
| `gG`        | NORMAL mode, the vim `g` prefix                      |
| `<ctrl+g>G` | INSERT and NORMAL mode, the prompt-local `^G` prefix |

Both surfaces already exist and already render a which-key hint panel. `G` is unclaimed
on both, and `gG` is not a vim command, so nothing is shadowed.

If the cursor sits inside a highlighted glossary term when the panel opens, that term is
selected. Otherwise the panel opens on the first term.

### Layout

```
┌ GLOSSARY · sase · 24 terms · project 1/3 ────────────────────────────────────┐
│                                                                              │
│  ┌ TERMS ─────────────────┐  ┌ Agent Hood ──────────────────────────────┐    │
│  │   Agent Clan           │  │  hood · agent neighborhood               │    │
│  │   Agent Family         │  │                                          │    │
│  │ › Agent Hood           │  │  An agent hood is a group of agents that │    │
│  │   Agent Instruction F… │  │  are all named with the same `<name>.`   │    │
│  │   Agent Neighbor       │  │  prefix. For example, agents named       │    │
│  │   Agent Node           │  │  `foo.bar` … The agent `foo`, if it      │    │
│  │   Agent Shell          │  │  exists, is also considered part of the  │    │
│  │   Agent Tribe          │  │  `foo` agent hood.                       │    │
│  │   Artifact Reference   │  │                                          │    │
│  │   Feature Flag         │  │  ────────────────────────────────────    │    │
│  │   Flag Bead            │  │  SEE ALSO       ① Sase Agent             │    │
│  │   …                    │  │  REFERENCED BY  ② Agent Neighbor         │    │
│  │                        │  │                 ③ Agent Clan             │    │
│  │                        │  │                                          │    │
│  │                        │  │  PROJECT  sase                           │    │
│  │                        │  │  ALIASES  2 configured · 6 effective     │    │
│  │                        │  │  SOURCE   sase/sase.yml:18               │    │
│  └────────────────────────┘  └──────────────────────────────────────────┘    │
│                                                                              │
│  j/k term   <tab> relation   l follow   h back   / filter   p/P project      │
│  a add   d delete   o edit   y copy   ? help   esc close                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

When a trail exists, a breadcrumb strip appears above the footer:

```
  TRAIL  Artifact Reference › Sase Agent › Agent Hood
```

### The two navigation axes

The prompt asks for term-by-term navigation _and_ travel through related terms. These
are deliberately two axes that stay synchronized, not two modes.

**Axis 1 — alphabetical.** `j`/`k` move the term-list cursor. The definition card
follows (debounced, per the TUI perf rules). `g`/`G` jump to the first/last term. `/`
opens a filter over terms and aliases; `.` toggles the filter to also match definition
bodies, mirroring `sase glossary list --definitions`.

**Axis 2 — relational.** The definition card carries two numbered chip rows:

- **SEE ALSO** — terms this definition references (depth-1 outbound, from
  `resolve_glossary_closure(..., depth=1)` filtered to `origin == "related"`).
- **REFERENCED BY** — terms whose definitions reference _this_ term (inbound). This is
  new; it is what makes the graph walkable upward as well as downward.

Numbering is continuous across both rows (① ② ③ …) so a digit is never ambiguous.

- `<tab>` / `<shift+tab>` move a chip cursor across both rows; the focused chip is
  highlighted and echoed in the footer as `→ Sase Agent`.
- `l` or `<enter>` travels to the focused chip, or to ① when no chip is focused.
- `1`–`9` travel directly to that numbered chip. This matches the muscle memory
  `GlossaryPreviewModal` already teaches.
- `h` or `<backspace>` walks back along the trail.

**Travel keeps the axes in sync.** Following a relation moves the _term-list cursor_ to
the target term, so you always know where you landed alphabetically, and `j`/`k`
continue from there. Traveling pushes the previous term onto a bounded trail (32
entries). If the target is hidden by an active filter, travel clears the filter rather
than failing, and toasts why.

### Project cycling

`p` and `P` cycle forward and backward through the enabled-project ring. Only the
selected project's terms are listed.

The ring contains every enabled project that has a glossary configured, plus the project
the panel was opened from even when it has none — so `a` can bootstrap an empty
project's first term. Ring order is by project **display name**; the header and every
label render the configured `PROJECT_NAME:`, never the `ProjectSpec` key. The header
shows the position (`project 2/3`) so a single-project setup reads as `project 1/1`
rather than looking broken.

Switching projects clears the trail and the filter, and restores that project's
last-selected term for the life of the panel.

### Adding and deleting

`a` opens a form: **Term** (required), **Aliases** (comma-separated, optional),
**Definition** (required, multi-line). The form validates live against the candidate
entry set through the Rust validator, so a duplicate term or a colliding alias is
reported under the offending field _before_ anything is written. `<ctrl+s>` submits;
`<esc>` cancels.

`d` opens a confirmation showing the term, its aliases, the first line of its
definition, and its inbound REFERENCED BY list — so "3 definitions reference this term"
is visible before you confirm rather than discovered afterward.

Both write through the same engine the CLI uses, run as tracked procs, refresh the
affected project's catalog, and offer a config commit for the written `sase.yml` exactly
the way the Models panel already offers one. A delete toasts the exact
`sase glossary add …` command that restores the entry.

### CLI

```
sase glossary add TERM DEFINITION [-a ALIAS]... [-f FORMAT] [-I] [-p REF]
sase glossary del TERM [-f FORMAT] [-I] [-n] [-p REF]
```

Required values are positionals and every long option has a short alias, per the CLI
rules. Subcommands register alphabetically: `add`, `del`, `list`, `log`, `read`, `show`.
Bare `sase glossary` still delegates to `list`.

`del` resolves `TERM` through the same lookup `show` and `read` use, so aliases,
slug-forms, and unique prefixes all work, and it prints the restore command instead of
prompting — a non-interactive-safe undo that costs one copy-paste.

Both commands regenerate the project's agent instruction files afterward, because a term
that is not in `AGENTS.md`'s `GLOSSARY TERMS` block is invisible to agents and the write
is therefore incomplete without it. `-I/--no-init` skips that step.

## Architecture

### Where the code lives

| Concern                          | Location                                                                |
| -------------------------------- | ----------------------------------------------------------------------- |
| Add/delete engine, reverse index | `src/sase/glossary/mutation.py`, `src/sase/glossary/relations.py`       |
| CLI handlers                     | `src/sase/glossary/cli_add.py`, `src/sase/glossary/cli_del.py`          |
| CLI parser and dispatch          | `src/sase/main/parser_glossary.py`, `src/sase/main/glossary_handler.py` |
| TUI catalog service and ring     | `src/sase/ace/tui/glossary_panel_catalog.py`                            |
| Panel and its modals             | `src/sase/ace/tui/modals/glossary_panel*.py`                            |
| Panel keymap scope               | `src/sase/ace/tui/keymaps/`, `src/sase/default_config.yml`              |
| Prompt entry point               | `src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py`        |

Split modules the way the existing panels do — `models_panel*.py` and `zoom_panel*.py`
are the precedent — rather than growing one large file.

### Reuse, do not reimplement

This epic is mostly assembly. The following already exist and must be used as-is:

- `sase.core.glossary_facade` — `validate_glossary_entries`, `build_glossary_catalog`,
  `compile_glossary_catalog`, `scan_glossary_spans`, `lookup_glossary_span`.
- `sase.xprompt.glossary_catalog` — `editor_glossary_catalog_for_project` and the
  `EditorGlossaryCatalog` shape the panel renders from.
- `sase.glossary.resolution` — `resolve_glossary_closure`,
  `normalize_glossary_reference`, and the alias/prefix lookup behind
  `GlossaryLookupError`.
- `sase.glossary.cli_common` — `resolve_glossary_cli_project` and its error taxonomy.
- `sase.ace.tui.modals.glossary_preview_render` — `build_glossary_title`,
  `build_alias_chips`, `build_see_also_chips`, `build_property_grid`,
  `glossary_definition_markdown`, `glossary_cross_references`, `glossary_source_path`,
  `glossary_definition_position`, `glossary_card_accent`. The panel's definition card is
  these helpers in a two-column frame, so the card and the preview modal cannot drift.
- `sase.ace.tui.modals._source_file_actions.SourceFileActionsMixin` — `o` open in
  editor, `Z` open in viewer.
- `sase.ace.tui.modals.config_commit` — `build_config_commit_offer`,
  `push_config_commit_prompt`.
- `sase.config._edit_yaml` — `set_key`, `unset_key` (source-preserving, surgical when it
  can be, round-trip otherwise).
- `sase.content_layout` — `resolve_project_config_write_path`.
- `sase.core.project_lifecycle_facade.list_project_records` — enabled projects.

### Rust core boundary

**No `sase-core` change is required, and that is a deliberate reading of the boundary
rule, not an omission.**

The glossary's shared domain logic already lives in Rust: what a valid entry set is,
what collides, how aliases expand, and how phrases match are all owned by
`glossary_validate`, `glossary_catalog`, and `compile_glossary_catalog`. This epic calls
those unchanged. What it adds on the Python side is exactly what the boundary already
assigns to Python — the module docstring of `src/sase/core/glossary_facade.py` states it
outright: _"Python callers own file discovery, source-preserving YAML parsing, and
editor presentation."_ Locating a project's `sase.yml`, splicing a key into it without
disturbing neighboring bytes, and rendering a panel are all on that side of the line.

Two pieces deserve an explicit check rather than an assumption:

1. **Reverse references.** Computed from `resolve_glossary_closure`, which is already
   Python (`src/sase/glossary/resolution.py`) and already the shared closure walker
   behind `sase glossary show`/`read`/`list`. The reverse index is one more traversal of
   the same spans, so it belongs beside its forward sibling.
2. **Insertion ordering.** Where a new term lands in the YAML map is a
   source-preservation question, not a semantic one — the catalog Rust builds is
   order-independent.

The tripwire for a phase worker: if you find yourself writing new rules about _what a
glossary means_ — which references are legal, how aliases expand, what makes two terms
the same — stop and push that into `../sase-core` instead. Anything about _where bytes
go_ or _what the screen shows_ stays here.

### No feature flag

The rule is that user-reaching behavior gets a flag before it is ready. Nothing here
reaches a user before it is ready, because the only entry point — `G` on the prompt `g`
prefix — lands in the `entry` phase, after the panel, its travel keys, and its write
surfaces are all complete. The CLI commands land whole in a single phase. A `wip` flag
would guard a path no user can reach.

The real risk this creates is Symvision failing on public symbols whose consumer arrives
a phase later. Handle that the sanctioned way: add `--epic-symbol <bead_id>(<symbol>)`
to the Symvision invocation in the `Justfile`, and delete each entry in the phase that
gives the symbol a real consumer. The `polish` phase verifies none are left.

### TUI performance contract

Every phase touching the TUI must hold these, which come from the project's TUI
performance rules:

- Catalog loads, config writes, `sase memory init` runs, and git-status probes never run
  on the event loop. Push them off-thread and marshal results back.
- Panel writes are tracked procs (`_submit_tracked_proc()`), not fire-and-forget
  coroutines, so they appear in the proc indicator and count at quit.
- Catalog snapshots are cached keyed by `(config path, mtime_ns, size)`. Render paths
  never stat, glob, or parse YAML.
- The term-list cursor repaints immediately; the definition card updates through
  `DetailPanelDebouncer`.
- Programmatic `OptionList.highlighted` assignments are wrapped in a
  `ProgrammaticSelectionGuard` cleared in a `finally:`.
- Re-capture selection after every `await`; a catalog that lands late must not move a
  cursor the user has since moved.

## Phase 1 — Shared glossary add/delete engine

`src/sase/glossary/mutation.py` exposes two operations and one shared resolution step.

```python
@dataclass(frozen=True, slots=True)
class GlossaryMutationOutcome:
    project_name: str        # display name, never the ProjectSpec key
    config_path: str
    term: str
    aliases: tuple[str, ...]
    definition: str
    created_section: bool    # memory.glossary did not exist before
    restore_command: str     # exact `sase glossary add …` that undoes a delete
    referenced_by: tuple[str, ...]   # inbound terms, for delete blast radius

class GlossaryMutationError(RuntimeError): ...      # base
class GlossaryValidationError(GlossaryMutationError):
    diagnostics: tuple[GlossaryDiagnostic, ...]
class GlossaryConflictError(GlossaryMutationError): ...   # file changed under us
```

`add_glossary_term(project_ref, term, definition, aliases)` and
`delete_glossary_term(project_ref, reference)` both:

1. Resolve the project. Reuse `resolve_glossary_cli_project` where a catalog must exist
   (delete), and fall back to the project-record path for add, which must work when
   `memory.glossary` does not exist yet. Preserve the existing error taxonomy that
   distinguishes an unknown project from a project with no glossary.
2. Resolve the write target with `resolve_project_config_write_path(workspace_dir)`. The
   glossary is project-scoped only — `_load_editor_glossary_catalog` reads a project's
   own config and nothing else, so there is no home or chezmoi layer to consider here.
   Read the file's bytes once and keep them.
3. Build the candidate entry list — the current entries with the new one inserted, or
   with the target removed — and run `validate_glossary_entries` on it. Raise
   `GlossaryValidationError` with the Rust diagnostics on any `error`-severity result.
   **Nothing is written when validation fails.**
4. Apply `set_key(text, ("memory", "glossary", term, …), …)` or
   `unset_key(text, ("memory", "glossary", term))` to the text.
5. Write atomically with the same guard `apply_config_edit` uses: re-read the file's
   bytes immediately before replacing, compare against the bytes read in step 2, and
   raise `GlossaryConflictError` on a mismatch instead of clobbering a concurrent edit.
   Write to a temp file in the same directory, preserve the mode, then replace.

Reference `sase/config/_edit_plan.py::apply_config_edit` for the exact atomic-write and
token-comparison shape and follow it. Prefer routing through `plan_config_edit` /
`apply_config_edit` outright if the phase worker confirms that the inventory/layer
machinery can be aimed at an arbitrary project's config file; that machinery exists to
resolve layered precedence (home vs project vs chezmoi), which the glossary does not
have, so a direct `set_key`/`unset_key` write against the resolved project path is the
expected outcome. Whichever route is taken, the stale-write guard is not optional.

**Insertion placement.** New terms are inserted so the map stays sorted by term,
matching how every existing glossary reads today. If `set_key`'s surgical path cannot
place a key in sorted position, fall back to the round-trip writer for insertions and
keep the surgical path for removals; a correctly-ordered file is worth the wider diff.
Multi-line definitions are written as folded block scalars (`>-`), matching the existing
entries.

**Term text rules.** Reject a blank term, a term containing a newline, and a term that
is only separators, with a clear message. Everything else — collisions, alias overlap,
normalization — comes from the Rust validator; do not re-derive it here.

`src/sase/glossary/relations.py` adds:

```python
def glossary_reverse_references(
    catalog: GlossaryCatalog, compiled: CompiledGlossaryCatalog
) -> Mapping[int, tuple[str, ...]]:
    """Map each entry index to the terms whose definitions reference it."""
```

built from one pass of `scan_glossary_spans` over every entry's definition, keyed by
`entry.index`, with self-references dropped. Both the CLI's delete blast radius and the
panel's REFERENCED BY row read it.

**Tests** (`tests/main/` and `tests/`): add creating the first term in a project with no
`memory.glossary`; add into an existing sorted map lands in sorted position; add with
aliases; duplicate term rejected with the Rust diagnostic and the file untouched; alias
colliding with an existing term rejected; delete by exact term, by alias, and by unique
prefix; delete of an unknown term raises the existing lookup error with candidates;
comments and unrelated keys in `sase.yml` survive both operations byte-for-byte outside
the edited region; a file mutated between read and write raises `GlossaryConflictError`
and leaves the file alone; reverse references are correct including the self-reference
drop.

## Phase 2 — sase glossary add and del commands

Register `add` and `del` in `register_glossary_parser`, moving the existing
registrations so all six subcommands appear alphabetically. Give the group epilog
examples for both.

```
add   TERM DEFINITION   -a/--alias (repeatable)  -f/--format {json,rich}
                        -I/--no-init  -p/--project REF
del   TERM              -f/--format {json,rich}  -I/--no-init
                        -n/--dry-run  -p/--project REF
```

Handlers land in `src/sase/glossary/cli_add.py` and `src/sase/glossary/cli_del.py` and
dispatch from `handle_glossary_command`, whose usage line grows to
`{add,del,list,log,read,show}`.

Output is colored and scannable, matching `cli_list.py`'s use of
`sase.cli_show_palette`. `add` prints the project, the term, its effective aliases, and
the config path written. `del` prints the same plus the inbound reference count and the
restore command. `-n/--dry-run` on `del` prints exactly that block and exits without
writing. `-f json` emits a stable machine-readable object for both.

Validation failures print each Rust diagnostic on its own line with the config path and
key path, and exit non-zero. A `GlossaryConflictError` prints the reload-and-retry
message and exits non-zero.

**Agent instruction regeneration.** On success, unless `-I/--no-init`, regenerate the
target project's instruction files by calling the in-process entry point in
`src/sase/main/init_memory_handler.py` (`run_init_memory`) — not a subprocess. This is
the same regeneration the post-commit hook already performs, so it is idempotent. Print
one line naming what was regenerated. A regeneration failure is reported as a warning
with the manual `sase memory init` follow-up, and does **not** roll back or fail the
write, which already succeeded.

**Completion.** Add `(("glossary", "del"), "term"): ValueKind.GLOSSARY` to the mapping
in `src/sase/completion/kinds.py` so `sase glossary del <TAB>` completes terms and
aliases through the existing candidate provider. `add`'s `TERM` is new text and gets no
completion.

**Docs.** Update `docs/cli.md` and the glossary section of `docs/memory.md` with both
commands, the restore-command undo, and the automatic regeneration.

**Tests** (`tests/main/`): reuse `glossary_cli_helpers.py`. Cover help text listing
subcommands alphabetically; `add` and `del` happy paths in rich and json; `--dry-run`
writes nothing; `--no-init` skips regeneration and the default performs it; a validation
failure exits non-zero with the diagnostic and no write; unknown project and unknown
term messages; the parser/handler dispatch test grows both subcommands; a completion
test for `del`.

## Phase 3 — Multi-project glossary catalog service for the TUI

`src/sase/ace/tui/glossary_panel_catalog.py` is the panel's only data source.

```python
@dataclass(frozen=True, slots=True)
class GlossaryProjectRef:
    key: str            # identity and storage only
    display_name: str   # everything the user sees
    workspace_dir: str
    has_glossary: bool

@dataclass(frozen=True, slots=True)
class GlossaryProjectSnapshot:
    project: GlossaryProjectRef
    catalog: EditorGlossaryCatalog | None
    reverse_references: Mapping[int, tuple[str, ...]]
    diagnostics: tuple[str, ...]
```

- `build_glossary_project_ring(launch_project_ref)` returns the ordered ring: every
  enabled project with a glossary configured, plus the launch project even when it has
  none, sorted by `display_name.casefold()`, de-duplicated by key. Build it from
  `list_project_records(..., ("enabled",), include_home=False, projects_only=True)`,
  mirroring the filtering `_enabled_project_records` already applies in
  `src/sase/xprompt/glossary_catalog.py` — factor that helper into something both
  callers share rather than copying it.
- `load_glossary_project_snapshot(ref)` calls `editor_glossary_catalog_for_project` and
  `glossary_reverse_references`, and is **only ever called off the event loop**.
- A module-level snapshot cache keyed by `(config_path, mtime_ns, size)` with a bounded
  LRU (8 projects) and a short re-stat throttle. Model it on
  `src/sase/ace/tui/glossary_reads.py`, which already implements exactly this shape for
  the read log.
- `invalidate_glossary_project(key)` drops one project's snapshot after a write.

The ring itself is disk work (project records + one config stat per project) and is
built off-thread once when the panel opens, not per keypress.

**Diagnostics are data, not exceptions.** A project whose glossary fails to parse yields
a snapshot with `catalog=None` and populated `diagnostics`; the panel renders that as an
error state and `p`/`P` still work. One broken project never breaks the ring.

Also expose `glossary_entry_relations(snapshot, entry)` returning the ordered outbound
and inbound term lists the card renders, so the numbering is computed in one place.

**Tests** (`tests/ace/tui/`): ring composition and ordering with a mix of
glossary/no-glossary projects; the launch project is present even with no glossary;
display names are used for ordering and no `ProjectSpec` key leaks into a user-facing
field; the snapshot cache re-reads on an mtime change and not otherwise; invalidation
drops exactly one project; a malformed glossary produces diagnostics rather than
raising.

## Phase 4 — Glossary panel shell, term list, filter, and project ring

### Keymap scope, first

`load_statistics_keymaps` and `load_gate_keymaps` in
`src/sase/ace/tui/keymaps/scopes.py` are near-identical — defaults lookup,
unknown-action warning, per-field canonicalization, invalid-key revert, duplicate-key
revert. Factor them into one generic loader parameterized by scope name, dataclass, and
defaults loader, and re-express both existing scopes through it (including the gate
scope's `activate_control` deprecation shim, which stays a gate-specific pre-step). Do
the same for the three near-identical `_builtin_*_defaults` functions in
`src/sase/ace/tui/keymaps/defaults.py`. Only then add the third scope. Existing
behavior, including every warning message, must not change; a third copy-paste is not
acceptable.

Add `ace.keymaps.glossary` to `src/sase/default_config.yml` with a comment matching the
`statistics` block's, and a `GlossaryPanelKeymaps` dataclass in `app_keymaps.py`:

```yaml
glossary:
  next_term: "j"
  prev_term: "k"
  first_term: "g"
  last_term: "G"
  scroll_definition_down: "ctrl+d"
  scroll_definition_up: "ctrl+u"
  filter_terms: "slash"
  toggle_definition_filter: "full_stop"
  next_relation: "tab"
  prev_relation: "shift+tab"
  follow_relation: "enter,l"
  travel_back: "backspace,h"
  next_project: "p"
  prev_project: "P"
  add_term: "a"
  delete_term: "d"
  open_source: "o"
  open_viewer: "Z"
  copy_definition: "y"
  copy_source_path: "Y"
  refresh: "r"
  help: "question_mark"
```

Comma-separated alternatives are already supported (`split_key_alternatives`, as
`open_command_palette: "colon,semicolon"` uses). Declare **every** action in this phase,
including the ones `travel` and `actions` implement, with stub methods, so the keymap
surface is settled once. Add the labels to `src/sase/ace/tui/keymaps/metadata.py`.

Digits `1`–`9` are fixed bindings, not configurable, matching `GlossaryPreviewModal`.

### The panel

`GlossaryPanel(ModalScreen[None])` in `src/sase/ace/tui/modals/glossary_panel.py`, split
across focused modules the way `models_panel*.py` is. Composition: a header `Static`, a
`Horizontal` holding the term `OptionList` and a `VerticalScroll` definition card, an
optional trail strip, and a footer `Static`.

This phase delivers:

- **Open and load.** Mount instantly with a loading state; build the ring and load the
  first snapshot off-thread; apply on the UI thread. Accept an optional
  `initial_term`/`initial_project` so `entry` can seed from the cursor.
- **Header.** `GLOSSARY · <display name> · <n> terms · project <i>/<N>`, using
  `glossary_card_accent(self.app.current_theme)` so light and dark both read well.
- **Term list.** Alphabetical by term. Rows show the term and a dim alias summary; long
  terms ellipsize. `j`/`k`/arrows/`ctrl+n`/`ctrl+p` move; `g`/`G` to ends.
- **Definition card.** `build_glossary_title`, `build_alias_chips`,
  `glossary_definition_markdown`, and `build_property_grid` from
  `glossary_preview_render`, in a two-column frame. Chip rows arrive in `travel`.
- **Filter.** `/` opens an inline filter input over terms and aliases (the same
  predicate `_filter_entries` in `cli_list.py` uses — extract and share it rather than
  re-writing the casefold logic). `.` toggles definition-body matching. `<esc>` closes
  the filter and keeps the selection when it is still visible. An empty result renders
  `no terms matched: <pattern>`.
- **Project cycling.** `p`/`P` step the ring, showing a loading state for an uncached
  project, restoring that project's remembered selection, and clearing trail and filter.
  Re-capture the current project after the load lands and drop a result the user has
  since cycled past.
- **Empty and error states.** A project with no glossary renders a centered invitation
  naming the project and pointing at `a`. A project with diagnostics renders them plus
  the config path. Both keep `p`/`P`, `a`, and `?` live.
- **Passive actions.** `y` copy definition, `Y` copy source path, `o` open in editor,
  `Z` open in viewer — via `SourceFileActionsMixin` and `schedule_copy_delivery`,
  matching `GlossaryPreviewModal`. `r` re-reads the current project, bypassing the
  cache.
- **Styles.** Add a `GlossaryPanel` block to `src/sase/ace/tui/styles.tcss` near the
  existing `GlossaryPreviewModal` rules, sharing its accent and border language.
- **Help.** `?` opens a panel-scoped help overlay listing the effective keys, following
  `statistics_help_modal.py`.

**Tests** (`tests/ace/tui/modals/`): the generic keymap loader preserves both existing
scopes' behavior including warning text, and loads the new one with overrides, invalid
keys, and duplicate keys; the panel mounts and selects the first term; `j`/`k` move and
the card follows; filter matches terms, aliases, and — with `.` — definitions; empty
filter state; `p`/`P` cycle in display-name order and only the selected project's terms
render; a no-glossary project renders the invitation; a diagnostics project renders the
error; no snapshot or catalog load happens on the event loop.

## Phase 5 — Related-term travel, relation chips, and the back trail

Extend the definition card and add the second navigation axis.

- **Chip rows.** SEE ALSO from `glossary_cross_references`, REFERENCED BY from the
  snapshot's reverse index, both from `glossary_entry_relations` so the numbering is
  computed once. Render with `build_see_also_chips` extended (or a sibling helper) to
  take a starting index and a focused index, so numbering continues across the two rows
  and the focused chip is visually distinct. A row with no members is omitted, not shown
  empty.
- **Chip cursor.** `<tab>`/`<shift+tab>` move across the concatenated chip list and
  wrap; the footer echoes `→ <term>`. Moving the term cursor resets the chip cursor.
- **Follow.** `l`/`<enter>` follows the focused chip, or ① when none is focused and at
  least one chip exists. `1`–`9` follow that number directly. Following:
  1. pushes the current term onto the trail (bounded at 32, oldest dropped);
  2. clears the filter when the target is not currently visible, with a toast naming
     why;
  3. moves the **term-list** cursor to the target under a programmatic-selection guard;
  4. resets the chip cursor and scrolls the card home.
- **Back.** `h`/`<backspace>` pops the trail and restores that term the same way. With
  an empty trail it is a no-op, not an error.
- **Trail strip.** Rendered only when the trail is non-empty: `TRAIL  A › B › C`,
  eliding the middle with `…` when it exceeds the width, always showing the first and
  the two most recent.
- **Cross-project safety.** The trail holds terms within one project. Cycling projects
  clears it. A trail entry whose term no longer exists (deleted from another surface) is
  skipped rather than raising.

**Tests**: chip numbering is continuous across both rows and stable; digits follow the
right chip including when SEE ALSO is empty; `<tab>` wraps; follow moves the term-list
cursor and pushes the trail; follow through an active filter clears it and lands; back
restores the previous term and pops exactly one; back on an empty trail is a no-op; the
trail is bounded at 32; project cycling clears the trail; a deleted trail entry is
skipped; reverse references make an inbound-only term reachable (a term nothing links to
from its own definition still lists what points at it).

## Phase 6 — Panel add and delete surfaces

**Add (`a`).** `GlossaryTermAddModal` with a Term `Input`, an Aliases `Input`
(comma-separated), and a Definition `TextArea`. `<ctrl+s>` submits, `<esc>` cancels,
`<tab>` moves between fields.

Validation runs on a debounce against the candidate entry set through
`validate_glossary_entries`, and renders each diagnostic under the field it names.
Submit is refused while an error-severity diagnostic stands. Validation is pure
computation over already-loaded entries — no disk — so it can run inline; the write
cannot.

**Delete (`d`).** A confirmation built on the existing confirm-modal pattern showing the
term, its aliases, the first line of the definition, the config path, and the inbound
REFERENCED BY list rendered as `3 definitions reference this term: A, B, C`. Default
focus is Cancel.

**Writing.** Both submit through `_submit_tracked_proc()` calling `add_glossary_term` /
`delete_glossary_term` off-thread and returning a typed outcome. On the UI thread:

1. `invalidate_glossary_project(key)` and reload that project's snapshot.
2. Reselect: the new term after an add; after a delete, the nearest surviving neighbor —
   the row that took the deleted term's index, or the last row when it was last.
3. Invalidate the prompt-side glossary catalog so prompt highlighting reflects the
   change without a restart. Find the existing warm/invalidate seam used by
   `PromptGlossaryContext` and `_prompt_glossary.py` and reuse it; do not add a second
   cache path.
4. Toast the outcome. A delete's toast carries the restore command.
5. Offer a config commit with `build_config_commit_offer(config_path, subject=…)` and
   `push_config_commit_prompt`, exactly as the Models panel does after a persistent
   edit. Building the offer shells out to git and stays off the event loop.

`GlossaryValidationError` and `GlossaryConflictError` surface as an error toast naming
the cause; a conflict additionally refreshes the panel so the next attempt plans against
current bytes. Neither leaves the panel in a half-updated state.

The panel does **not** regenerate agent instruction files inline — that is a slower,
wider-blast-radius operation than a TUI action should perform implicitly. The success
toast names the pending `sase memory init`. Revisit only if that proves annoying in use.

**Tests**: the add form validates a duplicate term and a colliding alias and refuses
submit; a valid add writes through the shared engine and the new term is selected;
delete confirmation lists inbound references and the count; cancel writes nothing; a
delete selects the correct neighbor including the last-row case; a conflict error toasts
and refreshes; a validation error toasts and leaves the file unchanged; the commit offer
is built off the event loop; both writes go through `_submit_tracked_proc`.

## Phase 7 — Prompt keymap entry point and focus handoff

One row in `_PROMPT_G_PREFIX_BINDINGS`
(`src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py`):

```python
_PromptGPrefixBinding(
    "G",
    "request_open_glossary_panel",
    "_g_prefix_label_glossary",
    "_g_prefix_available_glossary",
),
```

That single row is the whole key wiring, and it is worth understanding why rather than
adding a special case:

- `gG` — `_handle_normal_pending_key` in `_vim_normal_pending.py` gives host `g`
  continuations priority over vim's own, forwarding through
  `_dispatch_host_g_prefix_key` to `dispatch_g_prefix_key`.
- `<ctrl+g>G` — `_handle_insert_g_prefix_key` and `_handle_normal_g_prefix_key` in
  `_prompt_text_area_key_handling.py` both resolve the second key and call
  `dispatch_g_prefix_key(..., via_ctrl_g=True)`.
- The which-key hint panel is driven by the same table, so a hint row appears for free.

The binding is **not** `ctrl_g_only`, because the user asked for both surfaces.
`_g_prefix_label_glossary` returns `"glossary…"`. `_g_prefix_available_glossary` returns
`True` in prompt mode; the panel handles the empty and no-glossary cases itself and is
useful precisely when a project has no terms yet.

`request_open_glossary_panel` stays presentation-only per the prompt-bar boundary rule:
it captures the term under the cursor and posts a new `GlossaryPanelRequested` message
carrying the term (or `None`) and the bar's current mode. Add the message to
`_prompt_input_bar_messages.py` and re-export it from `prompt_input_bar.py` beside
`RestoreRequested`; handle it in a new module under
`src/sase/ace/tui/actions/agent_workflow/`, modeled on `_prompt_bar_stash_restore.py`.

**Cursor seeding.** Detect the term with the existing `lookup_glossary_span` path
already used by `_prompt_glossary.py` for the preview action — reuse that helper rather
than re-scanning. A hit seeds the panel's initial term; a miss opens on the first term.

**Focus handoff.** Record the focused pane and vim mode before pushing the panel and
restore both on dismiss, so `<ctrl+g>G` from INSERT returns to INSERT at the same cursor
position. This is the difference between the panel feeling like part of the prompt and
feeling like an interruption.

**Tests** (`tests/ace/tui/widgets/`): extend the existing g-prefix routing and hint
lifecycle tests — `gG` from NORMAL, `<ctrl+g>G` from INSERT, and `<ctrl+g>G` from NORMAL
all post the message; the hint panel lists the glossary row on both surfaces; the
message carries the term under the cursor when there is one and `None` otherwise; the
app handler opens the panel with that seed; dismissing restores focus and vim mode; no
other `g` continuation regressed.

## Phase 8 — Help, docs, and visual snapshots

- **Help modal.** Add the panel's keys to the ACE help modal, including the `gG` /
  `<ctrl+g>G` entry point. Respect the 57-character box width and 32-character
  description limits the ACE guidelines require.
- **Guide.** Document the panel in `docs/ace.md`: the two navigation axes, the trail,
  project cycling and its ring rule, and add/delete. Cross-link the CLI commands.
- **Footer.** Audit against the ACE footer convention — the footer shows _conditional_
  keymaps only. `d` and the relation keys are conditional (a term must be selected;
  chips must exist); `p`/`P` is conditional on a ring larger than one. Global keys
  belong in help only.
- **Onboarding and empty-state copy.** Final pass so the no-glossary invitation reads
  well and names the project by its display name.
- **PNG snapshots.** Add goldens under `tests/ace/tui/visual/snapshots/png/` for the
  panel in light and dark: a populated project with a trail and both chip rows, and the
  no-glossary empty state. Follow the existing visual fixtures' color and font pinning.
- **Cleanup.** Remove every `--epic-symbol` entry this epic added to the `Justfile` and
  confirm Symvision passes without them. Confirm `sase glossary --help` reads well and
  lists subcommands alphabetically.
- **Verification.** Run `just check-full` through a monitor and land on green.

## Verification

Each phase runs `just install` then `just check` before handing off. The epic's combined
tree runs `just check-full` through `/sase_monitor`, never inline. Phase 8 additionally
runs `just test-visual`.

Manual acceptance, end to end:

1. `sase glossary add "Test Term" "A test term that references Agent Hood." -a tt` —
   verify the sorted insert, the surviving comments, and the regenerated `AGENTS.md`
   block.
2. Open a prompt, type nothing, press `gG` — the panel opens on the first term.
3. `j`/`k` through terms; `/ agent` filters; `.` extends into definitions; `<esc>`
   closes the filter.
4. Select `Test Term`; `<tab>` focuses `Agent Hood` under SEE ALSO; `l` travels; the
   trail shows `Test Term › Agent Hood`; REFERENCED BY lists `Test Term`; `1` travels
   back down into it; `h` unwinds.
5. `p` cycles to another project; only its terms show; `P` returns; the trail is
   cleared.
6. `a` adds a term with a colliding alias — the error appears under the field and
   nothing is written; fix it and submit; the new term is selected and a commit is
   offered.
7. `d` on `Test Term` — the confirmation lists its inbound references; confirm; the
   neighbor row is selected and the toast carries the restore command.
8. `<esc>` — the prompt regains focus in the mode it had, cursor unmoved.
9. Type `Agent Hood` in a prompt, put the cursor inside it, press `<ctrl+g>G` — the
   panel opens on `Agent Hood`.

## Risks

- **The generic keymap-loader refactor touches two working scopes.** It is worth doing —
  a third copy of that 60-line body is how drift starts — but it must be
  behavior-preserving down to warning text. Land it with the existing scopes' tests
  green before the new scope is added.
- **Sorted YAML insertion may defeat the surgical writer** and produce a wider diff than
  a scalar edit does. Accepted, with the round-trip fallback; the tests pin that
  unrelated keys and comments survive.
- **Loading every enabled project's glossary is real I/O.** Contained by building the
  ring once off-thread, loading snapshots lazily per project, and caching by mtime. The
  regression to watch is a ring build creeping onto a keypress path.
- **Two caches now describe the same catalogs** — the panel's and the prompt's. Phase 6
  must invalidate both from one place; a stale prompt highlight after an add is the
  failure mode.
- **Automatic `sase memory init` from the CLI rewrites files.** It is idempotent and
  already runs as a post-commit hook, it is skippable with `-I`, and its failure is a
  warning rather than a rollback — the config write has already succeeded and must not
  be reverted.

## Non-goals

Recorded so a phase worker does not quietly absorb them:

- **Editing a term in place** (`sase glossary edit`, an in-panel editor). `o` opens the
  entry's exact line in `$EDITOR`, which covers the need. A dedicated edit flow is a
  reasonable follow-up bead.
- **Inserting the selected term into the draft prompt** from the panel. Natural, and
  tempting given where the panel opens from, but not requested; it also wants `<enter>`,
  which travel owns.
- **Home-level or cross-project glossaries.** The glossary is project-scoped today and
  stays that way.
- **Searching definitions across all projects at once.** `p`/`P` scopes to one project
  by design, per the prompt.
- **Changing the audited `sase glossary read` log.** Panel browsing is a human reading
  their own glossary, not an attributable agent read, and does not write audit events.
