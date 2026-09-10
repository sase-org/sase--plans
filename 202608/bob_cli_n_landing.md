---
tier: tale
title: Finish and land epic bob-cli-n
goal:
  Clear the macOS release gate, verify integration, process follow-ups, and close epic
  bob-cli-n with complete evidence.
size: small
proposed_by: bbugyi200.athena.bob-cli-n.land
bead: bob-cli-n
create_time: 2026-09-09 19:59:44
status: wip
---

- **PARENT:** [202608/obsidian_link_completion.md](obsidian_link_completion.md)
- **BEAD:** bob-cli-n

# Finish and land epic `bob-cli-n`

## Context

Epic `bob-cli-n` added the authoritative Obsidian wikilink protocol to `bob-cli` and
consumed it in the linked `bob-mac-capture` app. All three child beads are closed, but
land verification found that the latest macOS 26 CI run for the phase-3 commit still
fails while compiling `BobMacCaptureTests.swift`: the test uses unqualified
`.greatestFiniteMagnitude` values where the toolchain cannot choose between `CGFloat`
and `Double`. This is the same release-gate blocker recorded on the epic bead, and the
failure prevents the test, bundle, signature, launch-smoke, and install steps from
running.

The integration window has also been audited. In `bob-cli`, post-epic commit `7fa0658`
makes `clip.entries` consistently present in JSON. In `bob-mac-capture`, non-epic
commits `727b05d` and `60ac163` landed between phases 2 and 3; their real-caret,
hierarchical-preview, and tolerant-decoding changes are already composed with the
wikilink implementation and need no further source changes.

## Phase 1: Clear the macOS release gate

1. Open the linked `bob-mac-capture` repository with `sase repo open` and an audited
   reason; preserve unrelated worktree changes.
2. In `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, qualify both dimensions of
   the test panel's `NSSize` as `CGFloat.greatestFiniteMagnitude`, exactly matching the
   compiler's expected AppKit scalar type. Keep the change limited to this known
   pre-epic test defect.
3. Run all locally available checks from the linked repository, including
   `just format-lint`, `just build`, `just test`, `just bundle`, and `git diff --check`.
   If the host cannot execute an AppKit check, record that precisely and rely on the
   macOS 26 workflow for that portion rather than claiming success.
4. After the normal post-completion finalizer publishes the fix, inspect the resulting
   GitHub Actions run with `gh`. Require all workflow steps—including test, bundle,
   plist/signature verification, launch smoke, and install/reinstall—to pass. If a new
   failure appears, diagnose and fix it as part of this phase, then revalidate.

## Phase 2: Finish the audit and follow-up triage

1. Re-read the epic and child bead notes and confirm the CI fix addresses the epic-level
   discovered issue and the phase-2/phase-3 Mac validation proposals.
2. Exercise the focused wikilink protocol and app regression suites needed to confirm
   scanner/index behavior, exact byte replacements and `cursor_after`, real-caret
   completion, stale-range rejection, semantic row presentation, accessibility model
   output, and compatibility with the intervening bullet/JSON changes.
3. Process the phase-1 `PROPOSED FOLLOW-UP:` about the transient
   `capture_clip_failures_leave_vault_untouched` `ETXTBSY` failure through
   `/sase_new_task`, identifying `bob-cli-n.1` as the proposer. Record whether it was
   corroborated, attached to a related epic, newly created with an intentional size, or
   declined for lack of a distinct reproducible issue. The Mac validation proposal is
   not a separate task because it is resolved by Phase 1.
4. Review the final diffs against the original epic plan and ensure both repositories
   are clean apart from intentional, reported changes.

## Phase 3: Land and close the epic

1. Close `bob-cli-n` with `sase bead close bob-cli-n --note "..."`. The note must
   summarize every child and commit reviewed, source/test verification, the post-start
   integration audit, the successful macOS workflow, and every proposed-follow-up
   disposition (including reasons for any decline). Do not force a close; if it is
   rejected, deliberately finish or reopen the named descendant work.
2. After the close, run `just symvision` in `bob-cli` if the recipe exists. Remove only
   stale `bob-cli-n` symbol-whitelist entries and genuinely unused code it reports, then
   rerun the relevant checks and `git diff --check`.
3. Set `status: done` in the YAML frontmatter of the epic's canonical plan file,
   `202608/obsidian_link_completion.md`, using `apply_patch` and without changing its
   substantive plan history.
4. Report the landed state, validation evidence, follow-up outcomes, and any remaining
   owner-only manual appearance/VoiceOver/IME observations without overstating what CI
   validated.
