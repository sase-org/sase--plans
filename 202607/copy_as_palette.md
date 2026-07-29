---
tier: epic
title: Unified "Copy as…" palette
goal: 'Pressing the copy prefix anywhere in ACE opens one beautiful, discoverable
  "Copy as…" palette: every existing copy target keeps its key as an accelerator,
  new Markdown-link and metadata-JSON representations join the reference/path/contents
  targets, marked sets pluralize into paste-ready forms, the artifact-files modal
  gains the full file-kind representation set, and every copy in the TUI rides one
  delivery seam that pairs OSC 52 with the subprocess adapter, names exactly what
  was copied, and leaves the generated text selectable when no transport works.

  '
phases:
- id: delivery
  title: One clipboard delivery seam with OSC 52 and a selectable fallback
  depends_on: []
  size: medium
  description: 'delivery: add a unified copy-delivery module that pairs Textual''s
    OSC 52 write with the verifiable subprocess adapter (plus a tmux load-buffer candidate),
    reports a typed outcome, standardizes the success/failure toast vocabulary, opens
    a selectable-text fallback modal when no transport works, and sweeps every TUI
    copy call site onto the seam and off the message pump.

    '
- id: targets
  title: Representation targets and one copy-target registry
  depends_on:
  - delivery
  size: medium
  description: 'targets: collapse the duplicated copy-target label sets into one registry
    carrying footer labels, palette labels, categories, and plural forms; add uniform
    Markdown-link and metadata-JSON targets to the four Artifacts key groups (link
    also to changespecs); make marked-set output paste-ready per representation (newline-separated
    references, Markdown list, JSON array, skip-and-report partial failures); and
    size-cap the contents-shaped targets.

    '
- id: palette
  title: The Copy-as palette modal
  depends_on:
  - targets
  size: large
  description: 'palette: make the copy prefix open a compact grouped palette modal
    built from the copy-target registry — configured keys as live accelerators, warm-data
    previews, marked-set titles and plural rows, snapshot-after-dismiss, graceful
    unknown-key handling — reachable from every tab, sub-tab, and forwarding modal,
    with footer/help/docs/PNG coverage.

    '
- id: files
  title: File-kind representations in the artifact-files modal
  depends_on:
  - palette
  size: medium
  description: 'files: give the artifact-files modal the same palette on the copy
    prefix with the full file-kind representation set — @file reference, Markdown
    link, contents, stored path, source path (origin-labeled, missing-aware), and
    metadata JSON matching sase artifact show -j — for single rows and marked sets,
    keeping y/Y as accelerators.

    '
create_time: 2026-07-29 19:12:27
status: wip
---

# Plan: Unified "Copy as…" palette

## Context

This epic implements item 6 of the consolidated research report
(`research:202607/artifact_refs_and_inspector/artifact_refs_and_inspector.md`, §5.6): replace the growing set of fixed
copy keys with one "Copy as…" palette, keep today's keys as accelerators, make marked-set copies paste-ready, always
name what was copied, leave generated text recoverable on failure, and pair the palette with OSC 52 so copy works over
SSH.

Every prerequisite the research names has landed and must be spent, not rebuilt:

- **Copy mode, marks, and canonical references on all Artifacts sub-tabs** (epics `sase-as`, `sase-av`). The `%` prefix
  dispatches sub-tab-first through `_handle_copy_key` (`src/sase/ace/tui/actions/clipboard/_core.py:35-62`) into
  per-group handlers; `%@` copies the prompt-form reference and `%!` seeds an agent prompt on all four non-PR sub-tabs
  (`src/sase/ace/tui/actions/clipboard/_artifacts.py:120-211`); marks are keyed on `ArtifactEntryTarget` per sub-tab
  with visible-order marked copy (`_artifacts.py:458-475`).
- **The kind-tagged reference facade** `src/sase/artifact_refs.py`: `reference_for_entry_target` (`:505`),
  `resolve_artifact_ref` (`:179`), `artifact_ref_context` (`:60`, cached per workspace+project at
  `_artifacts.py:788-799`), and the cheap path-only `chat_reference` precedent
  (`src/sase/ace/tui/widgets/artifacts/chats_detail.py:292-306`).
