---
tier: tale
title: Delimit memory batch section headers
goal:
  Make every labeled Markdown section in a multi-selector memory read visually distinct
  with a leading blank line and a ten-dash header prefix.
size: small
proposed_by: bbugyi200.athena.0fx
---

- **AGENTS:**
  - [bbugyi200.athena.0fx](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fx.md)
- **COMMITS:**
  - [2069fce](https://github.com/sase-org/sase/commit/2069fcea0b78076ebfbf1cf071bdaecd0d8bc9e3)
    — feat(memory): delimit batch Markdown section headers

# Plan: Delimit memory batch section headers

## Intended behavior

For any selector batch that uses the combined Markdown renderer, render each flat-note
section header as `---------- MEMORY FILE: <canonical-path>` and each grouped memory-web
section header as `---------- MEMORY WEB: <slug>`. Start every such header on a new line
after one blank line, including the first header in the command output, so the first
section begins with `\n---------- MEMORY ...` and later sections remain visibly
separated from the preceding body.

Keep an exact single-note read byte-for-byte compatible and header-free. Do not change
selector resolution, web/strand grouping, audit metadata, note or strand bodies, the
blank line already present after each header, or the JSON and Rich renderers.

## Implementation

1. Update `src/sase/memory/selector_render.py` so the note and web Markdown section
   builders share the exact ten-dash prefix and leading-newline convention. Keep the
   decoration local to the combined batch Markdown path, which is shared by
   `sase memory read`, `sase memory show`, and generated memory-read reports, while
   leaving the single-note fast path and other output formats untouched.
2. Strengthen `tests/memory/test_memory_selector_render.py` with exact boundary
   assertions for multi-note and mixed note/web batches. Verify the leading blank line,
   the exact `---------- MEMORY FILE:` and `---------- MEMORY WEB:` spellings, the
   separator before subsequent sections, section/body ordering, and the unchanged
   single-note output. Retain coverage for nested child listings beneath their owning
   note header.
3. Update the CLI-level assertion in `tests/main/test_memory_read_selectors.py` and the
   downstream report assertion in `tests/test_memory_read_report.py` to require the new
   decorated headers, demonstrating that both audited command output and re-rendered
   reports inherit the shared format without changing their selector ordering or
   contents.
4. Refresh `docs/memory.md` to document the exact decorated `MEMORY FILE` and
   `MEMORY WEB` boundary syntax and its leading blank line, while preserving the
   documented single-note compatibility guarantee.

## Verification

1. Run the focused renderer, CLI selector, and read-report tests:
   `pytest tests/memory/test_memory_selector_render.py tests/main/test_memory_read_selectors.py tests/test_memory_read_report.py`.
2. Run `just fmt` and review its diff so formatting changes remain limited to the files
   in this plan.
3. Run the required repository gate, `just check`, and resolve any selected test, lint,
   type-check, or documentation failures before handing off the completed change.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact                              | Why                                                             | Uses |
| -------- | ------------------------------------- | --------------------------------------------------------------- | ---: |
| cited-by | [agent:bbugyi200.athena.0fx--code][1] | prompt reference @plan:202608/memory_batch_header_separators.md |    1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fx.md

<!-- sase:referenced-by:end -->
