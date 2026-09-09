---
tier: epic
title: Glossary migration to a core web
goal: "The `sase` and `bob-cli` glossaries stop living in `memory.glossary` and become
  file-backed core memory webs: `sase/memory/glossary.md` is a user-owned descriptor
  with a managed roster region, each term is a strand file under
  `sase/memory/glossary/`, the Rust glossary source wire addresses strand files so
  editor go-to-definition lands on a real Markdown note, config and files can never both
  be live, and `sase glossary *` survives one release as a deprecating alias over `sase
  memory`.

  "
phases:
  - id: wire
    title: File-backed glossary source wire
    depends_on: []
    size: medium
    description:
      "wire: generalize sase-core's GlossarySourceWire from config_path/config_key_path
      to source_path plus an optional key_path with keyword_range/body_range, bump
      GLOSSARY_WIRE_SCHEMA_VERSION to 2 keeping v1 keys accepted on read, and update the
      Python GlossarySource adapter and its readers to emit new names and accept both."
  - id: roster
    title: Inline roster parity with the generated glossary note
    depends_on: []
    size: small
    description:
      "roster: make the `roster: inline` managed region reproduce today's generated
      glossary roster byte for byte — Rust-derived display aliases instead of configured
      aliases, Markdown escaping, and wrapping at the configured print width."
  - id: source
    title: Strand-backed glossary catalog and fail-closed dual truth
    depends_on:
      - wire
    size: medium
    description:
      "source: build glossary catalog entries from strand files with per-strand source
      ranges, make editor_glossary_catalog_for_project and load_project_glossary_terms
      prefer the web and fail closed when config and web are both present, and delete
      the generated-glossary marker, collision blocker, and retirement path."
  - id: migrate
    title: The sase memory web migrate command
    depends_on:
      - roster
      - source
    size: medium
    description:
      "migrate: add `sase memory web migrate glossary [-n] [-p REF]`, which writes one
      strand per configured term, removes the memory.glossary block with a
      source-preserving YAML edit, and rewrites sase/memory/glossary.md as a user-owned
      web descriptor."
  - id: compat
    title: sase glossary as a deprecating alias
    depends_on:
      - source
    size: medium
    description:
      "compat: give `sase glossary read|show|all|list|log` a one-line deprecation notice
      naming its `sase memory` equivalent and delegate to it when the project has a
      glossary web, and make `sase glossary add|del` write and delete strand files
      instead of config entries when the web exists."
  - id: trees
    title: Migrate the sase and bob-cli trees
    depends_on:
      - compat
      - migrate
    size: medium
    description:
      "trees: run the migration for the sase project and the bob-cli project, regenerate
      memory in both, and prove the roster is byte-identical, closures match, and `sase
      memory init --check` is clean for sase, bob-cli, and home."
proposed_by: bbugyi200.athena.sase-sq.7
parent_bead: sase-sq.7
status: done
bead_id: sase-sq.7.1
create_time: 2026-09-09 19:50:38
---

- **PROMPT:**
  [prompts/202608/glossary_memory_web.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_memory_web.md)
- **PARENT:** [202608/memory_webs.md](memory_webs.md)
- **BEAD:**
  [sase-sq.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sq/sase-sq.7.1.md)

# Plan: Glossary migration to a core web

This is the child plan for phase `glossary` (`sase-sq.7`) of the epic **Memory webs and
strands** (`plan:202608/memory_webs.md`, bead `sase-sq`). Read the epic plan's `Design`
section before starting any phase here; this plan refines its `glossary` phase and does
not restate its rationale.

## Goal

`memory.glossary` stops being a source of glossary truth for the `sase` and `bob-cli`
projects. After this plan:

- `sase/memory/glossary.md` is a **user-owned core web descriptor** (`type: core`,
  `web: true`, `roster: inline`, `roster_label: GLOSSARY TERMS`, `strand_noun: term`,
  `closure: mentions`) whose managed `<!-- sase:strands -->` region renders the same
  roster line the generated note renders today, byte for byte.
