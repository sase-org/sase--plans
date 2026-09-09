---
tier: epic
title: Reject a malformed plan-header block at validation time instead of mid-launch
goal:
  A plan whose provenance header block is malformed is rejected the first time anything
  validates it — `sase plan validate`, `sase plan propose`, and the tale/epic approval
  gate — with a diagnostic that names the offending section and its line, so an approved
  epic can never reach `sase bead work` and abort at its archive step; and if the
  archive boundary is ever reached with a malformed document anyway, it fails with that
  same actionable diagnostic instead of a bare `validation:` exception from the Rust
  binding.
phases:
  - id: core-diagnostic
    title: A header-block validity rule in the Rust plan validator
    depends_on: []
    size: medium
    description:
      "core-diagnostic: teach `plan_validate` in sase-core to emit an error diagnostic
      when the document's leading plan-header block does not parse, carrying the
      parser's reason and the offending bullet's line, without touching the plan-header
      block wire schema; then land it and let release-plz publish the wheel."
  - id: links-parity
    title: Report an invalid header block from `sase plan links validate`
    depends_on: []
    size: small
    description:
      "links-parity: `_link_validation.py` silently skips a plan whose header
      disposition is INVALID while `plan_links_refresh` reports `header-invalid` for the
      same document; give the validator the same issue so the two commands agree on what
      a broken header block is."
  - id: core-adopt
    title: Adopt the release and pin the rule at every Python validation surface
    depends_on:
      - core-diagnostic
    size: small
    description:
      "core-adopt: raise the `sase-core-rs` floor to the release from core-diagnostic,
      pin with tests that the new diagnostic fires at `sase plan validate`, at the
      approval gate, and at `sase bead work` before any lock or store mutation, and
      update the `--explain` header-block note from advice into a stated rule."
  - id: archive-guard
    title: An actionable failure at the archive boundary
    depends_on:
      - core-adopt
    size: small
    description:
      "archive-guard: `archive_plan_file` projects header sections before it validates,
      so a malformed document escapes as a bare `validation: ...` ValueError with no
      path, line, or remedy; make the boundary refuse a malformed source before it
      mutates anything and fail with the diagnostic envelope."
  - id: land
    title: Land the plan-header validation epic
    depends_on:
      - core-diagnostic
      - links-parity
      - core-adopt
      - archive-guard
    size: small
    description:
      "land: verify the combined tree end to end against a real malformed plan, run
      `just check-full` and `just symvision`, deploy the skill/explain text if it
      changed, file the collected follow-ups with /sase_new_task, and close the epic."
proposed_by: bbugyi200.athena.ty
status: done
bead_id: sase-g4
create_time: 2026-09-09 19:51:04
---

- **PROMPT:**
  [prompts/202608/plan_header_validation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plan_header_validation.md)
- **BEAD:**
  [sase-g4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-g4/README.md)

# Plan: reject a malformed plan-header block before it can be approved

## Why this plan exists

The `selection_soundness` epic launch failed at 2026-08-06 08:48 with:

```
Error: could not archive epic plan /home/bryan/.sase/plans/202608/selection_soundness.md:
validation: unexpected trailing text in PARENT plan header section
Resume with:
  sase bead work /home/bryan/.sase/plans/202608/selection_soundness.md --yes
```

The plan's fourth line is a hand-authored parent bullet carrying a human annotation
after the Markdown link — the link followed by `(epic ...)` naming the parent epic bead
and its state. The Rust plan-header parser allows trailing text on `AGENTS` /
`ARTIFACTS` / `COMMITS` list entries but forbids it on the link-shaped `PLAN`, `PROMPT`,
`PARENT`, and `BEAD` sections, so that one suffix makes the whole header block
`invalid`.

That is not the interesting part. The interesting part is **when** it was discovered.
The rule that this bullet violates is already documented — `PLAN_HEADER_BLOCK_NOTE` in
`src/sase/main/plan_explain.py:6-10` tells every planning agent not to author the
provenance header block at all — and nothing enforces it. The malformed plan passed
`sase plan validate`, passed `sase plan propose`, passed the epic approval gate, passed
the launch's own up-front validation, passed the launch preview, and only detonated
inside `archive_plan_file`, at the first step of the mutation transaction, from a
binding-level `ValueError` with no path, no line, and no statement of what a correct
bullet looks like. The offered resume command would have failed identically forever.

