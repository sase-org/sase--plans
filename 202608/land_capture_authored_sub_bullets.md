---
tier: epic
title: Finish and land authored sub-bullets
goal:
  Close the remaining Bob Mac Capture contract and verification gaps found while landing
  bob-cli-m, integrate the result with concurrent capture work, and close the original
  epic only after its exact preview and native-editing promises are verified.
phases:
  - id: mac-contract-remediation
    title: Complete and verify the macOS capture contract
    depends_on: []
    size: medium
    description:
      "mac-contract-remediation: update bob-mac-capture on its latest origin/master so
      explicit Preview renders every returned block line in authoritative Bob order,
      Backspace intervenes only for an unmodified empty bullet row, the native edit
      paths have executable regression coverage, and the integrated macOS checks pass."
  - id: land-authored-sub-bullets
    title: Reverify and close bob-cli-m
    depends_on:
      - mac-contract-remediation
    size: small
    description:
      "land-authored-sub-bullets: re-audit the integrated CLI and macOS commits, record
      both proposed-follow-up outcomes, close bob-cli-m without force, run post-close
      symvision cleanup, and mark the original capture_authored_sub_bullets plan done."
proposed_by: bbugyi200.athena.bob-cli-m.land
parent_bead: bob-cli-m
create_time: 2026-09-09 20:00:11
status: wip
---

- **PROMPT:**
  [prompts/202608/land_capture_authored_sub_bullets.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/land_capture_authored_sub_bullets.md)
