---
tier: tale
title: Make the research filename example respect disabled suffixes
goal:
  Render research:202609/topic.md when suffixes are disabled and retain
  research:202609/topic__a.md when enabled, with consistent filename instructions.
size: small
proposed_by: bbugyi200.apollo.6f
create_time: 2026-10-10 12:29:13
status: wip
---

# Plan: Make the research filename example respect disabled suffixes

## Scope and evidence

This is a focused prompt-template correction that one coding agent can implement. The
implementation belongs in the linked `sase-research-artifacts` repository. Open it with
`/sase_repo` using
`sase repo open sase-research-artifacts -r "Implement the approved research suffix example correction"`
and use the returned checkout. Read its `AGENTS.md` before editing. All implementation
paths below are relative to that repository.

`src/sase_research_artifacts/xprompts/research.md` unconditionally includes
`research:202609/topic__a.md` in the artifact-registration instructions after all
filename-selection branches. Consequently, even ordinary `#research` suggests a suffixed
label.

There is a schema mismatch with the request's description: `suffix` currently has
`type: word` and `default: null`, rather than a boolean type. The swarm supplies real
identifiers such as `cdx` and `cld`. Rendering the current macro confirms that literal
`suffix=false` is a truthy word and currently requests `<stem>__false.md`. Merely
wrapping the example in `{% if suffix %}` would therefore miss the explicitly requested
`suffix=false` case.

Preserve identifier-valued suffixes and the existing default. Treat the exact lowercase
word `false` as an explicit disabled value, equivalent to omission. This reserves that
one identifier; it is the deliberate compatibility consequence of honoring the user's
requested spelling. Update both the filename-selection predicate and the example so they
agree. This work is confined to prompt content and its input description; shared macro
parsing, Rust bindings, and artifact storage need no changes.

## Implementation

1. In `research.md`, retain the input name, `type: word`, and `default: null`. Explain
   in the `suffix` description that omission or `false` disables suffixing, while
   another identifier requests `__<suffix>.md`. Keep `report_target` precedence.
2. Use `suffix and suffix != 'false'` as the suffix-enabled condition in the existing
   `{% elif suffix %}` filename branch. Omission and literal `false` then reach the
   ordinary new-file branch; actual identifiers retain their current instructions.
3. Make the registration example conditional on the same condition. A minimal form is:

   ```jinja2
   (for example `research:202609/topic{% if suffix and suffix != 'false' %}__a{% endif %}.md`)
   ```

   Keep `__a` as the generic enabled example requested by the user, including when the
   actual identifier is `cdx`. Continue telling the agent to use the actual report path
   for the registered label. The illustrative month remains `202609`.

4. Preserve the exact-target branch, collision handling, registration command, and
   instructions to use the actual repo-relative path and retain the source file. An
   explicit target remains authoritative even when `suffix` is supplied. Its
   registration example follows the suffix input but remains only an illustration.

## Verification and acceptance

Use the public macro parser and `expand_single_macro` to render the edited source
without executing its embedded shell commands or creating research artifacts. Parse this
checkout's file explicitly so an installed copy cannot mask the change. The same
in-memory candidate already rendered correctly during planning for every row below;
repeat the smoke check against the actual edited file.

| Inputs                                    | Required example              | Filename instructions                         |
| ----------------------------------------- | ----------------------------- | --------------------------------------------- |
| omitted                                   | `research:202609/topic.md`    | ordinary new report; no required suffix       |
| `suffix=false`                            | `research:202609/topic.md`    | ordinary new report; no `__false` requirement |
| `suffix=a`                                | `research:202609/topic__a.md` | require `<stem>__a.md`                        |
| `suffix=cdx`                              | `research:202609/topic__a.md` | require `<stem>__cdx.md`                      |
| `report_target=nested/x.md`               | `research:202609/topic.md`    | exactly `nested/x.md`                         |
| `report_target=nested/x.md, suffix=false` | `research:202609/topic.md`    | exactly `nested/x.md`                         |
| `report_target=nested/x.md, suffix=a`     | `research:202609/topic__a.md` | exactly `nested/x.md`; no suffix requirement  |

Check that each expansion contains only its expected example, has no unrendered Jinja
directives, and still includes the artifact-registration command. No new test module or
helper is needed for this small template correction. Existing coverage in
`tests/test_macro_loading.py` includes `test_research_prompt_declares_typed_input`,
`test_research_prompt_suffix_branch_renders_without_artifacts`, and
`test_research_registers_report_in_every_branch`.

Run the plugin's prescribed `sase tool run check` from its opened checkout after the
edit. Follow `/sase_monitor` if that verification needs a long-command handoff. If setup
requires opening the coordinated `sase-core` checkout, use `/sase_repo` before accessing
it. Do not globally install the plugin for this change. Review the final diff to confirm
that only `research.md` changed and report the smoke/check outcomes.

The planning turn changes only this scratch plan and submits it for approval. The
implementation starts after the plan is approved.
