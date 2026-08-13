---
tier: tale
title: Number every section heading in generated agent instruction files
size: medium
goal:
  Every section heading in a generated agent instruction file carries a hierarchical
  section number derived from the document's own heading tree — `## 1. Tier 1
  (short-term) Memory`, then `### 1.1 Build & Run Commands (build_and_run)`, then `####
  1.1.1 PNG Snapshot Tests`, then `## 2. Tier 2 (long-term) Memory` — so any part of
  `AGENTS.md` (and its provider shims) can be named by number instead of by prose title.
proposed_by: bbugyi200.athena.zn
create_time: 2026-08-13 12:39:36
status: wip
---

# Plan: Number every section heading in generated agent instruction files

## Context

`sase memory init` generates each root's agent instruction file (`AGENTS.md`) and then
writes the four provider shims (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`) as
byte-for-byte copies of it. Today only the _inlined short-term memory_ headings are
numbered, and that numbering is done at inline time by
`inline_memory_section()`/`_shift_body()` in `src/sase/amd/inline_memory.py:167` and
`:117`, which take a `number: int | None`. The template's own `##` headings are not
numbered at all.

Current project-root output:

```
# Structured Agentic Software Engineering (SASE) - Agent Instructions
## Tier 1 (short-term) Memory
### 1. Build & Run Commands (build_and_run)
#### 1.1 IMPORTANT: Two-Speed Verification — Run `just check` if you Made File Changes
#### 1.2 PNG Snapshot Tests
### 2. Code Conventions and Gotchas (gotchas)
### 3. Rust Core Backend Boundary (rust_core_backend_boundary)
### 4. SASE = Structured Agentic Software Engineering (sase)
#### 4.1 Ephemeral `sase_<N>` Workspace Directories
#### 4.2 Repositories
#### 4.3 File Discovered Work As Task Beads
## Tier 2 (long-term) Memory
```

The document is produced in one place — `render_agents_template()`
(`src/sase/amd/_template.py:16`) — from either the managed template
(`src/sase/amd/templates/AGENTS.template.md`, H1 + two H2 sections +
`{{ tier1_sections }}` / `{{ tier2_entries }}`) or the minimal create-if-missing
template (`AGENTS.minimal.template.md`, just H1 + `{{ tier1_sections }}`). Both
templates are user-overridable per project (`memory.agents_template`) or per home root
(`~/.config/sase/AGENTS.template.md`), so an override may contain arbitrary extra
headings. Two callers reach it, both in `src/sase/amd/_memory.py`:
`_render_managed_agents()` (`:304`) and `plan_minimal_agents_sync()` (`:353`). The
rendered text is then passed through `format_generated_memory_markdown()`
(`src/sase/main/init_memory/root_rendering.py:479`), which copies heading lines verbatim
and only rewraps prose.

### What reads these headings

- `parse_amd_agents_document()` (`src/sase/amd/_agents_doc.py:192`) locates the two tier
  blocks by **exact string match** against `_SHORT_SECTION_HEADINGS` /
  `_LONG_SECTION_HEADINGS` (`:12`, `:17`). It is used by `sase memory agent-docs list`
  (`src/sase/amd/inventory.py:_management_state_and_memory_counts`) to classify a doc as
  `managed` vs `custom`, by `_existing_agents_long_descriptions()`
  (`src/sase/amd/_memory.py:43`) to recover Tier 2 descriptions from a previous
  `AGENTS.md`, and by `_render_managed_agents()` itself (`:313`) to validate the freshly
  rendered document. **This is the one hard blocker**: numbering the `##` headings
  without loosening these matchers turns every managed doc into `custom` and makes
  rendering fail its own structural-anchor check.
- `_SHORT_MEMORY_HEADER_RE` (`src/sase/amd/_agents_doc.py:31`) already tolerates a
  number prefix (`^### (?:.* )?\((?P<name>...)\)$`), so note-level headers keep parsing.
- `_H2_RE = ^##\s+` (`:22`) bounds a section; `### Foo` does not match it (the third
  character is `#`), and `## 1. Foo` still does. No change needed.
- Nothing else parses these headings. The Rust core has no `AGENTS.md` heading code —
  its only `AGENTS.md` references are a filename constant in `content_layout.rs:1031`
  and memory-note frontmatter fixtures in `xprompt_catalog.rs`. The ACE glossary
  catalogs build from `sase.yml`, and `#memory/<note>` expansion reads the canonical
  note body, not the inlined copy.

### Rust core boundary

This stays in Python. Agent-instruction-file rendering is Python-side generation that no
other frontend consumes; `crates/sase_core` contains none of it (verified against the
opened `sase-core` linked checkout). No wire, binding, or `sase-core` change is needed.

## Goal

```
# Structured Agentic Software Engineering (SASE) - Agent Instructions
## 1. Tier 1 (short-term) Memory
### 1.1 Build & Run Commands (build_and_run)
#### 1.1.1 IMPORTANT: Two-Speed Verification — Run `just check` if you Made File Changes
#### 1.1.2 PNG Snapshot Tests
### 1.2 Code Conventions and Gotchas (gotchas)
### 1.3 Rust Core Backend Boundary (rust_core_backend_boundary)
### 1.4 SASE = Structured Agentic Software Engineering (sase)
#### 1.4.1 Ephemeral `sase_<N>` Workspace Directories
#### 1.4.2 Repositories
#### 1.4.3 File Discovered Work As Task Beads
## 2. Tier 2 (long-term) Memory
```

The H1 document title is never numbered. Everything below it is.

## Non-goals

- **No change to canonical notes under `sase/memory/`.** Numbering stays a render-time
  transform; the notes on disk stay unnumbered and remain the single source of truth, so
  `sase memory read`, `sase memory list`, the memory review TUI, and `#memory/<note>`
  expansion are all unaffected.
- **No change to `sase/memory/README.md`.** It is generated from
  `src/sase/main/init_memory/templates/memory-README.template.md` and is not an agent
  instruction file.
- **No numbering of Tier 2 entries.** ``**`sase/memory/x.md`**`` lines are bolded
  description-list items, not headings. "Section header title" does not cover them, and
  numbering them would collide with `_LONG_MEMORY_ENTRY_RE`.
- **No touching of unmanaged agent docs.** Hand-written files such as the chezmoi repo's
  own `AGENTS.md`, `home/lib/CLAUDE.md`, and `home/.config/nvim/CLAUDE.md` are not
  generated by `sase memory init` (the fallback path is `create_if_missing`) and stay
  exactly as they are.
- **No change to `validate_short_memory_structure()`'s contract.** H3-before-H2 stays
  legal and no new `sase memory init` blocker is introduced.
- **No change to heading shifting, fence handling, H1 consumption, the no-title
  fallback, or Tier 2 rendering.**

## Design

### 1. Move numbering to a single document-level pass

Numbering must move out of `inline_memory_section()` and become one fence-aware pass
over the **fully rendered** document. Reasons:

- The note-level number can no longer be computed in isolation: it now depends on which
  `##` section the note lands under, which is a property of the template (and templates
  are user-overridable).
- "Every section header title" includes template headings, which `inline_memory_section`
  never sees.
- Two numbering authorities would have to agree on a prefix; one authority cannot
  disagree with itself.

Add `src/sase/amd/_section_numbers.py` exporting a single public function:

```python
def number_agent_document_sections(text: str) -> str:
    """Return *text* with every heading below the title hierarchically numbered."""
```

A private module with public symbols matches the package's existing shape (`_memory.py`,
`_agents_doc.py`, `_template.py`), keeps symvision happy (a real non-test consumer
exists in `_template.py`), and still lets tests import it.

