---
tier: epic
status: done
title: sase glossary command and on-demand glossary context
goal: "Agents fetch glossary definitions on demand with `sase glossary read <term>`,
  which prints the term plus the transitive closure of terms its definition depends on
  and records an audited, visible read; the always-loaded glossary memory note is gone,
  replaced by one concise Tier 2 instruction block, so the glossary can grow without
  growing every agent's context.

  "
phases:
  - id: core
    title: Glossary resolution core and read-log foundation
    depends_on: []
    size: medium
    description:
      "core: add the shared term-reference lookup and recursive reference-closure
      resolver over the Rust glossary matcher, plus the glossary read-event model, JSONL
      store, and summaries; make the ACE preview modal's cross-references delegate to
      the shared resolver."
  - id: init
    title: Retire the generated glossary note for a Tier 2 instruction block
    depends_on: []
    size: medium
    description:
      "init: stop generating `sase/memory/glossary.md`, delete the previously generated
      note, and render a `**GLOSSARY TERMS:**` block into the AGENTS.md Tier 2 section
      whenever a project configures glossary entries."
  - id: cli
    title: sase glossary group with list and show
    depends_on:
      - core
    size: medium
    description:
      "cli: register the `sase glossary` command group with project selection, the
      filtered multi-format `list` subcommand, and the recursive `show` subcommand with
      its provenance-annotated rendering."
  - id: audit
    title: sase glossary read and log
    depends_on:
      - core
      - cli
    size: medium
    description:
      "audit: add the reason-requiring `read` wrapper that appends an audited event
      before printing show output, and the `log` dashboard that summarizes recorded
      reads by term, by agent, and by event."
  - id: panel
    title: GLOSSARY lane in the agent metadata panel
    depends_on:
      - audit
    size: medium
    description:
      "panel: add the per-agent glossary-read loader and render a GLOSSARY sub-section
      in the SASE CONTEXT section of the agent metadata panel, showing each read's
      terms, related-term count, and reason."
  - id: docs
    title: Documentation, completion spec, and end-to-end sweep
    depends_on:
      - core
      - init
      - cli
      - audit
      - panel
    size: small
    description:
      "docs: update the CLI, configuration, init, memory, and ACE documentation for the
      new command and the retired note, regenerate the completion spec snapshot, and run
      the exhaustive verification lane over the combined tree."
proposed_by: bbugyi200.athena.050
bead_id: sase-op
create_time: 2026-09-09 19:50:37
---

- **PROMPT:**
  [prompts/202608/glossary_command.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_command.md)
