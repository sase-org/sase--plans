---
tier: epic
title: 'Artifacts → Files sub-tab: browse the artifact-file store'
goal: 'The ACE Artifacts tab gains a sixth "Files" sub-tab that makes every indexed
  artifact file browsable by kind, project, agent, origin, and time — with the same
  marks, copy mode, references, filters, and jump machinery as its sibling panes —
  and puts each file one keypress from the preview reader or the rich terminal viewer,
  so the 4,000-file store stops being reachable only through the one agent run that
  produced each file.

  '
phases:
- id: scaffold
  title: Register the Files sub-tab across TUI plumbing
  depends_on: []
  size: medium
  description: 'scaffold: register `files` in the shared Artifacts sub-tab constants,
    compose a real-but-empty ArtifactsFilesPane on digit 6, and wire actions, gating,
    keymaps, copy-mode routing safety, marks state, placeholder/onboarding/help/palette
    surfaces, and styles.'
- id: list
  title: Files list, kind icons, and off-thread loading
  depends_on:
  - scaffold
  size: medium
  description: 'list: load the index off-thread through the Rust-backed query facade
    with two-phase paging, render date-grouped rows with view-mode icons and origin
    badges, implement the shared entry-navigator contract with stable-target cursor
    restore, and build the kind summary chips and hints strip.'
- id: detail
  title: Files detail panel with reference, metadata, and liveness
  depends_on:
  - list
  size: medium
  description: 'detail: build the debounced, worker-backed detail panel — reference
    line with resolved stored path, file metadata with graceful unenriched fallbacks
    and the doctor hint, origin sentence, path liveness badges, and a bounded text
    preview for text-like rows.'
- id: filters
  title: Files filter bar, kind cycle, and in-memory filtering
  depends_on:
  - list
  size: medium
  description: 'filters: add the Files filter bar with kind/project/agent/workflow/
    origin/since/until tokens and free text, pure in-memory filtering over the loaded
    snapshot, completion sources, the s kind-cycle key, and the / edit-query arm.'
- id: open-actions
  title: Smart open, viewer hand-off, external open, and agent jump
  depends_on:
  - list
  size: medium
  description: 'open-actions: implement smart enter dispatch — preview reader for
    text-like files, rich terminal viewer for media — plus Z viewer hand-off for one
    row or the marked set, o open-external, and a jump-to-producing-agent with revival.'
- id: copy-refs
  title: Copy verbs, the % menu, and the file reference branch
  depends_on:
  - open-actions
  size: medium
  description: 'copy-refs: give Files real copy verbs — y reference, Y anchored stored
    path, a % copy menu with contents/path/source/label/JSON targets and marked-set
    support — extend reference_for_entry_target with a files branch, and share the
    modal''s anchored path-copy semantics through one helper.'
- id: visual-docs
  title: Visual snapshots and documentation polish
  depends_on:
  - detail
  - filters
  - copy-refs
  size: small
  description: 'visual-docs: add populated and empty PNG snapshot coverage for the
    Files pane, verify icons and chips render distinctly, and finish the docs/ace.md,
    help, onboarding, and quickstart sweeps including six-way sub-tab numbering and
    the stale path-copy doc line.'
create_time: 2026-07-29 19:13:43
status: wip
---

# Plan: Artifacts → Files sub-tab

## Why

The artifact-file store (`~/.sase/artifacts/` + `index.jsonl`) holds ~4,000 files across 500+ agent runs, and the only
TUI door into it is the per-agent picker modal on the Agents tab (`a`). The consolidated research report
(`research:202607/artifact_refs_and_inspector/artifact_refs_and_inspector.md`, §5 item 5) calls for a sixth **Files**
sub-tab on the Artifacts panel, and explicitly prefers it over merely lifting the Agents-tab gate: a sub-tab makes the
store browsable by project, kind, and time instead of by "which of 5,000 agent runs produced this," and it is the
surface where copy mode, marks, references, and the preview reader all apply for free.

Every enabling layer has already landed — this epic is pure presentation:

- **Data**: `query_artifact_files` (`src/sase/core/artifact_file_query_facade.py:21-59`) is the Rust-backed, wire-
  checked query API over the index (filters: kinds, project, agent, since, explicit_only, query, limit; sorted
  `created_at` DESC). The PyO3 binding releases the GIL, so it is safe from a thread worker. `sase artifact list`
  (`src/sase/artifact_cli/listing.py`) already renders the canonical column vocabulary: KIND · REF (`file:<id>`) · LABEL
  · PROJECT · AGENT · SIZE · CREATED.
- **References**: `file:` is a builtin ref kind (`src/sase/artifact_ref_models.py:12`); a row's `id` (`default:<24hex>`
  / `explicit:<24hex>`) is already the canonical `file:` payload.