### 2. Numbering algorithm

Two passes over the lines, both skipping fenced content:

1. **Find the base level.**
   `base = min(level for level in heading_levels if level >= 2)`. If there is no such
   heading, return _text_ unchanged.
2. **Number.** Maintain a `counters: list[int]`. For each heading at `level`:
   - `level < base` (i.e. the H1 title): emit unchanged.
   - otherwise `depth = level - base + 1`; grow `counters` with zeros to `depth`,
     truncate anything deeper, increment `counters[depth - 1]`, and render the prefix as
     `".".join(counters)` — plus a trailing `.` when `depth == 1`.

The base-level rule is what keeps the minimal create-if-missing document correct. Its
headings are H1 + H3 (`### 1. sase (sase)`) with no H2 at all; `base = 3` reproduces
today's exact output there, whereas hard-coding H2 as the root would emit
`### 0.1 sase (sase)`. In the managed document `base = 2`, giving the Goal shape above.

The zero-fill is deliberate and preserves the precedent already set for inlined notes: a
heading whose parent level is absent reads `0` in that position (an H3 before the first
H2 becomes `0.1`). This is deterministic, obviously anomalous to a reader, matches
Pandoc's `--number-sections`, and costs nothing. Do not add a validator error for it —
that would turn a currently tolerated authoring style into a hard `sase memory init`
blocker in roots this change does not otherwise touch.

