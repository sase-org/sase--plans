---
tier: tale
title: Wipe every %name:! owner in multi-segment TUI launches
goal: 'Swarm launches submitted through the ACE TUI with per-segment %name:! directives
  force-reuse every requested agent name instead of only the first segment''s name,
  eliminating spurious "Agent name ''<n>'' is taken" collision failures.

  '
create_time: 2026-07-18 17:04:58
status: wip
prompt: 202607/prompts/fix_multi_segment_force_reuse.md
---

# Plan: Wipe every `%name:!` owner in multi-segment TUI launches

## Problem

Submitting a multi-segment (swarm) prompt through the ACE TUI where **every** segment carries a forced-reuse name
directive (`%name:!<n>`) fails with a name collision, e.g.:

```
Agent name 'sase-6v.4' is taken. Try 'sase-6v.41'.
```

This should never happen: `%name:!` is the explicit "replace the existing owner of this name" form, and on a trusted TUI
submit each requested name is supposed to be wiped from the agent-name registry before validation runs.

## Root cause (confirmed by reproduction)

The TUI launch body `run_agent_launch_body` in `src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py` performs
forced-reuse cleanup **before** the prompt is split into segments:

1. It calls `force_reuse_owner_names([prompt])` with the entire submitted multi-segment prompt as a single list element.
2. `force_reuse_owner_names` (in `src/sase/agent/launch_validation.py`) feeds each list element through
   `_extract_explicit_name`, which by contract returns only the **first** `%name`/`%n` directive found in that string.
   The contract assumes callers pass already-split per-segment prompts — every other caller does (e.g.
   `prepare_bead_work_force_reuse` in `src/sase/bead/cli_work_cleanup.py` splits on `\n---\n` first).
3. So for a swarm prompt, only the **first segment's** name is returned and wiped via `wipe_names_for_forced_reuse`.
4. `rewrite_force_reuse_name_directives(prompt)`, however, iterates **all** matches and strips the `!` marker from every
   segment.
5. The prompt is then parsed with `parse_multi_prompt`, and `validate_launch_name_requests(multi.segments)` runs with
   the default `allow_force_reuse=False`. Segments beyond the first now carry plain `%name:<n>` directives whose names
   were never wiped, so the first still-reserved name raises `AgentNameLaunchCollisionError` — exactly the reported
   error.

Reproduction (whole-prompt vs. per-segment extraction):

```python
from sase.agent.launch_validation import force_reuse_owner_names
prompt = "%name:!a one\n---\n%name:!b two\n---\n%name:!c three"
force_reuse_owner_names([prompt])                    # ['a']  <- TUI path today
force_reuse_owner_names(prompt.split("\n---\n"))     # ['a', 'b', 'c']
```

The asymmetry — wipe-first-name-only vs. rewrite-all-names — is the bug. The CLI `sase bead work` relaunch path is
unaffected because it splits segments before extraction; non-TUI surfaces reject `%name:!` outright with
`AgentNameReuseConfirmationRequiredError`, so the TUI multi-segment path is the only surface that can hit this.

## Fix design

Fix at the TUI call site in `src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py`: split the submitted prompt
into segments before extracting forced-reuse owner names, so the wipe set matches the set of names the rewrite de-marks.

- Use `parse_multi_prompt` from `sase.agent.multi_prompt` for the split (it is frontmatter-aware and protects fenced
  code blocks, unlike a naive `split("\n---\n")`, so a `---` inside a fenced block cannot create phantom segments), and
  pass the resulting segments to `force_reuse_owner_names`.
- Guard the early parse: if `parse_multi_prompt` raises (e.g. `LocalXPromptNameError` for a bad local-xprompt name),
  fall back to treating the prompt as a single segment. Parse errors must keep surfacing from the existing later
  `parse_multi_prompt` call inside the `prompt_parse` timer stage so their reporting behavior is unchanged.
- Everything downstream stays as-is: `wipe_names_for_forced_reuse` receives the full name list,
  `rewrite_force_reuse_name_directives` already rewrites all segments, history still records the rewritten (no-`!`)
  prompt, and both the in-body `validate_launch_name_requests(multi.segments)` call and the later launcher-level
  validation pass because every requested name was wiped.

Also update the docstrings of `force_reuse_owner_names` and `_explicit_launch_name_requests` in
`src/sase/agent/launch_validation.py` to state explicitly that inputs must be per-segment (already-split) prompts and
that only the first `%name` directive per element is considered, so the next caller cannot repeat this mistake silently.

Do not change `_extract_explicit_name`'s first-directive-only semantics: it is correct for per-segment inputs and shared
by launch-name validation.

This is Python launch-orchestration glue around the existing validation module; no Rust core (`sase-core`) API is
involved.

## Testing

- **Regression test (primary)** in `tests/ace/tui/test_agent_launch_non_blocking.py`, modeled on
  `test_finish_agent_launch_force_reuse_schedules_original_prompt_and_worker_rewrites`: submit a multi-segment prompt
  where every segment has `%name:!<n>` and assert that `wipe_names_for_forced_reuse` is called with **all** requested
  names and that the segments reaching `validate_launch_name_requests` (and the queued multi-prompt launch) carry the
  rewritten plain `%name:<n>` forms.
- **Fenced-block guard**: a case where `---` appears inside a fenced code block within a segment, asserting the split
  used for extraction does not treat it as a separator (wipe set unchanged).
- **Parse-failure fallback**: a prompt that makes the early parse raise still launches down the existing error path (no
  behavior change vs. today) and wipes at most the single-prompt extraction result.
- Keep the existing single-segment force-reuse tests green unchanged — they pin the wipe-in-worker, rewrite, and
  history-recording contracts this fix must preserve.

## Verification

Run `just install`, then `just check` (lint + mypy + tests) before finishing. Targeted runs while iterating:
`pytest tests/ace/tui/test_agent_launch_non_blocking.py tests/test_agent_launch_validation.py`.

## Risks and out of scope

- **Xprompt-expanded forced reuse**: `%name:!` directives that only appear after xprompt swarm expansion (expansion runs
  after the forced-reuse block) are still not wiped and will still raise `AgentNameReuseConfirmationRequiredError` at
  validation. That is pre-existing behavior for a different surface area and is intentionally not addressed here;
  literal user-typed swarm segments are the reported and fixed case.
- **Duplicate names across segments** are deduplicated by `force_reuse_owner_names` (first-seen order), so the wipe list
  stays stable.
- The early split adds one extra `parse_multi_prompt` call per TUI submit that contains `%name:!`; parsing is cheap
  relative to the launch body and only runs on force-reuse prompts.
