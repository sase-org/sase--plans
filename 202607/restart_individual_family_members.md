---
tier: tale
title: Restart individual agent-family members with `,x`
goal: The Agents tab can kill or dismiss one selected family member and relaunch that
  exact member from a correct forced-reuse prompt.
create_time: 2026-07-23 10:37:25
status: done
---

- **PROMPT:** [202607/prompts/restart_individual_family_members.md](prompts/restart_individual_family_members.md)

# Restart individual agent-family members with `,x`

## Goal

Make the Agents-tab `,x` action kill or dismiss the focused real agent-family member and seed the prompt input with a
relaunchable prompt for that same member. In the motivating case, focusing `sase-8u.4.2--code` must dismiss only that
completed code row immediately, retain the earlier `--plan` member, and open a prompt that will recreate the `--code`
member rather than collapsing the request to the family container `sase-8u.4.2`.

The editable prompt must express confirmed name reuse with `!` while preserving family semantics. Use the family-attach
spelling, for example `%id(!code, family=sase-8u.4.2, bead=sase-8u.4.2)`, rather than treating `sase-8u.4.2--code` as an
ordinary explicit name. That allows the normal family-attach launch path to restore parent timestamps, family/clan
metadata, workspace context, model aliases, and plan association.

## Current behavior and constraints

- `src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py` already routes focused `,x` through
  `_kill_and_edit_agent`, and existing cleanup methods can target a selected running or completed child. The keymap
  itself does not need a new binding.
- `prepare_kill_and_edit_prompt()` currently normalizes any canonical `--` phase name to its family base. That
  intentionally turns `sase-8u.4.2--code` into `sase-8u.4.2`, which is the opposite of this feature.
- Plan-chain follow-up artifacts such as the example `--code` row can lack `raw_xprompt.md`. They do retain the executed
  prompt in a non-finalizer `*_prompt.md` artifact (for the example, `workflow-gh-main_prompt.md`), so the current
  raw-only relaunch lookup aborts with “No prompt found.”
- The low-level forced-reuse helper only marks an existing `%id`/`%i`; it does not insert one when a follow-up prompt
  has no name directive.
- Direct user names containing `--` are correctly rejected. Do not weaken that guard globally or launch a standalone
  agent that merely happens to have a family-shaped name.
- Keep disk work off rendering paths, reuse the existing mtime-aware `ArtifactFileCache`, preserve optimistic/selective
  dismissal, and leave persistence in the tracked cleanup/launch task machinery. Do not add a full agent-list reload or
  a new refresh path.

## Implementation

1. **Resolve a restartable prompt for every real family member.**
   - Add a model/artifact-layer prompt-source helper used by the relaunch action. Prefer the stored `raw_xprompt.md`;
     when it is absent, select the relevant non-commit-finalizer `*_prompt.md` through the existing cached
     prompt-selection logic already used by the detail panel.
   - Treat a historical step-prompt fallback as the body, not automatically as a complete launch prompt: workflow
     expansion may already have consumed its VCS and model directives. If the fallback body lacks them, recover the
     selected member's VCS tag through the existing agent/ancestor VCS resolver and prepend a provider-qualified model
     plus reasoning effort from the selected row's metadata. Preserve any rollover workflow references that remain
     available in the member's embedded-workflow metadata. This prevents a historical GitHub coder restart from silently
     normalizing to `#git:home` or the current default model.
   - Use the same helper in the focused and marked `,x` paths so the action collects every prompt before any
     kill/dismiss mutation. If any requested prompt cannot be resolved, keep the existing all-or-nothing behavior:
     notify and do not kill or dismiss anything.
   - Do not introduce synchronous glob/read work on the key handler. Reuse already-cached prompt/detail data when
     available; perform a cache miss or historical reconstruction off the Textual message pump, then re-resolve the
     captured agent identity before presenting confirmation or applying optimistic dismissal. A stale/removed selection
     must abort cleanly.
   - When a plan is accepted and the runner creates a coder follow-up artifact, store `state.current_prompt` with the
     existing `store_followup_prompt_artifact()` facility after the complete coder prompt is assembled. This gives new
     code-family members a canonical, pre-execution relaunch source containing the inherited VCS workflow, model/effort
     directive, plan reference, custom coder instructions, and rollover workflows. The cached `*_prompt.md` fallback
     remains necessary for historical artifacts such as the screenshot.

2. **Rewrite the selected member as an exact forced family attachment.**
   - Refactor the shared kill-and-edit prompt preparation boundary to receive enough selected-agent metadata to
     distinguish a family member from a clan member or ordinary agent: prompt-facing agent/family names, role suffix,
     and phase bead association.
   - For a real family member, replace or prepend the effective `%id` directive with the member suffix attached to the
     prompt-facing family base and mark the suffix with `!`. Preserve an existing `bead=` keyword, or inject the
     selected phase bead when the follow-up prompt did not carry a name directive. Drop obsolete clan/tribe naming
     syntax from that directive; the family parent is authoritative and the family-attach resolver carries the inherited
     clan context.
   - Preserve current behavior for ordinary named agents, name templates, clan members, unnamed prompts, fenced/disabled
     directives, and machine qualification. In particular, strip only the local machine hood from editable names and
     never misread a dotted clan member as a family phase.
   - Apply this one helper to both focused and marked-agent prompt panes so single and bulk `,x` cannot diverge.

