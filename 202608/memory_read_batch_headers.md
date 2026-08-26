---
tier: tale
title: Label files in batched reference-memory reads
goal:
  Agents can unambiguously associate every flat memory file with its content in batched
  Markdown output without changing single-note output.
size: small
proposed_by: bbugyi200.athena.0ef
---

- **AGENTS:**
  - [bbugyi200.athena.0ef](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ef.md)
- **COMMITS:**
  - [95dc60c](https://github.com/sase-org/sase/commit/95dc60c5aa2fc1ddf649d3bd46fac6e9c0e75c01)
    — feat(memory): add MEMORY FILE header to multi-note selector batch renders

# Label each reference-memory file in batched Markdown reads

## Goal

Make the default Markdown output from `sase memory read` unambiguous when a resolved
selector batch contains multiple reference-memory sources: every flat memory-note body
must begin with a clear header naming that note. Preserve the existing byte-for-byte
output for the common single-note invocation.

## Current behavior

`sase memory read` and `sase memory show` share selector resolution and rendering.
`src/sase/memory/selector_render.py` delegates a batch containing exactly one flat note
and no web sections to the legacy single-note renderer, but its Markdown batch path
currently concatenates `memory_note_markdown(note)` results with blank lines. Because
those note bodies commonly begin with their own topical headings, an agent cannot
reliably identify where one memory file ends and the next begins. Memory-web sections
already identify themselves with `MEMORY WEB: <slug>`; Rich note output already has a
`MEMORY <path>` header; JSON already carries each note's path as structured data.

The shared Markdown batch helper is also used when materializing deferred memory-read
reports, so fixing it should keep live output and recorded reports aligned without a
second rendering implementation.

## Implementation

1. In `src/sase/memory/selector_render.py`, add a small Markdown note-section renderer
   that prefixes a resolved flat note with a stable, human-readable header containing
   its canonical memory path, then includes the existing `memory_note_markdown(note)`
   result so child-note listings and newline normalization remain centralized.
2. Use that labeled note section for every flat note handled by `_batch_markdown`. Keep
   the `batch.is_single_note` fast path unchanged so `sase memory read foo.md` remains
   byte-identical, while batches of two or more notes and mixed note/web batches receive
   headers. Retain the current deterministic note/web ordering and the existing
   blank-line separation between sections.
3. Do not alter Rich or JSON output: Rich already labels each note, and JSON consumers
   already receive explicit `path` and `canonical_path` fields. Because `read` and
   `show` intentionally share rendering, do not fork behavior between the two commands.

## Tests

1. Extend `tests/main/test_memory_read_selectors.py` with a CLI-level regression test
   that seeds two reference notes with distinguishable bodies, invokes
   `handle_memory_read_command` with both selectors in one command, and asserts that the
   Markdown output contains one correctly named note header immediately before each
   corresponding body in deterministic order. Continue to exercise the audited `read`
   entry point named in the user-facing requirement.
2. Add focused renderer assertions (in a dedicated selector-render test module if that
   keeps responsibilities clearer) covering:
   - a single flat-note batch remains exactly the original note Markdown with no new
     wrapper header;
   - a mixed flat-note/memory-web batch labels the note and retains the existing
     `MEMORY WEB: <slug>` separator;
   - note rendering still includes any generated `## Children` section beneath the
     correct file header.
3. Confirm the existing memory-read report tests still pass, since reports consume
   `memory_selector_batch_markdown`; strengthen the mixed report assertion to check the
   flat note's file header if needed to make that inherited behavior explicit.

## Verification

Run the focused selector, CLI, and report tests while iterating, then run the repository
default verification gate required for changed SASE files:

```bash
pytest tests/memory/test_memory_selector_render.py tests/main/test_memory_read_selectors.py tests/test_memory_read_report.py
just check
```

If the focused renderer coverage is placed in an existing module rather than the
suggested new file, adjust the focused pytest path accordingly.

## Acceptance criteria

- A Markdown `sase memory read first.md second.md -r "..."` result clearly labels both
  `first.md` and `second.md`, with each label adjacent to its own content.
- A mixed note/web Markdown batch clearly labels both the flat note section and the
  existing memory-web section.
- A one-note Markdown read has exactly the same bytes as before this change.
- Rich and JSON output contracts are unchanged.
- Deferred memory-read reports use the same labeled Markdown sections without a
  duplicate implementation.
- Focused tests and `just check` pass.
