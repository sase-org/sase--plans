---
status: done
tier: epic
title: Memory webs and strands
goal: "A keyed memory collection is a first-class SASE memory kind: one flat web
  descriptor note plus a sibling directory of strand files, configured by users adding
  files, rendered as core or reference memory per web, and read with `sase memory read
  <web>:<keyword>`. The glossary, task types, and a new decision log all run on that one
  substrate, and the config-backed glossary is gone.

  "
phases:
  - id: tiers
    title: Core and reference memory vocabulary
    depends_on: []
    size: large
    description:
      "tiers: rename short-term/long-term memory to core/reference across the Rust tier
      wire, note frontmatter, AGENTS.md anchors, templates, skills, docs, and prose,
      accepting the old spelling forever and migrating existing notes in place."
  - id: substrate
    title: Web and strand substrate
    depends_on:
      - tiers
    size: large
    description:
      "substrate: add the web/strand domain model, provider-based discovery, the
      fail-closed validator, managed roster regions, and core/reference web rendering
      behind the memory_webs beta flag."
  - id: cli
    title: Selector-based memory read and the web command group
    depends_on:
      - substrate
    size: medium
    description:
      "cli: make `sase memory read`/`show` variadic over note, web, and web:keyword
      selectors, add `sase memory web list|show`, and unify the read audit event around
      a note/web/strand kind discriminator."
  - id: ace
    title: ACE memory pane webs and strands
    depends_on:
      - cli
    size: medium
    description:
      "ace: teach the ACE memory pane to browse webs and their strands, preview strand
      bodies, and reuse the existing memory read audit path."
  - id: decisions
    title: Decision web and flag removal
    depends_on:
      - cli
      - ace
    size: medium
    description:
      "decisions: remove the memory_webs beta flag and close its flag bead, then ship
      the `decisions` core web with six authored decision records."
  - id: task_types
    title: Generated task-type web
    depends_on:
      - decisions
    size: medium
    description:
      "task_types: add the generated web provider driven by the task-type registry and
      convert the task_types core note into a generated core web with one strand per
      agent-creatable type."
  - id: glossary
    title: Glossary migration to a core web
    depends_on:
      - decisions
    size: large
    description:
      "glossary: generalize the Rust glossary source wire to file-backed strands, add
      the one-shot config-to-strand migration, fail closed on dual truth, and migrate
      the sase and bob-cli glossaries."
  - id: retire
    title: Retire the config glossary
    depends_on:
      - task_types
      - glossary
    size: large
    description:
      "retire: delete the config-backed glossary package, schema, completion source, and
      CLI group, fold the ACE glossary pane into the memory pane, and finish the docs."
proposed_by: bbugyi200.athena.0cb
bead_id: sase-sq
create_time: 2026-09-09 19:50:53
---

- **PROMPT:**
  [prompts/202608/memory_webs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/memory_webs.md)
- **BEAD:**
  [sase-sq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sq/README.md)

# Plan: Memory webs and strands

## Goal

Generalize the glossary into a memory kind. A **memory web** is a hub note that names a
keyed collection; a **memory strand** is one small note inside it. Users create a web by
adding files — `sase/memory/<web>.md` plus `sase/memory/<web>/<slug>.md` — and choose
per web whether it reaches agents as core memory or reference memory. Agents read
strands with `sase memory read <web>:<keyword>`.

By the end of this epic:

- `sase/memory/glossary.md` is a user-owned core web whose 34 terms are strand files,
  and `memory.glossary` no longer exists in any `sase.yml`.
- `sase/memory/task_types.md` is a generated core web whose strands come from the
  task-type registry, so plugins control which strand files `sase memory init` writes.
- `sase/memory/decisions.md` is a core web holding six real decision records.
- "Short-term memory" and "long-term memory" are called **core memory** and **reference
  memory** everywhere.
- The `memory_webs` beta flag has been removed and its flag bead closed.

## Design

### Two orthogonal axes

This is the single most important sentence in the design, and every later sentence
depends on getting it right:

> What a memory **is** (note, web, strand) and how a memory **renders** (core,
> reference) are independent. A web is not "a core memory"; a web _renders as_ core
> memory.

Today those axes are conflated in one `type:` field because there is only one kind of
memory. After this epic:

| Axis      | Values                  | Declared by                                  |
| --------- | ----------------------- | -------------------------------------------- |
| Kind      | `note`, `web`, `strand` | `web: true` frontmatter; strands by location |
| Rendering | `core`, `reference`     | `type:` frontmatter on the note or web       |

A strand has no rendering of its own: it is never in a generated document. Write both
axes explicitly in `sase/memory/README.md` and in the `/sase_memory_read` skill, or
every downstream sentence is ambiguous.

### Vocabulary

- **core memory** replaces "short-term memory". The Tier 1 preamble already says "core
  (always loaded) context", so this rename is free.