### 3. Prefix format

- Depth 1 keeps a trailing period: `## 1. Tier 1 (short-term) Memory`.
- Depth 2+ has no trailing period: `### 1.1 Build & Run Commands (build_and_run)`.

This is not a new convention — it is exactly today's `### 1. Title` / `#### 1.1 Title`
split, lifted one level. The bare digit needs the period to separate it from the title;
a dotted number does not. Keep the existing `_numbered_heading()`-style join that
filters empty parts, so a heading with no text (`##` alone matches `_HEADING_RE`, which
accepts `#+` followed by whitespace _or_ end of line) renders `## 1.` with no trailing
space.

Do **not** strip pre-existing number prefixes before renumbering. Stripping would make a
legitimate heading like `## 2026 Roadmap` silently lose its text, and there is no
feedback loop that could reintroduce a number: the document is rebuilt from the template
plus canonical (unnumbered) notes on every run, including under `--check`, so the pass
is idempotent at the file level without it. Cover the residual case in docs (design item
7).

### 4. Share the fence-aware heading primitives

`inline_memory.py` already owns fence-aware `_fence_marker()`, `_heading_level()`, and
`_iter_headings()` (`:28`, `:37`, `:49`). The numbering pass needs the same logic, and a
third copy inside `sase.amd` (there are already independent copies in
`init_memory/formatting.py` and `history/chat.py`) is a maintenance hazard on a function
whose whole job is "do not mistake a `# comment` in a bash fence for a heading".

Extract them into `src/sase/amd/_headings.py` as public `fence_marker()`,
`heading_level()`, and `iter_headings()`, and import them from both `inline_memory.py`
and `_section_numbers.py`. Symvision forbids importing `_`-prefixed _symbols_ across
files, so the moved helpers must lose their underscore; the module stays private. This
is a pure move — no behavior change, no signature change.

### 5. Wire the pass in and de-number `inline_memory`

- `src/sase/amd/_template.py`: apply `number_agent_document_sections()` to the rendered
  text before returning it from `render_agents_template()`. That single seam covers the
  managed path, the minimal fallback path, and any template override, and it runs before
  `format_generated_memory_markdown()`, which passes heading lines through verbatim.
- `src/sase/amd/inline_memory.py`: drop the `number` parameter from
  `inline_memory_section()` and `_shift_body()`, and drop the section/subsection
  counters. `_numbered_heading()` becomes a plain shift helper (rename to
  `_shifted_heading()`). Update the module docstring and `inline_memory_section()`'s
  docstring, both of which currently describe numbering, to say that numbering now
  happens document-wide in `_section_numbers`.
- `src/sase/amd/_memory.py`: drop `number=index + 1` (`:288`) and `number=1` (`:352`).
  The `enumerate()` at `:287` collapses back to a plain iteration over `bodies.items()`;
  the sort in `_short_memory_bodies()` still fixes the order, and now the numbers follow
  from that order via document position.

### 6. Loosen the tier-heading matchers

In `src/sase/amd/_agents_doc.py`, replace the `_SHORT_SECTION_HEADINGS` /
`_LONG_SECTION_HEADINGS` frozensets with compiled regexes that accept an optional
numeric prefix, applied to the same normalized line as today:

```python
_SHORT_SECTION_RE = re.compile(r"^##\s+(?:\d+(?:\.\d+)*\.?\s+)?Tier 1 \(short-term\) Memory$")
_LONG_SECTION_RE = re.compile(r"^##\s+(?:\d+(?:\.\d+)*\.?\s+)?Tier 2 \(long-term\) Memory$")
```

`_section_bounds()` takes the matcher instead of the frozenset. **Both the numbered and
the unnumbered form must keep matching**: every already-generated `AGENTS.md` in every
root and linked repo stays unnumbered until `sase memory init` next runs there, and
those docs must keep reporting as `managed` in `sase memory agent-docs list` and must
keep yielding their Tier 2 descriptions to `_existing_agents_long_descriptions()`.

Leave the two "missing structural anchor" error strings in `_memory.py:318`/`:324`
unchanged — they name the canonical heading text, which is still what an author needs to
put in a custom template.

### 7. Documentation

