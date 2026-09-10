---
tier: epic
status: done
title: Complete the multi-item capture landing
goal:
  Prove the newly integrated Bob Mac Capture tip on macOS and in the signed installed
  app, then close bob-cli-t.4 with every proposal and post-close artifact resolved.
phases:
  - id: validate_latest_mac_integration
    title: Validate the latest macOS integration tip
    depends_on: []
    size: medium
    description:
      "validate_latest_mac_integration: verify the combined multi-item capture and
      task-ID completion behavior after dcbc6b7, fix any regression it exposes, and
      require the complete exact-head macOS release workflow to pass."
  - id: exercise_signed_batch_experience
    title: Exercise the signed installed-app batch experience
    depends_on:
      - validate_latest_mac_integration
    size: medium
    description:
      "exercise_signed_batch_experience: guide and record the owner-assisted signed-app
      matrix for batch capture, keyboard editing, previews, notifications, target
      opening, failure retention, and later-item task-ID assignment."
  - id: land_multi_capture_followup
    title: Close bob-cli-t.4 and finish its landing artifacts
    depends_on:
      - exercise_signed_batch_experience
    size: small
    description:
      "land_multi_capture_followup: perform the final drift and note audit, record every
      proposal outcome, close bob-cli-t.4, run post-close symvision cleanup if
      available, and mark plan:202608/land_multi_capture.md done."
proposed_by: bbugyi200.athena.bob-cli-t.4.land
parent_bead: bob-cli-t.4
create_time: 2026-09-09 19:59:57
---

- **PROMPT:**
  [prompts/202608/complete_multi_capture_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/complete_multi_capture_landing.md)
- **PARENT:** [202608/land_multi_capture.md](land_multi_capture.md)

# Complete the multi-item capture landing

## Why this plan exists

The land audit confirmed that all four children of `bob-cli-t.4` are closed with
resolution `done` and that their published implementation exists:

- `bob-cli` commit `3beae5b` adds the later-batch `capture-complete --all-tasks`
  regression for shared item parsing, global UTF-8 ranges, missing-ID candidates,
  ordering, and separator no-op behavior.
- `bob-mac-capture` commit `fc1c16b` makes only the pure bullet-indentation resolver
  chain nonisolated. The `bob-cli-t.4.1` note incorrectly names then-current base commit
  `a1ae3fd`; the actual actor-isolation fix is `fc1c16b`.
- `bob-mac-capture` commits `49f0037`, `181a644`, `db0460d`, `d877624`, and `a9ffab9`
  cover and stabilize one-argument later-batch completion, global task-ID replacement,
  preview and capture behavior, cancellation, server/transport/stale failures, and draft
  retention.
- macOS 26 CI run `31894453684` passed every release step at `a9ffab9`, proving the
  indentation fix and closing unrelated prerequisite task `bob-cli-x`.

The current `bob-cli` tip remains `3beae5b`; `git diff --check` and `just all` pass with
only the three warnings already tracked by unrelated task `bob-cli-v`. A fresh
`bob-mac-capture` fetch found two commits after the child-phase audit:

- `a35003a` changes only canceled-stash deletion behavior and does not intersect batch
  capture.
- `dcbc6b7` directly overlaps task-ID completion. It makes cached concrete-route
  completions authoritative, preserves Bob completion for the task side of `+`, and adds
  editor/prompt focus requests. Source review shows the later-item global range and
  one-process contracts remain intact. At `dcbc6b7`, `git diff --check`, fake-Bob syntax
  validation, `swift build --target CaptureCore`, and
  `swift build --target CaptureCoreTests` pass on Linux with only warnings tracked by
  unrelated task `bob-cli-w`. Exact-head macOS CI run `31895172100` was still in
  progress during this audit.

Two `PROPOSED FOLLOW-UP:` entries were collected:

1. `bob-cli-t.4.3` proposed the signed installed-app UI/notification matrix because its
   Linux agent could not exercise it. This was an explicit phase acceptance criterion,
   so it remains epic work and is Phase 2 below rather than a separate task.
2. `bob-cli-t.4.4` proposed repairing pre-existing plan-link drift. The required
   `/sase_new_task` search, recent-task sweep, and active-epic check found no duplicate
   or causal epic. Ready large task `bob-cli-y` now owns migration of the 57 legacy
   prompt Markdown files. The separate `parent-unpublished` warning for
   `202608/bob_cli_n_landing.md` was declined because it is an intentional transient
   reference that resolves when its parent plan lands.

