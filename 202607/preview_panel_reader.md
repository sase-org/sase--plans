---
tier: epic
title: Make PreviewPanelModal a real reader
goal: 'The full-screen preview modal that Plans, Chats, and the prompt bar open behaves
  like a real document reader: copy contents/path, the % copy menu, open in $EDITOR,
  hand off to the rich terminal viewer, rendered Markdown for documents, and in-document
  search — matching the standard CommitViewModal already sets.

  '
phases:
- id: reader-core
  title: Reader core — copy, editor, viewer hand-off, reference-aware chrome
  depends_on: []
  size: medium
  description: 'reader-core: add a reference field to PreviewPayload (chats/plans
    callers fill it best-effort), adopt CopyModeForwardingMixin for the % menu, add
    y/Y copy actions, o open-in-$EDITOR, Z terminal-viewer hand-off via a shared path
    helper, a reference→path title line, a dynamic footer, docs/help sweeps, and unit/pilot/PNG
    tests.'
- id: rendered-markdown
  title: Rendered Markdown view with R toggle
  depends_on:
  - reader-core
  size: small
  description: 'rendered-markdown: add default_view to PreviewPayload (plans rows
    open rendered), compose a hidden Textual Markdown widget toggled with R for markdown
    payloads, fence leading YAML frontmatter, enforce plain-render size caps with
    source fallback, style the widget, and cover with pilot/unit/PNG tests plus docs.'
- id: reader-search
  title: In-document search with slash, n, N
  depends_on:
  - reader-core
  - rendered-markdown
  size: medium
  description: 'reader-search: add a hidden commit-on-enter search input with smartcase
    substring matching over the source view, worker-computed match lines and wrapped-row
    offset maps, all-match highlighting through a lazy_renderable highlight_lines
    extension, centered n/N jumps with wraparound, an escape ladder, rendered-to-source
    auto-switch, perf guards, and unit/pilot/PNG tests plus docs.'
create_time: 2026-07-29 16:59:00
status: wip
bead_id: sase-aw
---

- **BEAD:** [sase-aw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-aw/README.md)
- **PROMPT:** [prompts/preview_panel_reader.md](prompts/preview_panel_reader.md)
- **AGENTS:**
  - [bbugyi200.athena.ot](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ot/README.md)
  - [bbugyi200.athena.sase-aw.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aw.1/README.md)
  - [bbugyi200.athena.sase-aw.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aw.2/README.md)
  - [bbugyi200.athena.sase-aw.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aw.3/README.md)
  - [bbugyi200.athena.sase-aw.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-aw.land/README.md)

# Make PreviewPanelModal a Real Reader

## Context

`PreviewPanelModal` (`src/sase/ace/tui/modals/preview_panel_modal.py`) is the primary full-screen read surface for three
flows: Plans rows (`action_plans_view_selected`), Chats transcripts (`action_chats_view_selected`), and the prompt bar's
xprompt/skill/file preview (`PromptPreviewMixin`). Its entire binding set today is scroll and close. This epic
implements item 4 of the consolidated research report
(`research:202607/artifact_refs_and_inspector/artifact_refs_and_inspector.md`, section 5.4): bring the reader up to the
standard `CommitViewModal` already sets — copy, open-in-editor, viewer hand-off, rendered Markdown, and in-document
search.

Two prerequisites from that report have **already landed** and must be spent, not rebuilt:

- Copy mode, marks, and canonical artifact references on all Artifacts sub-tabs (commits `7d41d17a0`, `d867a44ff`,
  `d16fe1dcd`). The app-level `%` copy menu (`src/sase/ace/tui/actions/clipboard/_artifacts.py`) already offers
  per-sub-tab targets including reference copy (`@`) and agent hand-off (`!`).
- The kind-tagged reference facade `src/sase/artifact_refs.py` backed by Rust (`parse_artifact_ref`,
  `canonicalize_artifact_ref`, `reference_for_entry_target`), plus `chats_detail._chat_reference` which canonicalizes a
  transcript path with a cheap minimal context.

Verified enabling fact: Textual's `App.query_one` resolves against the **default screen** (`App._get_dom_base` returns
`self.default_screen`), so app copy-mode handlers reached through `CopyModeForwardingMixin` still find the artifact
panes _beneath_ an open modal. This is exactly how `CommitViewModal` already forwards `%`, and it means the modal gets
the full sub-tab copy menu (including canonical reference copy) for free.

## Design decisions (apply to all phases)

