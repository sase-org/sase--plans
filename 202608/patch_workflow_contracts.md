---
tier: tale
title: Canonicalize Patch and stitch workflow contracts
goal:
  Make Patch and stitch canonical across non-TUI workflows, CLI, and machine metadata
  while preserving legacy entry points and stored data compatibility.
size: medium
proposed_by: bbugyi200.athena.sase-hn.3
bead: sase-hn.3
create_time: 2026-08-08 17:14:06
status: wip
---

- **PARENT:**
  [202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_and_stitch_terminology.md)
- **BEAD:**
  [sase-hn.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.3.md)

# Canonicalize Patch and stitch workflow contracts

## Goal

Complete phase `sase-hn.3` by making Patch/stitch the canonical vocabulary in SASE's
non-TUI lifecycle workflows, automation, CLI, and machine-facing metadata while
preserving every existing ChangeSpec/commit-entry compatibility entry point and all
workflow semantics.

Phase 2 has already established `sase.ace.patch`, `Patch`, `Stitch`, `.stitches`, patch
locks/writers/discovery, canonical Rust wire records, and thin legacy aliases. This
phase consumes those APIs. It does not perform the ACE TUI/config rename, linked
repository work, documentation/memory/skill-source migration, or the final
cross-repository terminology audit owned by later epic phases.

## Compatibility and semantic constraints

- Keep lifecycle states, parent/child rules, archive placement, suffix behavior, branch
  mapping, proposal IDs, hook/mentor/comment eligibility, rebase/rewind behavior, and PR
  submission semantics unchanged.
- Continue to treat actual VCS commits, SHAs, commit logs/statistics, and the
  `sase commit` command as commits. Rename only Patch history entries and their IDs to
  stitches.
- A real tracked VCS commit creates a numeric stitch; a proposal creates a commitless
  letter-suffixed stitch such as `2a`.
- A Patch without `PR:` remains valid. Existing local PR-creation/submission paths must
  still record the resulting URL, and this phase must not add external PR import.
- Read durable legacy names including `changespec_name`, `commit_changespec_name`,
  `commit_entry_id`, `meta_changespec`, ChangeSpec-tagged payloads, legacy bead/agent
  marker fields, and legacy completion/query values. Expose canonical Patch/stitch
  properties and new metadata where compatibility permits; preserve or dual-emit stable
  legacy serialized fields needed by existing consumers so attribution is never lost.
- Preserve `sase changespec`, `--changespec`, old Python imports/functions, and existing
  machine-output behavior as tested aliases. New help, diagnostics, and source call
  sites use `sase patch`, `--patch`, Patch, and stitch.

## Implementation

### 1. Canonicalize lifecycle and workflow internals

- Move non-TUI callers from `sase.ace.changespec` and legacy core conversion names to
  `sase.ace.patch` and the canonical Patch/Stitch APIs introduced by phases 1 and 2.
- Rename the workflow helpers/modules and internal arguments that manipulate Patch
  history: Patch creation/reservation/querying, ProjectSpec mutations, commit tracking,
  proposal acceptance/rejection, rewind/renumber, rebase, restore/revert, status
  transitions, deltas, refs, comments, hooks, mentors, timestamps, running fields,
  archive/mail operations, and bare-git workspace submission.
- Keep deliberately small compatibility modules/aliases at the old import paths. Ensure
  all hook, mentor, suffix, proposal, and timestamp cross-references use the same
  canonical stitch ID, including renumbering and proposal acceptance.
- Update human-facing non-TUI diagnostics and query/search rendering to Patch/stitch,
  without touching ACE widget/action/config surfaces assigned to phase 4.

### 2. Add the canonical CLI and option aliases

- Register `sase patch` as the canonical command group and retain `sase changespec` as
  an exact alias routed to the same parser destinations and handler implementation.
  Provide the existing `current`, `search`, `ref`, `sync-deltas`, and
  `migrate-extension` behavior through both spellings.
- Rename the handler/parser modules and user-facing usage/help to Patch terminology.
  Keep legacy module-level imports as compatibility facades where public callers or
  tests depend on them.
- Add canonical `-p/--patch` targeting where commands currently expose `--changespec`,
  choosing non-conflicting short aliases in command contexts that already reserve `-p`;
  accept `--changespec` into the same normalized destination. Keep the command inventory
  sorted and every new public long option paired with a short option per CLI policy.
- Test full and narrow parser construction, command dispatch, help output, exit codes,
  JSON/plain/markdown output equivalence, ref operations, current/search behavior, delta
  sync, and extension migration through both command names.

### 3. Migrate automation and durable machine metadata compatibly

- Introduce canonical Patch/stitch names in agent scan/completion/result markers, runner
  setup/cleanup, xprompt workflow outputs, history/MRU records, branch maps,
  query/completion wire conversions, stats, notifications, mobile helper responses, bead
  models/database projections/JSONL, and plugin/provider-facing Python protocols owned
  by this repository.
- Add canonical properties/constructors and explicit dual-read adapters for existing
  agent archives, result markers, notification fixtures, database rows, JSONL, and
  persisted workflow state. Only dual-write where an installed legacy consumer still
  requires the old key; otherwise retain the stable serialized key behind a canonical
  API property to avoid gratuitous schema churn.
- Add or extend migrations and golden fixtures where stored schemas change. Verify old
  records load with identical Patch attribution and that canonical records survive round
  trips without silently dropping IDs, project names, PR URLs, or stitch links.
- Update bundled automation scripts and xprompt workflow definitions that execute these
  contracts, but leave generated skill sources and linked integration repos for their
  dedicated later phases.

### 4. Prove invariants and compatibility

- Add focused tests for canonical and legacy workflow imports, Patch lifecycle
  transitions, stitch renumber/cross-reference behavior, commit/proposal tracking,
  agent/bead/notification/mobile metadata compatibility, and CLI alias parity.
- Add explicit invariant coverage for Patch-without-PR validity, local PR URL recording,
  no external PR discovery, numeric committed stitches, and commitless lettered proposal
  stitches.
- Run `just install` before checks. Run targeted suites while iterating, including the
  CLI/parser, commit/accept/rewind, hooks/mentors/status, workspace-provider,
  agent-marker, bead JSON/database, notification/mobile, query/history, and xprompt
  tests.
- Run `just check`. If the scoped selector escalates, reports unusual selection, or this
  phase touches a broadening-set file, run `just check-full` as required. Re-run focused
  compatibility and invariant tests after final cleanup and inspect the final diff for
  accidental TUI/config/docs/integration scope creep.
- Record any out-of-scope defect as a `PROPOSED FOLLOW-UP:` note on `sase-hn.3` rather
  than creating a task bead. When all verification passes, close only `sase-hn.3` with a
  note listing the successful checks; do not close the parent epic.

## Expected result

New code and non-TUI output use Patch/stitch, `sase patch` is the documented command,
and machine-facing consumers can use canonical Patch/stitch properties without losing
access to existing data. Legacy commands, options, imports, stored records, and
machine-output semantics remain compatible, while real VCS commit concepts remain
unchanged.