- **BEAD:**
  [sase-op](https://github.com/sase-org/sase--beads/blob/main/pages/sase-op/README.md)

# Plan: `sase glossary` and on-demand glossary context

## Problem

The project glossary lives in `memory.glossary` in `sase/sase.yml`. Today
`sase memory init` renders every entry into a generated `sase/memory/glossary.md` long
note, and that note's frontmatter description — an indexed bullet list of all 27 terms
and their aliases — is inlined into the Tier 2 section of `AGENTS.md` and every provider
instruction shim. Every agent therefore pays for the full term index on every launch,
and reading a single definition costs a whole-file `sase memory read glossary.md`, which
returns all 27 definitions.

That pricing model punishes exactly the behavior we want. Adding a glossary entry today
means adding tokens to every agent's context forever, so the glossary stays small when
it should grow.

## Outcome

- `sase glossary read "Agent Hood" -r "<why>"` prints that one definition plus only the
  terms it transitively depends on, and clearly shows why each extra term appeared.
- The always-loaded cost drops to a single concise `**GLOSSARY TERMS:**` block in Tier 2
  that names the terms and tells agents how to fetch one.
- Every glossary read is attributable: it is recorded with its reason, surfaced in
  `sase glossary log`, and rendered in the agent metadata panel next to `MEMORY`.

## Design decisions

### Where the closure logic lives

`sase/memory/rust_core_backend_boundary` requires shared backend and domain behavior to
live in the sibling Rust core repo. The glossary's genuine domain logic — alias
normalization, derived plurals, phrase-boundary matching — already lives there
(`crates/sase_core/src/glossary.rs`), reaches Python through
`sase.core.glossary_facade`, and this plan keeps using it unchanged: every reference
edge comes from `scan_glossary_spans()` on the Rust-compiled matcher.

The new code is the breadth-first walk over those spans plus the provenance a reader
needs ("this term appeared because Agent Hood's definition says _agents_"). That adds no
matching semantics of its own, it is a thin adapter over the binding, and the depth-1
case already lives in Python here today as `glossary_cross_references()` in
`src/sase/ace/tui/modals/glossary_preview_render.py`.

**Decision:** implement the resolver as one shared Python module in this repo and route
both the new CLI and the existing ACE preview modal through it, so the two can never
disagree. Keeping it here also keeps the feature in a single repo instead of serializing
it behind a `sase-core` release and a `sase-core-rs` version-window bump.

**Alternative if the user prefers strict boundary adherence:** add
`resolve_glossary_closure` to `crates/sase_core/src/glossary.rs`, expose it as a
`CompiledGlossaryCatalog.resolve(...)` pyo3 method, and have `sase.core.glossary_facade`
wrap it. The Python-side call sites in this plan are unchanged by that swap, so it can
be done later without reworking the CLI; file it as a follow-up task bead if a
non-Python frontend needs the closure.

### One `read` event per invocation

`sase memory read` records one event per file read. A glossary read has a requested set
and an expanded set, so the record keeps both: `terms` (what the agent asked for) and
`related_terms` (what the closure added). One event per invocation keeps the audit story
honest ("this agent ran this read, for this reason") and lets both the log dashboard and
the panel show the closure's shape.

### No feature flag

`sase/memory/sase_flags.md` requires a flag for user-reaching behavior that is not
ready: a disabled beta, an early landed path, or a deprecation whose old branch must
stay reachable. This is a complete cutover that is ready when it lands, and the removal
of the generated note is the intended user-visible result, not a fallback that must stay
reachable. No flag; no flag bead.

### Deviation from "options must not be required"

`sase/memory/cli_rules.md` says a value required for execution belongs in a positional.
`sase glossary read` deliberately keeps `-r/--reason` as a required option to match
`sase memory read <path> -r "<why>"` exactly, because both commands are audited reads
that agents invoke from instruction text and muscle memory. Both are the same shape or
neither should be; this plan chooses consistency and records the deviation here.

---

## Phase `core`: Glossary resolution core and read-log foundation

Create a new package `src/sase/glossary/` (this coexists with the existing
`src/sase/glossary_config.py` module).

### `src/sase/glossary/resolution.py`

Term-reference lookup and the recursive closure.

**Term-reference normalization.** Normalize a user- or agent-supplied reference by
casefolding and collapsing runs of `[-_\s]+` into a single space, so `Agent Hood`,
`agent hood`, `agent-hood`, and `agent_hood` all resolve. This is a CLI-side convenience
layer and is separate from the Rust matcher's own normalization.

**Lookup order** against a loaded `GlossaryCatalog`:

1. Exact normalized match on `GlossaryEntry.normalized_term`.
2. Exact normalized match on any `effective_aliases` entry (this includes the
   Rust-derived plurals), resolving to that entry's canonical term.
3. Unique normalized prefix match on a term or alias.
4. Otherwise raise a lookup error carrying up to five near-miss candidates (normalized
   substring matches on terms and aliases) so callers can print a "did you mean" list.

**Closure resolution.**
`resolve_glossary_closure(catalog, compiled, roots, depth=None)`:

- Seed a FIFO queue with the resolved root entries, in the order the caller requested
  them, at depth 0 and origin `requested`.
- Pop an entry, scan its definition with `scan_glossary_spans(compiled, definition)`,
  and walk the spans in byte order.
- Skip spans that resolve to the entry itself.
- For an unseen target, enqueue it at `depth + 1` with origin `related` and provenance
  `(referrer_term, matched_text)`; for an already-seen target, append the referrer to
  that node's `also_referenced_by` list instead of re-enqueuing.
- Stop descending past `depth` when a limit is given; `depth=0` yields only the roots.
  Mark the result `truncated` when the limit actually cut off unexplored references, so
  the renderer can say so.
- A `seen` set makes cycles terminate; a term that is both requested and reachable keeps
  origin `requested`.

Return a frozen `GlossaryClosure` with `nodes` in discovery order, each node carrying
`entry`, `depth`, `origin`, `referrer` (term plus matched phrase, `None` for roots),
`also_referenced_by`, and the node's own outgoing `spans` (kept so renderers can
highlight the linking phrases without rescanning), plus `roots`, `depth_limit`, and
`truncated`.