1. **`CommitViewModal` is the template.** `PreviewPanelModal` adopts `CopyModeForwardingMixin`
   (`src/sase/ace/tui/modals/base.py`), hardcoded modal `BINDINGS` (modal-level keys are not remappable anywhere in the
   codebase — `CommitViewModal`, `ReportModal` — so `src/sase/default_config.yml` intentionally needs **no** change), a
   dynamic footer rebuilt per state, and `run_worker(thread=True)` / pump-free tasks for anything slow, per
   `sase/memory/tui_perf.md` rules 1–3.
2. **The payload stays the contract.** `PreviewPayload` (`src/sase/ace/tui/widgets/_prompt_preview_target.py`) gains
   optional, defaulted fields only (`reference`, later `default_view`), so every existing constructor keeps working. The
   modal never reaches into panes for data; callers enrich the payload.
3. **Key vocabulary matches the panel conventions** already documented in the research (§5.9): `y` copy primary
   contents, `Y` copy path, `%` copy menu (via forwarding), `o` open externally, `Z` rich-viewer hand-off, `R`
   rendered/source toggle, `/` `n` `N` search. Uppercase letters are bound in both spellings (`"Y"` and `"shift+y"`),
   following the existing `G`/`shift+g` pattern.
4. **Actions degrade with specific toasts, never crash**: no `source_path` ⇒ warning toast naming the missing
   capability; clipboard tool unavailable ⇒ the existing "Copy failed — clipboard tool not available" error path.
5. **Boundary check** (`rust_core_backend_boundary`): everything in this epic is presentation — modal bindings,
   rendering, scroll math, search UI. Reference strings come from the existing Rust-backed facade. No `sase-core`
   changes.

## Phases

### Phase 1: Reader core — copy, editor, viewer hand-off, reference-aware chrome {#reader-core}

**Size:** medium **Depends on:** nothing

Make the modal act on its document: copy it, open it, hand it off, and say what it is.

1. **Payload:** add `reference: str | None = None` to `PreviewPayload`.
   - Chats: in `action_chats_view_selected` (`src/sase/ace/tui/actions/artifacts_chats.py`), compute the reference
     inside the existing `asyncio.to_thread` load using `_chat_reference` from
     `src/sase/ace/tui/widgets/artifacts/chats_detail.py` (wrap `read_full_chat` + `_chat_reference` in one off-thread
     callable; export `_chat_reference` under a public name).
   - Plans (`preview_for_row` in `src/sase/ace/tui/widgets/artifacts/plans_navigation.py`): bead rows use `issue.design`
     when it parses as a document reference (`parse_artifact_ref`, sub-ms Rust call, Enter-only code path); archive rows
     render `f"{role}:{plan.relpath}"` mirroring `_reference_for_plan_row` in `src/sase/artifact_refs.py`; proposal rows
     stay `None` (canonicalization needs a heavyweight context — the absolute `plan_path` is already shown). Best-effort
     by design: `None` simply omits the reference line.
   - Prompt-bar payloads keep `reference=None`.
2. **Copy keys:** `y` copies `payload.content` ("Copied {kind_label} contents"); `Y` copies `payload.source_path`
   ("Copied path" / warning toast when `None`). Both run the clipboard call off the pump (pump-free task +
   `asyncio.to_thread(copy_to_system_clipboard, ...)`, mirroring `_schedule_artifacts_copy` in
   `src/sase/ace/tui/actions/clipboard/_artifacts.py`); register tasks on the modal and cancel them on unmount
   (`cancel_pump_free_tasks`).
3. **`%` copy menu:** subclass `CopyModeForwardingMixin` (first in MRO, exactly like `CommitViewModal`). No other
   plumbing — the app dispatcher already branches on `current_artifacts_subtab`, and pane lookups resolve against the
   default screen beneath the modal. From the prompt bar, `%` falls through to the active tab's menu, the established
   behavior of every forwarding modal.
4. **`o` open in `$EDITOR`:** replicate `ReportModal.action_open_in_editor` (`src/sase/ace/tui/modals/report_modal.py`):
   `build_editor_args(os.environ.get("EDITOR") or "nvim", [source_path])` inside
   `suspend_for_external_tool(self.app, action="preview_open_editor", tool_kind="editor", ...)`; warning toast when
   `source_path` is `None`.
5. **`Z` rich-viewer hand-off:** add a small shared helper (suggested:
   `src/sase/ace/tui/actions/artifact_viewer_handoff.py`) that opens one _path_ in the terminal viewer, mirroring the
   shape of `_open_artifact_file` in `src/sase/ace/tui/actions/agents/_panel_artifact_files.py`: `is_tmux_session()` ⇒
   `view_artifact_files_in_tmux_pane((ArtifactFileViewSpec(path),))` (track the returned pane id via the app's tracker
   when present, using `getattr`); otherwise `suspend_for_external_tool(...)` around `view_artifact_file(path)`. Surface
   `result.warning` as a toast. Warning toast when `source_path` is `None`. Do **not** refactor the registered-artifact
   panel path in this phase; the helper is additive.
