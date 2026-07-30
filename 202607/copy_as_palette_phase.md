---
tier: tale
title: Add the ACE “Copy as…” palette
goal:
  The copy prefix opens one registry-driven, warm-only palette everywhere in ACE while preserving configured
  accelerators, dispatch behavior, and snapshot correctness.
bead: sase-az.3
create_time: 2026-07-29 20:34:42
status: done
---

- **PROMPT:** [202607/prompts/copy_as_palette_phase.md](prompts/copy_as_palette_phase.md)
- **PARENT:** [202607/copy_as_palette.md](https://github.com/sase-org/sase--plans/blob/main/202607/copy_as_palette.md)
- **BEAD:** [sase-az.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-az/sase-az.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-az.3--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-az.3.md#member-code)
  - [bbugyi200.athena.sase-az.3--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-az.3.md#member-plan)
- **COMMITS:**
  - [3da9140](https://github.com/sase-org/sase/commit/3da9140b4968f590f84ace1db91f21c565746381) — feat(ace): add
    contextual Copy as palette

# Plan: Add the ACE “Copy as…” palette

## Goal

Replace ACE’s footer-only copy-prefix state with one compact, discoverable “Copy as…” modal while preserving every
configured two-key copy accelerator and the existing copy dispatch implementations. The modal must open from every ACE
tab, Artifacts sub-tab, and copy-forwarding modal; use only already-loaded UI state on its open path; represent marked
Artifacts sets accurately; and dismiss before snapshot capture.

## Current state and constraints

- Clipboard delivery, fallback handling, copy labels, Markdown-link/metadata-JSON representations, marked-set
  formatting, and the shared copy-target registry are already implemented. This work consumes those seams rather than
  moving copy behavior into the modal.
- `action_start_copy_mode` currently sets `_copy_mode_active` and paints the `COPY` footer. `_handle_copy_key` owns
  target dispatch, unknown-key messages, flag clearing, and footer restoration.
- The copy-target registry supplies presentation order, palette/footer labels, category, plural label, and
  `accepts_marks`; the active keymap registry supplies configured keys. The command catalog and its pure availability
  predicates already express target applicability.
- The modal open path must not resolve artifact references, read files, stat or glob paths, invoke subprocesses, or
  perform other data-scaled work. Context, title, and previews must be derived only from current pane objects, cached
  snapshots, and already-rendered values. Missing warm data yields an empty or type-level preview.
- The normal `COPY` footer remains visible beneath the modal. Cancel and selection must clear `_copy_mode_active` and
  restore the correct normal footer. Configured target keys take precedence over `j`, `k`, and `q` navigation bindings.

## Implementation

1. Add a dedicated `CopyAsModal` and immutable row/context records under `src/sase/ace/tui/modals/`.
   - Render “Copy as…” plus a single-line context subtitle.
   - Group rows in registry order beneath Identity, Location, Content, Data, and Actions headings.
   - Render each configured key with `footer_key_display`, the registry palette label (or plural label for marked sets),
     and a dimmed best-effort preview.
   - Support mouse selection, arrows/`j`/`k`, Enter, Escape/`q`, and direct configured accelerators. Test and document
     that configured accelerators win when rebound to navigation or cancel keys.
   - Keep unknown printable keys inside the modal, emit the existing context-named “Unknown copy key (…)” warning, and
     leave both modal and copy mode active.
   - Add responsive styling to `styles.tcss` consistent with existing picker modals and usable in narrow terminals and
     both themes.

2. Add a warm-only palette-context builder at the clipboard action boundary.
   - Map the live tab/sub-tab to `changespecs`, `artifacts_<kind>`, `agents`, or `axe`, and join configured keys to the
     matching copy-target registry rows.
   - Reuse command-catalog availability predicates with a minimal in-memory context so unavailable PR fields, missing
     selections, and row-specific targets are filtered without invoking broader command-context extraction that can
     discover files.
   - Derive display-name-aware subtitles from current ChangeSpec, agent, AXE item, or Artifacts pane/snapshot state. For
     Artifacts marks, use visible marked-target order and render “N marked <kind>” plus registry plural row labels.
   - Supply only cheap previews already present in current row/pane state: identifiers, names, paths, short excerpts,
     cached references, or type/size hints. Do not call reference resolution, serializers, transcript readers, clipboard
     transports, or filesystem helpers.
   - Reject empty non-PR Artifacts panes with the established “No <sub-tab> entry to copy” warning; retain the existing
     empty-ChangeSpec guard and analogous agent/AXE selection guards.

3. Wire `ClipboardCoreMixin.action_start_copy_mode` to paint the existing copy footer and push `CopyAsModal`.
   - On cancel, route through the existing escape dispatch so copy state and footer restoration remain centralized.
   - On selection, dismiss first and then dispatch the configured key through `_handle_copy_key`, leaving all copy
     engines unchanged.
   - For the snapshot target, defer the synchronous dispatch until the next refresh after modal dismissal so the
     captured pane cannot contain the palette. Keep the refresh callback thin and non-awaiting.
   - Preserve command-palette `copy_mode_key` execution as a modal-bypassing path, and rely on `CopyModeForwardingMixin`
     so `%` stacks the palette above existing forwarding modals without API changes.
   - Export the new modal through the modal package only where existing import conventions require it.

4. Synchronize user-facing guidance.
   - Add one “the copy prefix opens the Copy as… palette” row to each ChangeSpecs/Artifacts, Agents, and AXE help copy
     section while retaining the complete configured accelerator references.
   - Rewrite the relevant `docs/ace.md` copy-mode introductions around the palette, direct accelerator behavior,
     keyboard precedence, marked-set pluralization, cancellation, and forwarding-modal behavior.
   - Add a current-release changelog feature entry.

5. Add focused behavioral tests.
   - Unit-test row construction, category grouping, configured key display, availability filtering, warm-only previews,
     display-name context, marked titles/plural labels, and duplicate/rebound accelerator handling.
   - Pilot-test `%` opening on PRs, every non-PR Artifacts sub-tab with a live row, Agents, AXE, and a forwarding modal;
     use a scaffold/controlled pane where a production pane intentionally has no rows yet.
   - Verify direct accelerator and Enter dispatch parity, configured navigation-key precedence, Escape/`q` lifecycle
     restoration, unknown-key warning with the modal retained, and the empty-selection warnings.
   - Verify the command-palette executor still dispatches without opening the new modal.
   - Verify snapshot dispatch occurs only after the palette is unmounted and captured content excludes it.

6. Add deliberate PNG coverage and inspect every accepted diff.
   - Capture the Commits palette with one selection and with marked entries.
   - Capture the palette above `PreviewPanelModal`.
   - Cover dark and light themes and a narrow layout where useful to prove responsive sizing.
   - Generate new goldens only after the behavioral tests pass, inspect actual/expected/diff artifacts, and accept only
     the intended palette visuals.

## Validation

1. Run `just install` before repository checks because the workspace is ephemeral.
2. Run focused nonvisual copy/palette, command-catalog, help, and forwarding modal tests during implementation.
3. Run `just test-visual`, inspect the palette PNGs in both themes and the snapshot diffs, and update goldens only for
   intentional changes.
4. Run `just check` as the final full validation, address every lint, type, unit, and snapshot failure, then rerun it to
   a clean pass.
5. Recheck the worktree diff for accidental generated or unrelated changes, confirm the phase bead remains the only bead
   being closed, and close `sase-az.3` with a note naming the focused, visual, and full checks that passed.
