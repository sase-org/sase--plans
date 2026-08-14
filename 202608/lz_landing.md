---
tier: tale
title: Finish and land epic sase-lz
goal:
  The Models panel selector builder rejects nested custom selectors, the docs are
  accurate, all epic work is verified and committed, and sase-lz is closed with its plan
  marked done.
size: small
proposed_by: bbugyi200.athena.sase-lz.land
bead: sase-lz
create_time: 2026-08-14 12:56:19
status: done
---

- **PARENT:**
  [202608/models_panel_pool_authoring.md](https://github.com/sase-org/sase--plans/blob/main/202608/models_panel_pool_authoring.md)
- **BEAD:**
  [sase-lz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-lz/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-lz.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-lz.land.md)
- **COMMITS:**
  - [6ee3347](https://github.com/sase-org/sase/commit/6ee334708e366715046bc7f871ca66f234794126)
    — fix(ace): reject typed selectors in builder members

# Finish and land epic sase-lz

## Context and verified audit

Epic `sase-lz` adds guided model-pool and fallback authoring to the ACE Models panel.
Its four phase commits are:

- `adea6b1df` (`sase-lz.1`): reject selector syntax in temporary Override.
- `a605d5c09` (`sase-lz.2`): shared parse-based selector helpers and prefilled custom
  Edit input.
- `877465a5a` (`sase-lz.3`): guided selector builder, picker entry point,
  member-operation alias safety, tests, styles, and PNG snapshots.
- `4d5598eaf` (`sase-lz.4`): ACE and LLM documentation.

All child beads are closed and their notes were reviewed. The one `PROPOSED FOLLOW-UP:`
from `sase-lz.4` is unrelated monitor work. Landing triage found exact duplicate task
`sase-ll`; a verified-after-close `+1` from `sase-lz.land` reopened it to READY, and the
recurrence was also recorded as a `DISCOVERED ISSUE:` on causally owning active epic
`sase-kp`. Do not create a second task.

The commits landed on a linear `master` alongside unrelated monitor, workspace-claim,
commit-finalizer, and provider-scoped `%model` completion changes. The completion change
edits the prompt directive completion catalog, not the Models-panel picker; it neither
duplicates nor conflicts with the builder. No integration rewrite is needed there.

The source audit found two remaining defects caused by this epic:

1. `SelectorBuilderModal._on_member_custom_picked()` only calls `member_rejection()`. A
   typed value such as `claude/opus | codex/o3` therefore enters the builder as one
   displayed row, but `compose_selector()` silently turns it into multiple semantic
   members. The authoritative validator accepts the flattened final expression, so the
   builder violates its promise that one custom entry is one non-selector member and
   that nested selectors are gated.
2. `docs/ace.md` says "overrides are config-only" in the Override paragraph. Selectors
   are config-only; temporary overrides are single-target. Correct the subject without
   changing the documented behavior.

## 1. Reject selector syntax in custom builder members

- Update `src/sase/ace/tui/modals/models_panel_selector_builder.py` to route the
  stripped custom-member value through the existing `parse_selector_for_display()`
  helper before `member_rejection()` or the effort picker.
- If parsing reports malformed selector syntax, show that parser message as a warning
  and leave the member list unchanged.
- If parsing returns a pool or fallback, reject it with a concise warning that a builder
  member must be a single target. Do not flatten it, open the effort picker, or mutate
  builder state.
- Continue to use `member_rejection()` for a single `@alias`, including an alias chain
  that reaches a selector. Keep final `validate_model_alias_selector_value()` validation
  authoritative.
- Extend `tests/test_models_panel_selector_builder.py` with focused regression coverage
  for typed pool/fallback member rejection and malformed mixed syntax. Assert no member
  is appended and the builder remains active.

## 2. Correct documentation and verify integration

- In `docs/ace.md`, change the sentence that says overrides are config-only to say
  selectors are config-only and overrides take a single target.
- Re-read the Models-panel selector sections in `docs/ace.md`, `docs/llms.md`, and
  `docs/configuration.md` against current source. Keep `docs/configuration.md` unchanged
  unless a concrete contradiction remains.
- Re-check commits after `adea6b1df`, excluding the four `sase-lz` commits, and confirm
  none now needs to call the builder/helper or duplicates its picker behavior. Record
  that conclusion in the epic close note.
- Run the focused builder, Edit, Override, and picker-alias tests. Run
  `just test-visual` because phase 3 added/shifted PNG goldens. Then run `just check`.
- Run `just check-full` only through `/sase_monitor`, with an explicit current lane to
  avoid the separately tracked implicit-lane defect `sase-ll`, and a `--next` action
  that resumes this landing workflow. Investigate any failure enough to distinguish an
  epic regression from pre-existing unrelated work; epic-caused failures must be fixed
  before proceeding.

## 3. Commit, close, and finalize the epic

- Review the final diff and use `/sase_git_commit` to commit the source, test, and
  documentation corrections as `sase-lz` landing work. Do not include unrelated user or
  concurrent-agent changes.
- Close the epic normally with `sase bead close sase-lz --note "..."`. The note must
  summarize all four child/source/commit verifications, the post-start integration
  audit, verification commands and outcomes, both landing fixes, and the sole follow-up
  outcome: proposing bead `sase-lz.4` matched `sase-ll`, whose verified-after-close `+1`
  reopened it to READY, with the same recurrence also attached to active epic `sase-kp`.
  Do not use `--force` unless a genuinely unfinished descendant is deliberately canceled
  or superseded for a documented reason.
- After the close succeeds, run `just symvision`. Remove any expired `sase-lz`
  epic-symbol whitelist entries and unused code it reports, verify the cleanup, and
  commit it through `/sase_git_commit` if it changes the repo.
- Finally edit the linked epic plan `plan:202608/models_panel_pool_authoring.md` so its
  frontmatter says `status: done` (not `wip`). Preserve all other plan content. Confirm
  `sase bead show sase-lz` reports CLOSED and the plan file reports done.