- **reference memory** replaces "long-term memory". "Structured" is the S in SASE, so a
  "structured memory" parses as "any SASE memory"; it also fails to contrast with
  "core", since core memory is structured too. The Tier 2 preamble already says
  "detailed **reference** material", and "a reference web" reads correctly where "a
  structured web" does not.
- Frontmatter values become `type: core` and `type: reference`. `short` and `long` stay
  accepted **forever** on read.
- The H2 anchors stay Tier 1 / Tier 2 as the user requires, with the parenthetical
  renamed: `## Tier 1 (core) Memory` and `## Tier 2 (reference) Memory`.

### Layout: flat descriptors, no `webs/` segment

```text
sase/memory/
  glossary.md          # web descriptor: type: core, web: true, managed roster region
  glossary/
    agent-hood.md      # strand: keyword "Agent Hood", aliases [hood, agent neighborhood]
    stitch.md
  decisions.md         # web descriptor: type: core, web: true
  decisions/
    single-turn-agents.md
  task_types.md        # generated web descriptor
  task_types/
    bug.md             # generated strand
  cli_rules.md         # ordinary reference note, unchanged
```

Dropping the `webs/` segment is worth roughly half the implementation cost, and the
reason is mechanical. Six path matchers in the document layer hard-code a memory path as
`memory/[A-Za-z0-9_.-]+\.md`, a character class that excludes `/`:

| Location                                     | Purpose                                      |
| -------------------------------------------- | -------------------------------------------- |
| `src/sase/amd/_agents_doc.py:22`             | legacy `- @memory/<f>.md` core bullet        |
| `src/sase/amd/_agents_doc.py:28`             | `### Title (<basename>)` inlined-core header |
| `src/sase/amd/_agents_doc.py:30`             | `**\`memory/<f>.md\`\*\*` Tier 2 entry       |
| `src/sase/amd/_agents_doc.py:33`             | H3/H4 `` `memory/<f>.md` `` form             |
| `src/sase/amd/inventory.py:60`               | memory-path reference scanning               |
| `src/sase/memory/inventory_references.py:25` | note-to-note link parsing                    |

With flat descriptors none of them change, `#memory/<web>` keeps working with no new
derivation, today's `sase/memory/glossary.md` _becomes_ the glossary web descriptor in
place, and the `### {title} ({basename})` round-trip ambiguity between `glossary.md` and
`webs/glossary.md` cannot arise. The only cost is that a web is told from a note by
frontmatter rather than by path, which is one field. Directories already coexist with
notes under `sase/memory/` (`assets/`), so the layout is not novel.

**The load-bearing property:** strands never appear in a generated document at all, so
the document layer never parses a nested path. Note discovery (`discover_memory_notes`,
`iter_memory_files`) keeps globbing `*.md` one level and keeps ignoring strands; only
the web reader and the validator walk into `<web>/`.

### Frontmatter

Web descriptor:

```yaml
type: core # or reference
web: true
description: Project vocabulary needed to interpret SASE prompts and artifacts.
roster: inline # or list; default inline
roster_label: GLOSSARY TERMS # optional; default derived from strand_noun
strand_noun: term # optional display noun; default "strand"
closure: mentions # or none; default none
```

Strand:

```yaml
keyword: Agent Hood # optional; defaults to a title-cased slug
aliases: [hood, agent neighborhood]
summary: One line, required only when the web uses roster: list
metadata: {} # opaque; generic memory code must never interpret it
```

Rules that follow:

- A strand declares no `type:`. A strand's rendering is its web's rendering, and letting
  a strand override it reintroduces exactly the per-item tier confusion webs exist to
  remove.
- A strand declares no `parent:`. Its parent is its web.
- No `references:` or strand-local link field in v1. See "Link model" below.
- `metadata:` is where a web's own lifecycle fields live (an ADR's `decided:` date and
  `status:`, for example). Generic memory code passes it through untouched.
- One caveat worth writing into the README: note `keywords:` was removed in `21e1640ee`
  as a _runtime trigger_ for the dynamic-memory engine. `keyword:` here is an
  _addressing alias_ evaluated at read time by an explicit command. Say so, or someone
  files a bead against it in six months.

### Identity: slug is identity, keyword is display

The filename stem is the strand's immutable identity. `keyword:` is mutable display.
Audit events and future artifact links always record the slug even when the user typed a
keyword, so renaming a display keyword never breaks a link or an audit trail.

Lookup precedence, resolved with the existing `normalize_glossary_reference` (casefold;
collapse runs of `-`, `_`, and whitespace to one space):

1. exact slug
2. exact keyword
3. exact alias (including Rust-derived plurals)
4. unique normalized prefix

Do **not** require that a keyword slugs back to its filename. That rule would make
renaming a display keyword require a file rename, which breaks links and audit history
for a purely cosmetic change.

### Rendering invariant

> **A core web inlines only its descriptor's own body. Strand bodies never reach a
> generated document.**