- **Copy/marks machinery**: per-sub-tab copy mode, marks, `%@` reference copy, and `%!` agent hand-off all shipped for
  the other non-PR sub-tabs (epics sase-as, sase-av).
- **Viewers**: the rich terminal viewer (kitten icat, mpv, markdown→PDF paging, tmux side pane) and `PreviewPanelModal`
  (now a real reader — copy, `$EDITOR`, rendered Markdown, viewer hand-off; epic sase-aw) need no changes. Rows from the
  query facade already satisfy the viewer's `ArtifactFileLike` protocol (`path` + `kind`).

The Chats sub-tab epic (sase-90, `plans:202607/artifacts_chats_subtab.md`) is the proven playbook for adding a sub-tab;
this plan follows its structure and names every registration point that has drifted since.

## Design pillars

- **Intuitive** — the pane speaks the panel's existing key vocabulary (`j/k`, `enter` view, `f` filter, `s` cycle facet,
  `a` agent, `o` external, `y`/`Y` copy, `Z` viewer, `m` marks, `%` menu, `R` refresh); `enter` always does "the best
  in-terminal view" for the row's type; kind icons come from the same classifier that picks the viewer, so an icon can
  never disagree with what actually opens.
- **Reliable** — durable `file:` references and anchored `~/…` paths (never bare workspace-relative); all index and file
  IO off the message pump with generation guards; every degraded case is a specific toast, never a crash or a silent
  misroute; copy-mode routing is wired in the scaffold phase so a half-landed epic can never leak PR copy semantics onto
  the Files pane.
- **Beautiful** — date-grouped rows with per-kind glyph + color (three-channel with the label text), an origin badge for
  explicit artifacts, kind summary chips in the info line, the sase-9z "logical ref → resolved path" presentation in the
  detail panel, and PNG-snapshot-verified rendering in both populated and empty states.

## Design decisions (apply to all phases)

1. **Append, never insert.** `ARTIFACTS_SUBTAB_ORDER` position drives the digit keys; `files` is appended so Commits
   through PRs keep `1`–`5` and Files becomes `6` (`src/sase/ace/tui/artifact_tabs.py:14-20`). A scaffold test pins
   `ARTIFACTS_SUBTAB_ORDER[:5]` unchanged.
2. **Accent `#FFAF5F` (warm amber), label `Files`.** Distinct from prs `#00D7AF`, commits `#FFD700`, bugs `#FF5F5F`,
   plans `#AF87FF`, chats `#5FAFFF`. The label matches the `Files:` field naming settled by the artifact-file rename
   tale. Row/badge colors are reserved for kinds; the accent is reserved for selection and chrome, so the two encodings
   never fight.
3. **The Chats pane is the template** — flat `OptionList` with date-group headers, stable-target cursor restore
   (`preferred_target=`, NOT the Plans option-id form), coalesced last-request-wins `_request_load` with
   `run_worker(thread=True)`, two-phase paging (bounded first page, then full extension via `spawn_pump_free_task`),
   `DetailPanelDebouncer` + exclusive detail worker, in-pane `#files-hints` strip (non-PR panes deliberately keep the
   conditional footer minimal). Copy `src/sase/ace/tui/widgets/artifacts/chats_pane.py:168-297` and its module split
   (`files_pane.py` / `files_data.py` / `files_list.py` / `files_rendering.py` / `files_navigation.py` /
   `files_detail.py` / `files_filtering.py` / `file_filter_bar.py`).
4. **One data source.** All loads go through `query_artifact_files` from `src/sase/core/artifact_file_query_facade.py` —
   never `read_artifact_file_index` directly — per the `rust_core_backend_boundary` memory: the CLI, the mobile gateway,
   and this pane must share list semantics. The initial load applies only the project scope; facet filtering is pure
   in-memory Python over the loaded snapshot (decision 8). **No sase-core changes anywhere in this epic.**
5. **Kind display derives from the viewer classifier.** Rows and detail render the _view mode_ from
   `artifact_file_view_mode(path, kind=)` (`src/sase/ace/tui/graphics/_viewer_render.py:118-137`) using the established
   glyphs (`src/sase/ace/tui/widgets/prompt_panel/_artifact_files.py:25-32`): `▨` image · `▶` video · `▤` document
   (pdf/markdown) · `•` text/other, each tinted with its established per-kind color. The stored `kind`
   (`chat|plan|image|markdown|pdf|file`) remains the filter facet; the icon guarantees "what you see is what opens"
   (videos are stored as `kind="file"` and only the classifier knows better).
6. **Identity is the index id.** `ArtifactEntryTarget` for a row is `("file", artifact_file.id)` — refresh-stable,
   mark-stable, and the literal `file:` ref payload. Synthesized `default-<ordinal>:` rows never appear here because the
   pane reads only the index, so every visible row is durably referenceable.
