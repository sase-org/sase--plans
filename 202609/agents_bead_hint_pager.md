---
tier: tale
title: Render Agents-tab bead hints in sase's pager
goal:
  Selecting a Beads-row hint after pressing v on the Agents tab opens the bead's live
  detail view in sase's pager instead of failing with "File no longer exists".
size: medium
proposed_by: bbugyi200.athena.0qo
create_time: 2026-09-24 10:29:39
status: wip
---

# Fix Agents-tab bead hints so `v` renders the bead in sase's pager

## Problem

On the Agents tab, pressing `v` numbers the rows of the SASE CONTEXT → ARTIFACTS
`Beads:` sub-section, but choosing one of those numbers does nothing useful. The TUI
shows two toasts:

- `File no longer exists: pages/sase-17d/sase-17d.8.md`
- `No selected files could be opened`

Expected behavior: choosing a bead hint opens the bead's live detail view (the same
document `sase bead show` renders, and the same one the pager already shows when you
follow a `bead:` link) inside sase's pager.

## Root cause (confirmed)

`src/sase/ace/tui/widgets/prompt_panel/_agent_bead_touches.py::_bead_hint_target`
registers `sase.bead_pages.paths.bead_page_path(bead_id)` as the hint target. That
function returns a **beads-sidecar-repo-relative** POSIX path (`pages/<root>/<id>.md`);
its module docstring says so and says the page may not be published yet. It is not an
absolute path, and it is not tied to any checkout.