Inlining the glossary's 34 definitions would add 1,831 words to a measured 2,273-word
core footprint — an 81% increase from one web. State the invariant loudly and enforce it
with a renderer test that asserts no strand body appears in any generated document.

Pleasantly, this falls out of the existing model with no new machinery: a web descriptor
is a note; a core note inlines its body; a reference note contributes its description;
strands are simply not part of instruction rendering. A reference web therefore renders
in Tier 2 exactly like an ordinary reference note — description only, no roster, no new
parsing risk.

### The roster: a managed region inside a user-owned note

Today a hand-authored `sase/memory/glossary.md` _blocks `sase memory init` entirely_
(`_glossary_collision_blocker`, `root_planning.py:142`) because the generator owns the
whole file. Invert that: the user owns the descriptor, SASE owns one delimited block
inside it.

```markdown
<!-- sase:strands -->

**GLOSSARY TERMS:** Agent Clan; Agent Family; Agent Hood (hood, agent neighborhood); …

<!-- /sase:strands -->
```

- `roster: inline` renders `roster_label` followed by `keyword (alias, alias)` entries
  separated by `; `. This reproduces today's glossary roster byte-for-byte, which is why
  the glossary migration costs no core-memory budget.
- `roster: list` renders one bullet per strand: `- **<keyword>** (\`<slug>\`) —
  <summary>`. Used by `decisions`and`task_types`, where the roster is the skimmable
  index and the slug is the address.
- If the descriptor has no region, `sase memory init` appends one at the end of the
  body. If it has one, init replaces its contents in place. Unbalanced markers are a
  blocker.
- Region updates are ordinary `update` changes, so `sase memory init --check` and
  `--diff` show them like any other drift.

This deletes `_glossary_collision_blocker`, the `sase_generated: glossary` marker, and
the retired-glossary-note deletion path, and it is the only shape that satisfies "users
configure webs by adding files".

### Discovery is a provider interface

Discovery returns `(descriptor, strands)` through an interface rather than a hard-coded
glob:

- `FileMemoryWebProvider` — the descriptor and strands are user-owned files. This is
  every web except `task_types`.
- `GeneratedMemoryWebProvider` — SASE owns the whole descriptor and every strand file,
  writes them as expected files, and retires stale strands. `task_types` uses this,
  driven by the task-type registry, which is exactly how the plugin registry ends up
  controlling which strand files `sase memory init` creates.

Keeping this seam is what stops SASE from carrying three implementations of one idea
forever. `artifact_relations` stays an ordinary generated core note in this epic, per
the user's "no other webs yet", but it becomes a one-file change later.

### Scope resolution: per-strand merge, project wins

`sase memory read` already resolves **per path** — project root, then home root
(`_memory_read_roots`, `read_log.py:161`). A project `foo.md` hides a home `foo.md`, but
a home `bar.md` with no project counterpart stays reachable. Per-strand merge is
therefore the choice _consistent_ with the existing contract, not a divergence from it.

One consequence worth handling explicitly: home and project instruction files are
separate documents, each rendering its own Tier 1 and Tier 2 from its own scope. A home
web and a project web of the same name each render their own roster into their own
document, so a keyword defined at both scopes appears twice with different bodies while
reads resolve to the project one. **Warn at `sase memory init` for cross-scope keyword
collisions; do not fail.**

### Link model: build neither closure proposal in v1

`resolve_glossary_closure` is _name-matching_ machinery. It works because "Patch"
literally occurs inside the definition of "Stitch". Decision-record titles do not occur
inside each other's prose, so a `decisions` web gets an empty closure and any generated
"batching is cheaper" prose becomes false for it.

So, in v1:

- Keep mention-closure exactly as it is, as the glossary web's behavior. Reuse
  `resolve_glossary_closure` and the Rust matcher unchanged; only discovery changes. The
  glossary descriptor declares `closure: mentions`.
- Ship every other web with `closure: none` (the default). Decision records are read by
  name; a reader who wants a superseded record follows a prose reference.
- Do **not** invent a strand-local link field.

When a web has enough strands that manual traversal hurts, adopt the **existing**
`supersedes` / `superseded-by` relations from the artifact relation registry rather than
a parallel link syntax. The address shape `<web>:<keyword>` is already artifact-grammar
compatible (`<kind>:<argument>`), so deferring costs nothing and guessing wrong now
costs everything.

### CLI

```bash
sase memory read <note>.md                     -r "<why>"   # unchanged
sase memory read <web>                         -r "<why>"   # every strand in the web
sase memory read <web>:<kw> [<web>:<kw> …]     -r "<why>"   # named strands + closure
sase memory read glossary:stitch cli_rules.md  -r "<why>"   # mixed batch
sase memory show <selector> …                                # identical, no audit event
```

- `.md` means the descriptor file; a bare name means the web's strands. That also
  resolves the otherwise-ambiguous `sase memory read glossary`.
- The positional becomes variadic and gains `-d/--depth`,
  `-f/--format {json,markdown,rich}`, and `-p/--project`, which already exist on
  `sase glossary read`. This is an enumerable gap, not a redesign.