7. **Show project names, never keys** (`gotchas` memory): the PROJECT column and detail panel resolve display labels
   through `ProjectRefDisplaySnapshot` exactly as `src/sase/artifact_cli/listing.py:60-66,103` does. Keys remain
   identity in targets and filters.
8. **Load wide, filter narrow.** The index is ~4k rows / 3.5 MB — small enough to hold as one snapshot. The Rust query
   applies the project scope (exact key match; rows with `project=None` therefore appear only in the all-projects scope,
   which the `p` picker reaches — same trade the CLI makes). Facets (`kind:`, `agent:`, `origin:`, `since:`, `until:`,
   free text) filter in memory like `chats_filtering.py`, which keeps keystroke latency at zero, supports `until:` (the
   Rust filter set has no upper bound), and needs no re-query per facet change.
9. **Enrichment fields degrade gracefully.** Most pre-existing rows have `sha256`/`size_bytes`/`mime_type` = `None`
   until `sase artifact doctor --fix` runs. Every renderer treats `None` as `-`; the detail panel adds one dim hint line
   ("run `sase artifact doctor --fix` to backfill") when the selected row is missing enrichment.
10. **Perf rules are non-negotiable** (`sase/memory/tui_perf.md`): the query takes a shared flock and parses 3.5 MB —
    thread workers only, never the pump; render paths never stat, hash, or read files (liveness is computed in the
    detail worker and cached by `(id, stored mtime/size)`); detail updates ride the 150 ms debouncer; programmatic
    highlight assignment is guarded (`_syncing_options` + `_programmatic_update`, cleared in `finally:`). Wrap the
    snapshot load in a `tui_trace` span per `docs/perf_runbook.md`.
11. **The Agents-tab picker stays.** `action_open_artifact_files` and its `current_tab == "agents"` gate
    (`src/sase/ace/tui/app.py:362`) are untouched — it remains the agent-scoped fast path; the Files pane is the global
    browser. No behavior change for `a` on the Agents tab.

## Phases

### Phase 1 — `scaffold`: register the Files sub-tab across TUI plumbing {#scaffold}

**Size:** medium · **Depends on:** nothing

Mount a real-but-empty, fully wired `ArtifactsFilesPane` so digit `6` works, gating is correct, and — critically —
copy-mode routing cannot fall through to PR semantics while later phases land.

1. **Shared constants** — `src/sase/ace/tui/artifact_tabs.py`: add `"files"` to the `ArtifactsSubTab` Literal (`:11`),
   append to `ARTIFACTS_SUBTAB_ORDER` (`:14-20`), `ARTIFACTS_PANE_IDS["files"] = "artifacts-files-pane"` (`:21-27`),
   `ARTIFACTS_ACCENTS["files"] = "#FFAF5F"` (`:28-34`).
2. **View** — `src/sase/ace/tui/widgets/artifacts/view.py`: `_ARTIFACT_LABELS["files"] = "Files"` (`:39-45`),
   `_DETAIL_SCROLL_IDS["files"] = "files-detail-scroll"` (`:50-55`), yield the pane in `compose()` (`:71-90`), and add
   the `ArtifactsFilesPane` fan-out loops to `set_keymap_registry()` and `set_project_scope()` (`:148-183`).
3. **Pane shell** — new `src/sase/ace/tui/widgets/artifacts/files_pane.py`: `ArtifactsFilesPane(...Lifecycle, Vertical)`
   with `can_focus = False`, composing a filter-bar slot, `Static#files-info` (`classes="artifacts-pane-info"`),
   `Horizontal#files-panels` with `#files-list-panel` (border title "Files"; `#files-empty`, `#files-status`, an
   OptionList `#files-list`) and `#files-detail-panel` (border title "Details"; `VerticalScroll#files-detail-scroll` →
   `Static#files-detail`), and `Static#files-hints`. Empty state: "No artifact files found." Export from
   `widgets/artifacts/__init__.py` and `widgets/__init__.py`.
4. **Actions** — new `src/sase/ace/tui/actions/artifacts_files.py` exposing `FILES_ARTIFACT_ACTIONS` and
   `ArtifactsFilesActionsMixin` (pane lookup via `#artifacts-files-pane`), mirroring `actions/artifacts_chats.py`. Wire
   `src/sase/ace/tui/actions/artifacts.py`: union `FILES_ARTIFACT_ACTIONS` into `NON_PRS_ARTIFACT_ACTIONS` (`:39-44`),
   add `"files"` to the hardcoded sets in `_non_pr_artifacts_active()` (`:200-206`) and `_artifacts_entry_navigator()`
   (`:223-237`) and to the `_clear_all_artifacts_marks()` tuple (`:279-282`), add `action_show_artifacts_files()`
   (`:487-508`), and mix the new mixin into `ArtifactsMixin`.
5. **Marks state** — `src/sase/ace/tui/actions/_state_init.py:335-342`: add `"files": set()` to the
   `_artifacts_marked_targets` literal.
