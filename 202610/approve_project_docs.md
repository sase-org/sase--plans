---
tier: tale
title: Document gateless approve project frontmatter and close sase-1jw
goal: "Say in the gateless-approval docs that a proposal's frontmatter project supplies
  the project when --project is omitted, then close epic sase-1jw.

  "
size: small
proposed_by: bbugyi200.athena.sase-1jw.land
bead: sase-1jw
status: done
---

- **PARENT:**
  [202610/plan_project_frontmatter.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_project_frontmatter.md)
- **BEAD:**
  [sase-1jw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jw/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1jw.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1jw.land.md)
- **COMMITS:**
  - [af4f0da](https://github.com/sase-org/sase--plans/commit/af4f0daed489f351272140e20c15b224d12b9e4c)
    — docs: mark plan_project_frontmatter epic done

# Document gateless approve project frontmatter and close sase-1jw

## Problem

Epic sase-1jw is otherwise complete. Its three phases are closed, and the behavior they
describe is on master:

- `sase plan propose` stamps `project` with the proposing project's display name
  (`effective_project_name`). The field stays on the machine-local plan and is stripped
  by `archive_plan_file`, `handle_sase_plan`, and `write_sdd_files`.
- Gateless `sase plan approve` reads that field immediately after `-P` and refuses an
  unknown value instead of falling through to the working directory. Recovery looks up
  the committed plan from the resolved project's primary checkout. The dry-run line
  prints that project's display name.
- `sase bead work <plan-file>` routes by the field when the working directory is outside
  that project. There is no `--project` flag on `sase bead work`. Reviewer decisions
  `project_value = display` and `bead_work_project_flag = no` are already honored. Do
  not implement the other branches.

Phase `approve` never updated its docs. `docs/sdd.md`'s direct-approval paragraph and
the `sase plan approve` row of `docs/cli.md` still mention only `--project`. They do not
say that frontmatter `project` supplies the project when `--project` is omitted.

## Changes

1. In `docs/sdd.md`, in the paragraph that begins "The command can also directly approve
   an archived or scratch plan with no live gate", keep the existing `--dry-run` /
   `--project` sentence and add this sentence immediately after it:

   When `--project` is omitted, a proposal's frontmatter `project` field supplies the
   project.

2. In `docs/cli.md`, in the `sase plan approve` table row, keep the existing sentence
   that `--project` identifies a gateless plan's project. Add the same fact in that
   cell: when `--project` is omitted, the proposal's frontmatter `project` field
   supplies the project.

Do not change command behavior, parsers, or tests. Do not add a `--project` flag to
`sase bead work`.

## Verification

Read `lint_and_test.md` with `sase memory read` and follow it. Format with `just fmt`.
Verify with `sase tool run check`. Do not run `just check-full`.

A docs-only diff should not select the pre-existing import-budget or git-identity
failures. If `just check` still fails only those nodes, they are already triaged on
sase-1jw (sase-1ic and sase-120) and are not this tale's defect. Treat any UNKNOWN
failure as yours.

## Closeout

Do this in the same turn as the doc edit, after verification. Do not wait for this
turn's commit. Do not use `--force` to make `sase bead close` succeed.

1. Run `sase bead epic-symbols sase-1jw`. The landing review found no entries. If any
   are listed, resolve each one (wire it up, privatize it, add a non-test pragma, or
   delete it) or re-key that Justfile line to a still-open later bead that still needs
   the exemption. Close refuses while any entry remains.

2. Close the epic:

```bash
sase bead close sase-1jw --note "<note>"
```

Use a note that records all of the following.

Verified on master, by reading the phase beads, the three `sase-1jw` commits, and the
current source: propose stamps the display name and archive/store writers strip it;
gateless approve resolves frontmatter `project` after `-P` and refuses an unknown value;
recovery uses the resolved project's primary checkout; bead work routes outside that
project and does not grow a `--project` flag. Both reviewer decisions are honored.
sase-core commit `96f7796d` accepts the field and is inside the `sase-core-revision.txt`
pin. Later display-snapshot and formatted-propose commits already share
`effective_project_name` and authoring-mode acceptance of `project`, so they needed no
further edit. The only commit after phase bead-work is the Agents Archive action
boundary, which does not touch plan routing.

This tale added the missing approve docs: frontmatter `project` supplies the project
when `--project` is omitted (`docs/sdd.md` and the `docs/cli.md` `sase plan approve`
row).

Follow-ups, already recorded on sase-1jw and repeated here: declined symvision helpers
(sase-1js and sase-1jp are closed; epic-symbols is empty), declined flag-candidate
registry (current test matches `feature_flag_definitions()` and passed in isolation),
declined completion-snapshot drift (both snapshot nodes passed; sase-1de stays closed),
declined marker-path and Bob dry-run (both passed in isolation on current master),
declined zsh smoke and tool-run store locks (not FAILED nodes in ToolRun
367c8c7a986e162fb25f532f98e93c05), declined the ACE reset-hook leak (passed on clean
HEAD; sase-13c already tracks it). Corroborated sase-1ic (+1, import budget 3516 vs
3513, not caused by this epic) and sase-120 (+1, git-identity TimeoutExpired in that
ToolRun, full lane not re-run).

3. Run `just symvision` and confirm it is clean.

4. Set `status: done` in the frontmatter of the epic plan file. Get the path from
   `sase bead read sase-1jw -r "Need the linked plan file"` (ref
   `plan:202610/plan_project_frontmatter.md`). Change `status: wip` to `status: done`.
   Do not rewrite the rest of the plan.

5. Read `sase bead read sase-1jw -r "Need the parent link"`. There is no parent bead. If
   that is still true, stop. If a parent phase or plan bead is present, handle it with
   the normal parent-landing rules: close a completed parent phase, or re-check and
   close a still-complete parent plan only after its own epic-symbol cleanup. Do not
   force a close.