- Resolve the entire batch before writing an audit event or emitting output. One unknown
  or ambiguous selector fails the whole request, matching current glossary batch safety.
- `sase memory read <web>` with no keyword prints **everything**, literally. An audited
  read must mean what its selector says; report strand and byte counts in JSON output
  and warn on very large results rather than truncating silently.
- A core note still cannot be read (it is already in context), and neither can a core
  web's _descriptor_. A core web's _strands_ can, because strands are never in context.
  This is the rule that makes `sase memory read glossary:stitch` legal.
- `sase memory read glossary/stitch.md` stays rejected by the flat-path rule, with the
  error suggesting `glossary:stitch`.

New group, following the default-`list` convention so bare `sase memory web` delegates:

```bash
sase memory web list                                  # webs: name, renders, scope, strands
sase memory web show <web> [PATTERN] [-b] [-f] [-p]   # strand index for one web
```

`sase memory web show` is the _index_ (this is where `sase glossary list`'s filterable
table lands, with `-b/--bodies` replacing `-d/--definitions`); `sase memory show <web>`
is the _content_. Say exactly that in both help strings. `sase memory list` gains a webs
section so the context inventory stays complete.

### Audit log

Unify rather than writing two shapes from one command. Bump `READ_LOG_SCHEMA_VERSION` to
2 and add a discriminated `kind: note | web | strand`, recording original selectors,
resolved slugs, transitively included slugs, effective depth, scope origin, and bytes.
Keep the v1 reader so historical JSONL stays queryable. `sase memory log --include`
gains a `glossary` choice that folds in the legacy `glossary_reads.jsonl` so the audit
history survives retirement.

### Validation is a first-class deliverable, not polish

Config gave keyword uniqueness structurally. Files do not. Every rule below must be
_written_, must run in both `sase memory init` and `sase doctor`, and must fail closed:

1. slug and keyword uniqueness within a web, per scope
2. alias non-ambiguity — feed strand-derived entries to `validate_glossary_entries`,
   which is reusable verbatim
3. case and normalization collisions
4. an orphan strand directory with no descriptor
5. reserved web names: every registered artifact kind (`stitch`, `patch`, `bead`,
   `agent`, `file`, plus plugin kinds such as `plan` and `research`), plus `assets` and
   `README`
6. a `<web>/` directory whose sibling `<web>.md` does not declare `web: true`
7. nested subdirectories under a web
8. symlink escape out of the memory root
9. malformed strand frontmatter, and a missing `summary:` under `roster: list`
10. unbalanced or duplicated `sase:strands` region markers
11. cross-scope keyword collision — **warning**, not failure

The entire Rust diagnostic surface, alias-plural derivation, and normalization carry
over unchanged. Discovery changes; validation does not.

### Rust boundary

Per `rust_core_backend_boundary.md`: shared catalog, normalization, validation,
matching, and closure belong in `sase-core`. Scope discovery, filesystem containment,
rendering, audit persistence, init planning, and compatibility adapters stay in Python.

Exactly two coordinated `sase-core` changes, each landing with its Python adapter and a
dependency-window bump (`sase-core-rs>=0.31.0,<0.32.0` today):

1. **`tiers`**: `MemoryTierWire::parse` accepts `core` and `reference` alongside `short`
   and `long`; `as_str()` emits the new names; the `memory_note_issue` message names
   them.
2. **`glossary`**: generalize `GlossarySourceWire` from `config_path` +
   `config_key_path` to a source path plus _optional_ key path, so a strand's source is
   a file with `keyword_range` / `body_range`, and bump `GLOSSARY_WIRE_SCHEMA_VERSION`
   from 1 to 2. This is what keeps editor go-to-definition working after migration — and
   it improves it, since it lands on a real Markdown note instead of a YAML scalar.

Renaming the Rust `glossary.rs` module is explicitly **out of scope**: it is cosmetic,
cross-repo, and costs another release. The module is the phrase-matching engine and the
name is defensible.

### Feature flag

One `beta` flag, `memory_webs`, created with `sase flag new` (which files its flag
bead). It gates web discovery and the new CLI surface while they are being built.

- **On**: `web: true` is honored, strands are discovered, web selectors resolve.
- **Off**: `web: true` is ignored, the descriptor behaves as an ordinary note, `<web>`
  and `<web>:<kw>` selectors are rejected as unknown.

The flag is removed in the `decisions` phase — _before_ any web descriptor is committed
to any repo. That ordering is deliberate: a committed web descriptor plus a
machine-global flag would make `AGENTS.md` content depend on a boolean rather than on
the tree, and generated files must be decided by the tree. Removal deletes the Off
branch, makes the On branch unconditional, drops the registry entry, and closes the flag
bead in the same change.

**The glossary migration itself needs no flag**, and adding one would make it worse. The
migration's safety comes from the tree: strand files present means files win, config
present means config wins, and both present is a fail-closed blocker. That is
per-project and deterministic, where a global boolean could leave a migrated tree
reading config.