6. **Title chrome:** when `reference` is set, append a line in the muted style `CommitViewModal` uses for its plan line:
   `<reference>  →  <source_path>` (reference alone when no path). Keeps the sase-9z "logical ref and resolved path"
   presentation.
7. **Footer:** replace the static footer string with a `_build_footer()` that lists
   `j/k scroll | ctrl+d/u page | g/G top/bottom | y contents | Y path | % copy | o editor | Z viewer | esc close`,
   omitting `Y`/`o`/`Z` when there is no `source_path`. Keep it one line.
8. **Docs & help:** update `docs/ace.md` — add a "Preview reader" subsection describing the modal keys (referenced from
   the Plans and Chats key tables), and sweep the `?` help modal content (`src/sase/ace/tui/modals/help_modal/`) for any
   preview-modal key listings that must stay in sync (per `src/sase/ace/CLAUDE.md` help-popup rule).
9. **Tests:**
   - Extend `tests/ace/tui/modals/test_preview_panel_modal.py`: `y`/`Y` with a monkeypatched `copy_to_system_clipboard`
     (success and no-path toast), `o` invokes the editor suspend path with the right argv, `Z` warns without a path,
     footer content varies with `source_path`, title shows the reference line.
   - Pilot test that `%` from inside the modal enters app copy mode (forwarding), mirroring existing `CommitViewModal`
     coverage in `tests/ace/tui/modals/test_commit_view_modal.py`.
   - Update/extend the PNG suite `tests/ace/tui/visual/test_ace_png_snapshots_preview_panel.py` (footer/title changed;
     add a fixture payload carrying a reference). Use `--sase-update-visual-snapshots` only for the intentional changes.

### Phase 2: Rendered Markdown view with `R` toggle {#rendered-markdown}

**Size:** small **Depends on:** #reader-core

Remove the oddity that pressing `enter` on a plan renders it _less_ readably than the detail pane behind it.

1. **Payload:** add `default_view: Literal["source", "rendered"] = "source"` to `PreviewPayload`. Plans rows
   (`preview_for_row`) pass `"rendered"` for all three row shapes (proposal, bead, archive) — they are authored Markdown
   documents and the pane behind already renders them with the Textual `Markdown` widget
   (`src/sase/ace/tui/widgets/artifacts/plans_pane.py`). Chats and prompt-bar payloads keep `"source"` (transcripts and
   xprompt/file sources are read as source; `R` remains one keypress away).
2. **Modal:** compose both `Static#preview-content` (existing lazy syntax path) and a Textual
   `Markdown#preview-rendered` widget (initially hidden) inside `#preview-scroll`. `R` toggles visibility for payloads
   with `lexer == "markdown"`; other lexers get a warning toast. Build the Markdown content once, on first entry to
   rendered mode (`Markdown.update(...)`; show the widget after the update completes, keeping the pump free — schedule
   via worker/`call_after_refresh` and guard with `is_attached`). Reset scroll to top on toggle, like `CommitViewModal`
   mode switches.
3. **Frontmatter:** before handing content to the Markdown widget, wrap a leading YAML frontmatter block
   (`---\n...\n---`) in a fenced ` ```yaml ` block so plans/research documents don't render their frontmatter as broken
   prose (the source view already handles this with `FrontmatterMarkdownLexer`). Pure helper + unit test.
4. **Size cap:** rendered mode requires content within `PLAIN_RENDER_MAX_BYTES` / `PLAIN_RENDER_MAX_LINES`
   (`src/sase/ace/tui/util/lazy_syntax.py`). Oversized content: a `default_view="rendered"` payload opens in source view
   silently; an explicit `R` explains with a toast ("too large to render as Markdown").
5. **Chrome:** title/footer reflect the mode (`R source` vs `R rendered`), footer search hint stays accurate once phase
   3 lands.
6. **CSS:** style `#preview-rendered` in `src/sase/ace/tui/styles.tcss` consistently with the plans detail pane
   Markdown.
7. **Tests:** pilot tests for the toggle (default rendered for plans payloads, `R` back to source, non-markdown toast,
   oversized fallback), frontmatter fencing unit tests, and PNG snapshots for rendered mode (plans fixture) plus an
   updated source-mode snapshot if chrome changed. Docs: extend the phase-1 "Preview reader" section.

### Phase 3: In-document search with `/`, `n`, `N` {#reader-search}

**Size:** medium **Depends on:** #reader-core, #rendered-markdown