- Every term is a strand file at `sase/memory/glossary/<slug>.md`.
- `memory.glossary` is gone from `sase/sase.yml` in both projects.
- Editor go-to-definition lands on the strand file rather than a YAML scalar.
- A project that somehow has both a `glossary` web and a `memory.glossary` block is a
  **fail-closed blocker**, never a merge and never a dual write.
- `sase glossary <sub>` still works, prints a deprecation notice, and reads the same
  strand-backed catalog.

## What already exists

Phases `tiers`, `substrate`, `cli`, `ace`, and `decisions` of the parent epic have
landed, so the following are present and must be **reused, not rebuilt**:

| Thing                                                          | Location                                                                |
| -------------------------------------------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------- |
| Web/strand models, frontmatter parsing, discovery, scope merge | `src/sase/memory/web/`                                                  |
| Managed `<!-- sase:strands -->` roster region rendering        | `src/sase/memory/web/roster.py`                                         |
| Eleven-rule fail-closed validator                              | `src/sase/memory/web/validation.py`                                     |
| Strand lookup (slug → keyword → alias → unique prefix)         | `src/sase/memory/web/lookup.py`                                         |
| Mention closure over strands, reusing the Rust matcher         | `src/sase/memory/web/closure.py`                                        |
| `sase memory read                                              | show <web>`/`<web>:<keyword>` selector batching                         | `src/sase/memory/selector.py`, `cli_read.py`, `cli_show.py`    |
| `sase memory web list                                          | show`                                                                   | `src/sase/memory/web/cli.py`, `src/sase/main/parser_memory.py` |
| `sase memory log --include glossary`                           | `src/sase/memory/cli_log.py`                                            |
| Web roster rendering wired into init planning                  | `_memory_web_root_plan` in `src/sase/main/init_memory/root_planning.py` |
| A real working web to copy (`decisions`, `roster: list`)       | `sase/memory/decisions.md` + `sase/memory/decisions/`                   |

The `memory_webs` feature flag was **removed** in the `decisions` phase. Nothing in this
plan is flagged; safety comes from the tree (see "Fail-closed dual truth").

## Design

### Rust: a source wire that can address a file

`crates/sase_core/src/glossary.rs` in the `sase-core` repo (open it with `/sase_repo`;
never edit it through a hand-rolled clone) currently declares:

```rust
pub const GLOSSARY_WIRE_SCHEMA_VERSION: u32 = 1;

pub struct GlossarySourceWire {
    pub config_path: Option<String>,
    pub config_key_path: Vec<String>,
    pub term_range: Option<EditorRange>,
    pub definition_range: Option<EditorRange>,
    pub aliases_range: Option<EditorRange>,
}
```

Every field name assumes the source is a YAML config node. Generalize to a source path
plus an _optional_ key path, at schema version 2:

```rust
pub const GLOSSARY_WIRE_SCHEMA_VERSION: u32 = 2;

pub struct GlossarySourceWire {
    /// File that declares this entry: a project config for config-backed
    /// entries, a memory-web strand note for file-backed ones.
    #[serde(default, alias = "config_path", skip_serializing_if = "Option::is_none")]
    pub source_path: Option<String>,
    /// Config key path. Empty for file-backed strand sources.
    #[serde(default, alias = "config_key_path")]
    pub key_path: Vec<String>,
    #[serde(default, alias = "term_range", skip_serializing_if = "Option::is_none")]
    pub keyword_range: Option<EditorRange>,
    #[serde(default, alias = "definition_range", skip_serializing_if = "Option::is_none")]
    pub body_range: Option<EditorRange>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub aliases_range: Option<EditorRange>,
}
```

The `serde(alias = ...)` attributes are load-bearing, not politeness: the wire crosses
into Python through `serde_json::from_value` in
`crates/sase_core_py/src/lib.rs::glossary_entries_from_pylist`, so a v1-shaped dict from
an older caller must keep deserializing. `entry_path` (`glossary.rs`) reads
`source.key_path` instead of `source.config_key_path`; its existing fallback to
`glossary.<term>` is what file-backed strands get, and that is correct — the diagnostic
path is relative to the entry, and Python prefixes the strand's real path.