### Sequencing rationale

`decisions` ships before the glossary migrates. The glossary is simultaneously the
motivating case and the single most coupled surface in the repo — 29 modules named
`*glossary*` at 5,866 LOC, 124 Python modules mentioning it, a 1,186-line Rust module,
an LSP payload, a completion source, an ACE pane, and core memory in three projects. A
broken decision web costs nothing. Prove the machinery where failure is free.

Phases 1–3 have disjoint blast radii and each is independently worth its cost even if
the next never lands. Do not bundle them.

### Explicit non-goals

- **Strand proposals.** `sase memory write --target <web>:<slug>` is not built here. Per
  the `corpus-before-mechanism` record this epic itself authors: do not ship the
  contribution path before agents are actually reading the corpus.
- **New webs beyond the three named.** No `gotchas`, `runbooks`, `commands`, or
  `artifact_relations` webs. Leave the provider seam; do not use it.
- **Typed strand links / artifact promotion.** Deferred to `supersedes` on the artifact
  relation registry.
- **Relocating bob-cli's four terms to home scope.** The research argues they are
  Obsidian vault vocabulary that belongs at home scope; that is true but it would push
  Obsidian vocabulary into every project's core roster, including `sase`'s. Migrate them
  in place; moving four files later is a user decision, not a mechanical one.

## Phases

### tiers: Core and reference memory vocabulary

Rename short-term/long-term to core/reference everywhere, accepting the old spelling
forever.

**`sase-core`** (land and release first): `MemoryTierWire::parse` accepts `"core"` and
`"reference"` in addition to `"short"` and `"long"`; `as_str()` emits
`core`/`reference`; `memory_note_issue`'s message says "`type: core` or
`type: reference`". Bump the crate, release, and widen the `sase-core-rs` dependency
window in `pyproject.toml` via `tools/ratchet_core_window`.

**Python data model** (`src/sase/memory/notes.py`): `MemoryNoteType` becomes
`Literal["core", "reference"]`; parsing accepts all four spellings and normalizes
`short`→`core`, `long`→`reference` so no comparison site has to know about legacy
values; `apply_memory_frontmatter` emits the new values. Update every
`note.type == "short"` / `== "long"` comparison (`read_log.py`,
`inventory_reachability.py`, `render.py`, `root_rendering.py`, `amd/_memory.py`,
`memory_panel_catalog.py`, `cli_list.py`, `cli_write.py`, `cli_review.py`,
`review_tui/`) — normalization means these become simple value swaps.
`xprompt/loader_memory.py`'s `_memory_type` and `xprompt/models.MemoryType` follow.

**Document anchors** (`src/sase/amd/_agents_doc.py`): `_SHORT_SECTION_RE` and
`_LONG_SECTION_RE` accept `(?:short-term|core)` and `(?:long-term|reference)`. **Add,
never replace** — `parse_amd_agents_document` reads _already generated_ documents, so a
newer `sase` must keep accepting old anchors or every not-yet-reinitialized project's
shims break. Rename the constants to `_CORE_SECTION_RE` / `_REFERENCE_SECTION_RE`.
`_render_managed_agents`'s anchor assertions and error strings follow.

**Templates and generated prose**: `AGENTS.template.md` emits `## Tier 1 (core) Memory`
and `## Tier 2 (reference) Memory`; `memory-README.template.md` documents `type: core` /
`type: reference` and reports "Core notes" / "Reference notes". Template variables
`short_notes`/`long_notes` become `core_notes`/`reference_notes` — pass **both** names
in the render context for this release and require only the new ones, so a user's
overridden README template does not break. `_LONG_MEMORY_INTRO` and
`render_children_section` prose say "reference material" without "long-term".

**Prose sweep**: 84 occurrences of "short-term"/"long-term" across `src/`, `tests/`, and
`sase/memory/`, including `parser_memory.py` help text, `notifications/senders.py`,
`src/sase/xprompts/skills/sase_memory_read.md`, and `docs/memory.md`, `docs/cli.md`,
`docs/init.md`. The skill rewrite must also introduce both axes and preview the coming
`<web>:<keyword>` grammar.

**Data migration**: `sase memory init` rewrites `type: short|long` to
`type: core|reference` in every managed root's notes as an `update` change, so `--check`
and `--diff` report it. Regenerate `sase`, `bob-cli`, and home;
`sase memory init --check` must be clean in all three afterward.

**Vocabulary lock** (do this in the same pass so there is one regeneration): add
glossary terms **Core Memory**, **Reference Memory**, **Memory Web**, **Memory Strand**,
and **Strand Keyword** with `sase glossary add`. They migrate to strands automatically
later.

Tests: an already-generated `AGENTS.md` with old anchors still round-trips; a note with
`type: short` still loads and now normalizes to `core`; the emitted anchors are the new
ones; the README template renders with a user override that only knows the old variable
names.

