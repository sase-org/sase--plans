---
tier: tale
title: Number short-term memory subsections in agent instruction files
goal:
  Every subsection of every short-term memory note carries a hierarchical section number
  when inlined into an agent instruction file — `#### 1.2 Section` for the 2nd section
  of the 1st Tier 1 memory, `##### 1.2.1 Subsection` beneath it — so agents and humans
  can name any part of `AGENTS.md` by number instead of by prose title.
size: medium
proposed_by: bbugyi200.athena.we.f0
create_time: 2026-08-09 09:28:53
status: wip
---

# Plan: Number short-term memory subsections in agent instruction files

## Context

`sase memory init` inlines every `type: short` memory note into the
`## Tier 1 (short-term) Memory` block of the managed `AGENTS.md`, then copies that file
byte-for-byte to each provider shim (`CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
`OPENCODE.md`).

The single seam that performs the inlining is `inline_memory_section()` in
`src/sase/amd/inline_memory.py:145`. It:

1. consumes the note's H1 into a section header, already numbered —
   `### {number}. {title} ({basename})`;
2. calls `_shift_body()` (`src/sase/amd/inline_memory.py:107`) to strip that H1 and
   shift every remaining heading down two levels (H2 -> H4, H3 -> H5), fence-aware, so
   `#` lines inside ` ``` ` blocks are copied verbatim.

`validate_short_memory_structure()` (`src/sase/amd/inline_memory.py:79`) guarantees the
input shape: exactly one H1, it comes first, and no heading is deeper than H3. So an
inlined note contributes at most two levels below its `###` header.

Only the top level is numbered today. The result reads:

```
### 1. Build & Run Commands (build_and_run)
#### IMPORTANT: Two-Speed Verification — Run `just check` if you Made File Changes
##### Exceptions
#### PNG Snapshot Tests
### 2. Glossary of Terms (glossary)
#### Agent Clan
```

There are exactly two callers, both in `src/sase/amd/_memory.py`, and both already pass
a number:

- `_render_managed_agents()` (`src/sase/amd/_memory.py:275`) passes `number=index + 1`
  over `_short_memory_bodies()`, which is sorted by root-relative path — that sort is
  what makes the numbering deterministic.
- `plan_minimal_agents_sync()` (`src/sase/amd/_memory.py:339`) passes `number=1` for the
  create-if-missing fallback document.

The request is to extend numbering to the sections _inside_ each inlined note, for all
short-term memories rather than only the glossary's entry sections.

### Why this is safe to change

Nothing parses these inlined subsection headings.

- `_SHORT_MEMORY_HEADER_RE` in `src/sase/amd/_agents_doc.py:30` is anchored to `^### `
  and therefore only matches the note-level headers.
  `tests/main/test_memory_agent_docs_list.py:277` exists specifically to pin that
  inlined `####` body headings are _not_ miscounted as short-memory references.
- The ACE glossary underline/definition feature does not read `sase/memory/glossary.md`.
  Both `src/sase/xprompt/glossary_catalog.py` and `src/sase/ace/tui/glossary_catalog.py`
  build their catalogs from the `glossary:` block of the project's `sase.yml`.
  Renumbering the glossary note's H2 term headings cannot affect term matching.
- The `#memory/<note>` xprompt expansion reads the canonical note body
  (`crates/sase_core/src/xprompt_catalog.rs`), not the inlined copy, so it is untouched.
- No source file, skill template, or doc cross-references a Tier 1 subsection by its
  heading text in a way that numbering would break.

### Rust core boundary

This stays in Python. I checked `../sase-core`: `crates/sase_core` has no `AGENTS.md`
short-memory inlining code at all — its only references to `AGENTS.md` are memory-note
frontmatter fixtures in `xprompt_catalog.rs` and a filename constant in
`content_layout.rs`. Agent-instruction-file rendering is Python-side generation that no
other frontend consumes, so `sase.amd.inline_memory` remains the right home and no
wire/binding change is needed.

## Goal

When a short-term memory note is inlined into an agent instruction file, each of its
sections carries a hierarchical number derived from the note's Tier 1 position:

```
### 1. Build & Run Commands (build_and_run)
#### 1.1 IMPORTANT: Two-Speed Verification — Run `just check` if you Made File Changes
##### 1.1.1 Exceptions
#### 1.2 PNG Snapshot Tests
### 2. Glossary of Terms (glossary)
#### 2.1 Agent Clan
#### 2.2 Agent Family
```

## Non-goals

- No change to the canonical notes under `sase/memory/`. Numbering is applied at inline
  time only; the notes on disk stay unnumbered and stay the single source of truth. This
  also keeps `sase memory read`, `sase memory list`, the memory review TUI, and
  `#memory/<note>` xprompt expansion unchanged.
- No change to the existing `### {number}. {title} ({basename})` note-level header,
  including its trailing period.
- No change to `validate_short_memory_structure()`'s contract. H3-before-H2 stays legal
  (see design item 3); no new blocker is introduced that could stop `sase memory init`
  in the home root or a linked repo.
- No change to heading shifting, fence handling, H1 consumption, the no-title fallback,
  or the Tier 2 rendering path.
- No renumbering of `sase/memory/README.md`'s own headings — that file is generated from
  `src/sase/main/init_memory/templates/memory-README.template.md` and is not a Tier 1
  inline target.

## Design

### 1. Thread the note number into `_shift_body()`

In `src/sase/amd/inline_memory.py`, give `_shift_body()` a keyword-only
`number: int | None = None` parameter and have `inline_memory_section()` forward the
number it already receives. Numbering must happen inside `_shift_body()`, not as a
post-pass over its output, because that function is the only place that knows which `#`
lines are headings and which are fenced content.

Keep the existing single-pass loop. Add two counters alongside `h1_consumed`:

- On a level-2 heading: increment the section counter, reset the subsection counter,
  prefix with `{number}.{section}`.
- On a level-3 heading: increment the subsection counter, prefix with
  `{number}.{section}.{subsection}`.

When `number is None`, emit the shifted heading exactly as today. Both production
callers always pass a number, so this branch only preserves the documented pure-function
contract that the unit tests exercise.

Levels other than 2 and 3 (a stray second H1, or an H4+ that
`validate_short_memory_structure()` would have rejected) are shifted but left
unnumbered. `_shift_body()` must stay total: it is a pure helper that tests call with
deliberately invalid bodies, and it should not raise or produce a nonsense number for
input the validator already guards.

### 2. Prefix format

Render as `#### {number}.{section} {title}` — a space between the number and the title,
and no trailing period.

This matches the numbers in the request (`1.2`, `3.1`) and is the conventional shape for
multi-level section numbers. The note-level header keeps its `### 1. Title` trailing
period; that period is what separates a bare digit from the title text, and a dotted
number does not need it. The asymmetry is deliberate, not an oversight.

Extract a small module-private helper rather than inlining the string surgery in the
loop, e.g.:

```python
def _numbered_heading(line: str, level: int, prefix: str | None) -> str:
    """Return *line* shifted two levels, with *prefix* inserted before its text."""
    text = line[level:].strip()
    hashes = "#" * (level + 2)
    parts = [part for part in (hashes, prefix, text) if part]
    return " ".join(parts)
```

Filtering empty parts handles a heading with no text (`##` alone is a legal match for
`_HEADING_RE`, which accepts `#+` followed by whitespace _or_ end of line) without
emitting a trailing space.

### 3. Numbering when an H3 precedes any H2

`validate_short_memory_structure()` permits an H3 before the first H2, and that shape
occurs in this repo today (`sase/memory/generated_skills.md:19` opens with
`### Commit First, Then Deploy`). It happens to be a long note, so it is not inlined —
but short notes in the home root and in linked repos are outside this repo's control,
and `inline_memory_section()` must not produce garbage or crash on one.

Number such a heading `{number}.0.{subsection}`, i.e. an absent parent section reads as
`0`. This is deterministic, obviously anomalous to a reader, matches how Pandoc's
`--number-sections` handles skipped levels, and costs nothing: the section counter
simply starts at `0` and is not special-cased. Do not add a validator error for this
shape — that would turn an existing, tolerated authoring style into a hard
`sase memory init` blocker in roots this change is not otherwise touching.

### 4. Docstrings

Update the module docstring (`src/sase/amd/inline_memory.py:1-15`) and
`inline_memory_section()`'s docstring, both of which currently describe the shift as a
pure level change. State that when a number is supplied, H2 and H3 headings are also
numbered `{number}.{section}` and `{number}.{section}.{subsection}`.

### 5. Documentation

Three prose passages claim short notes are inlined _verbatim_, which stops being true:

- `docs/memory.md:8-9` — also update the parenthetical, which still shows the
  pre-numbering `### Title (<file>)` shape and is already stale relative to today's
  `### N. Title (<file>)`. Show both levels of the real output.
- `docs/init.md:211-212`
- `docs/configuration.md:3513`

Keep the edits to one clause each; these are reference docs, not a tour of the numbering
scheme.

## Tests

All new unit tests go in `tests/main/test_inline_memory.py`, which already owns this
module's contract.

1. **Sections are numbered under the note number.** A body with two H2s, inlined with
   `number=3`, yields `#### 3.1 ...` and `#### 3.2 ...`.
2. **Subsections nest and reset.** A body shaped H2, H3, H3, H2, H3 yields `4.1`,
   `4.1.1`, `4.1.2`, `4.2`, `4.2.1` — this is the test that pins the counter reset,
   which is the easiest thing to get wrong.
3. **The realistic fixture round-trips.** Extend the existing `BUILD_AND_RUN_BODY`
   coverage with a numbered assertion over the whole rendered block (mirroring
   `test_inline_copies_code_fences_verbatim`), asserting both that headings are numbered
   and that the fenced `# Install in editable mode...` comment lines are still
   untouched.
4. **Fenced hashes are never numbered.** Assert directly that a `# shell comment` inside
   a fence survives a numbered inline unchanged, so the fence guard is pinned
   independently of the fixture test.
5. **`number=None` is unchanged.** The existing
   `test_inline_shifts_headings_down_two_levels` already covers this; add an explicit
   assertion that no digit-dot prefix appears, so a future default flip cannot pass
   silently.
6. **H3 before H2.** Assert the `{number}.0.1` fallback from design item 3.
7. **Empty heading text.** `"# T\n\n##\n"` with a number renders `#### 1.1` with no
   trailing whitespace.

Existing tests that should need no change — confirm rather than assume:

- `tests/main/test_init_memory_managed_agents.py`, `test_init_memory_agent_docs.py`,
  `test_init_memory_glossary.py`, `test_init_memory_agents_templates.py`,
  `test_init_memory_handler_repositories.py` all assert on `### N. Title (file)`
  note-level headers only. `test_init_memory_managed_agents.py:137` asserts a `####`
  line is _absent_, which stays true.
- `tests/main/test_memory_agent_docs_list.py` and `tests/test_memory_inventory.py` build
  `AGENTS.md` text by hand and only exercise the parser, which never sees the
  generator's output.

Add one integration-level assertion in `tests/main/test_init_memory_managed_agents.py`:
render a managed `AGENTS.md` from a short note that has an H2, and assert the numbered
`####` heading appears. Without it, every numbering test lives in the pure-helper module
and nothing pins that `_render_managed_agents()` actually reaches the numbered path.

## Verification

Run everything from the workspace checkout.

1. `just install` first — workspace virtualenvs go stale between sessions.
2. Regenerate: `.venv/bin/sase memory init --no-commit`.

   Use `.venv/bin/sase`, not bare `sase`. A globally installed `sase` shadows the
   workspace's editable install on `PATH`, so bare `sase` will silently regenerate from
   unmodified code and appear to be a no-op.

3. Confirm the project diff is exactly `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
   `OPENCODE.md`, with the four shims byte-identical to `AGENTS.md`. Expected in this
   repo: `build_and_run` gains `1.1`, `1.1.1`, `1.2`; `glossary` gains `2.1` through
   `2.17`; `sase` gains `5.1`, `5.2`, `5.3`. `gotchas` (#3) and
   `rust_core_backend_boundary` (#4) have no H2s and must be untouched — they use bolded
   standalone labels, not headings, and are the check that numbering did not leak into
   non-heading lines. No file under `sase/memory/` should change.
4. `.venv/bin/sase memory init --check` must exit clean afterward.
5. `just check`, then `just check-full` before landing. The full suite is warranted
   here: `sase validate` runs inside both, `AGENTS.md` generation is reached from
   several handler-level test modules, and the scoped selector infers blast radius from
   the import graph, which understates a change whose real output is a generated
   document.
6. `just fmt-md-check` runs inside `just check` and covers the regenerated Markdown.
   Prettier does not reflow ATX headings, so the longest affected line (82 chars,
   becoming 86) stays inside the 88-column width either way.

### The home root also changes — surface it, do not hide it

`sase memory init` plans two roots
(`src/sase/main/init_memory_handler.py:_memory_root_plans`): the project root and the
home root. With `use_chezmoi: true` the home root is `~/.local/share/chezmoi/home`, a
different repository. Its `sase.md` short note has two H2 sections, so this change also
renumbers `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and `OPENCODE.md` there as
`1.1 Repositories` and `1.2 File Discovered Work As Task Beads`, which `chezmoi apply`
would then propagate to `~/CLAUDE.md` and friends.

This is intended — the request is about all agent instruction files — but it is a
cross-repo write, so:

- Run with `--no-commit` while iterating. `deploy_to_chezmoi()` returns early on
  `no_commit`, so files are written into the chezmoi source tree but nothing is staged,
  committed, or applied.
- Never hand-edit anything under `~/.local/share/chezmoi/`. Only `sase memory init`
  writes there.
- Before finishing, report the home-root file list to the project owner and let them
  decide whether to let the normal `sase memory init` chezmoi deploy commit and apply
  it. Do not commit to the chezmoi repo unprompted.

### Committing

Commit the generator change, the tests, the doc edits, and the regenerated project-side
`AGENTS.md` plus the four provider shims together, through the `/sase_git_commit`
workflow, naming each file with its own `-f` flag.

Regenerating `AGENTS.md` and the provider shims is normally gated by the "Memory File
Edits Require Explicit User Permission" convention. That permission is present: the
project owner asked for this rendering change directly, which carries the regeneration
it implies. The convention still forbids hand-editing any of those files — every byte of
their diff must come from `sase memory init`.

## Rollout notes

This is a presentation-only change to generated files. Numbers are derived at render
time from the note ordering that `_short_memory_bodies()` already produces, so the
transformation is idempotent and there is no state, config, cache, or migration to
manage.

Section numbers are positional, not stable identifiers. Adding a short note whose path
sorts early, or adding a section mid-note, renumbers everything after it. That is
inherent to positional numbering and is the same property the existing `### N.` note
numbers already have; anything that needs a durable reference should keep quoting the
heading title.

Other repos with their own managed `AGENTS.md` (`sase-github`, `sase-telegram`,
`sase-nvim`, and the chezmoi home root) pick up the new format the next time
`sase memory init` runs there. Nothing breaks in the interim — the numbering is cosmetic
and the parser ignores these heading levels entirely — so no coordinated rollout is
required.
