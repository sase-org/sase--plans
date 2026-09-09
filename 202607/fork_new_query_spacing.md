---
tier: tale
title: Keep fork history markers on a line boundary
goal:
  Forked prompts render an unindented New Query heading in every prompt-part expansion
  path.
create_time: 2026-09-09 19:53:14
status: wip
---

# Plan: Keep fork history markers on a line boundary

## Problem

An inline launch such as:

```text
#gh:sase #fork:parent Continue the work
```

can render the fork envelope with a leading space before its `# New Query` heading. The
captured launch artifacts show that the submitted prompt has a workspace workflow
immediately before `#fork`, while the final agent prompt contains ` # New Query`.

The fork history builder itself emits an unindented heading. The whitespace is
introduced earlier: launch-deferred fork expansion is performed by
`expand_embedded_workflows_in_query()` in
`src/sase/main/query_handler/_embedded_workflows.py`. Unlike the ordinary xprompt
processor and the full workflow executor, this expansion path does not move prompt-part
content beginning with `%xprompts_enabled:false` onto a new line when its reference
appears mid-line. The intermediate prompt therefore contains:

```text
#gh:sase %xprompts_enabled:false
```

instead of a line-bounded disabled-region marker. When the empty workspace workflow
prompt part is removed and the prompt passes through disabled-region protection and
Markdown formatting, the separator whitespace can be retained in front of `# New Query`.

## Implementation

1. Establish one shared helper in `src/sase/xprompt/_disabled_regions.py` for ensuring
   that rendered content beginning with `%xprompts_enabled:false` starts on a line
   boundary. The helper should prepend exactly one newline only when the insertion point
   is mid-line and leave other content and already line-bounded insertions unchanged.

2. Use that helper in all three prompt-part expansion paths:
   - `src/sase/xprompt/processor.py`
   - `src/sase/xprompt/workflow_executor_steps_embedded_expand.py`
   - `src/sase/main/query_handler/_embedded_workflows.py`

   Replace the two existing local regex/conditional implementations rather than adding a
   third copy. Preserve the current heading separation behavior: an inline query after
   `# New Query` must remain in a separate Markdown paragraph, and source whitespace
   outside the workflow reference must not be broadly stripped.

3. Add focused unit coverage for the shared helper, including mid-line, line-start,
   non-marker, and empty-content cases.

4. Add a launch-deferred fork regression test using a prompt with a retained workspace
   workflow before `#fork`. Assert that the expanded disabled marker begins immediately
   after a newline rather than after `#gh:... ` on the same line.

5. Add or extend workflow-level coverage that follows the composed prompt through
   workspace-reference removal and late Markdown preprocessing. Assert that the final
   envelope contains an unindented `# New Query`, that the user's inline query remains
   below the heading, and that disabled-region markers are absent from the final prompt.
   Retain coverage for a fork at the beginning of a line so the fix does not add an
   extra blank line there.

## Validation

Run the focused tests for disabled-region handling, launch-deferred expansion, and the
fork workflow first. Then run the repository-required checks:

```bash
just install
pytest -q tests/test_disabled_regions.py tests/test_run_agent_runner_setup.py tests/test_fork_workflow.py
just check
```

Inspect the focused regression's final prompt representation as part of the test
assertions so success verifies the absence of leading whitespace rather than relying
only on rendered appearance.