### substrate: Web and strand substrate

Create the domain and wire it into memory init, behind
`sase flag new memory_webs -k beta`.

New package `src/sase/memory/web/` (keep every module under the 700-line `toobig` cap):

- `models.py` — `MemoryWeb`, `MemoryStrand`, `WebRosterStyle`, `WebClosureMode`.
- `frontmatter.py` — descriptor and strand frontmatter parsing, defaults (`keyword`
  derived from a title-cased slug, `roster: inline`, `closure: none`,
  `strand_noun: strand`), and opaque `metadata:` passthrough.
- `discovery.py` — the `MemoryWebProvider` protocol returning `(descriptor, strands)`,
  `FileMemoryWebProvider`, and `discover_memory_webs(root)` finding flat `*.md` notes
  with `web: true` plus their sibling directories.
- `scope.py` — per-strand merge across project-then-home roots, with a `WebStrandOrigin`
  recording which scope supplied each strand.
- `lookup.py` — slug → keyword → alias → unique-normalized-prefix resolution, reusing
  `normalize_glossary_reference` (re-export it from here so nothing new imports
  `sase.glossary`).
- `roster.py` — render the `<!-- sase:strands -->` region for both styles; append the
  region when absent; replace in place when present.
- `validation.py` — the eleven rules above, returning blockers and warnings.

Integration:

- `root_planning.memory_root_context` discovers webs, renders roster regions as
  `MemoryExpectedFile` updates, and surfaces validator blockers alongside existing ones.
- `sase doctor` gains a webs check running the same validator.
- Note discovery is untouched: `discover_memory_notes` and `iter_memory_files` keep
  globbing one level, so strands never enter the note inventory, reachability graph, or
  Tier 2.
- Reserved-name checking pulls registered artifact kinds so the list cannot drift.

Tests, including the two that matter most: a renderer test asserting **no strand body
appears in any generated document**, and a flag-off test asserting a `web: true` note
renders exactly as an ordinary note with no roster region written. Both flag states get
coverage, as every flag requires.

### cli: Selector-based memory read and the web command group

Make `sase memory read` and `sase memory show` variadic over three selector shapes,
resolving the whole batch before any output or audit write. Port `-d/--depth`,
`-f/--format {json,markdown,rich}`, and `-p/--project` from `sase glossary read`; keep
every public long option short-aliased and the subcommand and option lists sorted, per
`cli_rules.md`.

Rendering reuses `sase.memory.render` for notes and adapts `sase.glossary.render`'s
closure presentation for strand batches, so a glossary read looks the same after
migration as before it.

Add `sase memory web list` and `sase memory web show <web> [PATTERN]`, with
`-b/--bodies`, `-f/--format {json,names,table}`, and `-p/--project`. Bare
`sase memory web` delegates to `list` through the central `_default_list_subcommands()`
wiring — do not re-implement the delegation. Extend `sase memory list` with a webs
section.

Audit: `READ_LOG_SCHEMA_VERSION` 2 with `kind: note | web | strand`, original selectors,
resolved slugs, transitively included slugs, effective depth, scope origin, and bytes.
Keep the v1 event reader. `sase memory log` renders both, and `--include glossary` folds
in the legacy glossary JSONL.

Tests: batch failure atomicity (one bad selector prints nothing and logs nothing); a
core web's descriptor is refused while its strands are allowed; `<web>` with no keyword
prints every strand; mixed batches; v1 events still parse.

### ace: ACE memory pane webs and strands

Teach `MemoryPane` about webs. `memory_panel_catalog` snapshots gain web rows with
strand counts; a web row expands to its strands; selecting a strand previews its body
and metadata. Reads from the pane go through the same audited path as the CLI, and every
disk walk stays off the event loop — read `sase/memory/tui_perf.md` with
`/sase_memory_read` before touching this.

Add `memory` keymap entries for expanding a web and moving between strands in
`src/sase/default_config.yml`. Do not touch the `glossary` keymap scope yet; the
glossary pane is still live until `retire`.

Visual snapshots: if the pane's rendering changes, refresh the PNG goldens with
`just test-visual --sase-update-visual-snapshots`.

### decisions: Decision web and flag removal

**First, remove the flag.** Delete the Off branch, make webs unconditional, drop the
`memory_webs` registry entry, and close its flag bead in this change. This must precede
committing any descriptor, so that generated documents depend on the tree and not on a
boolean.

**Then author the web.** `sase/memory/decisions.md` — `type: core`, `web: true`,
`roster: list`, `strand_noun: decision`, `roster_label: DECISIONS`, `closure: none`. Its
body is short: what the web is for, that records are immutable and superseded rather
than edited, and the read command. Six strands in `sase/memory/decisions/`, each
**200–300 words**, `metadata: {status: accepted, decided: <ISO date>}`, and each
structured as claim → why → what it costs → what would reopen it. Every token in an
always-loaded roster either helps or hurts, so keep summaries to one line.