The fix is not to loosen the parser. The header block is a machine-owned projection —
`refresh_existing_parent_section` rewrites that bullet from its label on every archive,
so the annotation was destined to be deleted regardless. The fix is to make a rule that
is already documented and already enforced at a mutation boundary be enforced at the
**validation** boundary, where it costs an agent one edit instead of costing the user a
failed epic launch.

## What was measured before writing this plan

All measurements taken at `6b0976bcb` in an installed workspace, against the live host.

**The malformed bullet is the whole cause, and one deletion repairs it.** Parsing the
plan returns disposition `invalid`, reason
`unexpected trailing text in PARENT plan header section`. Deleting the parenthesized
suffix and reparsing returns `canonical` with a well-formed `PARENT` section. Nothing
else in the document is wrong.

**Every gate on the path to launch passed it.**

| surface                                                 | result on the malformed plan                                   |
| ------------------------------------------------------- | -------------------------------------------------------------- |
| `sase plan validate <plan>`                             | `Validation passed: ... (0 warnings)`, exit 0                  |
| `sase plan propose` (validates before archiving)        | would pass — same validator                                    |
| epic approval gate (`require_plan_approval_validation`) | would pass — same validator                                    |
| `sase bead work <plan>` up-front validation             | `✓ Validated  tier: epic · 5 phases · 5 dependency edges`      |
| `sase bead work <plan> --dry-run`                       | exit 0, including `✓ Archived ... (preview; no files written)` |
| `sase bead work <plan>` (real)                          | **exit 1** at `archive_plan_file`                              |

The dry run is the sharpest reading: it prints a green archive line for the exact step
that cannot succeed, because `--dry-run` computes `plan_archive_destination` and never
calls `archive_plan_file`.

**The whole corpus is clean, so a new hard error is safe.** Scanning every Markdown file
for header disposition:

| corpus                                     | files | canonical | missing | invalid                          |
| ------------------------------------------ | ----- | --------- | ------- | -------------------------------- |
| committed plans store (`sase/repos/plans`) | 3441  | 3124      | 317     | **0**                            |
| machine-local plans (`~/.sase/plans`)      | 1192  | 40        | 1151    | **1** (`selection_soundness.md`) |

Promoting an invalid header block to a validation error therefore fails exactly one
document today — the one that is already failing — and cannot retroactively break the
`validate-committed-plans` gate that `just check` and `just check-full` both run.

**One rule reaches every surface.** Every plan-validation entry point in this repo
funnels through the thin Rust adapter `sase.sdd.plan_validate.validate_plan{,_file}`:

- `sase plan validate` and `sase plan propose` via `read_and_validate_plan_file`
  (`src/sase/main/plan_validate_handler.py:78`)
- the tale/epic approval gate via `_validate_plan_for_approval`
  (`src/sase/_plan_approval_protocol.py:145-159`), which the TUI modal
  (`_notification_modals.py:302-313`) and `sase plan approve`
  (`plan_approve_handler.py:121`) both call before any notification or file is mutated
- `sase bead work <plan-file>` (`src/sase/bead/cli_work_from_plan.py:95`) and
  `epic_from_plan` (`epic_from_plan.py:93`)
- the archive's own `validate_plan_for_commit` and the repo-wide
  `validate-committed-plans` sweep

A single diagnostic in the Rust validator closes all of them at once. That is why this
epic's centre of gravity is one small core change rather than a Python check bolted onto
each caller.

**The failure was clean.** The launch lock is context-managed and the raise happens
before any bead, file, or graph mutation, so nothing is half-written. The blast radius
is a blocked epic and a misleading resume hint, not corruption.

## What is already true — do not redo it

- **The parser's strictness is deliberate and correct.** `validate_trailing_text` exists
  for list entries; link sections reject trailing text by construction. Do not add
  trailing-text support to link sections, and do not make the archive silently strip the
  annotation — that trades a loud failure for silent deletion of something the author
  meant to keep.
- **Only the document's leading header block is parsed.** Fenced examples and bullets
  appearing later in the body are ignored (verified: a document with a malformed parent
  bullet in prose and in a fenced block parses as `missing`). This plan's own examples
  are therefore safe, and so are the fixtures a phase worker writes.
- **The contract is already written down** in `PLAN_HEADER_BLOCK_NOTE` and surfaces
  through `sase plan validate --explain`. This epic does not need to invent guidance; it
  needs to enforce guidance that exists.