- **PARENT:**
  [202608/capture_authored_sub_bullets.md](https://github.com/sase-org/sase--plans/blob/main/202608/capture_authored_sub_bullets.md)

# Finish and land authored sub-bullets

## Why this plan exists

The `bob-cli-m` landing audit verified that both original phases are closed and that
their commits are present:

- `2d6b0afe9053ce9ce6ccc6ccb08f73d7948286d0` (`bob-cli-m.1`) owns the
  physical-line-aware Rust grammar, rendered authored children, complete-stdin
  contracts, documentation, and test coverage.
- `727b05d0be377490fd27b47d29a72613e449f4f9` (`bob-cli-m.2`) is on `bob-mac-capture`
  `origin/master` and consumes the additive `sub_bullets` contract, adds native
  Ctrl-J/Backspace editing, uses the real caret byte offset, and displays authored
  children.

The Rust tree is clean and its integrated verification passed:
`cargo test capture_language` (77 tests), all 14 authored-bullet CLI tests, focused
later-commit wikilink/clip-contract tests, `just all` (538 unit, 308 CLI, 27 Dataview,
31 Tasks, and 1 real-vault parity test), `just install-smoke`, and `git diff --check`.

The audit nevertheless found two original-plan gaps in `bob-mac-capture`:

1. `CaptureCommandSuccess` already decodes `clip.lines` and `scheduleLog.lines`, but
   `PreviewPane.previewContent` renders only `taskLine` and `subBullets`. Explicit
   Preview therefore does not mirror the exact parent -> authored children -> clipboard
   children -> schedule-log block promised by the original plan and README.
2. `CaptureKeyCommandRouter` maps Shift-Backspace to `.deleteBackward`, although the
   original plan limits placeholder-row intervention to plain Backspace and requires
   every modified delete to pass through to AppKit. Existing tests currently assert the
   wrong Shift-Backspace behavior, and the native edit paths lack the complete
   executable selection/caret/undo/pass-through coverage requested by the plan.

The macOS 26 workflow for the phase commit also failed before running tests because an
unrelated pre-epic autosizing test used ambiguous untyped `.greatestFiniteMagnitude`.
`/sase_new_task` found no task duplicate and attached the evidence to active
release-gate epic `bob-cli-n` as a `DISCOVERED ISSUE`; do not create another task or
silently absorb that distinct defect into `bob-cli-m`.

The other collected proposal, from `bob-cli-m.1`, was unrelated suite flakiness in
`tests/cli.rs::nightly_runs_shared_sync_once_then_wrapped_steps_in_order`. The duplicate
and active-epic sweep found no owner, so it is now ready task `bob-cli-o`, sized large
because the suspected parallel shim-recompile race is not yet proven.

## Phase `mac-contract-remediation`: complete and verify the macOS contract

Work only in the linked source repository, opened from the Bob CLI checkout with:

```sh
sase repo open bob-mac-capture -r "Finish bob-cli-m exact preview and native-edit verification"
```

Use the printed path for every read, write, and command. Start from a clean checkout and
integrate the latest `origin/master`; `bob-cli-n` is concurrently changing the same
completion/view surface, so review every commit after `727b05d0` and preserve its
behavior. Do not edit a deployed application copy.

### Exact preview model

1. Give `CaptureCommandSuccess` one testable, authoritative rendered-block projection
   whose order is exactly `taskLine`, `subBullets`, `clip?.lines`, then
   `scheduleLog?.lines`. Keep the server-provided strings byte-for-byte; do not
   re-indent, classify, flatten, deduplicate, or rebuild Markdown in Swift.
2. Make `PreviewPane` render that complete projection for live and explicit responses.
   Live preview still invokes `--no-clip`, so it naturally has no clipboard lines and
   must retain the literal-clipboard notice. Explicit Preview must display returned
   clipboard and schedule-log children after authored children.
3. Base the scroll/clamp decision on the complete rendered block, not merely whether
   `subBullets` is empty. Preserve the concurrent autosizing/auxiliary-overflow work,
   metadata row, text selection, accessibility, and monospaced exact indentation.
4. Update README wording so it explicitly states the full returned-block order and the
   live-preview `--no-clip` exception.

Add pure model tests covering parent-only; each optional source alone; and the combined
parent/authored/clip/schedule order. Extend fixtures and process/model tests with one
explicit-preview response containing all four layers, including the now-always-present
leaf `clip.entries: []` contract from Bob CLI commit `7fa06585`. Add a view-focused test
at the smallest stable seam available; avoid pixel snapshots.

### Native key behavior and tests

1. Route `.deleteBackward` only for an otherwise unmodified Backspace event. Shift,
   Option, Command, Control, range selections, nonempty rows, unrelated responders, and
   noneditable views must pass through untouched. Preserve Ctrl-J as control-only and
   preserve Shift/Option-Return as ordinary native newlines.
2. Exercise `perform(.insertBulletNewline)` against a real editable `NSTextView` for an
   end insertion and a middle selection replacement. Assert the resulting text and
   collapsed caret location; verify the edit uses the text system and is undoable.
3. Exercise `perform(.deleteBackward)` for middle, final, and first-line exact `- `
   placeholders, and assert actual text/caret results plus pass-through cases. Keep IME
   and accessibility native—no synthetic events and no direct `CapturePanelModel` string
   rewriting.

### Integrated verification

Run the strongest local checks the current host supports. Then require a green macOS 26
workflow on the exact final `bob-mac-capture` commit before handing off to the landing
phase. The workflow must complete formatting, build, tests, bundle, plist/signature,
launch smoke, and install/reinstall—not merely compile the app.

The known `.greatestFiniteMagnitude` compile failure belongs to `bob-cli-n`. Inspect
`sase bead show bob-cli-n` and its children before waiting for CI. If its fix has
landed, integrate it. If it has not, coordinate with that active epic and wait using the
SASE monitor workflow; do not duplicate the task and do not claim `bob-cli-m` verified
while tests are still blocked. On a Mac, also perform the focused original interaction
check: Ctrl-J at end/middle, empty-row Backspace, modified-Backspace pass-through, a
marker and wikilink completion on an earlier child at the real caret, and explicit
Preview with authored + clipboard + rolled-priority schedule lines in exact order.

Commit linked-repository changes through `/sase_git_commit` only if the phase runner is
explicitly authorized by its launch/finalizer workflow to commit. Leave both repository
worktrees clean and record the exact final commit and CI URL on the phase bead.

## Phase `land-authored-sub-bullets`: reverify and close `bob-cli-m`

This is the final phase and owns the complete landing sequence.

1. Rerun `sase bead show bob-cli-m`, `bob-cli-m.1`, and `bob-cli-m.2`; reread every note
   and the original plan at `plan:202608/capture_authored_sub_bullets.md`. Confirm the
   two proposals have these durable outcomes: the phase-1 flake is ready task
   `bob-cli-o`, and the phase-2 preview proposal was completed by
   `mac-contract-remediation` rather than filed separately because it was caused by the
   epic.
2. Review the final source and every Bob CLI commit after `2d6b0afe` plus every
   `bob-mac-capture` commit after `727b05d0`. Confirm later wikilink, clip JSON,
   completion-presentation, autosizing, draft-retention, and keyboard work is preserved
   and no duplicate parsing or rendering path was introduced.
3. From Bob CLI, rerun `cargo test capture_language`, `cargo test --test cli capture`,
   `just all`, `just install-smoke`, and `git diff --check`. Confirm the exact
   linked-repo final commit has a fully green macOS workflow and that its worktree is
   clean.
4. Close without force:

   ```sh
   sase bead close bob-cli-m --note "Verified both original phase commits and all child notes against source; integrated post-start wikilink, clip-contract, autosizing, draft-retention, completion, and keyboard changes; completed exact parent/authored/clip/schedule preview plus unmodified-Backspace behavior and native edit tests; Bob CLI checks and the final macOS workflow passed. PROPOSED FOLLOW-UP outcomes: bob-cli-m.1 -> ready task bob-cli-o; bob-cli-m.2 -> fixed inside this epic because the missing preview lines were original epic work. The unrelated macOS CI ambiguity was attached to active epic bob-cli-n and integrated after its fix landed."
   ```

   If the close is rejected, finish or reopen the named phase; never force merely to
   make the command succeed.

5. Only after the close succeeds, run `just symvision` when that recipe exists. Remove
   every stale `bob-cli-m` whitelist entry and any now-unused code it reports, rerun the
   relevant checks, and commit those changes only through `/sase_git_commit` when the
   finalizer authorizes it.
6. Finally, edit the canonical original plan file shown by `sase bead show bob-cli-m`
   and add `status: done` to its YAML frontmatter. This is the original
   `capture_authored_sub_bullets.md`, not this remediation plan. Verify
   `sase bead show bob-cli-m` reports closed, the original plan reports done, and both
   worktrees are clean.