Slugs are semantic, not numbered. Nygard's sequential IDs are a 2001-era filesystem
sorting artifact; SASE has a real index (the roster) and real ordering
(`metadata.decided`), and `decisions:corpus-before-mechanism` is a far better address
than `decisions:0004`.

| Slug                      | Keyword                                  | Aliases                                        | Claim                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------- | ---------------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `single-turn-agents`      | Agents Are Single-Turn                   | `single turn`, `one-turn agent`                | A SASE agent run is one provider turn; continuation is mechanical (`/sase_monitor`, `/sase_pipe`, plan and questions handoffs), never a promise to resume. Rejected: long-lived agent processes, because workspace claims and provider budgets are per-run.                                                                                                                                                                                                       |
| `rust-core-required`      | The Rust Core Is Required                | `no python fallback`, `rust core boundary`     | Shared backend and domain behavior lives in `sase-core` and is reached through `sase_core_rs`; there is no Python fallback and no env-var backend switch. Rejected: a mirrored Python implementation, because divergence between frontends is silent.                                                                                                                                                                                                             |
| `two-speed-verification`  | Verification Is Two-Speed                | `check vs check-full`, `scoped tests`          | `just check` is the agent default and `just check-full` gates landing and CI, because the binding constraint is host capacity with many agents in parallel, not test speed. Scoped selection is a heuristic backstopped by CI.                                                                                                                                                                                                                                    |
| `corpus-before-mechanism` | No Retrieval Mechanism Before Its Corpus | `mechanism before corpus`, `corpus first`      | SASE does not ship a retrieval mechanism before the content it retrieves exists and is in use. Evidence: dynamic memory removed in `e8c2f14bb`, episodes in `37973b8b3`, note `keywords:` in `21e1640ee`. This is the record that governs the epic that created it.                                                                                                                                                                                               |
| `host-owned-completion`   | Completion Is Host-Owned                 | `finalizer declaration`, `agents never commit` | An agent never creates commits, branches, or PRs; it submits a sealed declaration and host-owned finalizers act. Rejected: agent-authored git, because an agent's word that work is done is neither verifiable nor attributable.                                                                                                                                                                                                                                  |
| `memory-webs`             | Memory Webs                              | `memory strands`, `web-backed memory`          | A keyed memory collection is one flat descriptor note plus a sibling directory of strands, addressed `<web>:<keyword>`, rendering as core or reference per web. Why: the shape had been hand-built three times (glossary, task types, artifact relations) and the glossary note had moved between tiers four times because a collection's tier had nowhere to be declared. Flat over nested because six document-layer matchers hard-code a flat memory filename. |

`memory-webs` should cite `corpus-before-mechanism` in prose as the reason it ships with
three real corpora and defers closure and typed links.

The two research reports in the `sase--research` sidecar
(`202608/decision_web_seed_adrs.md` and `202608/decisions_web_seed_adrs.md`) carry the
evidence — commit SHAs, measurements, and rejected alternatives — for every record
above. Open them with `/sase_repo` and read them before writing.

Regenerate `sase`; confirm the core roster grew by roughly 170 words and no strand body
reached `AGENTS.md`.

### task_types: Generated task-type web

Add `GeneratedMemoryWebProvider` and convert `task_types`.

`root_rendering_task_types.py` currently renders one 612-word core note with an H5
section per type. Replace it with a provider that reads
`build_committed_task_type_snapshot_entries(get_task_type_registry())` and yields:

- a generated descriptor at `sase/memory/task_types.md` — `type: core`, `web: true`,
  `roster: list`, `strand_noun: task type`, `roster_label: TASK TYPES` — whose body
  keeps the always-loaded "File Discovered Work As Task Beads" instruction and the
  `/sase_new_task` requirement verbatim, since that is the part agents must always have;
- one generated strand per agent-creatable type at `sase/memory/task_types/<slug>.md`,
  carrying `when_to_use`, required and optional fields, and the
  `sase bead task-type show <slug>` pointer, with `summary:` supplying the roster line.

Strand retirement mirrors the existing note-retirement rule: a strand whose bytes match
the current packaged render but whose type is gone is deleted; a hand-edited file at the
same path is left alone. `sase/task_types.json` is unchanged.

This is the concrete answer to whether the plugin registry can control which memory
files `sase init` creates: it can, because the registry already includes builtin,
project-config, and required-plugin types, and the provider is just another
expected-file source. Optional plugin types stay live-only so two machines render the
same tree.

Regenerate `sase` and `bob-cli`. Core memory should drop by roughly 450 words.

### glossary: Glossary migration to a core web

**`sase-core` first**: generalize `GlossarySourceWire` to a source path plus optional
key path, add `keyword_range` / `body_range` for file-backed strands, bump
`GLOSSARY_WIRE_SCHEMA_VERSION` from 1 to 2, release, and widen the dependency window.
Update `core/glossary_facade.py`, `xprompt/_glossary_catalog_config.py`, and the LSP
payload so go-to-definition lands on the strand file.