`GlossarySourceWire` has no `deny_unknown_fields` and must not gain one. The wire is a
passthrough (`entry.source.clone()` in `catalog_from_entries`), so unknown keys are how
callers such as `src/sase/memory/web/closure.py` attach their own bookkeeping today.

**Do not rename the Rust `glossary` module.** The epic plan puts that explicitly out of
scope.

### The published-floor constraint

`pyproject.toml` declares `sase-core-rs>=0.31.12,<0.32.0`. Dev installs and the main CI
job build `sase_core_rs` from the `sase-core` checkout (`just install`, and the
`sase-org/sase-core` checkout step in `.github/workflows/ci.yml`), so the new wire is
live in both. But CI also runs a `release-core-floor-smoke` job that installs the exact
_published_ floor and then runs `tools/check_sase_core_rs_bindings`,
`tools/validate_sase_core_rs`, the `smoke_sase_core_rs_*` probes, and every test marked
`contract` from `tests/contract_manifest.txt`.

Two consequences, both mandatory:

1. **No new or changed test that asserts the v2 field names may be marked `contract`**,
   and none may be added to `tests/contract_manifest.txt`. Under the published floor
   those names do not exist yet.
2. **Python must read `entry.source` tolerantly.** Rust echoes the source back through
   whatever version is installed, so under the old floor a `source_path` key sent from
   Python is silently dropped on the way back. Every Python reader of `entry.source`
   must accept `source_path` or `config_path` and `body_range` or `definition_range`,
   and degrade to `None` rather than raising. Put that in exactly one helper.

No floor bump belongs in this plan — the release has not happened yet. Record a
`PROPOSED FOLLOW-UP:` on `sase-sq.7` to ratchet the floor with
`tools/ratchet_core_window` once the `sase-core` release carrying the v2 wire is
published, exactly as phase `tiers` did for its own wire change.

### Roster byte-identity

Today `sase/memory/glossary.md` is generated from
`src/sase/main/init_memory/templates/memory-sase-glossary.template.md` by
`render_generated_glossary_memory_body` (`root_rendering_notes.py:198`). That path:

1. takes `(term, display_aliases)` pairs from `build_glossary_catalog` — **display**
   aliases, which drop configured aliases the Rust matcher can derive as plurals;
2. renders each entry with `md_escape` as `Term (alias, alias)`;
3. joins with `"; "`;
4. runs the result through `format_generated_memory_markdown`, which wraps at
   `markdown_print_width()`.

That is why the committed roster reads `Proc (background task)` even though the
configured aliases are `[procs, background task, background tasks]`.

`render_strand_roster`'s `inline` branch (`src/sase/memory/web/roster.py:39`) does none
of those three things: it uses raw `strand.aliases`, does not escape, and does not wrap.
A 39-term inline roster rendered that way is one ~900-character line, which
`just fmt-md-check` (prettier) would rewrap on disk, which `sase memory init --check`
would then report as drift forever.

So the `inline` branch must:

- build a catalog from the web's strands (same shape as `lookup.py::_effective_aliases`)
  and render `entry.display_aliases`, not `strand.aliases`;
- apply `md_escape` to the keyword and each alias;
- wrap the finished `**LABEL:** …` line with
  `wrap_markdown(line, width=markdown_print_width())`, matching what the `list` branch
  already does per bullet.

Apply escaping and display-alias derivation to the **`inline` branch only**. The `list`
branch renders ``(`slug`)`` inside backticks, where escaping would corrupt the slug, and
`decisions.md` is already committed against the current list rendering.

Strand order is `_ordered_strands` (normalized keyword, then slug), which is the same
alphabetical order the sorted `memory.glossary` keys produce. The `trees` phase proves
that empirically with a `git diff`.

### Strand-backed glossary catalog

Add `src/sase/memory/web/catalog.py` with the single conversion every consumer shares:

- `memory_web_glossary_entries(web) -> tuple[GlossaryInputEntry, ...]` — one
  `GlossaryInputEntry(term=strand.keyword, definition=strand.body, aliases=strand.aliases, source=...)`
  per strand, in `_ordered_strands` order, where `source` is a `GlossarySource` carrying
  `source_path=str(strand.path)`, an empty `key_path`, `keyword_range` covering the
  strand's `keyword:` frontmatter value (or the strand's first body line when `keyword:`
  is absent and the keyword is slug-derived), and `body_range` covering the strand body.
- `glossary_source_from_wire(payload) -> …` — the one tolerant reader described under
  "The published-floor constraint".
- `memory_web_source_signature(web) -> …` — a filesystem signature over the descriptor
  and every strand file: `path` = the strand directory, `mtime_ns` = the maximum
  `st_mtime_ns` across descriptor plus strands, `size` = their summed sizes. This
  replaces the single-file `sase.yml` stat for strand-backed catalogs and is what keeps
  the ACE prompt-highlight cache and the completion invalidation source honest.
- `find_memory_web(root, slug) -> MemoryWeb | None` — descriptor lookup by slug over
  `discover_memory_webs(root)`.

Computing `keyword_range` / `body_range` needs the strand's body offset, which
`MemoryStrand` does not carry today (`MemoryWeb` has `body_start`; `MemoryStrand` does
not). Add `body_start: int` to `MemoryStrand` in `src/sase/memory/web/models.py` and
populate it in `parse_memory_strand`; both ranges are then derived from `raw_text`
without re-reading the file. Ranges are LSP-shaped zero-based
`{"start": {"line", "character"}, "end": {…}}` dicts, the same shape
`src/sase/xprompt/_glossary_catalog_ranges.py` already produces.

`editor_glossary_catalog_for_project` (`src/sase/xprompt/glossary_catalog.py:99`) then
resolves its entries as:

1. discover webs under the project workspace root; look for a `glossary` web;
2. if a `glossary` web **and** a declared `memory.glossary` are both present → return an
   `EditorGlossaryCatalogResult` with a single diagnostic naming
   `sase memory web migrate glossary`, and no catalog;
3. if only the web is present → entries from `memory_web_glossary_entries`,
   `config_path` set to the strand directory, `config_signature` from
   `memory_web_source_signature`;
4. otherwise → today's config path, unchanged.

Keep the `EditorGlossaryCatalog` field names `config_path` and `config_signature`. They
are read by `src/sase/ace/tui/modals/glossary_panel_actions.py`,
`src/sase/ace/tui/widgets/prompt_panel/_agent_xprompt_highlighting.py`, and several
tests; renaming them is `retire`'s work, not this plan's.

`load_project_glossary_terms` (`src/sase/main/init_memory/glossary.py:38`) applies the
same three-way rule from the memory root, returning `(None, ())` when the web owns the
glossary — the web's own roster region already renders in `_memory_web_root_plan`, so
the generated note must not also be produced.

Delete, per the epic plan, now that the managed region makes them obsolete:

- `_glossary_collision_blocker` (`root_planning.py:158`)
- `_retired_glossary_note_paths` (`root_planning.py:112`)
- `is_generated_glossary_memory_content`, `GENERATED_GLOSSARY_MARKER_KEY`, and
  `GENERATED_GLOSSARY_MARKER_VALUE` (`init_memory/glossary.py`), and the
  `sase_generated: glossary` frontmatter key `generated_glossary_memory_content`
  (`root_rendering_notes.py:231`) writes.

`render_generated_glossary_memory_body`, its template, and the config reader stay:
projects that have not migrated must keep generating their note until `retire` removes
the config glossary entirely.

### Fail-closed dual truth

There is exactly one rule and it is stated once, in one predicate, used by both the
editor catalog and init planning:

> A project's glossary comes from strand files if a `glossary` web exists, and from
> `memory.glossary` if it does not. If both exist, every command that would read either
> fails with a blocker naming `sase memory web migrate glossary`. Never merge, never
> dual-write, never prefer silently.

