---
tier: tale
title: Fix the ACE crash when opening a byte-free artifact file
goal:
  Pressing <enter> on a VCS-backed (byte-free) artifact row in the Agents-tab artifact
  picker materializes the file and opens it in the viewer instead of crashing sase ace
  with a TypeError.
proposed_by: bbugyi200.athena.ux
create_time: 2026-08-07 14:28:26
status: done
---

- **PROMPT:**
  [prompts/202608/ace_byte_free_artifact_view_crash.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ace_byte_free_artifact_view_crash.md)
- **AGENTS:**
  - [bbugyi200.athena.ux](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ux.md)
- **COMMITS:**
  - [28e8ed1](https://github.com/sase-org/sase/commit/28e8ed1cebc384f1a283cf7b010c27d981b7f49d)
    — fix(ace): materialize byte-free artifact rows before opening from Agents tab

# Plan: Fix the ACE crash when opening a byte-free (VCS-backed) artifact file

## Symptom

`sase ace` hard-crashes (unhandled `TypeError`, Textual tears the app down) when the
user presses `<enter>` on an image row in the artifact-file picker opened with the `a`
keymap on the Agents tab:

```
TypeError: argument should be a str or an os.PathLike object where __fspath__ returns a
str, not 'NoneType'
```

with these locals at the top frame:

```
artifacts = (ArtifactFileViewSpec(path=None, kind='image'),)
command   = ['.../python3', '-m', 'sase.ace.tui.graphics.viewer', '--kind', 'image']
```

## Root cause

The crashing row is a **byte-free artifact row**: an `ArtifactFile` whose bytes were
never pooled because the file was clean and tracked in a durable VCS ref, so the row
records `vcs_repo` / `vcs_sha` / `vcs_relpath` instead of a stored `path`.

The chain:

1. `ArtifactFile.path` is `str | None` (`src/sase/core/artifact_file_types.py`), and
   `ArtifactFile.is_vcs_backed` is exactly "no `path` + complete VCS provenance".
2. `store_default_artifact_file()` (`src/sase/core/artifact_file_explicit.py`) sets
   `stored_path = None` in `reference_mode` (when the capture policy returns
   `outcome="reference"`). This is the intended, lossless representation — not
   corruption.
3. The Agents-tab picker lists rows via
   `AgentPanelArtifactFileMixin._list_selected_artifact_files()` →
   `read_artifact_files_for_tui()` →
   `sase.core.artifact_file_facade.list_artifact_files()`, which returns those byte-free
   rows unchanged.
4. `<enter>` dismisses `ArtifactFileSelectionModal` into `_open_selected()` inside
   `action_open_artifact_files()`
   (`src/sase/ace/tui/actions/agents/_panel_artifact_files.py`), which calls
   `_open_artifact_files()` → `_open_artifact_file()` → the graphics layer with the
   **raw, unmaterialized row**.
5. `artifact_file_view_spec()` (`src/sase/ace/tui/graphics/_viewer_artifact_files.py`)
   copies `artifact_file.path` verbatim, producing
   `ArtifactFileViewSpec(path=None, ...)`.
6. Both launch paths then feed `None` into `pathlib.Path`:
   - tmux: `artifact_file_viewer_module_command()` in
     `src/sase/ace/tui/graphics/_viewer_tmux.py` — `str(Path(spec.path).expanduser())`.
   - non-tmux: `view_artifact_files()` in `src/sase/ace/tui/graphics/_viewer_launch.py`
     — `artifact_file_view_mode(specs[0].path, ...)` → `Path(path).suffix`.

   Nothing between the picker and `pathlib` validates `spec.path`, so the failure
   surfaces as an unhandled `TypeError` instead of a viewer warning.

**Why only the Agents tab crashes.** The Artifacts tab already resolves bytes first:
`ArtifactsFilesActionsMixin.action_files_open_viewer()` / `action_files_view_selected()`
/ `action_files_open_external()` (`src/sase/ace/tui/actions/artifacts_files.py`) check
`entry.path` and, when it is absent,
`await asyncio.to_thread(materialize_artifact_file_entries, ...)` before opening. The
Agents-tab picker never got that treatment. The picker's _copy_ verbs
(`src/sase/ace/tui/modals/artifact_files_modal_copying.py`) are also already safe — they
go through `artifact_file_materialized_stored_path()` /
`artifact_file_materialized_display_path()`. Only the picker's **open/zoom-open verbs**
are broken.

**Type-level reason it slipped through.** The viewer protocol `ArtifactFileLike`
(`src/sase/ace/tui/graphics/_viewer_types.py`) declares `path: str`, while the real
`ArtifactFile.path` is `str | None`; and the whole Agents-panel artifact plumbing is
typed `Any`, so mypy never compared the two.

## Evidence (measured, not assumed)

On the developer machine at the time of diagnosis:

- `~/.sase/artifacts/index.jsonl`: 5842 rows, **1403 with `path: null`** — every one
  `kind="image"` with complete VCS provenance (`vcs_repo` ∈ {`sase`, `research`,
  `plans`}), e.g. `tests/ace/tui/visual/snapshots/png/*.png` referenced from agent
  prompts.
- Scanning the 400 most recent agent artifact dirs through `list_artifact_files()`
  yields **1323 byte-free rows**, i.e. a large fraction of recent runs can reproduce
  this.
- Direct reproduction (both launch paths crash identically):

  ```python
  from sase.core.artifact_file_facade import list_artifact_files
  from sase.ace.tui.graphics._viewer_artifact_files import artifact_file_view_spec
  from sase.ace.tui.graphics._viewer_tmux import artifact_file_viewer_module_command
  from sase.ace.tui.graphics import view_registered_artifact_file

  rows = list_artifact_files("<an agent artifacts dir with a byte-free row>")
  row = next(r for r in rows if not r.path)          # is_vcs_backed is True
  spec = artifact_file_view_spec(row)                # ArtifactFileViewSpec(path=None, kind='image')
  artifact_file_viewer_module_command((spec,))       # TypeError  (tmux path)
  view_registered_artifact_file(row)                 # TypeError  (non-tmux path)
  ```

- The fix is viable: `materialize_artifact_file_entry(row)` (from
  `src/sase/ace/tui/models/artifact_file_clipboard.py`) resolves the same row to a real,
  existing PNG under `~/.sase/artifacts/vcs-cache/<xx>/<sha256>.png`.

## Fix

Two layers: make the Agents-tab picker resolve bytes like the Artifacts tab already
does, and make the graphics layer fail soft instead of raising `TypeError` if any future
caller regresses.

### Change 1 — Agents-tab picker materializes byte-free rows before opening

File: `src/sase/ace/tui/actions/agents/_panel_artifact_files.py`, in
`action_open_artifact_files()`'s `_open_selected()` callback (the single call site that
reaches `_open_artifact_files()` from the Agents tab; `action_view_image()` delegates
here too, and both the single-selection and marked/`A`/`z` selections funnel through
it).

