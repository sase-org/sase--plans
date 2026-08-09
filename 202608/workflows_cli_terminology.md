---
tier: tale
title: Sweep workflows, CLI, and non-ACE code to Patch/stitch terminology
goal:
  Make Patch and stitch canonical throughout the non-ACE Python source tree, CLI help,
  workflow messages, and corresponding tests while preserving every legacy command,
  option, import, wire, and serialized-data contract.
size: large
proposed_by: bbugyi200.athena.sase-hn.8.3
bead: sase-hn.8.3
create_time: 2026-08-09 00:38:17
status: wip
---

- **PARENT:**
  [202608/patch_terminology_completion.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_terminology_completion.md)
- **BEAD:**
  [sase-hn.8.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.8.3.md)

# Sweep workflows, CLI, and non-ACE code to Patch/stitch terminology

## Goal and measured scope

Complete phase bead `sase-hn.8.3` by removing current-concept ChangeSpec vocabulary from
every canonical `src/sase/**` path outside `src/sase/ace/**`, from CLI and workflow
presentation, and from the corresponding non-ACE test surface. Preserve old spellings
only where they are externally pinned compatibility contracts.

The tightened content-aware audit is the authoritative inventory. At the phase baseline
it reports 1,502 defects in non-ACE `src/sase/**`, 2,439 defects in tests outside
`tests/ace/**`, and one defect in `tools/validate_sase_core_rs`. The source
concentration is in workflows, Axe, main/CLI, core adapters, the status state machine,
workspace providers, bead integration, and stats. The tools occurrence is the existing
version-5 `work["changespecs"]` wire key: keep the key and explicitly document it as a
legacy serialized compatibility field.

## Compatibility and scope constraints

- Do not edit `src/sase/ace/**`, `tests/ace/**`, Rust core, linked repositories,
  documentation/history, memory files, generated provider instructions, or the
  terminology classifier. Those belong to sibling or landing phases.
- Use `Patch`/`Patches` in user-facing prose, `patch`/`patches` in canonical locals and
  private helpers, and `stitch` only for Patch history entries. Actual VCS commits,
  commit SHAs/logs, and the `sase commit` command remain commits.
- Preserve `sase changespec`, `--changespec`, `changespec_name`, `changespec_bug_id`,
  `commit_changespec_name`, `meta_changespec`, the `changespecs` tab and stats-wire
  keys, `COMMITS:` parsing, compatibility modules and imports, existing JSON/database
  fields, and the Python compatibility symbol `changespec_to_wire`. Do not rename files
  or public APIs merely to satisfy the audit.
- Keep legacy command exit codes, parser destinations, machine output, saved state,
  notification action values, and wire formats unchanged. Make retained spellings pass
  the content-aware audit with a narrow nearby legacy/compatibility explanation; never
  loosen the classifier or change a pinned value.
- Follow the Symvision hierarchy for any symbol rename. Prefer existing Patch-named
  APIs, keep private helpers file-local, and leave an old public name only as a real
  compatibility alias with a non-test consumer or justified pragma.
- Treat `JumpToChangeSpec` as a serialized notification action unless all consumers
  prove it is safe to migrate. The current phase should retain it and add explicit
  legacy-wire context while canonical nearby payload/local names use Patch.

## Implementation

1. Regenerate and partition the audit JSON before each sweep. Work from the exact
   non-ACE source findings rather than a blind repository-wide replacement. For each
   occurrence, decide whether it is canonical prose/code to migrate or one of the
   enumerated retained contracts. Use token-aware edits so `ChangeSpecI` is corrected to
   `CLI`, `ChangeSpec(s)` becomes `Patch(es)` only when it names the current domain
   object, and identifiers embedded in legacy field names or import paths are not
   corrupted.

2. Fix the user-visible contract surface first. Update `main/parser_commit.py`,
   `main/commit_handler.py`, `main/parser_ace.py`, `main/parser_commands.py`, search and
   Patch handlers, rewind/accept/commit workflows, mentor output, workspace-provider
   diagnostics, status-wire errors, bead launch errors, integration helper labels, and
   Axe messages so current concepts say Patch/stitch. Correct all three known
   `ChangeSpecI` garbles to `CLI`. Update the stale Justfile status-state-machine
   comment to its current Patch-named symbol. Preserve argument names and dispatch
   behavior while making `sase commit --help`, `sase patch --help`, and the retained
   `sase changespec --help` alias describe Patches.

3. Sweep canonical implementation prose and internal names through workflows, Axe, main,
   core adapters, status-state-machine code, workspace providers, stats, bead, doctor,
   integrations, running-field, xprompt, and remaining non-ACE packages. Migrate
   comments/docstrings, type aliases, locals, collections, and private helpers to
   Patch/stitch vocabulary and existing canonical imports. Rename definitions and all
   callers together only when no public, serialized, or linked contract pins the name.
   For compatibility modules, data readers, dual-read adapters, and old public aliases,
   retain the exact spelling and add concise nearby compatibility prose where the audit
   needs it.

4. Update tests outside `tests/ace/**` alongside the owned source surface. Current
   behavior, fixture prose, local names, and expected user-visible output should use
   Patch/stitch. Keep focused assertions that exercise old commands, options, imports,
   serialized headings, fixture inputs, and wire keys; mark those as legacy
   compatibility cases rather than rewriting their expected values. Do not rename stable
   test paths solely to hide an audited token. In `tools/validate_sase_core_rs`,
   preserve both `REQUIRED_PYTHON_COMPAT_SYMBOLS = ("changespec_to_wire",)` and the
   version-5 `work["changespecs"]` probe, with explicit legacy-contract context.

5. Iterate until the audit reports zero defects for non-ACE `src/sase/**`, zero defects
   for tests outside `tests/ace/**`, and zero unclassified tools findings. Review every
   remaining retained candidate in the edited surface to ensure it is an actual
   compatibility boundary, not a marker added to conceal current prose. A nonzero global
   audit remains expected while the parallel ACE and linked-repository phases are
   unfinished; record the exact remaining out-of-scope buckets in the close note.

## Verification

Run narrow tests while editing, then complete all of the following:

1. Run the terminology audit in JSON mode and assert the three owned slices are empty:
   non-ACE `src/sase/**`, tests outside `tests/ace/**`, and tools other than the audit
   contract. Confirm remaining defects, if any, belong only to `src/sase/ace/**`,
   `tests/ace/**`, `sase-core`, or linked repositories owned by sibling phases.
2. Capture and inspect `sase commit --help`, `sase patch --help`, and
   `sase changespec --help`: all current prose must say Patch, both canonical and legacy
   command forms must still exit successfully, and parser destinations/options must
   remain compatible.
3. Run focused CLI/parser, workflow (commit/accept/rewind/mentor), Axe,
   status-state-machine, core-wire, workspace-provider, bead, stats, integration,
   xprompt, and compatibility tests affected by the edits. Re-run failures at their
   narrowest test file before broader verification.
4. Run `just _lint-symvision` after symbol cleanup, then `just check`. If scoped
   selection escalates, reports unusual selection, or a broadening-set file is touched,
   run `just check-full`.
5. Run `git diff --check` and review the final diff for accidental contract changes,
   ACE/linked-repo scope overlap, marker-only audit suppression, and unintended
   replacements of real VCS commit terminology.

Close only `sase-hn.8.3` with a note listing the zero-defect owned-slice audit, CLI
help/alias checks, and every verification command that passed. Record genuinely
out-of-scope work as `PROPOSED FOLLOW-UP:` notes on this phase bead; do not create task
beads and do not close the parent epic.