The walk must be deterministic: BFS order, spans in byte order, no set iteration in any
output path.

### `src/sase/glossary/read_log.py`

Mirror `src/sase/memory/read_log.py`, which is the reference implementation for locking,
schema-versioned rows, and malformed-row tolerance.

- `GlossaryReadEvent`: `schema_version`, `id`, `timestamp`, `project`, `cwd`,
  `agent_name`, `agent_source`, `artifacts_dir`, `reason`, `terms` (requested canonical
  terms), `related_terms` (expanded canonical terms in discovery order), `depth_limit`,
  `definition_bytes` (summed UTF-8 length of every printed definition, so the log can
  quantify what a read actually cost), and `source_path` (the glossary config path, for
  panel hint targets).
- `normalize_read_reason()` and `require_agent_identity()` reuse the shared helpers from
  `sase.agent.identity`; reject a blank reason and demand an agent identity exactly as
  memory reads do.
- `glossary_read_log_path(project)` returns
  `sase_projects_dir() / <resolved project> / "glossary_reads.jsonl"`, resolving aliases
  through `resolve_project_alias_ref` like the memory log does.
- `append_glossary_read_event()` appends one JSON row under an exclusive `fcntl` lock
  via `sase.memory.locks.locked_file`; `read_glossary_read_events()` reads under a
  shared lock and skips rows that are malformed or carry a different `schema_version`.
- `filter_glossary_read_events(events, term=None, agent_name=None)` where the term
  filter matches a normalized term in either `terms` or `related_terms`.
- `summarize_glossary_reads_by_term()` and `summarize_glossary_reads_by_agent()` return
  sorted summary tuples carrying counts, distinct-counter counts, and the latest read's
  timestamp, counterpart, and reason.

### ACE preview modal delegation

Rewrite `glossary_cross_references()` in
`src/sase/ace/tui/modals/glossary_preview_render.py` to call the shared resolver at
depth 1 and return the first nine related entries. Its current signature, the nine-entry
cap, and its `spans` fast-path stay intact so `GlossaryPreviewModal` and its existing
tests are unaffected.

### Tests

New unit tests for: alias, plural, prefix, and slug-form lookups; ambiguous and unknown
references with their candidate lists; closure over a diamond (a term reachable by two
paths appears once with both referrers recorded); a cycle; `depth=0`, a mid-range depth
with `truncated` set, and unlimited depth; multi-root ordering; and determinism across
repeated runs. Read-log tests cover append/read round-trips, concurrent appends under
the lock, malformed and wrong-schema row skipping, filters, and summaries. Existing
`tests/ace/tui/modals/test_glossary_preview_render.py` must keep passing unchanged.

---

## Phase `init`: Retire the generated glossary note for a Tier 2 instruction block

### Stop generating the note

In `src/sase/main/init_memory/glossary.py`, replace `load_project_glossary_memory()`
with `load_project_glossary_terms(config_path)` returning
`(ProjectGlossaryTerms | None, errors)`. `ProjectGlossaryTerms` holds an ordered tuple
of `(term, display_aliases)` pairs taken from the validated `GlossaryCatalog`. Keep the
existing config parsing, shape checking, and Rust validation paths exactly as they are —
only the rendering changes. Delete `_render_glossary_memory()`,
`_glossary_memory_description()`, and `GeneratedGlossaryMemory`.

Keep `is_generated_glossary_memory_content()`: it is now the migration hook.

In `src/sase/main/init_memory/root_planning.py` and `root_rendering.py`:

- Drop `generated_glossary` from the expected-files path so no glossary note is ever
  written, and remove `generated_glossary_memory_relative_path()` from the rendering
  path.
- Keep `_retired_glossary_note_paths()` and make it unconditional: on every init, if
  `sase/memory/glossary.md` exists and carries the `sase_generated: glossary` marker,
  delete it. This is what migrates existing checkouts.
- Delete `_glossary_collision_blocker()` and its tests. Nothing is written to that path
  any more, and an unmarked hand-authored `sase/memory/glossary.md` is intentionally
  left alone by the retirement check, so it simply remains an ordinary long note.