- `docs/memory.md:8-12` — the passage already shows worked examples
  (`### 1. Build & Run Commands (build_and_run)`, `#### 1.1 IMPORTANT: ...`). Update
  them to `### 1.1 ...` / `#### 1.1.1 ...` and say numbering now spans the whole
  document, starting at the `## 1. Tier 1 (short-term) Memory` heading.
- `docs/init.md:213` and `docs/configuration.md:3724` — both say "with generated heading
  numbers" for Tier 1 only. Widen to "every heading in the generated document".
- `docs/configuration.md:~493` (the template-override paragraph that already documents
  strict variable rendering) — add one clause: generated agent documents are numbered
  automatically, so custom `AGENTS.template.md` headings should not carry their own
  numbers.

Keep each edit to a clause or two; these are reference docs, not a tour of the scheme.

## Tests

### New — `tests/main/test_section_numbers.py`

Unit coverage for `number_agent_document_sections()`:

1. **Managed shape.** H1 + two H2s with H3/H4 children yields `## 1.`, `### 1.1`,
   `#### 1.1.1`, `## 2.` — the whole Goal shape in one assertion.
2. **Counters reset.** H2, H3, H3, H2, H3 yields `1.`, `1.1`, `1.2`, `2.`, `2.1`. This
   is the easiest thing to get wrong.
3. **H1 is never numbered**, including a document with several H1s.
4. **Base level is the shallowest heading below the title.** H1 + H3s (the minimal
   template shape) numbers the H3s `1.`, `2.` — pins the rule that keeps
   `plan_minimal_agents_sync()` output unchanged.
5. **Fenced hashes survive.** A ` ```bash ` block whose lines start with `#` (and a
   `~~~` block) is copied verbatim.
6. **Absent parent reads 0.** An H3 before the first H2 becomes `0.1`.
7. **Empty heading text.** `##` alone becomes `## 1.` with no trailing whitespace.
8. **No headings below the title / no headings at all** returns the text unchanged.
9. **Idempotence is not claimed.** Assert instead that running the pass on already
   numbered input double-prefixes, pinning the deliberate no-strip decision from design
   item 3 so a future change to it is a conscious one.

### Changed — `tests/main/test_inline_memory.py`

Drop every `number=` argument (`:101`, `:114`, `:130`, `:133`, `:140`, `:186`) and
assert that `inline_memory_section()` now emits unnumbered `### Title (stem)` headers
and plainly shifted body headings. The numbering assertions those tests carried are
replaced by the new module's tests, not deleted outright.

### Changed — parser and generator tests

These currently assert the unnumbered headings and will fail until updated; each is a
real assertion about generated output, so update rather than loosen:

- `tests/main/test_init_memory_managed_agents.py:130,141` — assert
  `## 1. Tier 1 (short-term) Memory` / `## 2. Tier 2 (long-term) Memory`. Also add the
  integration assertion that a short note with an H2 renders `### 1.1 ...` and
  `#### 1.1.1 ...`, so something pins that `_render_managed_agents()` actually reaches
  the numbered path. Note `:137` asserts a `####` line is _absent_ for a note without
  H2s — still true.
- `tests/main/test_init_memory_glossary.py:46-52` — splits the document on the two tier
  headings; update both split keys.
- `tests/main/test_init_memory_validation.py:145-146`,
  `tests/main/test_init_onboarding_memory.py:231`,
  `tests/main/test_init_memory_agent_docs.py:78`.
- `tests/main/test_init_memory_agents_templates.py:24,26,38,40,70,74` — hand-written
  expected documents and custom templates. The custom templates at `:70`/`:310` are
  _inputs_ and stay unnumbered; only the expected _outputs_ gain numbers. The
  structural-anchor blocker assertions at `:321-331` keep their current error strings.

### New — parser tolerance

In `tests/main/test_memory_agent_docs_list.py` (which already builds `AGENTS.md` text by
hand), add cases proving `parse_amd_agents_document()` recognizes both
`## Tier 1 (short-term) Memory` and `## 1. Tier 1 (short-term) Memory` (same for Tier
2), and that a numbered document still classifies as `managed` with the right memory
counts. The existing normalization case at
`tests/main/test_init_memory_agent_docs.py:41` (trailing whitespace) must keep passing.

### Expected to need no change — confirm, do not assume

`tests/test_memory_inventory.py` and the remaining `test_memory_agent_docs_list.py`
cases build unnumbered documents by hand and exercise only the parser; they are exactly
the regression net for the "legacy unnumbered docs still parse" requirement.

## Verification

Run everything from the workspace checkout.

