---
tier: epic
status: done
title: Finish and land multi-item capture
goal:
  Integrate every change that landed while bob-cli-t was in progress, restore the full
  macOS release gate, verify the real single- and multi-item experience, and close the
  original epic with every child proposal and landing artifact accounted for.
phases:
  - id: restore_macos_test_compilation
    title: Restore the macOS test and release pipeline
    depends_on: []
    size: small
    description:
      "restore_macos_test_compilation: resolve bob-cli-x by making pure indentation
      helpers safely nonisolated so the macOS test target compiles without weakening
      main-actor protection around AppKit and model mutation."
  - id: integrate_batch_task_id_flow
    title: Integrate later-item task-ID assignment with batch capture
    depends_on: []
    size: medium
    description:
      "integrate_batch_task_id_flow: add cross-repository regression coverage for
      all-task completion and inline block-ID assignment in a later batch item, fixing
      any loss of global ranges, earlier draft items, or one-process capture semantics."
  - id: validate_integrated_release
    title: Validate the integrated CLI and installed macOS experience
    depends_on:
      - restore_macos_test_compilation
      - integrate_batch_task_id_flow
    size: medium
    description:
      "validate_integrated_release: verify both repositories at their current tips,
      require a green complete macOS workflow, and run the owner-assisted installed-app
      batch, keyboard, preview, notification, and opening matrix."
  - id: land_original_multi_capture_epic
    title: Close bob-cli-t and clean its expired symbol allowances
    depends_on:
      - validate_integrated_release
    size: small
    description:
      "land_original_multi_capture_epic: perform the final drift and note audit, record
      every follow-up outcome, close bob-cli-t, run post-close symvision cleanup, and
      mark plan:202608/multi_capture.md done."
proposed_by: bbugyi200.athena.bob-cli-t.land
parent_bead: bob-cli-t
create_time: 2026-09-09 20:00:11
---

- **PROMPT:**
  [prompts/202608/land_multi_capture.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/land_multi_capture.md)
- **PARENT:** [202608/multi_capture.md](multi_capture.md)

# Finish and land multi-item capture

## Why this plan exists

The three original phases of `bob-cli-t` are closed and their implementation commits are
present:

- `bob-cli` `a8c9ad8` (`bob-cli-t.1`) adds the shared blank-line item grammar,
  item-aware parse/completion protocol, additive ordered result shapes, staged
  multi-file planning, rollback, clip cleanup, documentation, and focused tests.
- `bob-mac-capture` `4c22525` (`bob-cli-t.2`) normalizes batch protocol results,
  preserves one subprocess per parse/preview/submit operation, adds the native Control-J
  resolver, presents ordered previews, and retains/open all aggregate results.
- `bob-mac-capture` `c95ba0e` (`bob-cli-t.3`) adds semantic single/batch notifications,
  ordered de-duplicated target metadata and opening, singular/plural actions, and
  focused tests.

The land audit found two blockers to an honest close:

1. macOS 26 workflows `31890751259`, `31892336680`, and `31892613742` all pass format
   and build, then fail compiling the test target because the pure
   `CapturePanelController.bulletIndentationEdit` helper is implicitly main-actor
   isolated. This predates `bob-cli-t`, is tracked as ready task `bob-cli-x`, and blocks
   test, bundle, signature, launch-smoke, and install/reinstall evidence.
2. Concurrent epic `bob-cli-u` landed `bob-cli` `2037307` and `bob-mac-capture`
   `dff08a7`. The current implementation appears to preserve item-aware completion and
   draft-global replacement ranges, but neither repository pins the combined case where
   an `--all-tasks` missing-ID candidate belongs to a later batch item and the mac app
   assigns its ID without changing the earlier item.

Two unrelated proposals were triaged through the new-task workflow and must remain out
of this epic unless they independently become necessary for correctness:

- `bob-cli-v` (ready, small), proposed by `bob-cli-t.1`: remove three existing Clippy
  warnings in `src/native/plugins.rs` and `src/native/projects.rs`.
- `bob-cli-w` (ready, medium), proposed by `bob-cli-t.2`: remove existing Swift 6
  Sendable warnings in `CaptureTargetsCache` and `BobProcessClient`.

## Phase 1: Restore the macOS test and release pipeline

Work in `bob-mac-capture`, opened through `sase repo open`. Start from the latest
`origin/master`; do not assume `dff08a7` remains the tip.

1. Reproduce the exact `ActorIsolatedCall` diagnostics from workflow `31892613742` and
   inspect the full actor boundary of `CapturePanelController`.
2. Implement task `bob-cli-x`. Mark the pure synchronous indentation resolver and only
   the immutable/pure helper chain it calls `nonisolated`. Keep responder lookup,
   `NSTextView` mutation, completion dismissal, and every `CapturePanelModel` operation
   main-actor isolated. Do not silence the diagnostic by moving all pure tests onto the
   main actor: those tests deliberately pin callable pure behavior.
3. Retain the existing indentation matrix for `-`, `*`, and `+`, zero/two-space depth,
   placeholder rows, CRLF, Unicode selections, malformed rows, same-line selections,
   multiline rejection, and editable/noneditable responders. Add a focused compile-time
   or runtime assertion only if needed to make the isolation contract unmistakable.
4. Run `git diff --check`, `just format-lint`, and every locally available Swift build
   and test target. Record Linux SDK/toolchain limitations precisely; the dependent
   validation phase owns proof from macOS CI.
5. Leave `bob-cli-x` open with an implementation note if the exact published commit has
   not yet completed macOS CI. The validation phase closes it only after the Test step
   and the rest of the workflow pass.

## Phase 2: Integrate later-item task-ID assignment with batch capture

Work in both `bob-cli` and `bob-mac-capture`; open the linked mac repository through
`sase repo open`. Synchronize both current tips and review any commits newer than
`2037307` and `dff08a7` before editing.

1. In `bob-cli`, add focused CLI coverage for `capture-complete --all-tasks` with a
   blank-line-separated draft whose active `@route+query` is in a later item. Assert
   that Bob sends the whole draft through the shared item grammar, returns the
   missing-ID candidate fields, uses draft-global UTF-8 replacement/cursor ranges,
   preserves identified-before-unidentified ordering, and still returns an empty success
   on the separator. Fix shared completion code if the test exposes a regression; do not
   add a second splitter.
2. In `bob-mac-capture`, extend process-client coverage so the same later-item draft is
   passed as one argv element with `--all-tasks`, and its aggregate/global completion
   response decodes without client-side splitting.
3. Add model coverage that starts with a non-ASCII first item plus a later missing-ID
   parent-task completion. On successful `capture-task-id`, assert that only the saved
   global range in the later item is replaced, the first item and separators remain
   byte-for-byte intact, the caret lands after the canonical ID, and analysis reruns for
   the complete batch. Assert that cancel, server failure, transport failure, and a
   stale range preserve the entire draft and selected task state.
4. Pin the subprocess contract: ID assignment is one explicit pre-capture mutation;
   preview and final capture still each submit the full draft once, and a capture
   failure reports no partial capture success or draft clearing. Clarify this boundary
   in the README if the combined workflow is currently ambiguous.
5. Re-run `cargo fmt --check`, `cargo clippy --all-targets --all-features`,
   `cargo test`, the relevant manual single/two-item parse/complete/dry-run comparisons,
   `git diff --check`, fake-Bob syntax/JSON checks, and locally available Swift core
   targets. Existing diagnostics tracked by `bob-cli-v` and `bob-cli-w` are not failures
   of this phase unless the phase changes or worsens them.

## Phase 3: Validate the integrated CLI and installed macOS experience