Behavior:

- Normalize the selection exactly as today (`_normalize_artifact_file_selection`), and
  return early (still restoring focus) when the selection is empty.
- **Fast path, unchanged:** if every selected row has a truthy `path`, call
  `self._open_artifact_files(selected, zoom=zoom)` synchronously inside `try/finally`
  with `self._focus_agent_list_after_artifact_modal()` in the `finally` — byte-anchored
  rows must keep their current, fully synchronous behavior so existing tests and the
  tmux notify-pid/pane-tracking sequence are untouched.
- **Byte-free path:** build a coroutine that
  1. `materialized = await asyncio.to_thread(materialize_artifact_file_entries, tuple(selected))`
     — materialization shells out to git through the Rust binding, so it must not run on
     the UI thread;
  2. on `OSError` / `RuntimeError`,
     `self.notify(f"Could not open artifact file: {exc}", severity="warning")` and stop
     (the raised message already names the `<repo>@<sha>:<relpath>` locator);
  3. otherwise call `self._open_artifact_files(list(materialized), zoom=zoom)` on the
     loop thread, so the existing tmux-pane / suspend logic is reused verbatim;
  4. restores focus via `_focus_agent_list_after_artifact_modal()` in a `finally`.

  Spawn it with
  `spawn_pump_free_task(self, coro, name="sase-agent-artifact-materialize", registry_attr="_pump_free_async_tasks")`
  (`src/sase/ace/tui/util/pump_tasks.py`), matching `artifacts_files.py`. Import
  `materialize_artifact_file_entries` from
  `sase.ace.tui.models.artifact_file_clipboard`, mirroring the alias style used in
  `artifacts_files.py`.

- **No-loop fallback:** `spawn_pump_free_task()` returns `None` and closes the coroutine
  when no event loop is running. Do not let that become a silent no-op: when it returns
  `None`, run the same materialize-then-open sequence synchronously (a small shared
  helper method used by both branches keeps this from duplicating the notify/focus
  handling). The real app always has a loop; this keeps non-async unit tests and any
  future non-loop caller honest.

Keep the mixin's existing `# type: ignore[attr-defined]` conventions for `notify`/
`push_screen`; do not widen typing beyond what the surrounding code already does.

### Change 2 — graphics viewer layer fails soft on a path-less spec

Files: `src/sase/ace/tui/graphics/_viewer_launch.py`,
`src/sase/ace/tui/graphics/_viewer_tmux.py`,
`src/sase/ace/tui/graphics/_viewer_artifact_files.py`.

- `view_artifact_files()` and `view_artifact_files_in_tmux_pane()`: right after the
  existing empty-`specs` guard, if any spec has a falsy `path`, return
  `viewer_result_from_warnings((ArtifactFileViewerWarning("missing_artifact_path", "Artifact content is not available on disk", ...),))`.
  The callers already surface `result.warning` through `self.notify(...)`, so the user
  gets a warning toast instead of a crash.
