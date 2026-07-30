---
tier: tale
title: Finish Files integration for the unified Copy as palette
goal:
  The complete Files copy vocabulary is discoverable and correct in the unified Copy as palette, and epic sase-az is
  verified, cleaned up after closure, and fully landed.
bead: sase-az
create_time: 2026-07-29 22:05:31
status: done
---

- **PROMPT:** [202607/prompts/copy_as_files_integration.md](prompts/copy_as_files_integration.md)
- **PARENT:** [202607/copy_as_palette.md](https://github.com/sase-org/sase--plans/blob/main/202607/copy_as_palette.md)
- **BEAD:** [sase-az](https://github.com/sase-org/sase--beads/blob/main/pages/sase-az/README.md)

# Plan: Finish Files integration for the unified Copy as palette and land `sase-az`

## Goal

Finish the integration that became necessary while epic `sase-az` was in flight: the new Artifacts Files sub-tab must
surface its complete copy vocabulary through the registry-driven **Copy as…** palette, including a Markdown-link
representation, without regressing the direct chords that landed with `sase-b0.6`. Then re-verify the entire epic and
perform the required landing sequence: close `sase-az`, run post-close Symvision cleanup, and mark
`plans:202607/copy_as_palette.md` done.

## Verified baseline and remaining defect

The land-agent audit established the following before this plan was proposed:

- `sase bead show sase-az` reports four children, all closed with resolution `done`: `sase-az.1` (delivery), `.2`
  (registry/representations), `.3` (palette), and `.4` (artifact-files modal).
- The phase commits are `77ec8798e` (unified delivery), `cf844c3e5` (copy registry and paste-ready representations),
  `3da9140b4` (Copy as palette), and `132bd79c7` (artifact-file modal representations). Their source and tests match the
  phase notes: ACE has one `schedule_copy_delivery` seam over subprocess/tmux plus guarded OSC 52, a selectable fallback
  modal, one copy-target registry, bounded/partial marked representations, a warm-only grouped palette, and the
  artifact-files modal's `@`/link/contents/stored/source/JSON/snapshot rows with `y`/`Y` compatibility.
- A repository-wide source search finds no direct `copy_to_system_clipboard` use under `src/sase/ace` outside
  `actions/clipboard/_delivery.py`. The delivery, registry, artifacts copy, palette, modal, and help-focused baseline
  suite passes: 144 tests.
- `master` and `origin/master` are aligned. The commits after the first epic commit include interleaved work from the
  grouped `@` menu, the artifact clipboard split, the new Files list/detail/filter/open/copy phases, xprompt provenance,
  and a completion-renderer split. The non-overlapping work needs no changes. The overlapping Files copy commit
  `fec7898b2` correctly adopted the epic's delivery seam, target registry, command catalog, reference resolver, and
  copy-mode dispatcher, but it landed after the palette phase and did not update all palette-specific structures.
- The concrete regression is reproducible from current source: for a live Files pane, `build_copy_as_context()` returns
  only `reference`, `handoff`, and `snapshot`. Yet the Files keymap, registry, footer, and help expose `contents`,
  `reference`, `handoff`, `path`, `source`, `label`, `json`, and `snapshot`, and their direct chords dispatch correctly.
  `actions/clipboard/_palette.py` still has the pre-`sase-b0.6`
  `_DISPATCH_ORDER["artifacts_files"] = ("snapshot", "reference", "handoff")`, so the winner filter silently removes
  every later target from the modal. `docs/ace.md` repeats the stale two-target view.
- The generic artifact representation path already supports Files Markdown links (`reference_items_for_targets`,
  `resolve_artifact_selection`, and `format_artifact_representation("link", ...)`), but the Files keymap/registry never
  added a `link` target. Files also contains unreachable duplicate JSON branches in `_copy_file_target` and
  `_copy_marked_file_targets`: JSON is intercepted earlier by `_handle_artifacts_copy_key` and always goes through the
  shared representation path. Marked Files contents skip binary entries before `_schedule_marked_copy`, so a partial
  copy currently omits the skipped count from its toast.

## Phase 1: Integrate the Files sub-tab with the completed epic

### Palette and key vocabulary

1. Update `_DISPATCH_ORDER["artifacts_files"]` in `src/sase/ace/tui/actions/clipboard/_palette.py` to mirror the actual
   dispatcher precedence: `snapshot`, `reference`, `handoff`, `link`, `json`, then `contents`, `path`, `source`, and
   `label`. This table decides the winner only when users configure colliding accelerators; registry order still decides
   presentation order.
2. Add an `artifacts_files.link` entry to the one copy-target registry in `src/sase/ace/tui/copy_targets.py`, with
   Location category, Markdown-link singular/plural labels, and marked-set support. Add the matching default key to both
   `CopyModeKeymaps` and `src/sase/default_config.yml`.
3. Preserve every Files chord that `sase-b0.6` already shipped. In particular, keep `%l` for the artifact-file label and
   `%j` for metadata JSON; use `%L` for the newly integrated Markdown link. The older four artifact groups retain their
   uniform `%l`/`%J` link/JSON keys. This resolves the post-start collision without silently changing a landed
   accelerator.
4. Keep palette construction warm-only. Use fields already present on the in-memory `ArtifactFile` records and pane
   state—id, label, kind, size, stored/source path strings, and selected view mode—to provide useful previews and to
   suppress single-selection rows that cannot dispatch successfully (for example, source with no recorded source path,
   or contents for a binary view). For marked sets, keep a target when at least one warm record is representable and
   show counts/hints without statting, reading, resolving, parsing JSON, or spawning work on palette open.

### Reuse and partial-result behavior

1. Route Files links through the existing shared artifact representation action. Single selection must produce
   `[label](file:<id>)`; marked selection must produce a Markdown bullet list in visible order and retain the existing
   partial-reference accounting.
2. Delete the unreachable Files JSON branches and their now-unused imports from
   `src/sase/ace/tui/actions/clipboard/_artifact_targets.py`; JSON must have one implementation via the shared
   representation resolver/formatter, matching `sase artifact show -j`.
3. Make marked Files contents account for binary/unrepresentable rows in the delivery label. If one text/Markdown file
   and one binary file are marked, copy the representable bounded content and report both the success and unavailable
   count. Preserve the current immediate warning when no marked file has copyable text and preserve the shared
   per-item/total size caps and explicit truncation banner.
4. Do not merge the main Files pane with `ArtifactFileSelectionModal`: they are distinct surfaces with different
   app-configured versus modal-local accelerators. Continue sharing the canonical path helpers, CLI JSON serializer,
   delivery seam, and representation formatter where their behavior is genuinely common.

### User-facing surfaces

1. Update the Files Copy Mode section in `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py` to include the
   configured Markdown-link row.
2. Update `docs/ace.md` so the non-PR Artifacts table lists the complete Files palette/chord set: contents, reference,
   Markdown link, stored path, source path, label, metadata JSON, handoff, and the shared snapshot. Keep the separate
   artifact-files modal documentation unchanged except for wording needed to distinguish its modal-local `l`/`J` keys
   from the Files sub-tab's compatibility-preserving `%L`/`%j`.
3. Add a concise changelog bullet only if the repository's changelog policy considers this user-visible completion
   distinct from the existing Copy-as entry; otherwise amend the existing unreleased bullet rather than duplicating it.

## Phase 2: Regression coverage and complete epic verification

Add focused tests that would have failed before Phase 1:

- An exact Files `build_copy_as_context()` row-set assertion covering all default targets and keys, plus warm previews
  and single-selection availability for missing source/binary contents.
- A configured-key collision case proving `_DISPATCH_ORDER` matches `_handle_artifacts_copy_key` precedence.
- Single and marked Files Markdown-link dispatch, including visible-order bullet formatting.
- Marked mixed text/binary contents reporting the skipped count.
- Registry ↔ default-keymap equality, footer/help rows, command-catalog coverage, and `mode_keymaps.py` ↔
  `default_config.yml` equality.
- Retain the existing modal tests proving the artifact-files modal still uses its full modal-local representation set
  and `y`/`Y` accelerators.

Run `just install` first in the implementing workspace, then run the focused clipboard/palette/files/help tests used by
the audit:

```text
tests/test_clipboard_utils.py
tests/ace/tui/actions/test_clipboard_delivery.py
tests/ace/tui/test_copy_targets.py
tests/ace/tui/test_artifacts_copy_mode.py
tests/ace/tui/test_artifacts_copy_marked.py
tests/ace/tui/test_artifacts_copy_references.py
tests/ace/tui/test_copy_as_palette.py
tests/ace/tui/modals/test_artifact_files_modal_copy.py
tests/test_keymaps_display_help.py
```

Then run `just check`. If PNG goldens change, inspect the actual/expected/diff artifacts and run `just test-visual`;
palette styles are not changing, so do not update visual goldens merely to make a test pass. Re-sweep `src/sase/ace` for
direct `copy_to_system_clipboard` calls and confirm the only result is the delivery seam. Re-read the four child bead
descriptions/notes and the four epic commits, and record the focused/full validation evidence for the final close note.

## Phase 3: Land `sase-az` (must be last)

Only after Phases 1–2 are complete:

1. Run `sase bead show sase-az` and each child one last time. Close the epic normally—never with `--force`—using:

   ```text
   sase bead close sase-az --note "<verified phase commits/source/tests; Files integration; final check evidence>"
   ```

   If descendant validation unexpectedly rejects the close, finish or deliberately reopen the named work; do not force a
   `done` resolution.

2. **After the close succeeds**, run `just symvision`. The epic-symbol exemptions for `sase-az` have now expired. Follow
   the Symvision decision hierarchy: remove stale `--epic-symbol` entries, delete genuinely dead symbols and their dead
   helpers/tests, or make same-file-only symbols private and wire their real callers. Do not add a replacement whitelist
   or test-only pragma. Re-run the exact Symvision lane until clean, then rerun `just check` after any code or Justfile
   cleanup.
3. Open the plans sidecar through the sanctioned repository workflow (`sase repo open plans -r "<audit reason>"`), edit
   the linked epic plan `202607/copy_as_palette.md`, and change only its frontmatter `status: wip` to `status: done`.
   Verify the plans checkout diff, then confirm `sase bead show sase-az` reports `closed` / `done` and the linked plan
   frontmatter reports `status: done`.

## Boundaries

- No `sase-core` change is needed: this is Python/TUI presentation and glue over already canonical references and JSON
  serializers.
- Do not redesign copy-mode key names or remove the direct Files `y`/`Y` actions.
- Do not add synchronous filesystem, subprocess, resolver, or JSON work to palette construction.
- Do not close the epic before integration and validation, and do not mark the epic plan done before the post-close
  Symvision cleanup is complete.