3. **Support confirmed reuse in the family-attach launch pipeline without opening general `--` naming.**
   - Extend `%id(<suffix>, family=<parent>)` parsing so a leading `!` on the suffix is recorded as forced reuse and
     stripped before normal suffix validation. Carry that bit through the family-attach directive model.
   - Teach launch preflight/forced-owner extraction to derive the exact member owner (`<parent>--<suffix>`) from a
     forced family attachment. A non-TUI launch must still raise the existing confirmation-required error; arbitrary
     direct `%id:foo--bar` and `%id:!foo--bar` forms must remain syntax errors.
   - In the confirmed TUI launch body, preflight the whole prompt before mutation, wipe the exact derived member owner,
     then remove the `!` and continue through the ordinary family-attach resolver. Do not pass a family-shaped plain
     name through the internal-name bypass: the resolver must produce the normal `SASE_AGENT_FAMILY_ATTACH` metadata and
     launch context.
   - Reuse the existing name-wipe implementation and registry/index APIs. Verify that restarting `--code` removes the
     old code owner and any dependent descendants while preserving the family container, the earlier `--plan` owner, and
     unrelated siblings. Adjust only if the focused-member test exposes broader cleanup.

4. **Keep the Agents-tab lifecycle scoped to the selected row.**
   - Retain the current completed/pidless fast path through `_dismiss_done_agent()` and the running-member confirmation
     path through `_do_kill_agent()`. Capture prompt and identity data before either path mutates the visible list.
   - On cancel, a missing prompt, failed preflight, or failed wipe, do not mount a misleading relaunch pane and do not
     broaden cleanup to the family container. On success, mount the prompt input using the existing home-mode prompt
     context and preserve the current tracked task, optimistic refilter, focus, and refresh behavior.
   - Update command/help wording only if needed to clarify that `,x` restarts the focused member; do not change the
     configured `,x` keymap.

## Tests

- Extend `tests/ace/tui/test_retry_edit_agent_name.py` with exact family-member cases for:
  - an existing base-name `%id` rewritten to a forced `--code` family attachment;
  - a follow-up prompt with no `%id`, which must gain the forced attachment;
  - local machine qualification, a family nested in a clan, role suffixes other than `--code`, and
    preservation/injection of `bead=`;
  - unchanged ordinary-agent, clan-member, template, fenced, disabled, and unnamed behavior.
- Add artifact prompt-source coverage proving raw prompt preference, cached fallback to the matching workflow/follow-up
  `*_prompt.md`, and exclusion of commit-finalizer prompts. For historical fallback, assert that a consumed VCS workflow
  and model/effort are restored exactly once and an already-complete prompt is not duplicated. Add runner coverage that
  accepted coder follow-ups store their complete restartable prompt artifact.
- Add an Agents-tab integration regression modeled on the screenshot: select a completed `sase-8u.4.2--code` child with
  no `raw_xprompt.md`, press `,x`, assert that only that child is dismissed, and assert that the mounted
  `PromptInputBar` contains the correct code body, recovered `#gh` and model context, and the exact forced family-attach
  directive. Cover the running-member confirmation/cancel paths, a selection that goes stale during background prompt
  resolution, and marked-family-member panes as well.
- Extend launch-validation tests to prove that forced family attachment is rejected without TUI confirmation, accepted
  by the confirmed TUI preflight, rewritten to a normal family attachment after the exact owner is selected, and does
  not relax direct reserved-`--` names.
- Add a launch-body/family-attach integration test that records the exact name passed to `wipe_agent_name_for_reuse()`
  and verifies the resulting launch carries the original family base, role, parent, clan, workspace, bead, and plan
  metadata.
- Extend `tests/test_agent_name_wipe.py` with a family rooted at `<base>--plan` and a `<base>--code` descendant.
  Wiping/restarting `--code` must preserve `--plan` and the family reservation while removing the old code lineage.
- Run `just install`, the focused unit/integration tests above, and then the repository-required `just check`. No visual
  golden update is expected because the keymap and rendered layout are unchanged.

## Acceptance criteria

- With the screenshot’s `sase-8u.4.2--code` row focused, `,x` dismisses that row, leaves `sase-8u.4.2--plan` visible,
  and opens the implementation prompt in `PromptInputBar` with the original GitHub and model/effort launch context.
- The prompt visibly contains a `!`-forced attachment for the exact `code` member and submits successfully as a member
  of the existing `sase-8u.4.2` family, not as the family base or a standalone family-shaped name.
- Running family members require confirmation; cancellation and missing-prompt cases are non-destructive.
- Focused and marked `,x` flows share the same exact-member prompt rewrite.
- Direct user-authored `--` names remain invalid, and non-TUI forced reuse still requires confirmation.
- Restart cleanup does not erase the family container, earlier family members, unrelated siblings, or unrelated agent
  history.
- Targeted tests and `just check` pass without a TUI responsiveness regression or a new full-refresh path.