- `view_registered_artifact_files()` and `view_registered_artifact_files_in_tmux_pane()`
  (which receive rows, not specs): apply the same guard _before_ building specs so the
  message can be row-aware (name the label / VCS locator rather than a bare path).
- `artifact_file_viewer_module_command()`: raise a clear
  `ValueError("artifact view spec requires a path")` instead of letting `pathlib` raise
  `TypeError` from a generator expression. It is an internal command builder; the
  `view_*` guards above are what user-facing callers hit.
- `artifact_file_view_spec()`: document and enforce that a byte-free row is a
  programming error at this layer (raise `ValueError`), so the spec type's
  `path: str | Path` contract is real.
- `ArtifactFileLike.path` in `src/sase/ace/tui/graphics/_viewer_types.py`: change to
  `str | None` so the protocol matches `ArtifactFile` and the guard above is
  type-motivated rather than defensive noise. Fix any mypy fallout this exposes in the
  graphics package.

## Tests

Add regression coverage at both layers. Follow existing conventions —
`asyncio_mode = "auto"` is set in `pyproject.toml`, so `async def` tests need no
decorator.

1. **Agents-tab picker (new or extended in
   `tests/ace/tui/actions/test_artifact_file_image_open.py`; a new
   `tests/ace/tui/actions/test_artifact_file_vcs_open.py` is fine if the existing module
   gets unwieldy)**, using the `_ImageActionApp` harness from
   `tests/ace/tui/actions/_artifact_file_image_helpers.py`:
   - byte-free row (`path=None`, `kind="image"`, VCS provenance) → `<enter>` callback
     materializes and opens: assert the object handed to
     `view_registered_artifact_file[_in_tmux_pane]` has a non-`None` `path`, and assert
     the viewer is **never** called with `path=None` (this is the exact crash guard).
   - materialization raising `OSError` → exactly one `notify(..., severity="warning")`,
     no exception escapes, and agent-list focus is restored.
   - marked/multi-selection and the `z` zoom-open result
     (`ArtifactFileSelectionResult(..., zoom=True)`) still forward `zoom=True` after
     materialization.
   - regression: a selection whose rows all have `path` opens **synchronously** (no
     materialization call, no spawned task) — locks in the unchanged fast path.
2. **Graphics layer** — `tests/ace/tui/artifact_file_viewer/test_launch.py` and
   `tests/ace/tui/artifact_file_viewer/test_tmux.py`:
   - `view_artifact_files((ArtifactFileViewSpec(None, "image"),))` and
     `view_artifact_files_in_tmux_pane(...)` return `ok is False` with warning code
     `missing_artifact_path`, and raise nothing.
   - `view_registered_artifact_file(<byte-free row>)` returns the same structured
     warning.
   - `artifact_file_viewer_module_command` raises `ValueError` (not `TypeError`).
3. **End-to-end, optional but preferred**: reuse the real-git fixture pattern already in
   `tests/ace/tui/test_artifact_file_vcs_clipboard.py` (`_vcs_row()` builds a committed
   file and a matching byte-free `ArtifactFile`; `_install_materialization_context()`
   patches `sase.artifact_ref_context.launch_artifact_ref_context` and
   `sase.core.artifact_file_vcs.default_artifact_files_root`) to prove the Agents-tab
   open flow resolves real bytes out of the `vcs-cache`. Import those helpers if a
   cross-module import is clean; otherwise lift them into a shared `tests/ace/tui/`
   helper module and update the existing test to use it — do not copy-paste them.

## Verification

- `just install` first (workspace venvs go stale), then `just check`.
- If the diff ends up touching the graphics package broadly, run `just check-full`.
- `just test-visual` is not expected to be affected (no renderer changes).
- Manual smoke on a real setup: open `sase ace`, Agents tab, select an agent whose run
  referenced a committed image (visual-snapshot PNGs are the common case), press `a`,
  then `<enter>` on the image row — inside tmux it must open the viewer pane, and
  outside tmux it must suspend and open; neither may crash. Repeat with `z` (zoom open)
  and with a marked multi-selection.

## Notes / proposed follow-ups (not in scope here)

- **All-or-nothing multi-open.** `materialize_artifact_file_entries()` raises on the
  first unresolvable row, so one dead VCS reference aborts an `A` (open all) batch. This
  plan keeps that behavior deliberately, to stay consistent with the Artifacts tab. A
  per-row tolerant variant (open what resolves, warn about the rest) is a reasonable
  separate task bead.
- **Misleading picker path for byte-free rows.** `artifact_file_preferred_path_text()`
  falls back to the repo-relative `vcs_relpath`, which
  `artifact_files_modal_rendering._display_path()` then resolves against the ACE process
  CWD — so the picker can display a plausible-looking path that does not exist.
  Cosmetic, separate from the crash; worth its own bead.
- **`Any`-typed artifact rows.** Typing the Agents-panel artifact plumbing as
  `ArtifactFile` instead of `Any` would have let mypy catch this class of bug; a larger,
  separate cleanup.