- **The read CLI and enriched records** (epic `sase-ax`): `sase artifact show -j` emits a stable envelope via
  `ResolvedArtifactReference.to_json_dict()` (`src/sase/artifact_cli/references.py:41-50`), and
  `artifact_file_json_dict` (`references.py:88-93`) serializes an `ArtifactFile` (16 fields including `sha256`,
  `size_bytes`, `mime_type`, `src/sase/core/artifact_file_types.py:54-73`) plus its `ref`. These serializers are the
  metadata-JSON representation, ready-made — the palette must reuse them so TUI-copied JSON and CLI JSON cannot drift.
- **The reader modal** (epic `sase-aw`): `PreviewPanelModal` carries `reference` and forwards `%` through
  `CopyModeForwardingMixin` (`src/sase/ace/tui/modals/base.py:7-33`, 13 consumers), so the palette reaches every
  forwarding modal for free.

### Verified current state

Checked against the working tree on 2026-07-29; cite this rather than re-deriving it.

**What `%` shows today is a footer strip, not a menu.** `action_start_copy_mode` (`_core.py:22-33`) sets
`_copy_mode_active` and repaints the keybinding footer via `update_copy_bindings`
(`src/sase/ace/tui/widgets/_keybinding_modes.py:393-488`) — a one-line `COPY`-badged hint. There is no overlay, no list,
no preview. The per-target human labels live in a _second_, independent hardcoded set, `_COPY_LABELS`
(`src/sase/ace/tui/commands/_mode_commands.py:49-106`), consumed only by the `:` command palette's `_iter_copy_commands`
(`:207-242`); no test ties the two label sets together.

**Keys are config-driven with an exact-equality sync test.** `CopyModeKeymaps`
(`src/sase/ace/tui/keymaps/mode_keymaps.py:50-113`) defines seven groups (`changespecs`, `artifacts_commits`,
`artifacts_plans`, `artifacts_chats`, `artifacts_bugs`, `agents`, `axe`) mirrored key-for-key by
`src/sase/default_config.yml:365-415`; `tests/test_keymaps_defaults.py:50-57` asserts exact dict equality, and the
schema (`src/sase/config/sase.schema.json:824-877`) is generic, so new targets need no schema change. Current keys per
group, verbatim:

| Group               | Targets                                                                                                    |
| :------------------ | :--------------------------------------------------------------------------------------------------------- |
| `changespecs`       | `raw`→`%` · `with_snapshot`→`!` · `bug`→`b` · `pr_number`→`c` · `name`→`n` · `spec`→`p` · `snapshot`→`s`   |
| `artifacts_commits` | `sha`→`%` · `reference`→`@` · `handoff`→`!` · `message`→`m` · `repo_sha`→`r` · `plan`→`p` · `snapshot`→`s` |
| `artifacts_plans`   | `reference`→`@` · `handoff`→`!` · `path`→`p` · `title`→`t` · `body`→`b` · `snapshot`→`s`                   |
| `artifacts_chats`   | `reference`→`@` · `handoff`→`!` · `path`→`p` · `agent`→`a` · `transcript`→`t` · `snapshot`→`s`             |
| `artifacts_bugs`    | `reference`→`@` · `handoff`→`!` · `number`→`b` · `url`→`u` · `title`→`t` · `prompt`→`p` · `snapshot`→`s`   |
| `agents`            | `chat`→`c` · `file_path`→`E` · `name`→`n` · `prompt`→`p` · `snapshot`→`s`                                  |
| `axe`               | `visible`→`o` · `full`→`O` · `snapshot`→`s`                                                                |