1. `just install` first — workspace virtualenvs go stale between sessions.
2. Regenerate: `.venv/bin/sase memory init --no-commit`.

   Use `.venv/bin/sase`, not bare `sase`. A globally installed `sase` shadows the
   workspace's editable install on `PATH`, so bare `sase` silently regenerates from
   unmodified code and looks like a no-op.

3. Confirm the project-side diff is exactly `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
   `QWEN.md`, `OPENCODE.md`, the four shims byte-identical to `AGENTS.md`, and every
   changed line a heading. Expected result is the Goal block verbatim. Nothing under
   `sase/memory/` may change.
4. `.venv/bin/sase memory init --check` must then exit clean — this is the idempotence
   check for design item 3.
5. `.venv/bin/sase memory agent-docs list` must still report the managed roots as
   `managed` with unchanged short/long memory counts. Run it _before_ step 2 as well, so
   the numbered-vs-legacy parser tolerance is exercised in both directions.
6. `just check`, then `just check-full` before landing. The full suite is warranted:
   `sase validate` runs inside both, `AGENTS.md` generation is reached from several
   handler-level test modules, and the scoped selector infers blast radius from the
   import graph, which understates a change whose real output is a generated document.
   Run `just check-full` only through `/sase_monitor` with a `--next` action.
7. `just fmt-md-check` runs inside `just check`. The longest affected line —
   `#### 1.1.1 IMPORTANT: Two-Speed Verification — Run `just
   check` if you Made File Changes` — grows from 86 to 88 characters, exactly the
   configured `printWidth`. Prettier never wraps ATX headings and
   `format_generated_memory_markdown()` copies heading lines verbatim, so this is safe
   either way; verify rather than assume.

### The home root also changes — surface it, do not hide it

`sase memory init` plans two roots
(`src/sase/main/init_memory_handler.py:_memory_root_plans`): the project root and the
home root. With `use_chezmoi: true` the home root is the chezmoi source tree, a
different repository. Its generated `home/AGENTS.md` becomes:

```
## 1. Tier 1 (short-term) Memory
### 1.1 SASE = Structured Agentic Software Engineering (sase)
#### 1.1.1 Repositories
#### 1.1.2 File Discovered Work As Task Beads
## 2. Tier 2 (long-term) Memory
```

plus the four home provider shims, which `chezmoi apply` would then propagate to
`~/CLAUDE.md` and friends. This is intended — the request is about all agent instruction
files — but it is a cross-repo write, so:

- Use `/sase_repo` to reach the chezmoi checkout; never touch `~/.local/share/chezmoi/`
  by hand. Only `sase memory init` writes there.
- Run with `--no-commit` while iterating. `deploy_to_chezmoi()` returns early on
  `no_commit`, so files land in the chezmoi source tree without being staged, committed,
  or applied.
- Confirm the chezmoi repo's own `AGENTS.md`, `home/lib/CLAUDE.md`, and
  `home/.config/nvim/CLAUDE.md` are untouched — they are hand-written, not generated.
- Before finishing, report the home-root file list to the project owner and let them
  decide whether to let the normal `sase memory init` chezmoi deploy commit and apply
  it. Do not commit to the chezmoi repo unprompted.

### Committing

Commit the source change, the tests, the doc edits, and the regenerated project-side
`AGENTS.md` plus the four provider shims together through the `/sase_git_commit`
workflow, naming each file with its own `-f` flag.

Regenerating `AGENTS.md` and the provider shims is normally gated by the "Memory File
Edits Require Explicit User Permission" convention. That permission is present: the
project owner asked for this rendering change directly, which carries the regeneration
it implies. The convention still forbids hand-editing any of those files — every byte of
their diff must come from `sase memory init`.

## Rollout notes

This is a presentation-only change to generated files. Numbers are derived at render
time from the document's own heading tree, so there is no state, config, cache, or
migration to manage, and `--check` is stable immediately after the first regeneration.

Section numbers are positional, not stable identifiers. Adding a short note whose path
sorts early, or adding a section mid-note, renumbers everything after it — the same
property today's `### N.` numbers already have. Anything needing a durable reference
should keep quoting the heading title.

Other roots with a managed `AGENTS.md` (`sase-github`, `sase-telegram`, `sase-nvim`,
`sase-research`, and any other SASE project) pick up the new format the next time
`sase memory init` runs there. Nothing breaks in the interim: the loosened matchers in
design item 6 accept both forms, so mixed-state roots keep classifying and parsing
correctly, and no coordinated rollout is required.
