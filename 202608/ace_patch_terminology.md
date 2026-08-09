---
tier: tale
title: Sweep the ACE surface to canonical Patch/stitch terminology
goal:
  ACE user-facing output and canonical implementation use Patch/stitch terminology,
  while legacy aliases, saved state, wire keys, and import paths remain compatible.
size: large
proposed_by: bbugyi200.athena.sase-hn.8.2
bead: sase-hn.8.2
create_time: 2026-08-09 00:34:29
status: wip
---

- **PARENT:**
  [202608/patch_terminology_completion.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_terminology_completion.md)
- **BEAD:**
  [sase-hn.8.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.8.2.md)

# Sweep the ACE surface to canonical Patch/stitch terminology

## Goal

Complete phase bead `sase-hn.8.2` by removing current-concept ChangeSpec/commit-entry
vocabulary from ACE console output, TUI presentation, canonical Python prose, and
canonical internal names. Preserve every documented compatibility boundary: the
`sase.ace.changespec` facade, public aliases consumed by older callers, serialized and
saved-state keys such as the `changespecs` tab ID, legacy command/import paths, and
historical `COMMITS:` parsing.

The tightened terminology audit currently reports 2,333 defects under `src/sase/ace/**`
outside the designated compatibility package. Those findings are the authoritative work
list. The phase also owns the corresponding ACE tests and the glossary prompt PNG
fixture called out by the epic design.

## Constraints and compatibility boundary

- Work only in `src/sase/ace/**`, corresponding `tests/ace/**` coverage, and the
  intentionally changed PNG golden. Do not absorb workflow/CLI, Rust-core, linked-repo,
  or final epic-landing work owned by sibling phases.
- Prefer existing canonical APIs (`Patch`, `Stitch`, `find_all_patches`,
  `parse_patch_project_file`, Patch TUI mixins/widgets/models, and `*_patch_*` helpers)
  over inventing parallel behavior.
- Keep old public aliases and facade modules operational. Where an old spelling must
  remain, make the compatibility intent explicit enough for the content-aware audit; do
  not broaden path-based allowlisting.
- Keep wire fields, exact persisted values, CSS/widget IDs, keymap/config contracts,
  notification actions, and saved state unchanged when their spelling is externally
  pinned. Nearby prose should say that the retained value is legacy compatibility.
- Follow Symvision's hierarchy for symbol changes: use canonical existing symbols,
  preserve truly consumed public aliases, keep private helpers file-local, and do not
  add a whitelist merely to silence a rename.
- Do not hand-edit generated provider instruction shims or memory files. No memory
  change is expected for this phase.

## Implementation

1. Rebuild the phase inventory from the audit JSON and classify each ACE occurrence as
   canonical code/prose or a retained boundary. Start with user-visible defects in
   `archive.py`, `revert.py`, `restore.py`, `change_actions.py`, hook/process messages,
   delta refresh logs, workflow handlers, TUI toasts, banners, labels, and widget
   exports. Change current-concept messages to “Patch” and current history-entry prose
   to “stitch” without changing exact copyable identifiers or command arguments.

2. Migrate canonical non-TUI ACE implementation to the existing Patch/Stitch surface.
   Update type imports, annotations, locals, collections, comments, docstrings, and
   private helpers across query, operations, comments, deltas, hooks, mentors,
   schedulers, timestamps, testing support, and lifecycle modules. Use canonical helper
   aliases where they already exist; when a canonical helper is missing but an old
   public function must survive, define the Patch-named implementation and leave the old
   name as an explicit compatibility alias. Preserve parser/storage behavior,
   project-file layout, and VCS identity semantics.

3. Finish the canonical ACE TUI migration. Route active code through Patch actions,
   group models, graph indexes, widgets, and state names; update visible tab labels,
   quick-start copy, error/toast text, and docstrings. Keep the legacy action/widget
   modules and properties as explicit facades, and retain persisted `changespecs` values
   and any pinned notification/wire action strings. Update tests alongside canonical
   renames so they exercise Patch APIs while retaining focused compatibility assertions
   for the old aliases and saved state.

4. Update `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py` so the demo prompt
   and glossary catalog use the real current term `Patch` and its current definition.
   Run the relevant visual test first without update mode, inspect the generated
   actual/expected/diff artifacts to confirm that only the intended glossary wording
   changed, then regenerate the affected golden with `--sase-update-visual-snapshots`.
   Re-run the visual lane normally and inspect the committed PNG/diff metadata before
   accepting it.

5. Iterate on the content-aware audit until no defect has a path under
   `src/sase/ace/**`. For retained ACE test fixtures and aliases, use narrow explicit
   compatibility prose or existing retained-marker rules rather than weakening the
   classifier. Review the final legacy-token report to ensure remaining hits are only
   the compatibility package, aliases, stable paths, serialized state, or deliberate
   fixtures.

## Verification

Run and re-run failures at the narrowest relevant level while editing, then finish with:

1. `./tools/audit_patch_stitch_terminology --repo-root . --json` and an exact check that
   its defect list contains zero `src/sase/ace/**` paths. A nonzero global exit remains
   expected until the parallel non-ACE and linked-repo phases finish.
2. Focused lifecycle, query, hook/scheduler, and TUI tests affected by the renames,
   followed by the non-visual ACE suite.
3. `just _lint-symvision` to catch dead or improperly exposed compatibility symbols.
4. `just test-visual` after the intentionally updated glossary snapshot; it must pass
   without update mode.
5. `just check`, as required for repository changes. If its scoped selector escalates or
   reports unusual selection, run `just check-full`.
6. `git diff --check` and a final diff review confirming that user-facing ACE text says
   Patch/stitch and that retained aliases, saved state, keymaps, and the `changespecs`
   tab ID still load through their existing compatibility tests.

Close only `sase-hn.8.2` with a note recording the zero-defect ACE audit and every
verification command that passed. Record any genuinely out-of-scope discovery as a
`PROPOSED FOLLOW-UP:` note on this phase bead; do not create a task bead or close the
parent epic.