This must also surface from `sase doctor`'s webs check and from `sase memory init`, both
of which already consume `validate_memory_webs`; the dual-source blocker belongs
alongside the existing web blockers so `--check` and `--diff` report it like any other.

### The migration command

```bash
sase memory web migrate <WEB> [-n|--dry-run] [-p|--project REF]
```

`WEB` is positional because the command cannot run without it (per `cli_rules.md`:
options are never required). In v1 the only accepted value is `glossary`; any other
value exits non-zero explaining that only the config glossary can be migrated. Both long
options carry a short alias, and the subcommand and option lists stay sorted. The group
already defaults to `list` through `_default_list_subcommands()`; do not touch that
wiring.

Steps, in order, aborting on the first failure and writing nothing until every check
passes:

1. Resolve the root: `resolve_memory_cli_project(-p)` when given, else the CWD root.
   Running with no `-p` from inside a workspace checkout migrates _that_ checkout, which
   is what the `trees` phase depends on.
2. Read `memory.glossary` from the project config (`resolve_project_config_read_path` +
   `resolve_glossary_config`). Absent → exit non-zero, "nothing to migrate".
3. Refuse if `sase/memory/glossary/` already contains strands, or if
   `sase/memory/glossary.md` already declares `web: true` — that is the dual-truth state
   and the migration is one-shot.
4. Shape and validate the entries with the existing `parse_glossary_entries` +
   `validate_glossary_entries` path. Any diagnostic aborts.
5. Compute each slug as `normalize_glossary_reference(term)` with spaces replaced by
   hyphens; abort on a slug collision rather than overwriting.
6. Write `sase/memory/glossary/<slug>.md`: frontmatter `keyword:` = the configured term,
   `aliases:` = the configured aliases (omitted when empty), no `type:`, no `parent:`,
   no `summary:` (the web is `roster: inline`); body = the configured definition.
7. Rewrite `sase/memory/glossary.md` as the descriptor: the frontmatter listed under
   "Goal", today's preamble prose with `sase glossary read <term>` replaced by
   `sase memory read glossary:<term>`, then the managed roster region rendered by
   `render_web_descriptor_with_roster`.
8. Remove the `memory.glossary` node from the project config with the source-preserving
   round-trip YAML machinery in `src/sase/glossary/mutation.py` (`_load_root_mapping`,
   `_key_location`, `_block_end`, `_splice_lines`, `_write_config_atomically` are the
   pieces to reuse). Remove the `memory:` mapping too if and only if it becomes empty.
9. Print what changed: strand count, each written path, the config path, and the
   follow-up (`run \`sase memory init\``). `-n/--dry-run` prints exactly that report and
   writes nothing.

Keep the handler in a new `src/sase/memory/web/cli_migrate.py` and the mechanics in
`src/sase/memory/web/migrate.py` so `cli.py` stays under the 700-line `toobig` cap.

### Compatibility: `sase glossary` for one more release

Once the catalog source prefers strands, every read-side `sase glossary` subcommand
already returns strand-backed data without any change, because they all resolve through
`editor_glossary_catalog_for_project`. What this plan adds is the deprecation notice and
true delegation where delegation is honest:

| Command                    | Notice names                         | Behavior                                                                  |
| -------------------------- | ------------------------------------ | ------------------------------------------------------------------------- |
| `sase glossary read TERM…` | `sase memory read glossary:<term>`   | delegate when the project has a glossary web; else run the legacy handler |
| `sase glossary show TERM…` | `sase memory show glossary:<term>`   | same                                                                      |
| `sase glossary all`        | `sase memory show glossary`          | same                                                                      |
| `sase glossary list [PAT]` | `sase memory web show glossary`      | same; `-d/--definitions` maps to `-b/--bodies`                            |
| `sase glossary log`        | `sase memory log --include glossary` | same                                                                      |
| `sase glossary add`        | —                                    | write a strand file when the web exists; else today's config insert       |
| `sase glossary del`        | —                                    | delete the strand file when the web exists; else today's config delete    |

