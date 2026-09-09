---
tier: epic
status: done
title: Finish reliable pager link landing
goal:
  Make pager link resolution honor its one-search contract without event-loop I/O and
  keep context identity exact
phases:
  - id: one-pass-dead-ends
    title: Resolve dead ends once off the event loop
    depends_on: []
    description:
      "one-pass-dead-ends: return the file-path search diagnostics needed by the pager
      from the same background resolution attempt, remove the synchronous/repeated toast
      search, and enforce the git output bound while preserving timeout and
      resolver-injection behavior."
    size: medium
  - id: event-loop-context
    title: Make pager context handling pure and identity-safe
    depends_on:
      - one-pass-dead-ends
    description:
      "event-loop-context: remove filesystem work from per-label context merging and ACE
      request preparation, build captured agent/patch contexts in the existing
      background materialization path, and include workspace numbers in dangling-ref
      identity."
    size: medium
proposed_by: bbugyi200.athena.sase-xy.land
parent_bead: sase-xy
bead_id: sase-xy.4
create_time: 2026-09-09 19:52:32
---

- **PROMPT:**
  [prompts/202609/pager_link_landing_corrections.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pager_link_landing_corrections.md)
- **PARENT:** [202609/pager_link_reliability.md](pager_link_reliability.md)
- **BEAD:**
  [sase-xy.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xy/sase-xy.4.md)

# Finish Reliable Pager Link Landing

## Context

Epic `sase-xy` added ordered link anchors, hard file search, line-suffix scanning, and
context threading through the pager. Its three phase commits are present and the focused
pager, ACE, artifact-read, bead-show, and pager-command tests pass. Landing review
nevertheless found event-loop and identity defects in the new paths.

For a missing two-component path, `resolve_ref()` calls the bounded Git suffix search in
its background thread and returns `None`. `PagerActionMixin._apply_resolution()` then
calls `file_path_unresolved_message()` on the Textual event loop; that helper repeats
the full filesystem and Git search. A direct reproduction with one anchor observes one
`_git_ls_files` call after `resolve_ref()` and two after constructing the toast. The
repeated-label fast path also reconstructs the message synchronously. This violates both
the epic's one-`git ls-files`-per-anchor-per-press contract and `tui_perf.md`'s
prohibition on synchronous disk/subprocess work in action handlers.

The Git byte limit is currently checked only after `subprocess.run()` has materialized
all stdout, so `_GIT_LS_FILES_MAX_BYTES` rejects oversized output but does not cap how
much output is captured. The corrected implementation must bound capture itself, kill
and reap the child cleanly on timeout/overflow, and preserve the prompt-free environment
and unique-suffix semantics.

Two other new event-loop paths need correction. `merge_link_context()` calls
`_dedupe_anchors()`, whose existence checks and path resolution run once per label
activation. ACE `_prepare_view_input()` also calls `agent_link_context()` or
`workspace_link_context()` synchronously while handling submitted hint input; those
helpers stat directories, read workspace markers, and resolve primary workspaces.
Finally, `_dangling_ref_key()` records only anchor directories. Two typed-ref contexts
with the same directory but different `workspace_num` values therefore alias even though
`artifact_ref_context()` can resolve them differently.

## Phase 1: Resolve Dead Ends Once Off the Event Loop

Refactor the resolution boundary in `src/sase/pager/resolve.py`,
`src/sase/pager/app.py`, `src/sase/pager/screen.py`, and
`src/sase/pager/_screen_actions.py` so one background attempt returns both the
`LinkTarget | None` result and any file-path dead-end diagnostics the UI needs. Keep the
public `resolve_ref()` convenience API usable by non-UI callers, and preserve the ACE
link-index fast path and caller-injected resolver contract. The exact result type is an
implementation choice, but the event-loop apply path must consume already computed data:
it must not call `Path.exists`, `Path.resolve`, Git, artifact resolution, or the search
helper again.

Store the computed unresolved message with the dangling-context key so pressing an
already-dangling label can toast without re-resolving or rebuilding diagnostics. A
single failed press must run `_git_ls_files` no more than once per unique anchor across
all path candidates. Copy and successful follow/edit behavior must remain unchanged.

Replace the post-hoc Git stdout length check with genuinely bounded capture. Retain the
short timeout, `GIT_TERMINAL_PROMPT=0`, NUL-delimited parsing, graceful degradation on
Git/process errors, and cleanup/reaping on every exit. Oversized output is a miss, not a
partial candidate list.

Add focused resolver and Textual tests that:

- count one Git invocation for a missing path from press through toast;
- prove the message is computed off the event-loop thread and reused on a repeated
  dangling press;
- retain the location-count wording and resolver-injection/link-index behavior;
- prove output at the byte limit is accepted, output over it is rejected without
  retaining the excess, and timeout/overflow children are cleaned up; and
- keep all existing direct-hit, ambiguity, copy, edit, and inherited-context tests
  green.

Run the focused pager tests, then the repository's required verification lane.

## Phase 2: Make Pager Context Handling Pure and Identity-Safe

After phase 1, make context combination in `src/sase/pager/link_context.py` a pure,
in-memory operation. Anchor factories may still perform their documented validation when
called off-thread or before the pager starts, but `merge_link_context()` as used by
`_activate_label()` must preserve order and deduplicate without filesystem access. Do
not weaken the existing first-anchor-wins behavior.

In `src/sase/ace/tui/actions/hints/_processing.py`, capture only stable agent/patch
inputs on the UI thread. Resolve the selected agent workspace, patch workspace,
workspace marker, and primary/default anchors inside the already pump-free/background
view-materialization path before the `PagerDocument` is built. Revalidate any UI state
needed after awaits according to `tui_perf.md`; do not move mutable widget or selection
objects into a worker. Preserve the exact agent ordering (agent workspace, primary,
defaults) and patch ordering (patch workspace, primary, defaults), including graceful
fallbacks.

Change dangling-ref identity to include every anchor's `(directory, workspace_num)`
pair. Add tests with identical directories and different workspace numbers showing that
a failed typed ref in one section does not suppress resolution in the other. Also add
tests that fail if label activation/merge performs filesystem normalization, and that
show the ACE preparation step merely snapshots data while context construction runs
off-thread. Keep existing agent, patch, section-anchor, trail, copy, and edit tests
green.

Run the focused pager/ACE tests, `just test-visual` because label-state behavior is
exercised, and the repository's required verification lane. Do not update visual goldens
unless a deliberate visible change is reviewed.

## Landing Handoff

This plan contains only the remaining implementation work caused by `sase-xy`. Its
parent association should return control to that epic's land agent after this child epic
lands. The child phases must not close `sase-xy`, modify the original epic plan's
status, or treat final Symvision/close checks as phase work.