**Migration command**: `sase memory web migrate glossary [-p REF] [-n/--dry-run]` reads
`memory.glossary`, writes one strand per term (slug from `normalize_glossary_reference`
with spaces to hyphens, `keyword:` = the configured term, `aliases:` = configured
aliases, body = the definition), removes the config block with the existing
source-preserving YAML mutation, and rewrites `sase/memory/glossary.md` as a user-owned
descriptor: `type: core`, `web: true`, `roster: inline`, `roster_label: GLOSSARY TERMS`,
`strand_noun: term`, `closure: mentions`, with today's preamble prose (updated to say
`sase memory read glossary:<term>`) above the managed region. The rendered roster must
be byte-identical to today's.

**Catalog source**: `editor_glossary_catalog_for_project` prefers strand files when the
web exists and falls back to `memory.glossary` when it does not. Both present is a
fail-closed blocker naming the migration command — never dual-write, never merge.
`init_memory/glossary.py`'s `load_project_glossary_terms` follows the same rule; delete
`_glossary_collision_blocker`, `_retired_glossary_note_paths`, and the
`sase_generated: glossary` marker, all of which the managed region makes obsolete.

**Compatibility**: `sase glossary read|show|all|list|log` become thin aliases that print
a one-line deprecation notice naming the `sase memory` equivalent and delegate.
`sase glossary add|del` write strand files when the web exists.

Migrate `sase` (34 terms) and `bob-cli` (4 terms). Verify `sase glossary read Stitch`
and `sase memory read glossary:stitch` produce identical closures, and that
`sase memory init --check` is clean in both projects and at home.

### retire: Retire the config glossary

Delete the config-backed implementation now that nothing reads it.

Remove `src/sase/glossary/` (3,280 LOC), `src/sase/glossary_config.py`,
`src/sase/xprompt/_glossary_catalog_config.py`, `_glossary_catalog_ranges.py`,
`src/sase/main/parser_glossary.py`, `src/sase/main/glossary_handler.py`, and
`src/sase/main/init_memory/glossary.py`, folding anything still needed into
`sase/memory/web/`. Drop the `glossary` and `glossaryEntry` blocks from
`src/sase/config/sase.schema.json`, the `glossary` key from `config/layers.py`'s
project-only set, and the `sase.yml`-mtime completion invalidation source. Retire the
`sase glossary` command group after one release of deprecation notices.

Fold `GlossaryPane` (4,773 LOC across `ace/tui/modals/glossary_*` and
`ace/tui/glossary_*`) into `MemoryPane`: term browsing becomes strand browsing, add and
delete become strand file creation and deletion, relation travel becomes closure travel
for `closure: mentions` webs, and the project ring becomes the existing memory scope
ring. Move the `glossary` keymap scope's bindings into the `memory` scope in
`src/sase/default_config.yml`, keep the `glossary` scope name accepted but inert for one
release, and have `sase doctor` warn when a user config still sets it.

Finish `docs/memory.md`, `docs/cli.md`, `docs/editor.md`, `docs/completion.md`, and
`docs/ace.md`; regenerate `sase/memory/README.md` so it documents kinds and rendering as
two axes; update the `/sase_memory_read` skill and deploy it per `generated_skills.md`
(commit the template, land it, then `sase skill init --force`).

Land with `just check-full` through `/sase_monitor`, and confirm
`sase memory init --check` is clean for `sase`, `bob-cli`, and home.

## Risks

| Risk                                                           | Severity | Mitigation                                                                                                    |
| -------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------- |
| A core web inlines strand bodies and blows up the core budget  | high     | The rendering invariant plus a renderer test asserting no strand body reaches any generated document          |
| Nested paths break document round-trip parsing                 | high     | Flat descriptors; strands never reach the document layer, so none of the six path matchers change             |
| Glossary migration lands broken across three projects          | high     | `decisions` proves the machinery first; fail-closed dual-source rule; migrate `bob-cli` in the same phase     |
| Validator gap admits duplicate or ambiguous keywords           | medium   | Validation is a phase deliverable, failing closed in both `sase memory init` and `sase doctor`                |
| Anchor rename breaks already-emitted shims in unmigrated trees | medium   | Accept old anchors forever; add, never replace                                                                |
| `sase-core` wire desync across repos                           | medium   | Exactly two coordinated changes, each landing with its adapter and window bump before the phase that needs it |
| Mechanism before corpus                                        | medium   | Every web ships with a real corpus: 6 decisions, the live task-type catalog, 34 glossary terms                |
| Dual truth between YAML and files                              | medium   | Fail closed when both are present; never dual-write                                                           |
| Flag state changes generated-file content                      | medium   | The flag is removed before any descriptor is committed                                                        |
| Vocabulary drift between "is a web" and "renders as core"      | low      | The two-axis rule, in the README and the skill before the first web ships                                     |