The "else run the legacy handler" branch is not hedging: `sase` and `bob-cli` migrate in
the `trees` phase, but any other project on this machine still has a config glossary,
and a deprecation alias that breaks unmigrated projects is worse than no alias. The
branch is one predicate (`find_memory_web(root, "glossary") is not None`) and it dies
with the whole command group in `retire`.

Delegation is mechanical: build an `argparse.Namespace` with
`selectors=[f"glossary:{term}" for term in args.term]` plus `depth`, `format`,
`project`, and `reason`, and call `handle_memory_read_command` / the `show`, `web show`,
or `log` handler. `sase glossary read`'s `-f/--format {json,markdown,rich}` values are
identical to `MemoryShowFormat`, so no mapping is needed. Delegated reads write the
**memory** read log, not `glossary_reads.jsonl`; that is intended, and
`sase memory log --include glossary` is what keeps the historical JSONL readable.

Print the notice to **stderr**, one line, so piped JSON output stays parseable.

For `add`/`del` against a web, keep returning a `GlossaryMutationOutcome` with
`config_path` set to the strand file that was written or removed, so
`src/sase/ace/tui/modals/glossary_panel_actions.py` and `cli_write.py` need no changes.
`restore_command` for a strand delete is the equivalent `sase glossary add` invocation.

### Non-goals

Everything below belongs to the `retire` phase (`sase-sq.8`) and must **not** be done
here:

- deleting `src/sase/glossary/`, `src/sase/glossary_config.py`,
  `src/sase/xprompt/_glossary_catalog_config.py`, `_glossary_catalog_ranges.py`,
  `src/sase/main/parser_glossary.py`, `src/sase/main/glossary_handler.py`, or
  `src/sase/main/init_memory/glossary.py`;
- dropping the `glossary`/`glossaryEntry` blocks from `src/sase/config/sase.schema.json`
  or the `glossary` key from `config/layers.py`;
- folding `GlossaryPane` into `MemoryPane` or moving the `glossary` keymap scope;
- finishing `docs/memory.md`, `docs/cli.md`, `docs/editor.md`, `docs/completion.md`,
  `docs/ace.md`, regenerating `sase/memory/README.md`, or redeploying the
  `/sase_memory_read` skill.

Also out of scope: renaming the Rust `glossary` module, bumping the `sase-core-rs`
floor, relocating bob-cli's four terms to home scope, and any new web beyond `glossary`.

## Phases

### wire: File-backed glossary source wire

**`sase-core` (open with `/sase_repo`, commit there, declare it in `/sase_final`).**
Apply the `GlossarySourceWire` change above in `crates/sase_core/src/glossary.rs`:
renamed fields with v1 `serde(alias = …)` compatibility, `GLOSSARY_WIRE_SCHEMA_VERSION`
2, `entry_path` reading `key_path`. Update the module doc comment, which currently says
"Python owns config discovery and source-preserving YAML parsing" — it now owns config
_and strand file_ discovery. Re-export names in `crates/sase_core/src/lib.rs` are
unchanged (`GlossarySourceWire` keeps its type name). Add Rust unit tests: a v1-shaped
JSON payload with `config_path`/`config_key_path`/`term_range`/`definition_range`
deserializes into the new fields; a v2 payload round-trips; `catalog_from_entries`
reports `schema_version: 2`; `entry_path` falls back to `glossary.<term>` when
`key_path` is empty. Run the crate's own gates from the checkout.

**`sase` adapter.** In `src/sase/core/glossary_facade.py`, rename `GlossarySource`'s
fields to `source_path`, `key_path`, `keyword_range`, `body_range` (keeping
`aliases_range`), and emit the new names from `to_wire`. Update
`src/sase/xprompt/_glossary_catalog_config.py::_glossary_source` to construct the new
names. Update `src/sase/memory/web/validation.py:117` (currently
`source={"config_path": str(strand.path)}`) and `src/sase/memory/web/closure.py`
(currently `{"slug": …, "path": …}`) to use `source_path`.

Add the tolerant reader in `src/sase/memory/web/catalog.py` (create the module in this
phase; the `source` phase fills in the rest) and route
`src/sase/ace/tui/modals/glossary_preview_render.py:250`'s `definition_range` lookup
through it so the preview works against both the dev build and the published floor.