- **The floor-bump workflow is well-trodden.**
  `build(deps): raise sase-core-rs floor to ...` has landed many times, most recently at
  `6b0976bcb`, and `just validate` runs
  `tools/validate_sase_core_rs_version --published-minimum`, so the floor can only name
  a version that PyPI already serves.
- **The launch path's error wrapping is fine as-is.** `cli_work_from_plan.py:294-299`
  wraps whatever `archive_plan_file` raises. The defect is the quality of what is
  raised, not the wrapper.

## Immediate unblock — independent of every phase below

The blocked epic does not need to wait for this work. Deleting the parenthesized
annotation from the `PARENT` bullet on line 4 of
`/home/bryan/.sase/plans/202608/selection_soundness.md`, leaving only the Markdown link,
restores a canonical header (verified), after which the printed resume command launches
normally:

```bash
sase bead work /home/bryan/.sase/plans/202608/selection_soundness.md --yes
```

Do this first, or in parallel; it is a one-line edit to a machine-local file and is not
a phase deliverable.

## Phase `core-diagnostic` — a header-block validity rule in the Rust validator

**Problem.** `plan_validate` validates frontmatter and requires a non-empty body, and
knows nothing about the header block that `plan::artifact_link` in the same crate
already parses.

**Deliverable.** In `crates/sase_core/src/plan/validate.rs`, the validator parses the
document's leading header block and pushes one `error` diagnostic when it does not
parse:

- code `header-invalid`, matching the issue code `plan_links_refresh` already uses for
  the same condition, so the two surfaces share one vocabulary
- `field_path` empty (this is a body-level problem, not a frontmatter field)
- `line` set to the offending header bullet's 1-based line in the whole document,
  including the frontmatter, falling back to the block's first bullet when the reason
  does not identify a section
- a message that carries the parser's reason **and** states the canonical form, so an
  agent can fix it from the message alone without reading Rust — for example, that a
  link-shaped section must be exactly a bolded key followed by one Markdown link and
  nothing else

At most one such diagnostic per document, and it must not short-circuit the frontmatter
diagnostics: the validator's contract is to report every problem in one pass.

**Hard constraint — do not change the plan-header block wire.**
`PLAN_HEADER_BLOCK_WIRE_SCHEMA_VERSION` is 3 and the Python adapter asserts equality
with the binding, while `pyproject.toml` admits any `sase-core-rs` below `0.19.0`.
Publishing a bumped header wire inside the `0.18.x` window would make every
already-installed workspace raise on every header operation. The line number must come
from a crate-internal helper (the parser already builds `physical_lines`);
`parse_sdd_plan_header_block`'s public payload must stay byte-identical. The
plan-validation wire needs no change either — `PlanDiagnosticWire` already carries
`line`.

**Acceptance.** Rust tests cover: trailing text on `PARENT`, on `PLAN`, and on `BEAD`; a
duplicate section; an unknown section key; a malformed link; a canonical header (no
diagnostic); an absent header (no diagnostic); and a document whose only header-shaped
bullets appear after the title or inside a fence (no diagnostic). The change lands as a
`feat` on sase-core master and release-plz publishes `sase-core-rs`; record the
published version for the next phase.

**Non-goals.** No parser leniency. No new wire fields. No Python changes in this phase.

## Phase `links-parity` — report an invalid header from `plan links validate`

**Problem.** `src/sase/sdd/_link_validation.py:85-88` reads the header block and, when
the disposition is `INVALID`, skips parent validation and emits nothing at all — so
`sase plan links validate` reports a malformed plan as healthy.
`plan_links_refresh.py:216-222` returns a `header-invalid` error for the same document.
The two commands disagree.

**Deliverable.** `sase plan links validate` emits an error issue with code
`header-invalid`, the plan's relative path, and the parser's reason when the disposition
is `INVALID`, then continues to skip the parent-target check (there is no parsed parent
to check). Every other issue this file reports is unchanged.

**Acceptance.** A test in `tests/main/test_plan_links_validate_handler.py` pins that a
plan with a trailing-text link section is reported with `header-invalid` and the
parser's reason, and that a canonical plan still reports nothing. The committed plans
store contains zero invalid headers, so this cannot turn an existing tree red.

**Non-goals.** No change to `plan links refresh` or `repair`; no auto-repair.

## Phase `core-adopt` — adopt the release and pin the rule at every surface