Thread `ProjectGlossaryTerms` from `_load_memory_inputs()` in
`src/sase/main/init_memory_handler.py` through `_plan_memory_root()` /
`_initialize_memory_root()` into the AMD sync instead of the retired note content. Only
the project root gets glossary terms; the home root never does.

### Render the Tier 2 block

In `src/sase/amd/_memory.py`, accept the project's glossary terms in
`plan_amd_memory_sync()` and pass them into `_render_managed_agents()`. When the term
list is non-empty, prepend the rendered block plus one blank line to the `tier2_entries`
string handed to `render_agents_template()`.

Prepending to `tier2_entries` rather than adding a new template placeholder is
deliberate: `render_markdown_template()` rejects both missing required variables and
unknown placeholders, so a new required variable would break every user-supplied
`AGENTS.template.md` override. The rendered output is identical to what a new
placeholder would produce.

The result lands exactly where requested — after the section's intro paragraph, one
blank line before the long-memory sub-sections:

```markdown
## Tier 2 (long-term) Memory

The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.

**GLOSSARY TERMS:** Run `sase glossary read <term> -r "<why>"` before relying on any of
these SASE terms; it prints that term's definition plus every term the definition
depends on. Terms (aliases follow in parentheses): Agent Clan; Agent Family; Agent Hood
(hood, agent neighborhood); Agent Instruction File (agents.md file); ...

### `sase/memory/cli_rules.md`

Read anytime new CLI subcommands or options are added.
```

Rendering rules:

- The block is one paragraph beginning with the literal `**GLOSSARY TERMS:** `.
- Entries are `;`-separated so the `,`-separated aliases inside parentheses stay
  unambiguous; a term with no display aliases has no parentheses.
- Terms appear in the catalog's canonical order with `md_escape()` applied, matching how
  the retired description rendered them.
- Emit nothing at all when the project configures no glossary entries.

`format_generated_memory_markdown()` already wraps this shape correctly and the result
is a formatting fixpoint, which the phase must assert in a test since `AGENTS.md` is
prettier-checked.

`_render_managed_agents()`'s existing structural assertions still hold: the block adds
no heading, so `parse_amd_agents_document()` skips it and `parsed_long_paths` is
unchanged. Add a regression test that proves this for a project that has glossary terms
and no Tier 2 notes at all, where the block becomes the entire `tier2_entries` value.

### Tests

Update `tests/main/test_init_memory_glossary.py`, `test_init_memory_formatting.py`, and
any AMD memory tests that assert the generated note. Cover: a marked
`sase/memory/glossary.md` is deleted on the next init; an unmarked one is preserved; the
block's presence, exact prefix, term list, and alias formatting; absence with no
configured entries; the home root never gets a block; `sase memory init --check` reports
drift for a stale block and is clean once refreshed; and the provider instruction shims
carry the block.

---

## Phase `cli`: `sase glossary` group with `list` and `show`

### Registration

- `src/sase/main/parser_glossary.py` with `register_glossary_parser()`.
- Register `"glossary": ("sase.main.parser_glossary", "register_glossary_parser")` in
  the lazy registry in `src/sase/main/parser.py`, and add `register_glossary_parser` to
  `COMMAND_REGISTRARS_BY_NAME` in `src/sase/main/parser_full_registrars.py`.
- `src/sase/main/glossary_handler.py` with `handle_glossary_command()` dispatching to
  lazily imported per-subcommand handlers, following `src/sase/main/memory_handler.py`.
- Dispatch `args.command == "glossary"` in `src/sase/main/entry.py`.

`sase glossary` with no subcommand delegates to `list` through the central
`_default_list_subcommands()` mechanism in `parser.py`; do not re-implement it. Say so
in the group description.

### Project selection

`-p/--project REF` selects the project by name, alias, or key; the default is the
project owning the current directory. Register it on the group parser **and** on every
subcommand, with `default=argparse.SUPPRESS` on the subcommand copies so a value given
before the subcommand is not clobbered by the subparser's default. Both
`sase glossary -p sase show Stitch` and `sase glossary show Stitch -p sase` then work;
assert both orders in tests.