Find things inside a plan or a transcript without leaving the TUI.

1. **UI:** a hidden one-line search `Input` between the scroll area and the footer (reuse the `FilterInput`
   readline-style subclass from `src/sase/ace/tui/modals/base.py`). `/` reveals and focuses it, prefilled with the last
   query; `enter` commits; `escape` in the input hides it without committing. Escape ladder when the input is closed:
   active matches ⇒ first `escape` clears the search, second closes the modal; `q` always closes.
2. **Semantics:** plain substring match over `payload.content` lines, smartcase (case-insensitive unless the query
   contains an uppercase character). Search always operates on the **source view**: pressing `/` while in rendered mode
   (phase 2) switches to source view first — one consistent, vim-like rule.
3. **Commit pipeline (off the pump):** on `enter`, a `run_worker(thread=True)` job (the `CommitViewModal` worker
   pattern, with identity guards and `is_attached` checks) computes: (a) the matched line numbers and count, and (b) a
   line→render-row offset map for the current content width. Applying results on the UI thread rebuilds the source
   renderable once with **all** match lines highlighted, jumps to the first match at/after the current viewport, and
   shows `k/N · line L` in the footer.
4. **Highlighting:** extend `lazy_renderable` / `LazySyntaxRenderCache` (`src/sase/ace/tui/util/lazy_syntax.py`) with an
   optional `highlight_lines: frozenset[int] | None` that (a) passes through to Rich `Syntax(highlight_lines=...)` on
   the highlight path and (b) applies a background style to matched lines in the capped plain-text path. Extend
   `_SyntaxRenderableKey` so cached entries can't leak across queries.
5. **Jump math:** `n`/`N` wrap around and only _scroll_ (no re-render). Because the source view word-wraps, add a small
   pure helper module (suggested: `src/sase/ace/tui/modals/preview_search.py`) that computes cumulative render-row
   offsets per content line using Rich's own wrapping (`rich.text.Text.wrap`) at the Syntax code width (content width
   minus the line-number gutter, tabs expanded). Cache per (content digest, width); invalidate on resize and recompute
   lazily off-thread. **Center the target row (~⅓ from the top) so any residual ±few-row drift between the estimate and
   Rich Syntax's actual wrap stays invisible** — matches are additionally always visibly highlighted. Matches that fall
   beyond the capped plain-render display (`PLAIN_RENDER_MAX_LINES`) produce a toast ("match at line N is beyond the
   displayed portion") instead of a silent wrong scroll.
6. **Perf guardrails** (`sase/memory/tui_perf.md`): match scanning and offset mapping run in thread workers; input
   keystrokes do no scanning (commit-on-enter only); `n`/`N` are O(1) scrolls; workers are cancelled on unmount and
   guarded against stale application after resize/toggle.
7. **Tests:**
   - Unit: smartcase matching, offset-map agreement with real `Syntax` renders across several widths and contents (long
     unbroken tokens, wide CJK/emoji characters, tabs) — render the `Syntax` to a sized console and compare row counts;
     assert the centered-jump tolerance covers observed drift.
   - Pilot: `/` commit → highlight + jump, `n`/`N` wraparound and counter, escape ladder (input → clear → close),
     auto-switch from rendered mode, search on an over-cap document warns for out-of-window matches.
   - PNG snapshot of an active search (input visible, highlighted matches, counter in footer).
   - Docs: finish the "Preview reader" section with the search keys; final help-modal sweep.

## Validation

Every phase: `just install`, then `just check` (lint + mypy + tests), plus `just test-visual` for the PNG suite when
snapshots change; inspect `.pytest_cache/sase-visual/` artifacts and use `--sase-update-visual-snapshots` only for
intentional visual changes.

Manual validation cases (from research §7, narrowed to this surface): a payload with no `source_path` (bead preview) —
`Y`/`o`/`Z` degrade to toasts and the footer omits them; a host without a clipboard binary — copy reports the error
toast; a narrow terminal — footer stays one line and search jumps still land on-screen; a multi-MB chat transcript —
open, search, and toggle stay responsive (workers, caps); tmux and non-tmux `Z` hand-off.

## Non-goals

- No new copy targets in the app-level `%` menus, no "Copy as…" palette (research item 6), no OSC 52 clipboard transport
  (item 6), no Files sub-tab (item 5), no CLI changes (item 3).
- No change to `CommitViewModal` or `ReportModal`; consolidating the three readers behind one mixin is a possible
  follow-up once this modal proves the pattern.
- No regex search, no cross-document search, no persistence of search state across modal opens beyond the last-query
  prefill.
- No refresh-after-edit: content edited via `o` is re-read only when the modal is reopened.