Begin only after both implementation phases have published. Fetch both repositories and
review their combined diffs and every newer non-epic commit for duplication, conflict,
or a newly missing integration point.

1. In `bob-cli`, run the complete formatting, Clippy, and test suites on the current
   tree. Manually compare `capture-parse`, `capture-complete`, and
   `capture --dry-run --format json` for single and two-item drafts, including nested
   authored bullets, UTF-8 text, completion and `--all-tasks` in the later item, and
   separator cursors.
2. In `bob-mac-capture`, require `just format-lint`, `just build`, `just test`, and
   `just bundle` to pass on macOS 26. The GitHub Actions run for the exact current
   commit must also complete every step: Test, Bundle, plist/signature verification,
   launch smoke, and install/reinstall. A skipped downstream step is not a pass. Fix and
   republish any defect caused by the original epic or the cross-feature integration,
   then require the next exact-head workflow to go green.
3. Close `bob-cli-x` with the exact green workflow/run evidence after the
   actor-isolation fix is proven. If a different unrelated failure appears, route it
   through `/sase_new_task`; do not quietly absorb `bob-cli-v` or `bob-cli-w` into this
   epic.
4. On the signed installed app, exercise and record:
   - one task and one note, a same-target batch, and a cross-target batch;
   - a batch containing first- and second-level authored bullets;
   - live versus explicit preview ordering and exact block lines;
   - Control-J on top-level/nested rows and both placeholder depths;
   - Return versus Command-Return, unique source-order target opening, and full-draft
     retention on command/transport failure;
   - foreground banner and Notification Center expansion, richer single content, full
     ordered batch body, singular/plural actions, and opening every intended note;
   - a missing-ID parent task in a later batch item, followed by successful preview and
     capture of the still-complete draft.
5. Append one verification note to `bob-cli-t` containing commit IDs, command outcomes,
   the exact green CI URL/run ID, installed-app results, and any deliberately deferred
   unrelated warnings. Do not close the original epic in this phase.

## Phase 4: Close bob-cli-t and clean its expired symbol allowances

This is the final phase and owns the complete landing sequence requested for the
original epic.

1. Run `sase bead show bob-cli-t`, show all three children again, and review every note
   added since this plan was written. Confirm all descendants remain closed with
   resolution `done`, all epic-owned discovered issues are resolved, and task
   `bob-cli-x` is closed with green-CI evidence.
2. Re-run the cross-repository commit audit from the first `bob-cli-t` commit to the
   current tips. Inspect every later non-`bob-cli-t` commit, including any successors to
   `2037307`/`dff08a7`, and complete any final integration they require. If new child
   `PROPOSED FOLLOW-UP:` entries exist, process each through `/sase_new_task` before
   closing.
3. Close `bob-cli-t` with `sase bead close bob-cli-t --note "..."`. The note must state
   what source and commits were reviewed, the final CLI/macOS/installed-app evidence,
   how concurrent task-ID work was integrated, and all proposal outcomes: `bob-cli-v`
   and `bob-cli-w` were created as ready unrelated tasks, while `bob-cli-x` was created
   for the unrelated gate blocker and completed as a landing prerequisite. If closing is
   rejected, finish or reopen the named descendants. Never force a successful resolution
   merely to bypass the guard; use forced canceled/superseded semantics only for a
   deliberate non-done outcome with a real reason.
4. Only after the close succeeds, run `just symvision` in `bob-cli` if the recipe
   exists. Remove every stale `bob-cli-t` symbol-whitelist entry and genuinely unused
   code it reports, rerun `just symvision`, and run the focused formatting/tests
   appropriate to any cleanup. Preserve unrelated worktree changes.
5. Resolve `plan:202608/multi_capture.md` through SASE and add `status: done` to its
   YAML frontmatter. Verify `sase bead show bob-cli-t` reports closed, the plan link
   remains healthy, both repositories are clean or contain only explicitly owned landing
   changes, and the follow-up IDs in the close note are accurate.
