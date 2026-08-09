---
tier: tale
title: Clear the ACE TUI test surface
goal:
  Eliminate every strict Patch/stitch terminology defect under tests/ace/tui while
  preserving legacy contracts and pixel-identical PNG snapshots.
proposed_by: bbugyi200.athena.sase-hn.8.6.2
bead: sase-hn.8.6.2
create_time: 2026-08-09 04:51:43
status: done
---

- **PROMPT:**
  [prompts/202608/clear_ace_tui_test_surface.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/clear_ace_tui_test_surface.md)
- **PARENT:** [202608/patch_audit_gate_repair.md](patch_audit_gate_repair.md)
- **BEAD:**
  [sase-hn.8.6.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.8.6.2.md)

# Clear the ACE TUI test surface

## Goal

Complete phase bead `sase-hn.8.6.2` by eliminating all 2,709 strict Patch/stitch
terminology defects currently reported under `tests/ace/tui/**`, while preserving the
retained compatibility contracts and keeping every PNG golden byte-for-byte unchanged.

## Context and constraints

- The phase is limited to the ACE TUI test surface. Do not sweep production code,
  unrelated tests, linked repositories, documentation, changelogs, archived plans, or
  the parent epics.
- The authoritative work list is:
  `./tools/audit_patch_stitch_terminology --repo-root . --allow-missing-linked-repos --strict-test-fixtures --json`.
  At the planning baseline it reports 2,709 defects below `tests/ace/tui/`.
- Canonical test code should use Patch vocabulary and existing canonical interfaces:
  `make_patch`, `PatchGroupingMode`, `build_patch_tree`, `enumerate_patch_group_keys`,
  `AcePage(patches=...)`, and the corresponding Patch-named variables, fields, helpers,
  test names, docstrings, and comments.
- Do not rename or remove compatibility identifiers whose spelling is itself the
  contract, including the `changespecs` tab/state/keymap IDs, legacy serialized keys,
  compatibility facade imports, selector aliases, or tests intentionally proving old
  aliases. Such occurrences must remain behaviorally exercised and be labeled with a
  nearby explicit `legacy`/`compatibility` comment so the content-aware classifier can
  distinguish them from current terminology.
- Do not loosen the audit classifier or add broad path exemptions. Prefer a local,
  explicit compatibility annotation for retained literals. Any production audit-rule
  change would require separate justification and dedicated classifier coverage.
- Test edits must be behavior- and pixel-inert. Never update visual snapshot goldens to
  make this terminology-only phase pass.

## Implementation

1. Run `just install`, regenerate the strict JSON report, and group the
   `tests/ace/tui/**` defects by file and matched token. Use that report—not an
   unrestricted text replacement—as the bounded source of edits.
2. Convert canonical call sites first:
   - import test fixtures and grouping models through their Patch-named modules and
     symbols;
   - replace legacy fixture keywords and result fields with canonical Patch forms;
   - rename local test doubles, helper methods, parameters, collections, indices, trees,
     fold registries, grouping modes, constants, and prose so they mirror the production
     Patch API;
   - update internal references atomically within each file so tests continue to
     exercise the same code paths and assertions.
3. Review every remaining strict finding rather than mechanically deleting it. For a
   genuine legacy alias, stable tab/selector/keymap ID, or serialized fixture value,
   preserve the literal and add a concise compatibility declaration within the
   classifier's adjacent-line context. Keep at least one focused test for each legacy
   alias named by the phase design.
4. Iterate the strict audit and focused pytest runs over edited clusters until the
   report contains zero defects under `tests/ace/tui/**`. Confirm that defects outside
   this phase's path remain untouched for the sibling phase.
5. Review the final diff for accidental source changes, renamed stable literal values,
   altered test intent, or modified PNG files. Run formatting only as needed for the
   edited Python tests.

## Verification

Run and require all of the following before closing the bead:

1. The strict JSON audit reports exactly zero defects whose path starts with
   `tests/ace/tui/`.
2. Focused pytest runs for the edited ACE TUI clusters pass, including explicit legacy
   alias coverage.
3. `just test-visual` passes without updating or changing any PNG golden.
4. `just check` passes after the required `just install` setup.
5. `git diff --check` passes, `git status --short` shows no unintended files, and a
   final strict-audit rerun still reports zero ACE TUI defects.

After recording any discovered out-of-scope work only as a `PROPOSED FOLLOW-UP:` note on
`sase-hn.8.6.2`, close that phase bead with a note summarizing the zero-defect audit,
visual snapshot result, `just check`, retained alias coverage, and confirmation that no
goldens changed. Do not close `sase-hn.8.6`, `sase-hn.8`, or `sase-hn`.