**The transport is subprocess-only and the failure path drops the text.** `copy_to_system_clipboard`
(`src/sase/core/clipboard.py:47-61`) blocks on `pbcopy`/`wl-copy`/`xclip`/`xsel` and returns a bool; no code anywhere
calls Textual's `App.copy_to_clipboard()` or emits OSC 52. The pinned Textual 8.0.1 implementation
(`textual/app.py:1697-1713`) writes `ESC ] 52 ; c ; <b64> BEL` fire-and-forget: no success signal, no tmux passthrough,
no size guard. Failure toasts come in three competing phrasings ("Copy failed — clipboard tool not available", "Copy
failed - …", "Failed to copy to clipboard") and two success dialects ("Copied <thing>" vs "Copied: <Thing>"); two call
sites still block the message pump (`src/sase/ace/tui/modals/artifact_files_modal.py:406,424`,
`src/sase/ace/tui/modals/commit_view_modal.py:278`); no failure path leaves the text recoverable.

**Marked-set copy exists only on the Artifacts sub-tabs, and marked reference copy is not paste-ready.**
`format_multi_copy_content` (`src/sase/ace/tui/actions/clipboard/_helpers.py:36-51`) wraps every item in a `### <label>`
header plus an unlabeled code fence — right for content dumps, wrong for `%@`: pasting a marked reference set into a
prompt today yields fences and headers instead of references (`_artifacts.py:175-188`). Marked resolution is also
all-or-nothing: one unrepresentable item aborts the whole copy (`_artifacts.py:806-807`). The artifact-files modal's
`y`/`Y` ignore its marks entirely (`artifact_files_modal.py:387,415`); its stored-vs-source path policy and origin
labels are already decided (`_artifact_file_clipboard_path`, `:120-142`).

**The command catalog already models the palette's row.** `CommandSpec` carries label, key sequence, tab scoping, and a
`copy_mode_key` executor that re-enters `_handle_copy_key` (`src/sase/ace/tui/commands/execute.py:71-75`); availability
filtering already scopes copy specs by sub-tab and row presence
(`src/sase/ace/tui/commands/availability.py:223-224,289-300,445-446,535-536`); catalog guard tests require a spec for
every `(group, target)` pair (`tests/test_command_catalog_guards.py:109-112`). `PropertyPickerModal`
(`src/sase/ace/tui/modals/property_picker_modal.py`) is the polished in-repo precedent for an accelerator-keyed,
j/k-navigable picker with a live detail line.

## Design decisions (apply to all phases)

1. **The palette is the copy-mode UI, not an addition to it.** The copy prefix opens the palette; there is no setting to
   keep the old footer-only chord, because the palette preserves it exactly: every configured key dispatches immediately
   from inside the palette through the existing `_handle_copy_key` path, so `%@` and `%s` remain the same two
   keystrokes. One behavior, one test matrix. The `COPY` footer strip still paints beneath the modal (it is the truth
   for what the keys are, and tests already contract on it).
2. **Existing dispatch is the engine; the palette is a front-end.** No copy behavior moves. The palette builds rows from
   the keymap registry plus the new copy-target registry and executes by key through `app._handle_copy_key(key)` after
   dismissing itself. The `_copy_mode_active` flag lifecycle is preserved (set on open, cleared by dispatch or
   dismissal) so `CopyModeForwardingMixin`, the `:` command palette's `copy_mode_key` executor, and existing tests keep
   working.
3. **One label registry.** The duplicated label sets — `_COPY_LABELS` and the hardcoded footer strings in
   `_keybinding_modes.py:425-487` — collapse into one copy-target registry that carries, per `(group, target)`: the
   short footer label, the palette label, the category (Identity · Location · Content · Data · Actions), and the plural
   label. Footer, `:` command palette, help modal, and the Copy-as palette all consume it, with a test tying them
   together.
4. **Toasts always name what was copied**, in the newer dialect: `Copied <thing>` / `Copied N <plural>`, em-dash failure
   phrasing, explicit severities. Partial marked-set success says both numbers ("Copied 3 references — 2 entries have no
   reference"). The delivery module owns the phrasing so it cannot drift again.
5. **Honest transport reporting.** Delivery attempts the verifiable subprocess path and the OSC 52 write; the toast
   reflects the strongest _verified_ outcome. Subprocess success → plain "Copied …". Subprocess unavailable but OSC 52
   emitted → "Copied … (OSC 52)". Nothing worked → no red-toast-and-drop: a fallback modal presents the generated text
   for terminal-level selection.
6. **Representations are presentation; serialization is shared** (`rust_core_backend_boundary`): reference strings come
   from the Rust-backed facade, metadata JSON comes from the `sase artifact` CLI serializers, and the only new logic
   here is string assembly and UI. No `sase-core` changes anywhere in this epic.
7. **Warm data only on the open path** (`sase/memory/tui_perf.md` rules 1-3, 8, 11 — every phase touching the TUI reads
   that memory first with the `/sase_memory_read` skill). The palette composes from pane state, the keymap registry, and
   already-cached contexts; previews are best-effort from warm caches and never stat, glob, read, or resolve on open.
   Expensive representations (contents reads, JSON resolution, index lookups) run off the pump after selection through
   the existing `spawn_pump_free_task` + `asyncio.to_thread` pattern, registered and cancelled per
   `preview_panel_modal.py:258-283`.
8. **Project display names, never keys**, in palette titles, row previews, and toasts (`gotchas` memory); bug references
   already render display names through the facade.
9. **Modal-level keys stay hardcoded; app-level keys stay configured.** New app-level targets (`link`, `json`) land in
   `mode_keymaps.py` + `default_config.yml` together (the equality test enforces it) and flow into `_COPY_LABELS`'
   successor so the catalog guard stays green. The artifact-files modal's palette rows use hardcoded keys, matching the
   modal-bindings convention `sase-aw` recorded.
10. **Coordination with in-flight epics.** `sase-aw.3` (preview-modal search) and `sase-ax.4` (artifact docs) are
    landing during this epic: phases that touch `docs/ace.md` or preview-modal PNG goldens must rebase before finishing
    and re-verify snapshots; nothing in this epic edits `PreviewPanelModal` beyond what `%` forwarding already provides.

## Phases

### Phase 1: One clipboard delivery seam with OSC 52 and a selectable fallback {#delivery}

Make every copy in the TUI reliable, honest, and recoverable — independently valuable even if the palette never lands.

1. **Transport candidates.** Extend `clipboard_commands` (`src/sase/core/clipboard.py:18-44`) with a
   `tmux load-buffer -w -` candidate, first in line, when `TMUX` is set — verifiable, and `-w` also forwards to the
   client terminal's clipboard via tmux's own OSC 52 handling. Pure addition; the doctor check
   (`src/sase/doctor/checks_tools.py:208-210`) and CLI copy paths inherit it.
2. **Delivery module** (suggested: `src/sase/ace/tui/actions/clipboard/_delivery.py`), the only place TUI code copies
   from now on:
   - `schedule_copy_delivery(owner, value, *, copied_label, task_name, on_failure="modal")` — resolves `value` (str or
     0-arg callable) off-thread, runs `copy_to_system_clipboard` off-thread, emits OSC 52 via `app.copy_to_clipboard` on
     the UI thread, and toasts per design decision 4/5. Generalizes `_schedule_artifacts_copy` (`_artifacts.py:647-680`)
     and `PreviewPanelModal._schedule_copy` (`:258-283`); spawned via `spawn_pump_free_task` with registry + unmount
     cancellation.
   - Typed outcome (`verified` / `osc52_only` / `failed`) returned for tests.
   - **OSC 52 size guard**: skip the OSC 52 write above a constant (~64 KB of base64 payload — common terminal buffer
     caps), documented at the constant; subprocess transports are unaffected.
   - `on_failure` policy: `"modal"` (default) pushes the fallback modal; `"toast"` keeps a plain error toast for
     surfaces where a modal would interrupt (vim yank); `"silent"` for best-effort paths.
3. **Fallback modal** (`CopyFallbackModal`): the generated text in a read-only, focusable `TextArea` with a title naming
   the failure ("No clipboard transport available — select the text below"), esc/q close. New pattern; keep it minimal
   and reusable.
4. **Sweep.** Route every TUI `copy_to_system_clipboard` call site through the seam and off the pump — the copy-mode
   handlers (`_changespec.py`, `_agents.py`, `_axe.py`, `_core.py` snapshot, `_artifacts.py`), the blocking holdouts
   (`artifact_files_modal.py:406,424`, `commit_view_modal.py:278`, `zoom_panel_content.py:230`), and the remaining modal
   call sites (report, gate-debug, plan-approval, tasks pane, user-question, mentor-review, notification-attachments,
   prompt-history). Vim yank (`_vim_visual_ops.py:21`, `_vim_normal_operator_exec.py:66-176`) uses `on_failure="toast"`.
   Unify all success/failure phrasing per design decision 4; delete the three variant failure strings. Non-TUI callers
   (`prompt/cli_copy.py`, `main/ace_handler.py`, `history/prompt_maintenance.py`) keep calling
   `copy_to_system_clipboard` directly.
5. **Tests.** Unit: candidate list with/without `TMUX`; outcome matrix (subprocess ok / missing / raises × OSC 52
   emitted / skipped-by-size / no driver) with a monkeypatched app; toast text per outcome; fallback modal receives the
   exact text. Pilot: a copy action on a host with no tools opens the fallback modal carrying the content; `y` in
   `CommitViewModal` no longer blocks (worker-scheduled). Update the existing copy tests that assert old phrasings.
6. **Docs**: a short "Clipboard transports" note in `docs/ace.md` (tools tried, tmux, OSC 52, the fallback modal).

### Phase 2: Representation targets and one copy-target registry {#targets}

Give the palette its vocabulary before its face, and make marked-set output paste-ready.

1. **Copy-target registry** (suggested: `src/sase/ace/tui/copy_targets.py`): one record per `(group, target)` with
   footer label, palette label, category, plural label, and an `accepts_marks` flag. Rebuild `_COPY_LABELS`
   (`_mode_commands.py:49-106`) and the footer branches (`_keybinding_modes.py:425-487`) as views over it (keep the
   `pr_number`/`cl_number` alias tolerance, `_core.py:105,113`). Add a test asserting registry coverage equals the
   keymap groups' targets — this replaces the untested duplication.
2. **New targets**, uniform keys across groups, added to `mode_keymaps.py` + `default_config.yml` together:
   - `link` → `l` on all four `artifacts_*` groups **and** `changespecs`: a Markdown link `[<label>](<ref>)` using the
     canonical (no-`@`) reference and a per-kind label — commit subject, plan title/row id, chat basename, bug title,
     humanized ChangeSpec name with its PR URL as the target. Entries with no reference (or a ChangeSpec with no PR URL)
     warn with the existing missing-reference message style (`_artifacts.py:825-832`).
   - `json` → `J` on the four `artifacts_*` groups: the metadata-JSON representation. Resolution and serialization reuse
     the `sase artifact` CLI layer (`ResolvedArtifactReference.to_json_dict`,
     `src/sase/artifact_cli/references.py:41-50`) over the cached `artifact_ref_context`, run off-pump after selection;
     export a public resolve helper from `sase/artifact_cli/references.py` if the current one is private. Toast names
     the kind: "Copied commit metadata JSON".
   - Dispatch: extend the uniform pre-handler checks in `_handle_artifacts_copy_key` (`_artifacts.py:61-69`) using
     `.get(...)` lookups (the `snapshot` `KeyError` trap at `:61` is a known hazard — do not add required lookups);
     `link` for ChangeSpecs lands in `_changespec.py` beside its siblings.
3. **Marked-set forms become paste-ready per representation:**
   - `reference` (marked): newline-separated prompt-form references — no fences, no headers. This deliberately changes
     today's fenced output (`_artifacts.py:178-188`); update its tests and note it in the changelog.
   - `link` (marked): a Markdown bullet list, one `- [label](ref)` per line.
   - `json` (marked): one JSON array of envelopes.
   - Content-shaped targets (`message`, `body`, `transcript`, `spec`, …) keep `format_multi_copy_content` — fenced
     boundaries are right for content dumps.
   - **Partial failure policy**: marked resolution skips unrepresentable items instead of aborting
     (`_artifacts.py:806-807`), and the toast reports both counts ("Copied 3 references — 2 entries have no reference").
     Zero representable items keeps today's warning.
4. **Size caps on contents-shaped targets**: cap per-item and total copy payloads at a shared constant (reuse the 512 KB
   `_MAX_PREVIEW_READ_BYTES` precedent, `src/sase/ace/tui/widgets/_prompt_preview_target.py:23`); over-cap content is
   truncated with an explicit `[Truncated at N bytes]` banner and the toast says so — never silent, never unbounded
   (today `transcript`/`body` reads are uncapped).
5. **Surfaces kept in sync** (per `src/sase/ace/CLAUDE.md`): footer branches gain `l`/`J`; help-modal copy sections
   (`help_modal/changespecs_bindings.py:424-562`) gain the new rows within the 57-char box budget; `docs/ace.md`
   copy-mode tables (`:98-114`, `:500-510`) updated; the `:` command palette picks the new targets up through the
   registry so the catalog guard (`tests/test_command_catalog_guards.py:109-112`) stays green.
6. **Tests**: registry↔keymap coverage; link/json builders per kind (fixture document role named `designs`, never
   `research`); marked forms per representation; partial-failure counts; cap truncation with banner; changespecs `link`
   with and without a PR URL; PR sub-tab behavior untouched.

### Phase 3: The Copy-as palette modal {#palette}

The face: one compact, grouped, previewing palette on the copy prefix, everywhere.

1. **`CopyAsModal`** (suggested: `src/sase/ace/tui/modals/copy_as_modal.py`), modeled on `PropertyPickerModal`'s
   interaction (`property_picker_modal.py`) but composed for beauty:
   - Title: "Copy as…" plus a context line naming the entry (display-name project, entry label) or the marked count ("3
     marked commits").
   - Rows grouped under subtle category rules (Identity · Location · Content · Data · Actions), each row: configured
     accelerator key (rendered via `footer_key_display`), palette label, and a dimmed right-aligned preview. Hand-off
     (`!`) and snapshot rows sit under Actions so copy and act-on aren't visually conflated.
   - Previews are warm-data-only: the already-rendered reference for the selection when the context cache is warm, a
     short value excerpt for cheap targets, a type/size hint otherwise — never computed I/O on open (design decision 7).
     A cold context renders the row with no preview.
   - Navigation: j/k/arrows + enter select, esc/q dismiss; **configured target keys always win over navigation keys** (a
     user who rebinds a target to `j` keeps their key; document the precedence). Unknown keys warn with the existing
     "Unknown copy key (…)" message and keep the palette open.
   - Styling in `src/sase/ace/tui/styles.tcss` consistent with existing pickers; both themes verified by PNG snapshots.
2. **Wiring.** `action_start_copy_mode` (`_core.py:22-33`) pushes the palette after its existing guards; the
   empty-selection case on non-PR sub-tabs warns ("No <subtab> entry to copy") instead of opening an all-dead palette.
   Key dispatch: dismiss, then `app._handle_copy_key(key)` — the flag lifecycle, footer restore (`_restore_copy_footer`,
   `_core.py:64-69`), and the `:` palette's `copy_mode_key` executor (`execute.py:71-75`, which bypasses the modal by
   design) all keep working. The palette works identically on `changespecs`/PRs, all four artifacts sub-tabs, `agents`,
   and `axe`, sourcing rows from each group's registry entries filtered by the availability rules the command catalog
   already encodes (`availability.py:223-300,445-446,535-536`).
3. **Snapshot special case**: selecting `snapshot` dismisses the palette first and captures the pane on the next
   refresh, so the snapshot never contains the palette itself. Pilot-test the captured content.
4. **Forwarding modals**: `CopyModeForwardingMixin` needs no API change — `%` inside `CommitViewModal`,
   `PreviewPanelModal`, help, and the other ten consumers now opens the palette above the modal. Verify the preview
   modal's advertised `% copy` footer hint still holds.
5. **Marked sets**: when the active sub-tab has visible marks, the title switches to the marked form and rows render
   their plural labels; rows with `accepts_marks=False` (e.g. `sha` on commits is superseded by the marked variant
   labels) follow the existing marked-handler behavior — the palette displays what dispatch will actually do.
6. **Surfaces and docs**: footer strip unchanged (it remains accurate); `?` help gains one line ("% opens the Copy as…
   palette") per tab section; `docs/ace.md` copy-mode sections rewritten around the palette (tables stay as the
   accelerator reference); changelog entry. Rebase over `sase-aw.3`/`sase-ax.4` doc changes before finishing (design
   decision 10).
7. **Tests**: pilot — `%` opens the palette on every tab/sub-tab and inside forwarding modals; each accelerator
   dispatches identically to pre-palette behavior (reuse the existing copy-mode test matrix,
   `tests/ace/tui/test_artifacts_copy_mode.py`); enter-on-row parity; esc restores the normal footer and clears the
   flag; unknown-key warning keeps it open; marked title and plural rows; snapshot content excludes the palette. PNG —
   palette on Commits (single), palette with marks, palette over `PreviewPanelModal`, light and dark; accept goldens
   deliberately via `just test-visual`.

### Phase 4: File-kind representations in the artifact-files modal {#files}

The richest artifact kind gets the full set.

1. **Palette entry**: bind the configured copy prefix in `ArtifactFileSelectionModal` (prefix is punctuation, so the
   1-9/a-z selector map at `artifact_files_modal.py:24-35` is untouched; read the prefix from the keymap registry the
   way `CopyModeForwardingMixin` does). It opens the same `CopyAsModal` with modal-local rows over the selected row or
   the marked set (`_marked_indexes`, `:287` — today consumed only by open flows).
2. **File-kind rows** (hardcoded keys per design decision 9, mirroring group conventions):
   - `@` reference — `@file:<id>` prompt form (launch expansion already resolves `file:` refs to paths).
   - `l` Markdown link — `[<label>](file:<id>)`.
   - `c` contents — existing Markdown-only rule (`_MARKDOWN_SUFFIXES`, `:26`) with the phase-2 size cap; non-Markdown
     rows show the target disabled with the reason.
   - `p` stored path / `P` source path — split today's single `Y` policy (`_artifact_file_clipboard_path`, `:120-142`)
     into two explicit rows: stored is always available and anchored; source is origin-labeled, shows "not recorded"
     when absent, and keeps the "(no longer exists)" warning when dead.
   - `J` metadata JSON — `artifact_file_json_dict(record)` (`references.py:88-93`) on the in-memory record; no index
     scan. Byte-parity with `sase artifact show -j`'s `file` object, asserted by test.
   - `s` snapshot — the shared target, dismiss-first.
3. **Marked sets**: references/links/paths/JSON pluralize exactly as phase 2 defines; contents concatenates
   Markdown-only items with `format_multi_copy_content` and reports skipped non-Markdown items in the toast.
4. **Accelerators kept**: `y`/`Y` bindings remain (routed through the delivery seam since phase 1), documented as
   accelerators for contents / stored-or-source path; the modal hint line (`:473-481`) gains the prefix key.
5. **Tests**: pilot — palette opens from the modal, each row copies the expected form, marked pluralization,
   disabled-contents reasoning, source-path missing labels, JSON parity with the CLI serializer; PNG — the palette over
   the files modal. Docs: the artifact-files section of `docs/ace.md` and the agents-tab help box; note the `sase-ax.4`
   docs rebase.

## Validation

Every phase: `just install` first (ephemeral workspaces), then `just check`; `just test-visual` whenever PNG goldens
change, inspecting `.pytest_cache/sase-visual/` and accepting only intentional diffs with
`--sase-update-visual-snapshots`.

Manual validation cases (research §7, narrowed to this surface): SSH + tmux with no `pbcopy`/`wl-copy`/`xclip`/`xsel` —
copy succeeds via tmux/OSC 52 with the honest toast; the same host minus tmux and OSC 52 support — the fallback modal
presents the text; a marked set spanning an entry with no reference — partial toast with both counts; a multi-megabyte
transcript copy — truncation banner, UI stays responsive; `%` inside `PreviewPanelModal` over a plans row — palette
shows the document's reference row; a narrow terminal — palette stays within width and the footer stays one line.

## Out of scope

- **Selected-excerpt-with-anchor copy** (the one item-6 representation not shipped here): ACE's read surfaces have no
  text-selection model to anchor from — Textual's mouse selection copies directly to the clipboard itself, bypassing any
  palette. It needs the reader-search/selection substrate and should ride a `sase-aw` follow-up; the palette's row model
  accommodates it later without redesign.
- Marked-set copy for ChangeSpecs/PRs and the Agents tab (research items 7/9 territory — marks exist there but no copy
  consumes them today; the palette shows their existing single-entry targets unchanged).
- A Files sub-tab (item 5), prompt-bar changes (item 8 landed elsewhere), key-vocabulary renames and Jump All (item 9).
- A `pr:`/ChangeSpec reference kind, new reference grammar, or any `sase-core` change.
- CLI changes (`sase artifact` is complete for this epic's needs; no new subcommands or flags).
- OSC 52 read-back/verification protocols and clipboard-history features.
- Re-theming or restyling existing pickers beyond the new palette's own styles.