6. **App gating** — `src/sase/ace/tui/app.py::check_action`: add `"show_artifacts_files"` to the sub-tab switch list
   (`:396-410`); add a `FILES_ARTIFACT_ACTIONS` arm pinning the actions to the `files` sub-tab (mirror the Chats arm at
   `:417-428`); add `"files"` to the `refresh`-disabled arm (`:388-395`) because `y` becomes a copy key here (phase
   `copy-refs`; `R` remains refresh throughout).
7. **Keymaps** — the pane-scoped action set (defaults; overlap with sibling panes' keys is the established convention):

   | Action                 | Default | Label                  |
   | ---------------------- | ------- | ---------------------- |
   | `files_next`           | `j`     | Next Artifact File     |
   | `files_prev`           | `k`     | Previous Artifact File |
   | `files_view_selected`  | `enter` | View Artifact File     |
   | `files_open_viewer`    | `Z`     | Open in Rich Viewer    |
   | `files_open_external`  | `o`     | Open Externally        |
   | `files_open_agent`     | `a`     | Open Producing Agent   |
   | `files_filters`        | `f`     | Artifact File Filters  |
   | `files_cycle_kind`     | `s`     | Cycle Kind Filter      |
   | `files_copy_reference` | `y`     | Copy Reference         |
   | `files_copy_path`      | `Y`     | Copy Stored Path       |
   | `files_refresh`        | `R`     | Refresh Artifact Files |

   Register each everywhere the `chats_*` family is registered — `src/sase/ace/tui/bindings.py` (a
   `# Files sub-tab actions.` block; the digit-6 binding is generated from `ARTIFACTS_SUBTAB_ORDER`), the keymap
   dataclass/metadata modules, `src/sase/default_config.yml` (a `# Files sub-tab` block after the Chats block —
   mandatory per the `gotchas` memory), `src/sase/ace/tui/commands/_app_metadata.py`, and `commands/availability.py`
   (`_FILES_ARTIFACT_COMMANDS` + gate arm). Grep `chats_copy_path` for the authoritative registration list; actions land
   as stubs here (navigation + refresh no-op on the empty pane) and gain behavior in later phases.

8. **Copy-mode routing safety** (the §1.2 lesson from the research — without this, `%` on Files enters the _ChangeSpecs_
   dispatcher):
   - `src/sase/default_config.yml` (`modes.copy_mode`, `:365-416`) **and** its dataclass mirror
     `src/sase/ace/tui/keymaps/mode_keymaps.py:50-113`: add an `artifacts_files` block with the generic keys only for
     now — `snapshot: s`, `reference: at`, `handoff: exclamation_mark` (file-specific targets land in `copy-refs`).
   - `src/sase/ace/tui/actions/clipboard/_artifacts.py`: add `"files"` to `_non_pr_artifacts_copy_active()` (`:47-53`)
     and an explicit `elif subtab == "files"` arm in `_handle_artifacts_copy_key` (`:55-118`) — the trailing `else:` is
     the Bugs arm and would `KeyError`. The generic `snapshot`/`reference`/`handoff` handling already precedes the
     chain; the files arm contributes an empty target dict until `copy-refs`.
   - `src/sase/ace/tui/widgets/_keybinding_modes.py`: add `artifacts_files` to the copy-footer group set (`:408-413`)
     and a bindings arm (`:425-462`) listing the generic keys.
   - `src/sase/ace/tui/commands/_mode_commands.py`: `_COPY_LABELS["artifacts_files"]` (`:49-106`) and
     `"artifacts_files": "changespecs"` in `tab_to_command_tab` (`:210-218`).
9. **Docs surfaces that hard-fail or lie if skipped**: `widgets/changespec_onboarding.py:35-41`
   `_ARTIFACT_DESCRIPTIONS["files"] = "Browse every artifact file agents have produced."` (a missing key raises in the
   Help guide) and the card subtitle listing the sub-tabs; `widgets/artifacts/panes.py:259-275` `_PLACEHOLDER_COPY`;
   `widgets/tab_quickstart.py` digit tuple and "Commits · Plans · Chats · Bugs · PRs" strings gain Files;
   `modals/help_modal/changespecs_bindings.py` — a "Files Pane" section (keys above + shared navigation rows) and the
   copy-section entry for `artifacts_files` (it asserts on `cm.keys[...]`), minding the column split in
   `binding_common.py:133-137`.
10. **Styles** — `src/sase/ace/tui/styles.tcss`: `#artifacts-files-pane` and the `#files-*` rules modeled on the
    `#chats-*` block so proportions match sibling panes.
11. **Tests**: extend `tests/ace/tui/test_artifacts_scaffold.py` — `6` selects Files, `[`/`]` cycles six panes,
    `ARTIFACTS_SUBTAB_ORDER[:5]` unchanged, `check_action` matrix for `files_*` actions (enabled on Files, disabled
    elsewhere, `refresh` disabled on Files); extend `test_artifacts_marking.py`'s literal marks dict; a copy-mode
    routing test asserting `%` on the Files sub-tab dispatches through the artifacts handler (not the ChangeSpecs
    branch) and that unknown keys degrade politely.

### Phase 2 — `list`: list, kind icons, off-thread loading {#list}

**Size:** medium · **Depends on:** #scaffold

Make the pane show the store, beautifully, without ever touching the pump.

1. **Data module** — `widgets/artifacts/files_data.py`: a `FilesSnapshot` dataclass (rows, project scope, `complete`
   flag, per-view-mode and explicit counts, `load_error: str | None`) and `load_files_snapshot(project, limit)` calling
   `query_artifact_files` (project scope only — decision 8). Catch `RuntimeError`/`ImportError` from the binding and
   return an error snapshot (pane renders the message in `#files-status`; no crash on a stale wheel). Compute each row's
   view mode once here via `artifact_file_view_mode(path, kind=row.kind)`.
2. **Loading** — copy the Chats shape verbatim (`chats_pane.py:168-297`): coalesced `_request_load(force, full)` with
   generation + project guards, `run_worker(thread=True, exclusive=False, exit_on_error=False)`, first page
   `FILES_FIRST_PAGE_LIMIT = 500`, then the full extension via `spawn_pump_free_task`; capture `selected_entry_target()`
   before adopting a new model; `_cancel_artifacts_jump_mode_for_model_change("files")`; `on_first_activate` loads,
   `on_activate` reloads on scope change, `on_refresh` forces. Wrap the load in a `tui_trace` span.
3. **Rows** — `files_list.py` + `files_rendering.py`: date-grouped (`── Today ──` / `── Yesterday ──` /
   `── YYYY-MM-DD ──`, disabled header options), newest first. Each row, columns padded for vertical alignment:

   ```
   ▤ 14:32  [sase]  sase-ax.3--code   ◆ artifact-read-cli design notes      12.4 KiB
   ▨ 14:07  [sase]  ov.study          render-farm teaser frame               882 KiB
   ▶ 09:51  [bob]   bob.4j--code      ingest demo capture                        -
   ```

   View-mode glyph tinted per kind · `HH:MM` · project badge (display name) · presented agent name (via
   `present_agent_name`, workflow as fallback) · `◆` origin badge (accent-colored, explicit rows only) · label · dim
   humanized size (`-` when unenriched). Option ids `file:<id>`; wrap prompts in `prepend_mark_glyph` then
   `prepend_jump_hint`.

4. **Navigation** — `files_navigation.py`: implement the full `ArtifactEntryNavigator` protocol (`entry_targets`,
   `selected_entry_target`, `select_entry_target`, `apply_entry_jump_hints`, `clear_entry_jump_hints`,
   `apply_entry_marks`) with target `("file", id)`, an option-id map, and the `_syncing_options`/`_programmatic_update`
   echo guards — Chats form, giving `'` jump hints, `ctrl+d/u`, `g`/`G`, and marks rendering for free.
   `files_next`/`files_prev`/`files_refresh` become real.
5. **Info line** — kind summary chips computed from the snapshot (never a render-path scan):
   `▨ 3,775 images · ▤ 210 documents · ▶ 12 videos · • 25 files · ◆ 188 explicit`, each chip in its kind color (`◆` in
   the accent), plus a trailing `filtered N/M` segment once phase `filters` lands.
6. **Hints strip** — `build_files_hints` in `files_rendering.py` listing the pane keys via `key_display_name(...)`, `a`
   dimmed when the selected row has no `agent_name` (pane convention: hints strip, not footer).
7. **Tests** — `tests/ace/tui/test_artifacts_files_loading.py` / `…_rendering.py` +
   `tests/ace/tui/_artifacts_files_helpers.py` (fake `ArtifactFile` rows; monkeypatch the pane-module symbol
   `files_pane.load_files_snapshot` — never the real binding): newest-first order and date grouping; icon/color per view
   mode incl. a video-suffixed `kind="file"` row; explicit badge; `None` size renders `-`; first page paints before
   extension completes; error snapshot renders in `#files-status`; cursor survives refresh by stable target; j/k moves
   without highlight echoes. Extend `test_artifacts_list_navigation.py` for the shared navigation contract.

### Phase 3 — `detail`: detail panel with reference, metadata, liveness {#detail}

**Size:** medium · **Depends on:** #list

The right-hand panel answers "what is this, where did it come from, and can I trust its paths" at a glance.

1. **Detail panel** — `files_detail.py`, rendered through `DetailPanelDebouncer` (150 ms) + an exclusive detail worker,
   cache keyed by `(id, stored-path mtime/size)`:
   - **REFERENCE** — the sase-9z presentation: `file:<id>` styled bold, then `→ <stored path>` dimmed.
   - **FILE** — label · kind + view mode · mime type · humanized size · `sha256[:12]…` (dim) · created (minute
     precision); `-` for missing values, plus the one-line doctor hint when enrichment is absent (decision 9).
   - **ORIGIN** — one plain sentence: "Registered explicitly by <agent> during <workflow> in <project>." vs "Captured
     automatically from <agent>'s run."; then project (display name), workflow, agent, artifacts dir.
   - **PATHS** — stored path (always present; `live` badge from the worker's stat) and source path labeled
     `live`/`missing` — never presented as equivalent to the stored path (31% of source paths are dead; research §2.4).
   - **PREVIEW** — for text-like view modes only: a bounded head (~40 lines, worker-read, cached) with a dim "… press
     enter for the full reader" tail. Media rows show no preview block.
2. **Tests** — `tests/ace/tui/test_artifacts_files_detail.py`: detail sections for explicit/default rows; live and
   missing source paths render their distinct badges (the missing copy never presents source and stored paths as
   equivalent); `None` enrichment renders `-` and shows the doctor hint; sha/size/mime render when present; the preview
   head appears only for text-like rows and is bounded; debounced updates coalesce under rapid j/k.

### Phase 4 — `filters`: filter bar, kind cycle, in-memory filtering {#filters}

**Size:** medium · **Depends on:** #list

1. **Filter bar** — new `widgets/artifacts/file_filter_bar.py` subclassing `FilterBar` (accent `#FFAF5F`), modeled on
   `chat_filter_bar.py:26-99`: keys `kind:` (static values `chat|plan|image|markdown|pdf|file`, repeatable), `project:`,
   `agent:`, `workflow:` (dynamic completion from the snapshot), `origin:` (`explicit|default`), `since:`/`until:` (the
   plan-search DATE forms — `YYYY-MM-DD`, `YYYY-MM`, `YYYYMM`, `Nd/Nw/Nm` — validated with the same patterns as
   `plan_date_arg`), and bare text ANDed over label, stored path, and source path. No negation (Chats precedent).
2. **Filtering** — new pure `widgets/artifacts/files_filtering.py` (`filter_files_snapshot`) mirroring
   `chats_filtering.py:14-88`, plus a `FilesFilterSessionMixin` mirroring `chats_filter_session.py` for bar
   open/apply/clear, completion sources, and the status segment ("filtered N/M" in the info line; active kind chip
   highlighted, others dimmed).
3. **Keys** — `files_filters` (`f`) opens the bar; add the `files` arm to `action_edit_query`
   (`src/sase/ace/tui/actions/base.py:441-477`) so `/` works; `files_cycle_kind` (`s`) cycles All → each kind _present
   in the snapshot_ → All, updating the filter state and the highlighted chip (fast path; the bar is the precise path).
4. **Interactions** — filtering preserves selection when the selected target survives, else selects the first row; marks
   are target-keyed so hidden marked rows stay marked (visible-marked semantics for bulk actions come from
   `_visible_marked_targets`, unchanged); date headers recompute over the filtered set; empty-filter state says "No
   artifact files match the active filters." with the clear-key hint.
5. **Tests** — `tests/ace/tui/test_artifacts_files_filtering.py`: each token narrows correctly (incl. `origin:`,
   `since:`/`until:` bounds, repeatable `kind:`), bare text spans label and both paths, `s` cycles only present kinds,
   chip highlight follows, selection preservation, filtered-empty copy.

### Phase 5 — `open-actions`: smart open, viewer hand-off, external open, agent jump {#open-actions}

**Size:** medium · **Depends on:** #list

The whole point of the pane: every row is one keypress from the right viewer.

1. **`enter` — smart dispatch** (`files_view_selected`), branching on the row's view mode:
   - `markdown`/`text` → `PreviewPanelModal`: read the **stored** file off-thread (`asyncio.to_thread`, tolerant
     decode), then push a `PreviewPayload` with `reference=f"file:{id}"`, `source_path=<stored path>`,
     `default_view="rendered"` for markdown (source for text) — the sase-aw reader supplies copy, search, `$EDITOR`, and
     `Z` hand-off inside the modal for free. Mirror `action_chats_view_selected` (`actions/artifacts_chats.py:64-102`).
   - `image`/`video`/`pdf` → the rich terminal viewer directly, exactly as the Agents-tab picker opens one record:
     inside tmux, `view_registered_artifact_file_in_tmux_pane(row)` wrapped in the notify-PID helper and tracked via
     `_track_artifact_file_tmux_pane` (both are app methods — reuse, do not duplicate:
     `actions/agents/_panel_artifact_files.py:118-147`); otherwise `suspend_for_external_tool(...)` around
     `view_registered_artifact_file(row)`. Surface `result.warning` as a toast (missing `kitten`/`mpv`/`pdftoppm`
     degrade with named tools). The PDF-with-live-markdown-source retargeting inside `artifact_file_view_spec` applies
     unchanged.
2. **`Z` — explicit rich-viewer hand-off** (`files_open_viewer`): marked rows on the Files sub-tab (visible-marked
   order) → `view_registered_artifact_files[_in_tmux_pane](rows)` as one multi-artifact paging session (`n`/`p`
   traversal); no marks → the selected row. This is the pane analogue of the picker's multi-open and works for text rows
   too (bat / safe dump path).
3. **`o` — open externally** (`files_open_external`): text-like view modes → `$EDITOR` on the stored path via
   `build_editor_args` + `suspend_for_external_tool` (the `ReportModal`/preview-reader pattern); media/pdf →
   `xdg-open --` when available, else a warning toast naming the tool. Always `--` argument boundaries.
4. **`a` — jump to the producing agent** (`files_open_agent`): resolve the row's agent by `agent_name` /
   `agent_artifacts_dir` against live and dismissed agents, reusing the exact semantics proven by `chats_open_agent`
   (revive through `_do_revive_agent` when dismissed, save tab position, switch to Agents, select by identity with
   raw-suffix fallback, re-capture selection after awaits). No resolvable agent → the specific toast "No local agent
   artifact backs this file." — intentional, not broken.
5. **Consistency guards**: all four actions no-op with a polite toast on header rows / empty pane; viewer and editor
   suspends never run inside a worker (the viewer loop is deliberately synchronous); `enter` from an open tracked viewer
   pane follows the picker's established toggle behavior only for `Z` (leave `enter` always-open).
6. **Tests** — `tests/ace/tui/test_artifacts_files_open.py`: the dispatch matrix (markdown→modal rendered, text→modal
   source, image/pdf/video→viewer) with monkeypatched viewer/suspend seams; marked-set `Z` passes rows in visible order;
   tmux vs non-tmux branches; warning surfacing; `o` editor argv and xdg-open fallback; agent jump (live → select;
   dismissed → revive exactly once; none → toast, no tab change).

### Phase 6 — `copy-refs`: copy verbs, the % menu, the file reference branch {#copy-refs}

**Size:** medium · **Depends on:** #open-actions

Make the durable name the thing you copy, and make every other representation one deliberate key away.

1. **Reference facade** — `src/sase/artifact_refs.py::reference_for_entry_target` (`:614-655`): add the `files`/`file`
   branch returning `file:<id>` (the bare id is already the canonical payload — `artifact_cli/references.py:24`). The
   branch needs no resolution context; in `actions/clipboard/_artifacts.py::_resolve_artifact_references`, short-circuit
   `files` targets before the heavyweight `artifact_ref_context` memoization so reference copy on Files is O(1).
2. **Pane verbs** — `files_copy_reference` (`y`): copies `file:<id>` with the toast naming it ("Copied reference
   file:default:…"); matches the panel convention that `y` copies the row's durable identity (commits→SHA,
   bugs→`#N url`). `files_copy_path` (`Y`): anchored stored-path copy. Extract the modal's
   `_artifact_file_clipboard_path` semantics (`modals/artifact_files_modal.py:107-142` — stored-path preference, the PDF
   source-path exception, home-anchored output, origin labeling, missing warning) into a shared helper module (suggested
   `src/sase/ace/tui/models/artifact_file_clipboard.py`) consumed by **both** the modal and the pane, so the two
   surfaces cannot drift; the pane calls it off-thread (the modal's inline `done.json` reads stay as-is for this epic).
   Both verbs run through `_schedule_artifacts_copy`-style pump-free tasks.
3. **`%` copy menu** — fill the `artifacts_files` group (config + `mode_keymaps.py` mirror + the
   `_handle_artifacts_copy_key` files arm + `_keybinding_modes.py` footer arm + `_COPY_LABELS`):

   | Key                | Target                                | Behavior                                                                                      |
   | ------------------ | ------------------------------------- | --------------------------------------------------------------------------------------------- |
   | `%%`               | contents                              | text-like view modes only, worker-read with a size cap; binary rows → warning naming the kind |
   | `%p`               | stored path                           | the shared anchored helper                                                                    |
   | `%o`               | source path                           | explicitly labeled, warning toast when missing (never silently wrong)                         |
   | `%l`               | label                                 | the human name                                                                                |
   | `%j`               | metadata JSON                         | `artifact_file_json_dict(row)` — record + ref                                                 |
   | `%s` / `%@` / `%!` | snapshot / reference / agent hand-off | generic handlers, already wired in scaffold                                                   |

4. **Marked-set copy** — `_capture_artifact_reference_selection` pane map (`_artifacts.py:213-262`) and a `files` arm in
   `_reference_items_for_targets` (`:696-782`) so `%@` over marks yields the multi-reference block and `%!` seeds one
   agent prompt with every marked ref; `_copy_marked_files_targets` for `%p` (newline-separated anchored paths)
   following the sibling `_copy_marked_*` shape (`:477-619`); `_missing_reference_message` files arm (`:825-832`).
   Toasts always name the count ("Copied 3 references").
5. **Tests** — extend `tests/ace/tui/test_artifacts_copy_mode.py` with the `_CopyHarness` files cases: every target
   above (single + marked), binary-contents refusal, missing-source warning, anchored-path format (`~/…`, never bare
   relative), reference short-circuit (no `artifact_ref_context` construction for files targets — assert via
   monkeypatch), footer copy-binding rows, and modal/pane path-copy parity through the shared helper.

### Phase 7 — `visual-docs`: PNG snapshots and documentation polish {#visual-docs}

**Size:** small · **Depends on:** #detail, #filters, #copy-refs

1. **PNG suite** — `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_files.py` and `…_files_empty.py` following the
   Chats template (`test_ace_png_snapshots_artifacts_chats.py`): `patch_startup_loaders`, monkeypatch
   `files_pane.load_files_snapshot` (and the detail loader) with fixture rows covering **every view-mode glyph**, an
   explicit row, a `None`-enrichment row, and ≥2 projects; press `6`, `expect_state("artifacts_subtab", "files")`,
   deterministic `wait_for` on the snapshot + `wait_for_svg_contains`, SVG assertions (glyphs, accent hex, chips) before
   `assert_page_png(..., "artifacts_files_populated_120x40")` / `"artifacts_files_empty_120x40"`. Accept goldens with
   `--sase-update-visual-snapshots` only for these intentional additions; verify from the rendered PNGs that the four
   glyphs and the origin badge are distinguishable and the columns align.
2. **Docs** — `docs/ace.md`: "five focused sub-tabs" → six; the numbered strip line gains `6 Files`; a "Files sub-tab"
   section (keys, filters, copy targets, the enter dispatch rule, doctor hint); the copy-mode and marks tables gain the
   `artifacts_files` rows; fix the stale `Y` "workspace-relative when possible" line (`docs/ace.md` §artifact-file
   modal) to describe anchored stored-path copy. `docs/cli.md`: confirm the `sase artifact` group section (landed with
   sase-ax) cross-references the new pane.
3. **Final sweeps** — `?` help modal, onboarding card, tab quickstart: six-way numbering and every Files key present and
   accurate; command palette lists the `files_*` and `copy.artifacts_files.*` specs exactly when the pane is active.

## Validation

Every phase: `just install`, then `just check`; `just test-visual` whenever snapshots change (inspect
`.pytest_cache/sase-visual/` artifacts; `--sase-update-visual-snapshots` only for intentional changes).

Manual validation cases (research §7, narrowed to this surface):

- A row whose source workspace was recycled: detail says `missing`, `%o` warns, `Y` still yields the live stored path.
- Two byte-identical artifacts from different agents: two rows, distinct ids/refs, equal `sha256` in detail.
- A host without `kitten`/`mpv`: media `enter`/`Z` degrade to named-tool warnings; text rows still open.
- A host without a clipboard binary: copy reports the existing error toast.
- tmux and non-tmux: viewer opens in a side pane vs suspend; `Tab` returns to ACE from the pane.
- Narrow terminal: hints strip and chips stay one line; rows elide labels, never glyphs or times.
- Filters: `kind:image origin:explicit since:7d` narrows live with zero keystroke lag on ~4k rows.
- An un-backfilled index: sizes render `-`, detail shows the doctor hint, nothing crashes.

## Non-goals

- **No sase-core changes** — the query API, ref grammar, and resolvers all landed; this epic only spends them.
- **No changes to the Agents-tab picker** (`a` gate, modal, tmux toggle) beyond consuming the shared path-copy helper;
  no removal of any existing surface.
- **No Jump All (`` ` ``) artifact entries** — that modal is index-based over PRs and needs a contract change (research
  item 9 polish, separate work).
- **No "Copy as…" palette or OSC 52 transport** (research item 6), no new stores/indexes/IDs (research §6), no `video`
  stored kind, no automatic digest backfill (the doctor owns it; the pane only hints).
- **No `until:` filter in the Rust query** — `until:` is served by the in-memory layer; pushing it down is a follow-up
  only if another consumer needs it.

## Cross-cutting requirements

- Ephemeral workspaces: run `just install` before `just check` in every phase.
- Do not modify `sase/memory/*.md`, `AGENTS.md`, or generated provider shims — nothing here requires it.
- Keep new modules in the established pane split and within repo file-size norms; Symvision governs symbol hygiene (see
  `sase/memory/symvision.md` on lint complaints).
- The pane is read-only: it never writes the index, the store, or any sidecar; its only mutations are the agent revive
  (phase `open-actions`) and clipboard writes.
- The Artifacts tab's internal id stays `changespecs` (`src/sase/ace/tui/tab_order.py:14-17`) — never rename.