Resolution and loading reuse `editor_glossary_catalog_for_project()` from
`src/sase/xprompt/glossary_catalog.py`, which already resolves a project by key, name,
alias, or containing workspace and returns the validated catalog plus the compiled
matcher and diagnostics. Surface its diagnostics as a clear error and exit 1;
distinguish "no such project", "project has no glossary configured", and "glossary
configuration is invalid" with distinct messages. Print the project's configured display
name, never the `ProjectSpec` key, per the project-name convention.

### `sase glossary list`

```
sase glossary list [PATTERN] [-d/--definitions] [-f/--format {json,names,table}]
                   [-p/--project REF]
```

- `PATTERN` is an optional positional, case-insensitive substring matched against each
  term and its display aliases. `-d/--definitions` extends the match into definition
  bodies.
- `-f/--format` defaults to `table`.
  - `table`: a Rich table with Term, Aliases, Refs (how many distinct glossary terms
    this definition depends on), and a truncated first-sentence summary; colored, with a
    header line naming the project and a footer count.
  - `names`: one canonical term per line, nothing else — pipe-friendly and the cheapest
    form for an agent.
  - `json`: the full structured records including aliases, definition, reference terms,
    and source location.
- A pattern that matches nothing exits 0 with an explicit "no terms matched" message in
  the human formats and an empty list in `json`.

### `sase glossary show`

```
sase glossary show TERM [TERM ...] [-d/--depth N] [-f/--format {json,markdown,rich}]
                  [-p/--project REF]
```

- `TERM ...` accepts one or more references resolved through the shared lookup. An
  unresolvable term exits 1 and prints the near-miss candidates.
- `-d/--depth N` caps recursion; the default is unlimited, and `-d 0` prints only the
  requested terms.
- `-f/--format` defaults to `rich`. `markdown` emits plain Markdown suitable for pasting
  into a prompt; `json` emits the closure with full provenance.

**Rendering contract.** Every printed term must make its reason for being printed
obvious at a glance:

```
GLOSSARY  sase                          3 terms · 1 requested · 2 related

● Agent Hood                                                        REQUESTED
  aka hood · agent neighborhood
  An agent hood is a group of agents that are all named with the same `<name>.`
  prefix. For example, agents named `foo.bar`, `foo.baz`, and `foo.bar.1` are
  all apart of the same `foo` agent hood.

  ○ Sase Agent                                              RELATED · depth 1
    ↳ mentioned as "agents" in Agent Hood
    aka agent
    A sase agent is ...

    ○ Agent Shell                                           RELATED · depth 2
      ↳ mentioned as "agent shell" in Sase Agent
      also mentioned by Agent Family
      An agent shell is ...

Related terms were printed because the definitions above mention them.
Use -d 0 to print only the requested terms.
```

Rules the implementation must honor:

- Requested terms carry a `REQUESTED` marker; related terms carry `RELATED · depth N`
  plus a `↳ mentioned as "<phrase>" in <referrer>` line naming the exact matched phrase
  and the referring term. This is the "why was this printed" answer.
- A term reachable from several places prints once, at its shallowest discovery, with an
  `also mentioned by <term>, <term>` line.
- Indentation encodes depth, two spaces per level, capped so deep chains stay readable;
  past the cap the depth marker carries the information instead.
- Inside each definition, phrases that link to another printed term are highlighted in
  the accent color, so the closure's edges are visible in the prose itself. Reuse the
  node's stored spans; never rescan.
- Alias chips, the header, and the footer counts are always present; the footer names
  `-d 0` so the reader knows how to opt out of the expansion.
- Use `sase.cli_show_palette` for section coloring and let Rich degrade to plain text
  when stdout is not a TTY, so agent-captured output stays clean.
- `markdown` and `json` carry the same provenance in their own idiom: `markdown` uses a
  heading per term with an italic provenance line; `json` emits `origin`, `depth`,
  `referrer`, and `also_referenced_by` per node plus top-level `truncated`.

Put the shared rendering in `src/sase/glossary/render.py` so `read` reuses it verbatim.

### Tests

Parser tests for both `-p` positions, the default-`list` delegation notice, and
alphabetical help ordering. Handler tests for every format of both subcommands, pattern
filtering with and without `-d`, empty results, unknown and ambiguous terms, depth caps,
diamond and cycle rendering, a project without a glossary, and an invalid glossary
config. Assert the `REQUESTED`/`RELATED` markers and the `mentioned as` provenance line
appear with the exact matched phrase.

