---
tier: tale
goal: Restore Jinja input substitution inside inline-code formatting for xprompt and
  prompt-input templates without weakening launch-time literal-zone escaping.
create_time: 2026-07-15 15:46:11
status: done
prompt: 202607/prompts/inline_jinja_template_literals.md
---

# Plan: Restore inline-formatted xprompt input rendering

## Context and root cause

The `toobig_split` chop in the linked chezmoi repository is behaving correctly at its boundary: it discovers the
oversized file and launches an agent with `#split_file:<path>`. The failed `split_file.projects_pane-2` artifacts also
record the expected positional `file_path` argument. The defect is in SASE's built-in xprompt rendering: expansion of
`#split_file:src/sase/ace/tui/modals/projects_pane.py` produces the body with literal `` `{{ file_path }}` `` text.

The regression was introduced when inline Markdown code became a launch-time literal zone. The legacy
`protect_fenced_blocks()` helper was deliberately broadened to protect both fenced and inline code so xprompt
references, directives, and alternatives inside backticks remain inert. `substitute_placeholders()` also calls that
helper before rendering a declared Jinja template, which unintentionally hides legitimate template expressions that are
merely presented with inline-code formatting. This affects the built-in `split_file` xprompt and the shared
declared-input rendering path; it is not a chop-script argument, project-resolution, or chezmoi configuration error.

## Implementation

1. Separate the two protection intents in the xprompt literal-zone utility. Keep the existing broad helper and its
   default behavior for launch parsing, directive extraction, metadata inspection, and top-level literal escaping. Add a
   narrowly named fence-only protection path that uses the existing canonical fenced-block ranges and the same
   placeholder/restore mechanism, avoiding a second fence parser.
2. Use fence-only protection while substituting declared xprompt/prompt-input Jinja values. This restores historical
   rendering for expressions inside inline Markdown code while fenced code remains verbatim. Do not change launch-time
   processing of already-rendered text: inline `#xprompt`, `%directive`, and alternative syntax must still be protected
   once generated content returns to the normal expansion pipeline.
3. Add focused regression coverage for the semantic boundary:
   - a typed xprompt input renders inside backticks;
   - an xprompt-looking token beside that rendered value but within the same inline-code span remains literal after
     recursive processing;
   - Jinja inside fenced code remains untouched;
   - the packaged `split_file` xprompt expands its colon-form path and does not leak `{{ file_path }}`. Preserve and run
     the existing inline-code literal-zone tests so the fix cannot regress the feature that exposed the bug.

## Validation

- Install the checkout as required by the repository instructions, then run focused xprompt/Jinja, inline-literal, and
  packaged-xprompt tests while iterating.
- Reproduce the original invocation through the shared processor and confirm the rendered prompt contains the actual
  `projects_pane.py` path while retaining Markdown backticks.
- Run `just check` for the full repository gate before completion.

No chezmoi source change is expected: its chop script and `sase_athena.yml` configuration already produce the correct
`#split_file:<path>` launch prompt. The fix belongs in SASE's shared Python xprompt template-rendering path; it does not
require a Rust wire/API change because it corrects the existing Python launch renderer rather than adding new backend
domain behavior.