Tests: `tests/test_core_glossary_facade.py` and `tests/xprompt/test_glossary_catalog.py`
move to the new names; add a case asserting the tolerant reader accepts a v1-shaped
`source` mapping. **Mark none of these `contract`.**

Note in the phase's commit body that the `sase-core` change must land and release before
sase's floor can be ratcheted, and leave a `PROPOSED FOLLOW-UP:` note on `sase-sq.7` for
the ratchet.

### roster: Inline roster parity with the generated glossary note

Change only `render_strand_roster`'s `inline` branch in `src/sase/memory/web/roster.py`,
exactly as specified under "Roster byte-identity": Rust-derived `display_aliases`,
`md_escape` on keyword and aliases, and
`wrap_markdown(..., width=markdown_print_width())` on the finished line. Factor the
strand→catalog conversion so `lookup.py::_effective_aliases` and this share one helper
rather than each calling `build_glossary_catalog` their own way.

Tests: a fixture web whose strands reproduce this repo's own glossary terms `Proc`
(aliases `procs`, `background task`, `background tasks`) and `Agent Hood` (aliases
`hood`, `agent neighborhood`) renders `Proc (background task)` and
`Agent Hood (hood, agent neighborhood)`; a wide fixture wraps at the configured print
width and every produced line is `<= markdown_print_width()`; the `list` branch and the
committed `decisions` roster are unchanged.

### source: Strand-backed glossary catalog and fail-closed dual truth

Implement `src/sase/memory/web/catalog.py` in full (`memory_web_glossary_entries`,
`memory_web_source_signature`, `find_memory_web`), add `body_start` to `MemoryStrand`
and populate it in `parse_memory_strand`, and rewire
`editor_glossary_catalog_for_project` and `load_project_glossary_terms` to the three-way
rule. Surface the dual-source blocker through `_memory_web_root_plan` and the
`sase doctor` webs check. Delete `_glossary_collision_blocker`,
`_retired_glossary_note_paths`, and the `sase_generated: glossary` marker and its
helper.

Point `src/sase/ace/tui/glossary_panel_catalog.py`'s cache-invalidation stat at
`memory_web_source_signature` when the project has a glossary web, so the ACE glossary
pane and prompt highlighting notice a strand edit. Read `sase/memory/tui_perf.md` with
`/sase_memory_read` before touching that file: the strand walk must not run on the event
loop.

Tests: a fixture project with only strands produces a catalog whose entries carry
`source_path` pointing at each strand file and a `keyword_range` on the `keyword:` line;
a fixture with only config is unchanged; a fixture with both yields exactly one
diagnostic naming `sase memory web migrate glossary` and no catalog, from both
`editor_glossary_catalog_for_project` and `sase memory init --check`; a strand edit
changes `memory_web_source_signature`; adding and removing a strand changes it too.

### migrate: The sase memory web migrate command

Add `src/sase/memory/web/migrate.py` and `src/sase/memory/web/cli_migrate.py`, register
`migrate` in `_register_memory_web_parser` (`src/sase/main/parser_memory.py`) keeping
subcommands sorted (`list`, `migrate`, `show`), and dispatch it from the memory handler.
Follow the nine steps under "The migration command" exactly, including the one-shot
refusal and the atomic "check everything, then write" ordering.

Tests: a fixture project with a config glossary migrates to N strand files plus a
descriptor and an empty-of-glossary config; `-n/--dry-run` reports the same plan and
leaves the tree byte-identical; migrating twice fails on the second run; a config whose
terms collide on slug fails before writing anything; a term with aliases round-trips its
aliases into strand frontmatter; the resulting descriptor passes `validate_memory_webs`
with no blockers; the resulting roster equals what
`render_generated_glossary_memory_body` produced for the same terms.

That last assertion is the whole point of the phase — write it first.

### compat: sase glossary as a deprecating alias

