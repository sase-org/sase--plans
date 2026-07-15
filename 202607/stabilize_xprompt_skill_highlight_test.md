---
tier: tale
goal: Make the xprompt syntax-highlighting test deterministic in a core-only development
  environment while preserving coverage of project-scoped slash-skill highlighting.
create_time: 2026-07-15 09:17:28
status: wip
prompt: 202607/prompts/stabilize_xprompt_skill_highlight_test.md
---

# Plan: Stabilize project-scoped xprompt skill highlighting coverage

## Context and root cause

`test_xprompt_highlight_overlay_marks_spans_and_registers_styles` seeds its warm skill catalog under the project key
`sase`, then asks the prompt widget to derive that key from a leading `#gh:sase` reference. The minimal
`CompletionTestApp` does not install or register the optional GitHub workspace-provider plugin; a core-only installation
exposes only the built-in `git` provider. Consequently, `extract_vcs_workflow_tag` treats `#gh:sase` as an ordinary
xprompt rather than a workspace tag, the widget looks in the default catalog instead of the seeded `sase` catalog, and
`/sase_plan` is not recognized as a known skill. The generic invocation, directive, separator, and theme assertions
still pass, which leaves only the skill assertion failing.

This is a test-isolation regression rather than a production tokenizer or cache defect. A controlled probe using the
built-in `#git:sase` reference resolves the same project key, retrieves the seeded warm catalog, and emits the expected
`xprompt.skill` span without synchronous catalog work.

## Implementation

- Update the focused overlay test in `tests/ace/tui/widgets/test_prompt_xprompt_highlight.py` to use the
  always-available built-in workspace reference while retaining the `sase` project-scoped catalog seed.
- Preserve the existing assertions for invocation and argument spans, directives and their arguments, segment
  separators, slash-skill recognition, and registered theme styles. Do not weaken the test by moving the skill entry to
  the default catalog or by teaching production code to recognize an unregistered provider.
- Keep production highlighting, provider discovery, and memoization unchanged because their current behavior correctly
  avoids synchronous loading and scopes catalogs through registered workspace providers.

## Validation

- Run the formerly failing test in isolation to confirm it no longer depends on an optional plugin installation.
- Run the complete xprompt highlight widget test module to cover cold-catalog behavior, project-aware memoization,
  protected regions, theme switching, and fail-open tokenization.
- Run `just check` after the tracked test change, as required by the repository, to verify the full lint, type-check,
  and test suite.

## Risk

The change is test-only and low risk. Using `#git:sase` still exercises project extraction and the same
highlighting/tokenization path, while making the fixture match the repository's declared core dependencies. The main
regression risk is accidentally reducing project-scoping coverage; retaining the `project="sase"` seed and skill-span
assertion prevents that.
