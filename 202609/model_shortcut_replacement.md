---
tier: epic
title: Replace existing model directives on shortcut completion
goal:
  Accepting =alias or ==model yields one standalone model directive per prompt segment
  while preserving alternation branches, with matching behavior in the prompt widget and
  LSP.
phases:
  - id: core_selection
    title: Implement shared model edits and protect alternation targets
    description:
      "core_selection: implement and test alternation-aware model and project-tag
      selection in sase-core, including LSP completion edits."
    depends_on: []
    size: medium
  - id: widget_adoption
    title: Apply shared edits in the prompt widget
    description:
      "widget_adoption: adopt the core selection result in the Python widget, update the
      core revision pin, and verify cross-frontend parity."
    depends_on:
      - core_selection
    size: medium
proposed_by: bbugyi200.apollo.1x
create_time: 2026-09-26 10:14:28
status: wip
---

- **PROMPT:**
  [prompts/202609/model_shortcut_replacement.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/model_shortcut_replacement.md)

# Outcome

The `=` alias and `==` concrete-model menus continue to filter and display the same
rows. Accepting either row still expands the shortcut to the canonical `%m:<value>`
spelling. When the active `---` prompt segment has a standalone `%model`/`%m` directive,
the selected value occupies the earliest such directive's position, the typed shortcut
disappears, and any further standalone model directives in that segment are removed.
This handles a directive before or after the shortcut without moving the selected
directive to the front. When there is no eligible standalone directive, preserve today's
token-local expansion, trailing-space policy, and caret position; in particular, do not
adopt the project-tag leading-insertion rule for model shortcuts.

An alternation body is an independent edit boundary. Existing model directives and
workspace targets within `%alt(...)`, `%(...)`, and `%{...}` must never be selected as
replacement targets or deleted by a completion outside that body. A shortcut or
project-tag trigger _inside_ an alternation expands at its own token without rewriting
other branches or the surrounding prompt. Existing resolved project tags and VCS refs in
alternation bodies remain intact when a project row is accepted elsewhere. Only
standalone, nonliteral targets in the same `---` segment are eligible; other segments
and frontmatter, fenced/inline code, disabled regions, and definition regions remain
untouched. Preserve the current treatment of unresolved tags as ordinary text.

# Phase 1: shared Rust selection and LSP

Open the linked `sase-core` repository through `sase repo open sase-core` before working
there. In `crates/sase_core/src/editor/model_alias_shortcut.rs`, retain the current
`=`/`==` trigger, catalog validation, and model-value checks, but extend the accept
planner to produce a deterministic, nonoverlapping edit set in original-document
coordinates plus the post-edit caret. Parse eligible `%model` and `%m` surface forms
using the existing directive grammar, including colon and parenthesized values, rather
than matching arbitrary `%m` text. Exclude alternation bodies and literal/definition
regions, and scope replacement to the shortcut's prompt segment. Preserve current
single-edit output exactly when no eligible directive exists. Evolve the Rust wire and
PyO3 bindings compatibly so old readers of the current shortcut response continue to
work where possible; if a schema bump is necessary, update both frontends and their
schema checks together.

Reuse the launch parser's alternation delimiters and matching rules as a shared lexical
helper, avoiding a second approximate `%alt` regex. Apply its protected ranges to the
target search in `crates/sase_core/src/project_tag/accept.rs` without making project
tags inert for highlighting or launch parsing. A `+` trigger inside an alternation stays
local; outside it, the existing target-position behavior still applies to eligible
standalone tags and refs. Update `crates/sase_xprompt_lsp` to place the shortcut
deletion in the primary `textEdit` and any existing-directive replacement and
extra-directive removals in `additionalTextEdits`, merging a coincident range if needed.
All LSP ranges must use UTF-16 coordinates and be nonoverlapping. Keep catalog order,
filtering, metadata, and `isIncomplete` behavior unchanged.

Add focused Rust core, PyO3 round-trip, and LSP tests for both shortcut kinds: directive
before and after the trigger, `%m`/`%model` forms, repeated standalone directives, token
in the middle of text, Unicode offsets, spacing/newlines, separate `---` segments, no
existing directive, and protected text. Exercise all three alternation spellings,
including named branches, nested arguments, and model directives or project tags/VCS
refs in branches, with triggers both inside and outside an alternation. Assert the
**applied document**, caret, and LSP edit disjointness, not only row labels. Run
targeted tests while iterating, then the linked repo's required `sase tool run check`
gate.

# Phase 2: prompt widget adoption and parity

In `src/sase/ace/tui/widgets/model_alias_completion.py`, `model_explicit_completion.py`,
and `_file_completion_accept_kinds.py`, consume the shared multi-edit/final-caret result
for `=` and `==`. Apply all edits as one logical acceptance with valid offsets and
preserve the widget's completion dismissal and undo behavior. Do not duplicate Rust
target scanning in Python. Keep the existing compatibility path for older shortcut
marker bindings unless the new wire explicitly supersedes it. Advance
`sase-core-revision.txt` past the core commit before relying on the new binding in sase
CI.

Add prompt-widget acceptance tests for the same before/after, alias/concrete-model,
no-existing-directive, alternation, project-tag, caret, and whitespace cases, using
existing model and project completion test modules. Verify both frontends produce the
same final prompt for representative inputs. Update relevant completion documentation if
the new acceptance behavior is documented there. Run focused widget tests and the
required `sase tool run check` for the sase checkout; do not initiate `check-full`
unless explicitly requested or needed to repair a reported failure.

# Acceptance examples

- `Use =la` with `@large` selected stays `Use %m:@large `: the new directive remains at
  the shortcut position.
- `%model:old Use =la` with `@large` selected ends with one model directive at the
  former `%model:old` position and no `=la` token.
- `Use ==gp then %m:old` with a concrete model selected ends with one `%m:<selected>` at
  the former `%m:old` position and no `==gp` token.
- `%{%m:opus | %m:sonnet} Use =la` preserves both branch directives; `+` selection
  elsewhere likewise preserves branch project tags and VCS refs.
- A shortcut or `+` trigger inside an alternation changes only its own token; sibling
  branches, enclosing text, and standalone targets outside the alternation are
  preserved.
