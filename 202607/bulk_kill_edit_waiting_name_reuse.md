---
create_time: 2026-07-14 11:23:59
status: done
prompt: 202607/prompts/bulk_kill_edit_waiting_name_reuse.md
tier: tale
---
# Ensure marked-agent kill-and-edit preserves explicit names

## Context and current finding

The Agents-tab leader `,x` action has two routes:

- With no marks, `_kill_and_edit_agent()` reads the focused agent's raw prompt, marks an explicit `%name` / `%n` value
  for forced reuse, kills or dismisses the agent, and mounts one editable prompt.
- With marks, `_bulk_kill_marked_agents_and_edit()` resolves the marked agents in mark order, reads every raw prompt,
  calls `force_name_reuse_in_prompt(..., replacement_name=agent.agent_name)`, performs one bulk confirmation, and mounts
  one prompt pane per agent.

Code inspection therefore denies a simple status-gating bug in the current implementation: the bulk rewrite happens
before agents are partitioned by status, so `WAITING` does not directly bypass it. Existing unit coverage also proves
forced reuse for a RUNNING `%n` prompt and a DONE `%name` prompt. However, that coverage uses fake agents and captures
the prompt list before the real `PromptInputBar` is mounted. It does not exercise WAITING agents loaded from artifacts,
template-derived concrete names, or the final widget pane values. The report can therefore still reflect a
loader/artifact discrepancy, a prompt-mount discrepancy, or an unprotected alternate path.

## Plan

1. Add a focused regression reproduction for the reported interaction.
   - Build realistic marked `Agent` rows with `status="WAITING"`, artifact-backed `raw_xprompt.md` prompts, live PIDs,
     and explicit names in both `%n` and `%name` forms.
   - Drive the marked-agent `,x` route through confirmation and through the real bulk prompt-bar mounting boundary.
   - Assert that there is exactly one pane per marked agent, pane order follows mark order, and the widget's
     stored/editable values contain `%n:!<name>` / `%name:!<name>`.
   - Include a template-derived explicit name case such as `%name:@.<suffix>` to verify that the concrete `agent_name`
     loaded for a waiting row is used when forced reuse cannot be expressed by prefixing the template itself.

2. Establish one shared kill-and-edit prompt preparation contract.
   - Extract or promote a helper that accepts an agent's raw prompt plus its resolved `agent_name` and returns the exact
     editable prompt to mount, using `force_name_reuse_in_prompt` consistently.
   - Route both the focused-agent and marked-agent `,x` paths through that helper so alias handling, idempotence,
     fenced/disabled-region behavior, and template replacement cannot drift between the two workflows.
   - Keep preparation before kill/dismiss mutations, preserve the current all-or-nothing behavior when any marked agent
     lacks a prompt, and avoid adding disk I/O or other blocking work to the Textual event loop.

3. Fix the layer exposed by the regression test.
   - If realistic WAITING rows lack the resolved name needed for a template directive, correct the running/waiting
     artifact enrichment so `agent_meta.json` name data is retained uniformly across statuses.
   - If the prepared prompt is correct before mounting but not in the widget, correct the explicit `initial_panes`
     handoff so pane construction remains verbatim apart from the existing VCS-ref display humanization.
   - If both layers already preserve the value, retain the regression test and shared-helper cleanup as the fix that
     closes the untested path, and document that current source behavior denies the original literal-name suspicion; no
     status-specific special case should be introduced.

4. Expand narrow behavioral coverage around the shared contract.
   - Cover `%n` and `%name`, literal and template-derived names, already-forced names, and prompts without explicit
     names.
   - Verify WAITING, RUNNING, and dismissable statuses use identical name-reuse preparation while preserving their
     existing confirmation/kill classification.
   - Preserve existing guarantees: cancel leaves marks intact, missing prompts kill nothing, embedded `---` remains
     inside its owning pane, and panes remain in mark order.

5. Validate the change.
   - Run the focused retry-name, bulk kill-and-edit, prompt-input initial-pane, and any new artifact-loading/widget
     regression tests during iteration.
   - Run `just install` followed by the required `just check` before completion, because implementation/test files in
     the SASE repository will change.

## Acceptance criteria

- Marking multiple WAITING agents and invoking `,x` produces one editable pane per marked agent with explicit names
  forced for reuse using `!`.
- `%n` and `%name` behave identically, including concrete replacement of name templates.
- Focused and marked `,x` routes share the same prompt preparation behavior and do not branch on runtime/provider or
  WAITING status.
- Bulk confirmation, mark-order, cancellation, missing-prompt, and embedded-separator behavior remain unchanged.
- Focused tests and the full `just check` suite pass.