## Phase 1: Validate the latest macOS integration tip

Open `bob-mac-capture` through `sase repo open`, fetch the current base branch, and do
not assume `dcbc6b7` remains the tip.

1. Review every commit newer than `dcbc6b7` and every non-`bob-cli-t.4` commit since
   `fc1c16b` for duplication or conflict with batch capture, cached route completion,
   later-item task completion, prompt focus, or full-draft preview and submit.
2. Preserve the intended combined boundary: a concrete route span with a usable cache is
   answered only by the cache, while the right side of `@route+query` still calls
   `capture-complete --all-tasks` with the entire draft as one argument. Late responses
   must not overwrite cached route results.
3. Run `git diff --check`, fake-Bob syntax/JSON checks, and every locally available
   Swift build/test target. On macOS 26, run `just format-lint`, `just build`,
   `just test`, and `just bundle`. Fix and publish any failure caused by the combined
   completion/batch behavior.
4. Require a green workflow for the exact final commit. Every release step must pass:
   formatting, build, tests, bundle, plist/signature verification, launch smoke, and
   install/reinstall. A skipped downstream step is not success. Record the commit, run
   ID/URL, and command outcomes on this plan's validation phase.

## Phase 2: Exercise the signed installed-app batch experience

Use the exact signed app proven by Phase 1. Guide the owner through the checks when
physical Mac interaction is required, and record concrete pass/fail results rather than
accepting a general statement that the app works.

1. Capture one task and one note, a same-target batch, and a cross-target batch. Include
   first- and second-level authored bullets and confirm the final vault blocks and
   source order.
2. Compare live and explicit preview ordering and exact block lines. Exercise Control-J
   on top-level, nested, and both placeholder depths, plus Return versus Command-Return.
3. Confirm unique source-order target opening and full-draft retention on Bob command,
   capture, and transport failure.
4. Verify the foreground banner and expanded Notification Center content for rich single
   and ordered batch results, singular/plural actions, and opening every intended note
   without duplicates.
5. In a later batch item, choose a missing-ID parent task, assign its ID, and then
   preview and capture. Confirm the earlier UTF-8 item and separator remain unchanged,
   focus returns to the editor after success and remains in the prompt after a failure,
   and the complete draft is submitted once for preview and once for capture.
6. If any behavior fails, fix it in the owning repository, rerun the focused automated
   regressions and exact-head release workflow, reinstall that exact build, and repeat
   the failed interactive check before closing this phase.

## Phase 3: Close bob-cli-t.4 and finish its landing artifacts

This is the final phase and owns the complete landing sequence requested for
`bob-cli-t.4`.

1. Fetch both repositories again. Starting with `3beae5b` in `bob-cli` and `fc1c16b` in
   `bob-mac-capture`, inspect every later commit and integrate any new duplication or
   conflict. Re-run the focused checks appropriate to any new tip or landing edit.
2. Run `sase bead show bob-cli-t.4` and show all four children. Review every note added
   since this plan was written. Process each new `PROPOSED FOLLOW-UP:` through
   `/sase_new_task`; epic-caused defects remain epic work and must be finished first.
3. Close with `sase bead close bob-cli-t.4 --note "..."`. The note must record the
   audited source and commits, final Rust/macOS/installed-app evidence, integration of
   `dcbc6b7` and any successors, the `fc1c16b` note correction, and every proposal
   outcome: signed-app validation completed as epic work; `bob-cli-y` owns the unrelated
   57-file migration; the transient parent warning was declined; `bob-cli-v` and
   `bob-cli-w` remain unrelated ready tasks; and `bob-cli-x` was completed as the gate
   prerequisite. If closing is rejected, finish or reopen the named descendants. Never
   force merely to bypass the close guard.
4. Only after close succeeds, run `just symvision` in `bob-cli` if the recipe exists.
   Remove stale `bob-cli-t.4` whitelist entries and genuinely unused code it reports,
   then rerun symvision and focused formatting/tests. If the recipe still does not
   exist, record that explicitly.
5. Resolve `plan:202608/land_multi_capture.md` through SASE and add `status: done` to
   its YAML frontmatter. Confirm `sase bead show bob-cli-t.4` reports closed, its plan
   link remains healthy, and both repositories are clean or contain only deliberately
   owned landing changes.