Any phase that changes the parser must run `just sync-completion-spec` and commit the
regenerated `tests/completion/snapshots/cli_spec.json`, or
`tests/completion/test_snapshot.py` fails.

---

## Phase `audit`: `sase glossary read` and `log`

### `sase glossary read`

```
sase glossary read TERM [TERM ...] -r/--reason TEXT [-d/--depth N]
                  [-f/--format {json,markdown,rich}] [-p/--project REF]
```

Identical to `show` in every respect except that it requires `-r/--reason` and records
the read. Implement it by sharing the resolution and rendering code path with `show`,
not by duplicating it.

Order of operations: validate the reason, require an agent identity, resolve the project
and terms, build the closure, append the event, then write the rendered output to
stdout. Any failure before the append exits 1 with a message on stderr and records
nothing; never print a definition without recording the read that produced it.

### `sase glossary log`

```
sase glossary log [-a/--agent NAME] [-f/--format {json,table}] [-i/--id READ_ID]
                  [-p/--project REF] [-t/--term TERM]
```

Model it on `src/sase/memory/cli_log.py`.

- No selector: a Rich dashboard with a summary panel (total reads, distinct terms
  requested, distinct agents, total definition bytes served, most recent read), a
  by-term table (term, reads, distinct agents, last read, last reason), a by-agent
  table, and a recent-events table. The reason is shown everywhere a read is listed —
  that is the point of requiring it.
- `-t/--term` and `-a/--agent` filter the event set; both are reflected in the dashboard
  header so a filtered view can never be mistaken for the whole log.
- `-i/--id` selects one event by id or unambiguous id prefix and prints its full detail:
  timestamp, agent, reason, requested terms, related terms, depth limit, bytes, and cwd.
  Ambiguous and unknown ids exit 1 with the candidate list, matching
  `sase memory log --id`.
- `-f/--format json` emits deterministic, sorted JSON for both the summary and the
  single-event view.
- An empty log is a clean exit 0 with an explicit empty-state message.

Use `-f/--format` rather than `--json` across all four glossary subcommands: consistency
inside the group matters more here than matching `sase memory log`'s older `--json`.

### Tests

Read: reason validation, missing agent identity, event contents including
`related_terms` and `definition_bytes`, no event on resolution failure, identical stdout
to `show` for the same arguments, and correct project scoping with `-p`. Log: empty
state, summaries, both filters, id selection including ambiguous and unknown ids, JSON
determinism, and tolerance of a corrupt row in the JSONL.

---

## Phase `panel`: GLOSSARY lane in the agent metadata panel

Add a `GLOSSARY` sub-section to the `SASE CONTEXT` section of the agent metadata panel,
directly after `MEMORY`. `MEMORY` is the reference implementation at every step.

### Loader

`src/sase/ace/tui/glossary_reads.py`, mirroring `src/sase/ace/tui/memory_reads.py`:
`GlossaryReadDisplayEvent(event, agent_label=None)`, the same mtime-and-size keyed
snapshot cache with a re-read throttle, the same `artifacts_dir`-then-`agent_name`
attribution, and `load_glossary_reads_for_agent_context()` for family aggregation. The
TUI's j/k navigation hot path depends on these caches; do not simplify them away.

### Wiring

- `src/sase/ace/tui/widgets/prompt_panel/_agent_glossary_reads.py`:
  `append_agent_glossary_reads_section()`.
- `_agent_context_common.py`: `COLOR_GLOSSARY_SUBHEADER`, `COLOR_GLOSSARY_GLYPH`,
  `COLOR_GLOSSARY_PRIMARY`, and a `GLOSSARY_GLYPH` visually distinct from `MEMORY`'s
  `◇`.
- `_agent_context.py`: add `"GLOSSARY"` to `CONTEXT_LANE_ORDER` after `"MEMORY"`, plus
  entries in `_LANE_LABEL_BACKING`, `_LANE_LABEL_PENDING_STYLE`, `lane_renderers`, and
  both `lane_ids` / `section_id` maps using the section id `"glossary-reads"`.
