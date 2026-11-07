---
tier: tale
size: small
title: Keep the coder Reviewer decisions block literal
goal:
  Planner-written decision text can never be expanded as prompt syntax in an approved
  plan's coder prompt.
decisions:
  vcs_error_detail:
    ask:
      Should the multiple-VCS-workflow error name each full ref and explain project-tag
      expansion?
    default: true
    why:
      The terse error hid this failure's cause; prompts naming two projects hit it too
    answer: true
proposed_by: bbugyi200.athena.19e
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-07 07:09:25
status: wip
---

# Keep the coder Reviewer decisions block literal

## Why the 19c agent failed

`19c--plan` (project `gh_bbugyi200__love`, tag `+love`) proposed a tale with three
`decisions:`. Two of the `ask` strings named the project by its tag, for example
`ask: Should the coder also add a one-line pointer to the +love project in ~/bob/love.md?`.
The reviewer approved in ACE. The host-built `19c--code` prompt then failed after 40s,
before the provider ran:

```
WorkflowExecutionError: Step 'main' failed: Multiple VCS-tagged workflows in one prompt: #gh, #gh, #gh.
```

Root cause:

- `prepare_accepted_plan_successor` in `src/sase/axe/run_agent_exec_plan_accept.py`
  builds the coder prompt as
  `{model_prefix}{vcs_prefix}@{plan}\n\n... Implement it now.{decisions_block}{coder_extra}\n{embedded_refs}`.
- `decisions_block` comes from `_reviewer_decisions_block` -> `coder_decisions_block`
  (`src/sase/sdd/plan_decision_handoff.py`) -> the core `plan_decisions_prompt_block`.
  That function inlines each row's `ask` text as `Context: "<ask>"` via `quoted()` in
  sase-core `crates/sase_core/src/plan/decisions/sheet.rs`. `quoted()` only flattens
  newlines and escapes double quotes, so the text stays live prompt syntax.
- At launch, project-tag expansion rewrote both `+love` inside the quotes to
  `#gh:gh_bbugyi200__love`. `_expand_embedded_workflows_in_prompt`
  (`src/sase/macro/workflow_executor_steps_embedded_expand.py`) then found three VCS
  workflows and raised.

The bug is deterministic, so a retry fails the same way. The user already worked around
it by launching `19d` with `#coder`. More generally, any `ask` text containing a project
tag (the normal way to name a project), a `#macro`, a `%directive`, or an `@path` either
breaks the coder launch or silently changes it. For example, `%model:x` in an ask would
switch the coder's model, and `@file` would inline a file.

## Fix approach

Wrap the rendered block in a disabled region at the prompt-composition site, using the
existing `wrap_disabled_region` helper (`src/sase/macro/_disabled_regions.py`). Every
other host-built prompt that embeds data already does this:
`gate_turn/followup_prompt.py`, `monitor/followup_prompt.py`,
`history/chat_fork/build.py`, and `main/qa_prompt.py`. Disabled regions are honored by:

- the Rust project-tag scan (via `prompt_literal_zone_ranges`),
- Python macro/workflow expansion (`protect_disabled_regions`),
- directive scanning,
- file-ref expansion.

`preprocess_prompt_late` (`src/sase/llm_provider/preprocessing.py`) strips the markers,
so the model sees the block unchanged. This was verified during diagnosis:
`expand_project_tags` rewrites `+love` outside a wrapped region and leaves it alone
inside one.

Rejected alternatives:

- **Inline backticks around the ask.** These protect project tags in the Rust scan, but
  `_expand_embedded_workflows_in_prompt` protects only fenced blocks and disabled
  regions. A `#gh:...` ref inside backticks would still be counted.
- **Changing `quoted()` or the block in sase-core.** The same core block is printed as
  plain `sase bead read` output for epic phase and land agents
  (`src/sase/bead/cli_detail_decisions.py`), where markers would be noise. Inertness is
  a property of embedding the block in a prompt, and that composition is Python host
  code. No sase-core change and no `sase-core-revision.txt` bump are needed.
- **Deduplicating identical VCS refs.** The expanded refs would still mangle the Context
  text, and deduplication would hide real two-target prompts.

## Changes

### 1. Wrap the decisions block (core fix)

In `_reviewer_decisions_block` (`src/sase/axe/run_agent_exec_plan_accept.py`), return
`f"\n\n{wrap_disabled_region(block)}"` instead of `f"\n\n{block}"`:

- Import `wrap_disabled_region` locally, as the other prompt builders do.
- Keep the fail-open contract unchanged: an empty or failed block still returns `""`.
- Extend the docstring: the block quotes planner-authored frontmatter text, so it must
  reach the coder literally.

Keep `coder_extra` (reviewer-typed "Additional instructions") and `embedded_refs`
outside the region. They are intentionally live, and
`test_approve_prompt_includes_custom_extra_text` relies on `#foo` there.

Check that the closing `%macros_enabled:true` marker still matches the region regex.
With `coder_extra` it is followed by `\n\nAdditional instructions:`; without it, by `\n`
plus embedded refs or end of text. Also check that `set_prompt_wait`, applied
afterwards, still produces a canonical `%wait` (it already protects disabled regions).

### 2. Tests for the fix

- Update `test_reviewer_block_helper_prefers_archived_copy` in
  `tests/test_plan_decisions_handoff_coder.py`. It should assert that the helper returns
  `"\n\n%macros_enabled:false\nReviewer decisions for this plan"` ...
  `%macros_enabled:true`.
- Add a regression test: a stamped tale (pattern after `STAMPED_TALE` in
  `tests/_plan_decisions_handoff_helpers.py`) whose `ask` strings contain a resolvable
  project tag, a second VCS ref such as `#gh:other/repo`, a `%model:` directive, and an
  `@some/path` ref. Compose the prompt the way `prepare_accepted_plan_successor` does
  (`#gh:<project> @plan.md\n\n... Implement it now.` plus the helper output). Assert all
  of:
  - project-tag expansion (the core `project_tag_expand` binding or
    `expand_project_tags_with_catalog` with a small catalog) leaves the Context text
    byte-identical;
  - `_expand_embedded_workflows_in_prompt` collects exactly one VCS workflow and does
    not raise (reuse the `_FakeExecutor` pattern from `tests/test_wraps_all.py`, e.g.
    `test_vcs_refs_inside_disabled_region_not_counted`);
  - `extract_prompt_directives` on the prompt does not pick up the ask's `%model:`;
  - `strip_disabled_region_markers` of the prompt still contains the original
    `Context: "..."` line verbatim.
- If the `run_plan_approval` harness in
  `tests/test_axe_run_agent_exec_plan_followup_coder_prompt.py` can take a stamped plan
  (see `tests/_axe_run_agent_exec_plan_followup_prompt_helpers.py`), also add an
  end-to-end assertion. `state.current_prompt` should contain exactly one disabled
  region holding the decisions block, with `#gh:sase`, `@plan.md`, and `%model:@small`
  still outside it.

### 3. Clearer multiple-VCS error (only if `vcs_error_detail = yes`)

> [!decision] vcs_error_detail = yes In Phase 2 of
> `_expand_embedded_workflows_in_prompt`, name each entry by its full reference text
> (`prompt[p.match_start:p.match_end]`, e.g. `#gh:gh_bbugyi200__love`) instead of
> `#{p.name}`. Append one sentence noting that project tags (`+name`) expand into VCS
> workflow refs, so a project tag in prose counts too, and that literal text belongs in
> a fenced block or a `%macros_enabled:false` region. Keep the existing
> `Multiple VCS-tagged workflows in one prompt:` prefix, because
> `tests/test_wraps_all.py` matches on it. Extend `test_multiple_wraps_all_raises_error`
> (or add a sibling test) to assert the full refs appear.

> [!decision] vcs_error_detail = no Leave `workflow_executor_steps_embedded_expand.py`
> unchanged.

### 4. Docs

In `docs/reference/work-units/plan-format.md`, near the sentence saying implementers
receive a host-written Reviewer decisions block (around line 134), add one sentence. It
should say that the block reaches the implementer as literal text: project tags, macro
refs, directives, and file refs inside an `ask` are never expanded. Read the
`docs_writing.md` reference memory before editing docs. Do not edit `CHANGELOG.md`,
which release-please generates.

## Verification

- Run the targeted tests first: `tests/test_plan_decisions_handoff_coder.py`,
  `tests/test_axe_run_agent_exec_plan_followup_coder_prompt.py`,
  `tests/test_wraps_all.py`, and `tests/plan_gate_turn/test_followup.py`.
- Then follow the `lint_and_test.md` reference memory (`just check` through
  `sase tool run`).

## Out of scope

- Re-running the failed `19c--code`. Its stored prompt is unwrapped, so a retry would
  fail again, and `19d` already carries the work.
- Any sase-core change. No memory note becomes false, so no memory decision is needed.