Add the deprecation notices and web-aware delegation for `read|show|all|list|log`, and
strand-file writes for `add|del`, per the compatibility table. Keep
`src/sase/main/parser_glossary.py`'s help text truthful: the group description still
says "configured under memory.glossary in sase/sase.yml" and must now say the glossary
lives in the `glossary` memory web, with `memory.glossary` accepted until the next
release.

Tests: with a web present, each read-side subcommand prints its one-line notice on
stderr and produces the delegated command's output on stdout; with no web present, each
prints the notice and still produces today's config-backed output; `sase glossary add`
against a web creates `sase/memory/glossary/<slug>.md` and leaves the config untouched;
`sase glossary del -n` against a web previews without deleting; JSON output stays
parseable with the notice on stderr.

### trees: Migrate the sase and bob-cli trees

Nothing new is built here; this phase proves the previous five.

1. In this repo's workspace checkout, run `sase memory web migrate glossary` with the
   workspace venv (`.venv/bin/sase`), then `sase memory init`.
2. `git diff sase/memory/glossary.md` must show **only** the frontmatter change and the
   preamble's `sase glossary read <term>` → `sase memory read glossary:<term>` edit. The
   `**GLOSSARY TERMS:** …` block must be byte-identical, including its line breaks. If
   it is not, fix `roster`, do not adjust the expected output.
3. `sase/sase.yml` loses its `memory.glossary` block, and `sase/memory/glossary/` gains
   one file per term (39 at the time of writing — take the live count from the config,
   do not hard-code it).
4. Verify parity: `sase glossary read Stitch -r "<why>"` and
   `sase memory read glossary:stitch -r "<why>"` produce the same closure and the same
   term bodies. `sase memory web show glossary` lists every term.
   `sase memory read glossary` prints every strand.
5. Open the `bob-cli` project with `/sase_repo` and repeat steps 1–3 there for its four
   terms (`Pomodoro`, `Schedule Log`, `Task Link`, `Work Log`), running this workspace's
   `.venv/bin/sase` with the bob-cli checkout as the working directory so the migration
   uses the code being landed rather than the released `sase` on `PATH`. Its
   `sase/memory/glossary.md` roster must likewise be byte-identical.
6. `sase memory init --check` must be clean for this repo, for bob-cli, and for home.
7. `just check-full` through `/sase_monitor`, plus `just test-visual` if any ACE
   rendering changed.

The bob-cli tree becomes a **repository obligation** for whoever runs this phase: it
must appear in that turn's `/sase_final` declaration alongside this repo and
`sase-core`.

## Risks

| Risk                                                                      | Severity | Mitigation                                                                                                                    |
| ------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| The migrated roster differs from the generated one and churns core memory | high     | `roster` lands before `migrate`; `migrate` asserts equality against `render_generated_glossary_memory_body`; `trees` diffs it |
| An unwrapped inline roster fights prettier and `init --check` forever     | high     | Wrap in the renderer at `markdown_print_width()`; assert every rendered line fits                                             |
| A v2-only test runs against the published core floor and fails CI         | high     | No new wire assertion is marked `contract` or added to `tests/contract_manifest.txt`; Python reads `source` tolerantly        |
| Config and strands both live, so a project reads the wrong glossary       | high     | One predicate, fail-closed, used by the editor catalog, init planning, and doctor; the migration refuses to run twice         |
| bob-cli is migrated with a `sase` that never lands                        | medium   | `trees` runs last, after `just check` is green here; the bob-cli change is declared in the same turn as this repo's           |
| Deleting the generated-glossary marker orphans an unmigrated project      | medium   | The config render path stays; only the collision/retire helpers go, and any web-plus-config project is a blocker, not a merge |
| The ACE glossary pane caches a stale strand-backed catalog                | medium   | `memory_web_source_signature` covers descriptor and strands by max mtime and summed size; a strand edit changes it            |
| `sase glossary` aliases break a project that never migrates               | medium   | Delegate only when the web exists; otherwise notice plus today's behavior                                                     |
| Strand slugs collide after normalization                                  | low      | The migration aborts before writing; `validate_memory_webs` catches it afterward                                              |