- `_agent_display_state.py`: add `"glossary"` to `DetailContextLane` and
  `ALL_DETAIL_CONTEXT_LANES`, and a `glossary_reads` field on `DetailHeaderSummary`.
- `_agent_display_header_summary.py`: `_LANE_FIELDS["glossary"] = ("glossary_reads",)`,
  add `"glossary"` to resolution batch 2 alongside `artifacts`/`memory`/`skills` (it
  parses an append-only store on a cache miss, exactly like `memory`), and resolve it
  under its own `tui_trace` span.
- `_agent_display_header.py`: pass `glossary_reads=summary.glossary_reads`.
- `_agent_clan_disk_aggregation.py` and `_agent_display_clan_context.py`: aggregate and
  hint-target the new event type so clan, family, and tribe views stay consistent.

`test_lane_resolution_batches_cover_every_lane_exactly_once` will fail until the new
lane is in exactly one batch — that is the intended guard, not an obstacle to work
around.

### Row shape

```
GLOSSARY · 2 reads · 3 terms
14:22:07 ◈ Agent Hood +2 related
           ↳ needed the hood/agent distinction for the bead prompt
14:19:41 ◈ Stitch
           ↳ confirming stitch vs commit before writing the plan
```

- Primary text lists the requested terms, truncated like `MEMORY` truncates paths, with
  a `+N related` suffix when the closure expanded — that suffix is the visible payoff of
  the whole feature.
- The reason renders on its own indented line through the shared
  `append_context_reason()` helper.
- The lane header counts reads and distinct requested terms; multi-agent families add
  the agent count, matching `MEMORY`.
- Register a hint number per row pointing at the recorded `source_path` so the reader
  can jump to the glossary definition in `sase/sase.yml`. Check whether the existing
  file-hint opener accepts a `path:line` target; if it does, include the definition line
  from the entry's source range, otherwise map to the file.
- Empty state matches `MEMORY`: the lane is skipped unless `show_empty` is set.

### Tests

Loader tests for attribution, caching, and throttling. Rendering tests for the row
shape, the `+N related` suffix, reason rendering, truncation, the overflow row, lane
ordering within `SASE CONTEXT`, the pending "resolving…" row, and family/clan
aggregation. Update the lane-coverage and fold-section tests.

---

## Phase `docs`: Documentation, completion spec, and end-to-end sweep

### Documentation

- `docs/cli.md`: add `sase glossary` and its four subcommands to the appropriate
  section.
- `docs/configuration.md#memoryglossary`: rewrite the generation paragraph.
  `sase/sase.yml` glossary entries no longer generate `sase/memory/glossary.md`; they
  generate the Tier 2 `**GLOSSARY TERMS:**` block and are served on demand by
  `sase glossary`. Keep the schema table and the "no global glossary" rule as they are.
- `docs/init.md`: replace the generated-note description with the block description and
  the one-time deletion of the previously generated note.
- `docs/memory.md`: drop the `#memory/glossary` xprompt-reference example, which no
  longer resolves, and point at `sase glossary read` instead.
- `docs/ace.md`: document the `GLOSSARY` lane in the `SASE CONTEXT` section, and note
  that the preview modal's cross-references and the CLI closure now share one resolver.

### Completion

Regenerate `tests/completion/snapshots/cli_spec.json` with `just sync-completion-spec`
after all parser work has landed. Add a `ValueKind.GLOSSARY` with `PATH_OVERRIDES`
entries for the `show`, `read`, and `log` term slots **only if** the emitters can
consume a new kind without a live-value source; `src/sase/completion/kinds.py` says
kinds land one provider at a time. If a provider is required, leave the kind unset and
file a task bead through `/sase_new_task` proposing glossary-term completion as
follow-up work.

### Sweep

Run `sase memory init` in this repo so `AGENTS.md`, the provider instruction shims, and
the memory README reflect the retired note and the new Tier 2 block, and confirm
`sase/memory/glossary.md` is deleted. Then verify the whole feature by hand: `list` in
each format, `show` with and without a depth cap, `read` with a reason followed by
`log`, and the `GLOSSARY` lane in `sase ace`.

Finish with `just check-full` through `/sase_monitor` over the combined tree, since this
epic touches the memory-generation and TUI broadening sets.