`src/sase/ace/tui/actions/hints/_view_processing.py::_materialize_selected_view_files`
then runs `os.path.exists(resolved_path)` on that relative string, so the check runs
against the TUI process's CWD. The check fails, the path lands in `missing_paths`, and
`_finish_view_request` emits both toasts above. The bug has been there since the Beads
sub-section landed (commit `2b3b37e89`, "feat(agents): render Beads sub-section in
ARTIFACTS lane"). The existing test pins the broken value:
`tests/ace/tui/widgets/test_agent_bead_touch_rows.py` asserts
`hint_mappings == {3: bead_page_path("sase-14j.5")}`.

Clan rows have the same bug. In
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_context.py::_typed_context_value_path`,
the `ARTIFACTS` branch for `BeadTouchEntry` also returns `bead_page_path(...)`.

Verified during planning: from a sase checkout,
`sase.pager.resolve.resolve_link("bead:sase-17d.8", context=default_link_context())`
returns a `LinkResolution` whose `target.kind` is `DOCUMENT`. Its document has one
`PagerSection` (identity `bead:sase-17d.8`, kind `bead`, `origin=PagerOrigin.BEAD`,
title `sase-17d.8 · Spread versus paged rendering`) that carries its own link anchors,
owner, and known kinds. This path goes through `sase.pager.beads.bead_link_resolution`,
which opens the bead store from the link context's anchors and routes foreign-prefix IDs
through `ShowStoreRouter`. It needs no generated page. That is the renderer to reuse.

## Design

Bead hints stop pretending to be file paths. The hint target becomes the canonical
artifact ref `bead:<id>`. The view pipeline then splits those refs out of the file list
and resolves each one to live bead detail sections, using the pager's existing `bead:`
resolver and the link context already captured for the selected agent. Those sections
are merged into the one pager document the `v` flow already builds.

No new side-map is threaded through `HeaderHintState` / `AgentHintRender` / the app
state. The `bead:` scheme prefix is itself the discriminator. Real hint targets are
absolute (or `~`) filesystem paths, so they never parse as `bead:` artifact refs.

### 1. One helper module for bead hint targets

Create `src/sase/ace/tui/bead_hint_targets.py`, a small pure module with no I/O:

- `bead_hint_target(bead_id: str) -> str | None`: returns
  `parse_artifact_ref(f"bead:{bead_id.strip()}").rendered` (from
  `sase.artifact_ref_operations`), or `None` on `ValueError`. This keeps today's
  behavior of giving no hint to IDs like `"not a bead id!!"` or empty strings.
  `parse_artifact_ref` is a Rust-binding parse costing about 7µs, so it is safe on the
  render path (the TUI perf rule: no disk I/O in render paths).
- `bead_id_from_hint_target(target: str) -> str | None`: returns
  `parse_artifact_ref(target).payload.id` when `target` starts with `bead:` and parses
  as a bead ref with a non-empty ID; otherwise returns `None`. Plain paths never reach
  the parser, and `ValueError` returns `None`.
- Reuse `BEAD_READ_REF_PREFIX` from `sase.ace.tui.bead_touches` for the prefix check, or
  define an equivalent local constant if importing it would create a cycle. Do not add a
  third spelling of `"bead:"`.
- Add `__all__` and a module docstring that explains why bead hints are refs, not paths.

### 2. Register refs, not page paths, at both render sites

- `_agent_bead_touches.py`: delete `_bead_hint_target` and call `bead_hint_target(...)`
  from the new module. Row rendering and hint numbering stay the same; only the mapped
  value changes.
- `_agent_display_clan_context.py`: in `_typed_context_value_path`, the `BeadTouchEntry`
  branch returns `bead_hint_target(value.bead_id)`. Drop the `bead_page_path` import
  there.
- Do not touch `sase.bead_pages.paths`. It is still correct for its real callers (page
  publication, hosted links, `sase bead pages`).

### 3. Route bead refs through the view pipeline (`_view_processing.py`)

`_prepare_view_input` (UI thread, no I/O):

- After `files = self._files_for_view_hints(...)`, split the selection: `bead_ids` is
  the ordered, de-duplicated tuple of `bead_id_from_hint_target(f)` for the entries
  where that returns a value, and `files` keeps only the remaining entries.
- Change the empty-selection guard to
  `if not files and not commit_hint_nums and not bead_ids`.
- Editor mode (`@`) with beads selected: a bead has no editable file. Notify one
  warning, `Beads cannot be opened in an editor: <ids>`, drop the bead IDs, and carry on
  with any remaining files or commits. If nothing remains, return `None`.
- Add `bead_ids: tuple[str, ...] = ()` to `_ViewRequest` and populate it.
  `request.files` must no longer contain `bead:` refs, so `selected_reports` and
  `selected_ref_items` stay unaffected.

`_materialize_selected_view_files` (already runs inside `asyncio.to_thread`):

- Add a `bead_ids: tuple[str, ...] = ()` parameter. Compute the `LinkResolutionContext`
  once (today it is computed at the `return`). Then, for each bead ID, call
  `resolve_link(f"bead:{bead_id}", context=link_context)` from `sase.pager.resolve`. On
  success (`target is not None and target.document is not None`), extend the bead
  sections with `target.document.sections`. On failure, record
  `resolution.unresolved_message`, falling back to
  `f"bead:{bead_id} could not be resolved"`.
- Extend `_MaterializedReports` with `bead_sections: tuple[PagerSection, ...] = ()` and
  `bead_failures: tuple[str, ...] = ()`.
- `_finish_view_request` passes `request.bead_ids` only when the request is a plain
  view: not copy and not editor. Bead-store I/O stays off the event loop and never runs
  for `%`.

`_finish_view_request` (worker coroutine; keep the existing `is_running` guards).
Reorder the tail so that each mode handles beads explicitly, then keep every non-bead
branch behaving as it does today:

1. Notify `failed_paths` and `missing_paths` as today, then each `bead_failures` message
   (severity `warning`).
2. **Copy (`%`)**: `items = [*request.bead_ids, *files]`. Copy bare bead IDs such as
   `sase-17d.8`, ahead of the paths. If `commit_specs` is set, call
   `_copy_commit_specs_to_clipboard(commit_specs, items)`. Otherwise, if `items` is
   non-empty, call `_copy_files_to_clipboard(items)`. If neither applies, show the
   existing `No selected files could be opened` warning. Return. Existing copy behavior
   with no beads must not change.
3. **Editor (`@`)**: unchanged. Bead IDs were already stripped in `_prepare_view_input`.
4. **View**: if there are no `files` and no `bead_sections`, keep today's behavior (warn
   `No selected files could be opened` unless `commit_specs` is set) and return. If any
   file is a supported image or video, keep the artifact-viewer branch. If
   `bead_sections` is also non-empty there, notify one warning that the selected beads
   are not shown alongside media. Otherwise build the pager document off-thread as
   today, passing `bead_sections=outcome.bead_sections`, and push it with
   `_view_files_with_pager_screen`.

### 4. Compose bead sections into the pager document (`_files.py`)

Extend
`build_pager_document(files, commit_specs=(), *, link_context=None, bead_sections=())`:

- Section order: commit manifest (if any), then bead sections in selection order, then
  file sections. This mirrors how the commit manifest is already prepended.
- Keep `origin=PagerOrigin.FILE` and the captured `link_context` on the combined
  document. Each bead section already carries `origin=PagerOrigin.BEAD`, and
  `section_origin()` honors a section's own origin, so bare bead-ID tokens inside bead
  sections still resolve as beads.
- Title: with no files and exactly one bead section, use that section's `title`, for
  example `sase-17d.8 · Spread versus paged rendering`. With several beads and no files,
  use `"N beads"`. For a mixed selection, use a short count title such as
  `"1 bead · 2 files"`. Any non-empty title is fine; `PagerDocument` rejects an empty
  one. `document_from_paths([])` already works, with zero sections and a count title the
  caller overrides.
- Update the docstring: bead sections come from the caller and are already resolved
  off-thread.

### 5. Docs

`docs/ace.md` (the SASE CONTEXT / ARTIFACTS paragraph, currently "A numbered hint opens
the bead's page"): say that a numbered hint opens the bead's live detail (the
`sase bead show` view) in sase's pager, that `%` copies the bead ID, and that `@` is not
supported for bead rows.

## Tests

Update:

- `tests/ace/tui/widgets/test_agent_bead_touch_rows.py`
  - `test_hints_map_bead_pages_and_skip_invalid_ids`: rename it, for example to
    `test_hints_map_bead_refs_and_skip_invalid_ids`, and expect
    `{3: "bead:sase-14j.5"}`. The invalid ID still gets no hint, and the counter
    becomes 4.
  - `test_clan_hint_target_returns_bead_page`: rename it and expect `"bead:sase-14j.5"`.
    `test_clan_hint_target_rejects_invalid_bead_id` stays `None`.
  - Drop the now-unused `bead_page_path` import.

Add:

- A unit test module for `bead_hint_targets`. Cover a valid ID (with surrounding
  whitespace) mapping to `bead:<id>`, invalid or empty IDs mapping to `None`, the
  round-trip `bead_id_from_hint_target(bead_hint_target(x)) == x`, and absolute paths,
  `~` paths, and malformed `bead:` strings mapping to `None`.
- View-pipeline tests next to `tests/ace/tui/actions/test_view_files_pager_dispatch.py`.
  Reuse `_make_app` from `_view_files_helpers.py`, and monkeypatch
  `sase.ace.tui.actions.hints._view_processing.resolve_link` to return a fake
  `LinkResolution(target=LinkTarget(kind=LinkTargetKind.DOCUMENT, document=...))`. The
  fake document holds one
  `PagerSection(identity="bead:sase-1", kind="bead", origin=PagerOrigin.BEAD, ...)`.
  1. `_make_app("bead:sase-1")` and `_process_view_input("1")` open the pager exactly
     once. The document's section identities are `["bead:sase-1"]` and its title is the
     bead section's title. No `File no longer exists` toast appears; this is the
     regression test for the reported bug.
  2. A mixed file + bead selection (`"1 2"`) produces sections ordered `[bead, file]`.
  3. An unresolved bead (the fake returns `unresolved_message`) produces a warning toast
     with that message and does not open the pager when nothing else was selected.
  4. Bead resolution runs off the event-loop thread. Record `threading.get_ident()`
     inside the fake `resolve_link`, as the existing
     `test_builds_the_document_off_the_event_loop_thread` does.
  5. `"1%"` copies `sase-1`, and `resolve_link` is never called. Assert on the scheduled
     clipboard content with the helpers used by the existing copy tests.
  6. `"1@"` shows the `Beads cannot be opened in an editor` warning, does not suspend
     for the editor when only a bead was selected, and never calls `resolve_link`.
- Keep every existing view-files test green: the commit, image, copy, report, and
  artifact-read-repair paths must not change.

## Verification

- Follow the repo's lint/test memory note: run `just check` through `sase tool run`.
- Manual check (optional; screenshots are not required): in `sase tui` on the Agents
  tab, select an agent whose SASE CONTEXT → ARTIFACTS shows `Beads:` rows, press `v`,
  and type a bead row's number. The pager opens the bead detail (header
  `sase-<id> · <title>`, the same content as `sase bead show <id>`), and no "File no
  longer exists" toast appears.

## Out of scope

- Opening a bead's generated sidecar page in `$EDITOR` for `@`. A follow-up can add it
  if wanted.
- Changing `sase.bead_pages.paths` or the BEAD lane's plan-file hint
  (`BeadSummary.actual_plan_path`). That hint is a real file and works today.
- No feature flag: this is a bug fix to a broken hint target, not a new or deprecated
  user-facing behavior.