**Problem.** The rule exists in core but this repo does not require the release that has
it, and nothing here would notice if a future core change dropped it.

**Deliverable.**

- Raise the `sase-core-rs` floor in `pyproject.toml` to the version published by
  `core-diagnostic` and reinstall.
- Tests that pin the behaviour at the surfaces that actually failed, using a plan whose
  only defect is a trailing-text `PARENT` bullet: `validate_plan` reports
  `header-invalid` as an error; `sase plan validate` exits non-zero and prints the
  location-bearing diagnostic; `require_plan_approval_validation` raises
  `PlanApprovalValidationError`; and `work_from_plan_file` raises `PlanFileWorkError`
  from its up-front validation — asserting specifically that it fails **before** taking
  the launch lock or reaching the archive, since failing late is the defect under
  repair.
- Rewrite `PLAN_HEADER_BLOCK_NOTE` (`src/sase/main/plan_explain.py:6-10`) from advice
  into a stated rule: SASE owns the block, a hand-authored bullet that deviates from the
  canonical form is now a validation error, and — because this is the exact mistake that
  produced the epic — a link-shaped section carries a link and nothing else.

**Acceptance.** `just check` passes, `just validate` accepts the new floor as published,
and the repaired `selection_soundness` plan validates clean.

**Non-goals.** No change to the `/sase_plan` skill's steps; the note reached through
`--explain` is the right place and already exists.

## Phase `archive-guard` — an actionable failure at the archive boundary

**Problem.** `archive_plan_file` (`src/sase/sdd/plan_archive.py:109-129`) formats,
stamps, and then calls `project_plan_header_sections` **before**
`validate_plan_for_commit`. Projection is what raises: it upserts the `PARENT` section,
the binding refuses to update a document whose existing block is invalid, and the caller
receives `validation: <reason>` — no path, no line, no remedy. That is the string the
user saw. The two approval-time archive callers (`plan_approval_actions.py:462` and
`_notification_plan_background.py:54`) swallow it into a warning log entirely.

**Deliverable.** The archive boundary refuses a malformed source before it mutates or
writes anything, and fails with the same diagnostic envelope the validation surfaces use
— path, line, code, reason, remedy — so that a malformed document reaching this boundary
by any route produces an error a human or agent can act on. Reordering validation ahead
of projection is the expected shape; projection only ever rewrites derived sections into
canonical form, so validating the pre-projection document cannot reject anything the
post-projection document would have accepted.

**Acceptance.** A test asserts that `archive_plan_file` on a malformed source raises an
error whose message names the source path and the parser's reason, and that no file was
written to the destination. A second test asserts the launch path's wrapped message is
still the actionable one end to end.

**Non-goals.** Do not auto-repair the header. Do not change the two approval-time
callers' swallow-and-log behaviour — after `core-adopt` the approval gate rejects a
malformed plan before either of them runs, and the failures those warnings actually
record today have a different cause (see follow-ups).

## Phase `land` — land the epic

Verify the combined tree against a real malformed plan end to end: `sase plan validate`
rejects it, the approval gate refuses it, `sase bead work` fails at validation rather
than at archive, and `sase plan links validate` reports it. Run `just check-full` and
`just symvision`. If `plan_explain.py` or any `src/sase/xprompts/skills/` source
changed, deploy it only from the clean, merged tree per the generated-skills workflow.
File the collected follow-ups with `/sase_new_task`, state honestly whether the corpus
is still free of invalid headers, and close the epic.

## Proposed follow-ups — out of scope here

- **The approval-time plan archive has been failing silently since 2026-07-31.**
  `~/.sase/logs/tui.log` holds 56 `Failed to archive approved plan` warnings, every one
  of them ending in
  `ValueError: No workspace plugin detected a workflow type for '/home/bryan/.sase/projects/.sase'`
  raised from `get_workspace_directory`. The project directory is being resolved to a
  bogus path, so approved plans are not being archived at approval time at all, and both
  call sites swallow it. This is a separate defect with a separate root cause; it did
  not cause the launch failure (the launch's own archive would have raised regardless),
  and fixing it inside this epic would mean surfacing 56 errors that belong to a bug
  this epic does not fix. It deserves its own task bead.
- **`--dry-run` previews an archive it never attempts.**
  `✓ Archived ... (preview; no files written)` is printed from a destination
  computation. After this epic the specific failure it papered over is impossible, but
  the preview still claims more than it checked.
