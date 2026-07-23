---
tier: tale
title: Confirm prompt launches that still contain TODOs
goal: ACE asks for an explicit y/n confirmation before launching a prompt with visible
  TODO draft markers, while cancellation preserves the exact prompt stack and confirmation
  preserves the existing launch behavior.
create_time: 2026-07-23 11:24:59
status: done
---

- **PROMPT:** [202607/prompts/confirm_todo_prompt_launch.md](prompts/confirm_todo_prompt_launch.md)

# Plan: Confirm prompt launches that still contain TODOs

## Context

ACE already treats bounded uppercase `TODO` annotations as visible draft markers in the prompt input: `TODO`, `TODO:`,
`TODO(owner)`, and `TODO(owner):` count when they occur outside inline or fenced code, while lowercase and identifier
substrings do not. The prompt border aggregates those markers across the prompt stack, but submission currently launches
the literal prompt immediately. The new safeguard should use that established definition instead of inventing a second
meaning of TODO.

The interception belongs in the `PromptInputBar` submission flow, before it posts `PromptInputBar.Submitted`. In
particular, selected-pane submission currently removes the pane before posting the message; opening confirmation later
in the app-level handler would make a negative response destructive. A rejected confirmation must leave a single prompt
or every pane of a prompt stack intact, with no launch message, history write, cancellation receipt, or prompt-context
mutation. A confirmed submission must then follow the same whole-bar, whole-stack, or selected-pane path as today and
must pass the prompt text through unchanged.

This is a TUI-only interaction requested specifically for the prompt input widget. CLI, mobile, approval-feedback, coder
prompt editing, and backend launch entry points should remain unchanged. The existing agent launch must still be
submitted through its tracked background task after the prompt widget emits the accepted submission.

## Implementation

1. Refactor the prompt-input submission actions into a prepare/confirm/commit flow. Synchronize the current widget text,
   identify the exact submission shape (single bar, selected pane with `keep_bar`, or joined whole stack), and calculate
   its TODO count before any pane is removed or any `Submitted` message is posted. Empty selected panes must retain
   their existing drop-without-launch behavior.
2. Reuse the literal-aware TODO detector from `_todo_highlight.py` so the confirmation agrees with the gold marker and
   `TODO N` capsule contract, including owner-qualified markers and code-literal exclusions. Count only the content that
   the current action would launch: the selected pane for a per-pane submission and all non-empty panes for a
   whole-stack submission. Keep feedback and approve-prompt modes outside the launch guard.
3. When the count is nonzero, open the canonical neutral `ConfirmActionModal` with explicit launch/keep-editing copy,
   pluralized TODO count, y/n bindings, and the cancel choice focused by default. Treat `n`, Escape, `q`, button
   cancellation, and a `None` dismissal as rejection. On rejection, refocus the originating prompt input without
   changing text, pane order/selection, stack frontmatter or source binding, prompt context, or history.
4. On `y`, commit the captured submission exactly once. For a selected-pane submission, remove only the pane whose
   launch was confirmed, rebuild the remaining stack in insert mode, retain shared frontmatter for the remaining panes,
   and post the accepted text with `keep_bar=True`. For whole-bar and whole-stack submissions, post the same literal
   value and flags used today so the app-level input-collection and tracked-launch paths remain authoritative. Guard the
   delayed modal callback against a stale or unmounted origin so it never launches different text or mutates a rebuilt
   prompt bar.
5. Update the ACE prompt-input documentation: TODO handling is no longer entirely presentation-only at submission;
   describe the y/n launch warning, the non-destructive keep-editing result, the unchanged literal prompt on approval,
   and the fact that code-literal/lowercase/non-boundary text follows the existing detector semantics. Do not add a
   configuration option or alter non-TUI launch surfaces.

## Tests

- Extend the prompt-stack submit tests with a single prompt containing a real TODO marker: submission opens the
  confirmation instead of emitting `Submitted`; `n`/Escape preserve the exact text and launch nothing; `y` emits one
  unchanged submission. Assert the modal's cancel-default focus and y/n contract.
- Cover selected-pane submission from both the direct `g<enter>` path and the multi-pane submit chooser. Before a
  decision, and after rejection, all panes, selection, frontmatter, and binding remain intact. Approval removes only the
  confirmed pane, emits the expected frontmatter-attached value with `keep_bar=True`, and leaves the other panes
  launchable.
- Cover whole-stack submission with TODOs in multiple panes: the displayed count covers only non-literal markers in the
  submitted stack, rejection preserves the stack, and approval emits the canonical `\n---\n` joined prompt once with the
  existing whole-stack flags.
- Add routing regressions showing that TODO-free prompts retain the immediate path; TODO-shaped text in inline/fenced
  code, lowercase text, and identifier substrings does not prompt; a current-pane launch ignores TODOs that exist only
  in unsent panes; and feedback/approve-prompt submissions are not treated as agent launches.
- Exercise stale/unmounted-origin handling directly so a modal callback cannot submit or remove a replacement pane after
  the prompt bar changes.

## Verification

Run the focused TODO detector/title tests, prompt stack submit/cancel tests, and app-level prompt submit/launch handler
tests. Since no new modal styling is planned, confirm that the existing confirmation-dialog visual suite remains
unchanged. Then run `just install` followed by the repository-required `just check`. Review the final diff and working
tree to verify that prompt text is never rewritten, the established tracked launch path is intact, no Rust/core or
non-TUI launcher behavior changed, and no memory or generated provider-instruction files were modified.

## Risks and safeguards

- The selected-pane path currently mutates before the app sees the submission. The confirmation must precede that
  mutation; restoring a removed pane after `n` is not an acceptable substitute because it can lose selection, binding,
  frontmatter, or edit state.
- A modal result arrives later than the keypress that opened it. Capture the intended text and pane identity, then
  validate the mounted origin before committing so a stale callback fails closed rather than launching newly edited
  content.
- Re-entering the ordinary submit action from a positive callback can reopen the same warning or launch twice. Separate
  confirmation from the one-shot commit step and test the exact message count.
- TODO scanning runs on a keystroke-triggered UI path. Reuse the existing cached/fast-path detector and do not introduce
  disk access, subprocess work, prompt expansion, or backend resolution before confirmation; all heavy launch work must
  remain in the tracked background task.
