---
tier: epic
title: Split Artifacts into a dedicated Beads sub-tab and a nested Files sub-tab
goal: "The Artifacts tab exposes a bead-only Beads sub-tab with full task-bead triage, and a Files sub-tab whose Plans,
  Chats, and Other sub-sub-tabs cycle with ( and ), with Plans dedicated to plan documents and bidirectional jumps
  between a plan file and the bead that links it.

  "
phases:
  - id: shell
    title: Sub-tab taxonomy, nested Files container, and keymap surface
    depends_on: []
    size: medium
    description:
      "shell: retype the Artifacts sub-tabs to commits/beads/bugs/prs/files, mount a nested tab strip inside Files that
      hosts Plans, Chats, and Other, re-key every mark/jump/detail/copy/footer store off a pane key instead of a
      sub-tab, and declare all new keymap fields, bindings, and action stubs up front."
  - id: beads_view
    title: Read-only Beads pane
    depends_on:
      - shell
    size: medium
    description:
      "beads_view: load beads off-thread behind an mtime-keyed snapshot, render a Tasks section and an expandable Epics
      tree in which every bead appears exactly once, build the detail panel from shared bead presentation metadata, and
      wire navigation, marks, and jump hints."
  - id: beads_filters
    title: Bead filter query and inline filter bar
    depends_on:
      - beads_view
    size: small
    description:
      "beads_filters: parse a bead filter query covering type, status, tier, size, project, people, has, and date terms,
      prefold it into a reusable record index, and drive an inline completion bar whose hide-closed default stays
      visible in the info line."
  - id: beads_actions
    title: Bead mutations, close-with-reason, and triage-gate settlement
    depends_on:
      - beads_view
    size: medium
    description:
      "beads_actions: run every bead write as a tracked background task through the shared store-mutation facade, add
      create, edit, note, status, launch, and close surfaces, and settle the matching pending triage gate when a task
      bead is closed or launched from the pane."
  - id: plans_focus
    title: Plans sub-sub-tab dedicated to plan documents
    depends_on:
      - shell
    size: medium
    description:
      "plans_focus: regroup the pane into pending proposals, plans linked from live beads, and the archive, delete the
      bead rows and bead-only actions, and retune the filter vocabulary, detail panel, and copy targets to plan
      documents."
  - id: crosslinks
    title: Bidirectional bead and plan jumps with conditional footer hints
    depends_on:
      - beads_filters
      - beads_actions
      - plans_focus
    size: small
    description:
      "crosslinks: resolve each row's counterpart across the two panes, switch sub-tabs and apply a pending selection
      that survives an unloaded target pane, and surface the jump keys as conditional footer entries."
  - id: polish
    title: Help, docs, onboarding, and visual snapshots
    depends_on:
      - crosslinks
    size: medium
    description:
      "polish: document the new layout in the help modal and the ace guide, refresh onboarding and empty-state copy,
      complete the copy-mode palette for the new group, and re-record the affected PNG goldens."
proposed_by: bbugyi200.athena.r7
create_time: 2026-08-01 09:52:31
status: wip
---

- **PROMPT:** [202608/prompts/artifacts_beads_and_files_subtabs.md](prompts/artifacts_beads_and_files_subtabs.md)

# Plan: Artifacts Beads sub-tab and nested Files sub-tab

## Why

The Artifacts **Plans** sub-tab is misnamed. It renders five row kinds — plan proposals, standalone task beads, epic
beads, phase beads, and committed plan documents — and most of its keymaps (`s` status, `e` edit bead, `w` launch epic,
`o` open bug) act on beads, not plans. Beads have no home of their own, so task-bead triage lives entirely in
notification gates, and there is no way to update or close a bead from the TUI.

This epic splits that pane along its real seam:

- **Beads** becomes a top-level Artifacts sub-tab dedicated to bead work items.
- **Plans** becomes a document view nested under **Files**, alongside **Chats** and **Other**.
- A single key jumps between a bead and the plan document it links, in both directions.

## Target layout

Top-level Artifacts sub-tabs (`[` / `]` cycle, `1`–`5` select):

| #   | Sub-tab | Accent    | Contents                                   |
| --- | ------- | --------- | ------------------------------------------ |
| 1   | Commits | `#FFD700` | unchanged                                  |
| 2   | Beads   | `#D787FF` | **new** — task, epic, and phase beads only |
| 3   | Bugs    | `#FF5F5F` | unchanged                                  |
| 4   | PRs     | `#00D7AF` | unchanged                                  |
| 5   | Files   | `#FFAF5F` | **container** — hosts three sub-sub-tabs   |

Files sub-sub-tabs (`(` / `)` cycle):

| Sub-sub-tab | Accent    | Contents                                                             |
| ----------- | --------- | -------------------------------------------------------------------- |
| Plans       | `#AF87FF` | plan documents: proposals, plans linked from live beads, the archive |
| Chats       | `#5FAFFF` | today's Chats pane, unchanged                                        |
| Other       | `#FFAF5F` | today's Files pane, unchanged                                        |

`#D787FF` is the task-bead accent already published by `sase.bead_type_presentation`, so the Beads sub-tab inherits the
color the rest of SASE already uses for bead work. Plans keeps `#AF87FF`; the two purples never appear in the same
strip, and together they read as one bead/plan family.

The default Artifacts sub-tab stays **Commits**. The default Files sub-sub-tab is **Plans**, and the nested view
remembers the last visited sub-sub-tab for the session, so the default only applies on first entry.

## Design rules that hold for every phase

1. **One row, one identity.** Every bead appears in exactly one Beads row and every plan document in exactly one Plans
   row. Marks, jump hints, filters, and cross-pane selection all key off `ArtifactEntryTarget`, and duplicates would
   silently corrupt all four.
2. **No silent filtering.** The Beads pane ships a hide-closed default filter. Whenever a filter is active — default or
   user-typed — the pane info line renders it and the section headers show `matched/total`, exactly as Plans does today.
3. **Keystroke paths never touch disk.** Bead reads, notification-store reads, and gate-bundle probes all happen on
   worker threads behind an mtime-keyed snapshot; detail panels debounce through `DetailPanelDebouncer`; every mutation
   goes through `_submit_tracked_task`. See `sase/memory/tui_perf.md`.
4. **Bead semantics come from the bead domain, never re-derived.** Statuses, types, tiers, glyphs, and colors come from
   `sase.bead.model`, `sase.bead_status_presentation`, and `sase.bead_type_presentation`. Writes go through
   `sase.bead.cli_common.bead_store_mutation` and `sase.bead.mutation_commit`, the same locked commit path the CLI and
   the TaskTriage gate use. Do not add a second bead-mutation implementation in the TUI.
5. **`default_config.yml` is the source of truth for keys.** Every `AppKeymaps` field must have an entry there, and
   every new action needs a matching `_BINDING_META` row and `DEFAULT_BINDINGS` entry.
6. **Footer shows conditional keys only.** Per `src/sase/ace/CLAUDE.md`, a key belongs in the footer if and only if its
   availability depends on the selected entry or transient state; everything else belongs in the help modal.

## Shared vocabulary

A **pane key** identifies a leaf pane that owns rows: `prs`, `commits`, `bugs`, `beads`, `plans`, `chats`, `other`. A
**sub-tab** identifies a top-level Artifacts tab: `prs`, `commits`, `bugs`, `beads`, `files`. `files` is a container and
is never a pane key. All per-pane app state (marks, jump-hint maps, jump history, detail scrolls, copy-mode groups,
footer state) is keyed by pane key.

---

## Sub-tab taxonomy, nested Files container, and keymap surface

**Goal:** the new tab structure exists, cycles, and routes lifecycle correctly, with a placeholder Beads pane and every
new key already declared. No bead or plan behavior changes yet.

### Types and constants — `src/sase/ace/tui/artifact_tabs.py`

```python
ArtifactsSubTab = Literal["prs", "commits", "bugs", "beads", "files"]
FilesSubTab = Literal["plans", "chats", "other"]
ArtifactsPaneKey = Literal["prs", "commits", "bugs", "beads", "plans", "chats", "other"]

DEFAULT_ARTIFACTS_SUBTAB: ArtifactsSubTab = "commits"
ARTIFACTS_SUBTAB_ORDER = ("commits", "beads", "bugs", "prs", "files")

DEFAULT_FILES_SUBTAB: FilesSubTab = "plans"
FILES_SUBTAB_ORDER = ("plans", "chats", "other")

ARTIFACTS_PANE_IDS = {
    "prs": "artifacts-prs-pane",
    "commits": "artifacts-commits-pane",
    "bugs": "artifacts-bugs-pane",
    "beads": "artifacts-beads-pane",
    "files": "artifacts-files-view",      # the nested container
}
FILES_PANE_IDS = {
    "plans": "artifacts-plans-pane",      # ids preserved so existing TCSS keeps applying
    "chats": "artifacts-chats-pane",
    "other": "artifacts-files-pane",
}
ARTIFACTS_ACCENTS: dict[ArtifactsPaneKey | Literal["files"], str]  # includes "beads" and "other"
```

Add a helper `artifacts_pane_key(subtab, files_subtab) -> ArtifactsPaneKey` in the same module (no widget imports, so it
stays usable by bindings and tests). `widgets/artifacts/types.py` re-exports the new names.

### Nested view — `src/sase/ace/tui/widgets/artifacts/files_view.py` (new)

`ArtifactsFilesView(ArtifactsPaneLifecycle, Vertical)` composes a
`PanelTabStrip(FILES_SUBTAB_ORDER, show_numbers=False, uppercase_active=True, id="artifacts-files-subtabs")` above a
`ContentSwitcher(id="artifacts-files-content-switcher")` holding `ArtifactsPlansPane`, `ArtifactsChatsPane`, and
`ArtifactsFilesPane` under their preserved ids.

It mirrors `ArtifactsView`'s contract one level down:

- `current_subtab` / `switch_to(files_subtab)` — deactivate the old child, switch, activate the new child, update the
  strip. Only touch lifecycle when the Artifacts tab **and** the Files sub-tab are both visible.
- `activate_current()` / `deactivate_current()` / `request_active_refresh()` implement `on_activate`, `on_deactivate`,
  and `on_refresh` so `ArtifactsView` can keep treating `files` as one pane.
- `entry_navigator(files_subtab)`, `detail_scroll(files_subtab)`, `set_keymap_registry`, `set_project_scope`,
  `set_project_ref_display` forward to children.
- Handle `PanelTabStrip.TabClicked` by assigning `AceApp.current_files_subtab` (never by mutating local state), so mouse
  and keyboard take the same path.

`ArtifactsView` yields `ArtifactsFilesView` for `files` and `ArtifactPlaceholderPane("beads")` for `beads`, drops the
direct Plans/Chats/Files children, and re-keys `_DETAIL_SCROLL_IDS`, `entry_navigator`, and `detail_scroll` to pane keys
— delegating `plans`/`chats`/`other` to the nested view. Add a `beads` entry to `_PLACEHOLDER_COPY` and remove the
now-unused `plans` entry.

### App state — `src/sase/ace/tui/app.py`, `actions/artifacts.py`, `actions/_state_init.py`

- New reactive `current_files_subtab: reactive[FilesSubTab] = reactive(DEFAULT_FILES_SUBTAB, recompose=False)` and a
  `watch_current_files_subtab` that mirrors `watch_current_artifacts_subtab`: cancel jump mode, call
  `ArtifactsFilesView.switch_to`, then `_sync_active_artifacts_entry_state()`.
- New read-only property `current_artifacts_pane_key` built from `artifacts_pane_key(...)`.
- Re-key `_artifacts_marked_targets`, `_artifacts_jump_history`, `_artifacts_jump_mode_subtab`, and
  `_cancel_artifacts_jump_mode_for_model_change` from sub-tab to pane key. `_clear_all_artifacts_marks` iterates the
  non-PR pane keys. `_artifacts_entry_navigator(pane_key=None)` resolves through the nested view.
- `_non_pr_artifacts_active()` becomes `current_artifacts_pane_key != "prs"` on the Artifacts tab.
- Switching the top-level sub-tab away from and back to `files` must **not** reset `current_files_subtab`.

### `check_action` gating — `src/sase/ace/tui/app.py`

- `PLANS_ARTIFACT_ACTIONS`, `CHATS_ARTIFACT_ACTIONS`, and `FILES_ARTIFACT_ACTIONS` now require
  `current_artifacts_pane_key == "plans" | "chats" | "other"`.
- `BEADS_ARTIFACT_ACTIONS` requires `current_artifacts_pane_key == "beads"`.
- `cycle_files_subtab` / `cycle_files_subtab_reverse` require the Artifacts tab **and**
  `current_artifacts_subtab == "files"`.
- `show_artifacts_beads` joins the numbered-selector guard; `show_artifacts_plans` / `show_artifacts_chats` are removed
  (their sub-tabs no longer exist at the top level).
- The `refresh` suppression set (`y` is a pane copy key, not refresh) becomes the pane keys that bind `y`:
  `{"bugs", "commits", "other", "chats"}`. Chats is missing from that set today even though `chats_copy_path` binds `y`;
  add it here rather than filing it separately, since this predicate is being rewritten anyway.
- `NON_PRS_ARTIFACT_ACTIONS` gains `*BEADS_ARTIFACT_ACTIONS`, `cycle_files_subtab`, `cycle_files_subtab_reverse`, and
  `show_artifacts_beads`.

### Keymaps — declared here, implemented later

Declaring every key in one phase keeps later phases from colliding in `app_keymaps.py`, `default_config.yml`,
`bindings.py`, and `metadata.py`.

Add to `AppKeymaps`, `default_config.yml` (`ace.keymaps.app`), `_BINDING_META`, and `DEFAULT_BINDINGS`:

| Field                             | Default             | Description            |
| --------------------------------- | ------------------- | ---------------------- |
| `cycle_files_subtab`              | `right_parenthesis` | Next Files View        |
| `cycle_files_subtab_reverse`      | `left_parenthesis`  | Previous Files View    |
| `beads_next` / `beads_prev`       | `j` / `k`           | Next / Previous Bead   |
| `beads_view_selected`             | `enter`             | View Bead              |
| `beads_filters`                   | `f`                 | Bead Filters           |
| `beads_expand` / `beads_collapse` | `l` / `h`           | Expand / Collapse Epic |
| `beads_cycle_status`              | `s`                 | Cycle Bead Status      |
| `beads_edit`                      | `e`                 | Edit Bead              |
| `beads_add_note`                  | `N`                 | Add Bead Note          |
| `beads_create`                    | `n`                 | Create Task Bead       |
| `beads_close`                     | `c`                 | Close / Reopen Bead    |
| `beads_launch_work`               | `w`                 | Launch Bead Work       |
| `beads_open_bug`                  | `o`                 | Open Linked Bug        |
| `beads_open_plan`                 | `L`                 | Go to Linked Plan      |
| `beads_refresh`                   | `R`                 | Refresh Beads          |
| `plans_open_bead`                 | `L`                 | Go to Linked Bead      |

`L` is deliberately the same key on both panes: one "link" mnemonic, and the visible pane decides the direction.

Rename the bead-only Plans fields to their Beads counterparts and delete them from `AppKeymaps`, `default_config.yml`,
`_BINDING_META`, and `DEFAULT_BINDINGS`: `plans_expand`, `plans_collapse`, `plans_cycle_status`, `plans_edit_bead`,
`plans_launch_epic`, `plans_open_bug`. Add all six ids to `_RETIRED_APP_KEYS` in `keymaps/registry.py` so an existing
user override is dropped quietly instead of logging an unknown-action warning. `plans_approve` and `plans_reject` stay
on Plans — a proposal is a plan document.

`key_validation.py` has no parenthesis entries, so `(` and `)` are currently rejected as invalid keys. Add
`"left_parenthesis": "("` and `"right_parenthesis": ")"` to `_KEY_DISPLAY`; those are the exact names Textual's
`_character_to_key` produces for those glyphs.

Create `src/sase/ace/tui/actions/artifacts_beads.py` with `BEADS_ARTIFACT_ACTIONS` and an `ArtifactsBeadsActionsMixin`
whose action methods all exist from this phase and resolve the pane through `_beads_pane()`. Against the placeholder
pane every lookup returns `None` and the action is inert, so no key can raise; later phases fill in the bodies. Mix it
into `ArtifactsMixin` alongside the plans/chats/files mixins.

Also add `cycle_files_subtab` actions to `ArtifactsMixin` (`_cycle_files_subtab(step)` with wraparound, guarded on the
Artifacts tab and the Files sub-tab), and `action_show_artifacts_beads`.

### Other call sites to re-key

`copy_targets.py` and `actions/clipboard/*` (add an `artifacts_beads` group, keyed by pane key),
`_keybinding_modes. update_copy_bindings(..., artifacts_subtab=...)` → pane key, `KeybindingFooter.show_artifacts_pane`,
`actions/marking.py`, `actions/base.py`, `modals/jump_all_modal.py`, `modals/command_palette_modal.py`,
`widgets/changespec_onboarding.py`, `ace/testing/ace_page.py` (expose `files_subtab` next to `artifacts_subtab`), and
`styles.tcss` (nested strip + `#artifacts-beads-pane`; reuse the Plans pane's layout rules).

### Tests

New `tests/ace/tui/test_artifacts_files_subtabs.py`: `(`/`)` cycle with wraparound; both keys are inert on every other
sub-tab; switching away from and back to Files restores the remembered sub-sub-tab; lifecycle counters show exactly one
child activated at a time; marks and jump history are isolated per pane key. Update `test_artifacts_scaffold.py`,
`test_artifacts_marking.py`, `test_artifacts_copy_mode.py`, `test_artifacts_list_navigation.py`, and the keymap registry
tests for the new fields, the retired ids, and `(`/`)` display.

---

## Read-only Beads pane

**Goal:** the Beads sub-tab lists and describes beads. No writes yet.

### Data — `widgets/artifacts/beads_data*.py`

`load_beads_snapshot(project, *, previous=None, force=False) -> BeadsSnapshot` runs on a worker thread and follows the
Plans loader's structure (`plans_data.py`): resolve projects with the existing project-record facade, read each store
through `sase.core.bead_read_facade`, and short-circuit on an unchanged mtime-based `source_key`.

`BeadsSnapshot` (frozen dataclass) carries: `project`, `projects`, `display_names`, `beads_dirs`, `workspace_dirs`,
`tasks`, `epics`, `phases_by_epic`, `ready_ids`, `blocked_ids`, `plan_links`
(`(project, bead_id) -> resolved plan path or ""`), `triage_gates` (`(project, bead_id) -> PendingTriage`),
`source_key`, and `errors`.

- `plan_links` resolves `Issue.design` through `sase.plan_documents.resolve_plan_path`, reusing the workspace/plans-root
  pairing already implemented in `plans_data_documents.py`. Resolve paths only; do not read plan bodies here.
- `triage_gates` comes from `sase.notifications.store.load_notifications(include_dismissed=False)` filtered to the
  `TaskTriage` action, resolved with `sase.notification_gates.paths.resolve_notification_bundle`, and matched to a bead
  by reading `payload.bead_id` / `payload.project` from `request.json`. Match on the payload, never by parsing the
  request-id string. `PendingTriage` records the notification id, the request id, and the gate's created timestamp.
- Include closed beads in the snapshot. Hiding them is a filter concern (next phase), not a load concern.

### Rows — `beads_list.py`, `beads_rendering.py`

Two sections, using `_section_option`-style headers with the `#D787FF` accent:

- `── Tasks (N) ──` with a `· ✦ N awaiting triage` suffix when any task has a pending gate. Order: pending triage, then
  `ready`, `in_progress`, `claimed`, `open`, `closed`; ties broken by recency then id.
- `── Epics (N) ──` with the existing blocked/ready/launched glyph legend, each epic expandable (`l`/`h`) to its phase
  beads in hierarchical id order. Expansion state lives in the pane, keyed by `(project, bead_id)`.

Row anatomy, left to right: mark glyph, jump hint, type glyph from `bead_type_presentation`, a `✦` chip when a triage
gate is pending, a `▤` chip when the bead links a plan document, the bead id, the title, the status glyph and label from
`bead_status_presentation`, then dim metadata (assignee, age, project badge when browsing all projects). Epics
additionally show `phases closed/total` and the blocked/ready/launched state glyph. Phase rows are indented under their
epic exactly as the Plans pane renders them today.

Row identity: `("bead", project, kind, bead_id)` where kind is `task | epic | phase`. `row_option_id` namespaces by
project when the snapshot spans all projects, mirroring `plans_list.row_option_id`.

### Detail — `beads_detail.py`

Move the bead-specific helpers out of `plans_detail.py` (`bead_properties_header`, `bead_body_markdown`,
`_dependencies_text`, `_epic_phase_sizes`, `_dependency_state`, `_status_chip`, `_readiness_chip`, and the chip helpers
they need) into `beads_detail.py`, and extend them. `plans_focus` deletes whatever is left unused on the plans side, so
this phase should copy-then-extend rather than edit the plans module in place, to keep the two phases from fighting over
one file.

Properties header: id, type chip, status chip, tier or size, resolution and close reason when closed, assignee, owner,
model, created and updated timestamps, project, dependencies with their resolved state, `refs`, ChangeSpec name and bug
id, and the linked plan path (or `design` reference plus a "cannot resolve" note when resolution fails). Body:
description then notes, rendered as Markdown. When a triage gate is pending, render a callout above the body naming the
bead and the keys that resolve it.

### Pane, navigation, options

`ArtifactsBeadsPane` mirrors `ArtifactsPlansPane`: `BeadFilterBar` placeholder slot, `#beads-info` scope line, a
`#beads-list-panel` (`Beads` border title) beside a `#beads-detail-panel` with `#beads-detail-scroll`, and a
`#beads-hints` line. `beads_navigation.py` and `beads_options.py` port the plans mixins: `_syncing_options` guard around
`replace_options`, preferred-target preservation across reloads, `entry_targets` / `selected_entry_target` /
`select_entry_target` / `apply_entry_jump_hints` / `clear_entry_jump_hints` / `apply_entry_marks`, and
`_cancel_artifacts_jump_mode_for_model_change("beads")` on every model swap.

Implement `action_beads_next/prev/view_selected/expand/collapse/refresh` in `actions/artifacts_beads.py`.
`beads_view_selected` opens `PreviewPanelModal` with the bead's rendered detail, matching Plans' `enter`.

Loading, empty, and error states match the other panes: a `Loading beads…` disabled row while the first load runs, a
distinct empty state that names the filter key when a filter is active, and per-project error lines from
`snapshot.errors`.

### Tests

`test_artifacts_beads_loading.py` (source-key reuse, force reload, per-project errors, triage-gate matching by payload),
`test_artifacts_beads_rendering.py` (section ordering, chips, one row per bead, expansion), and
`test_artifacts_beads_navigation.py` (selection preserved across reloads, jump targets, marks).

---

## Bead filter query and inline filter bar

**Goal:** beads are filterable by everything that matters, with a visible default.

Add `src/sase/bead/filter_query.py` modeled on `sase/plan_search/filter_query.py`: `BeadFilterValues`,
`parse_bead_filter_query`, `BeadFilterQueryError`, `to_query_tokens`, `to_query_string`, and `bead_completion_context`.
It lives next to the bead domain rather than in the TUI because `sase bead list` and any future frontend should be able
to speak the same query; the vocabularies must be derived from `sase.bead.model` and the shared presentation modules,
never re-listed as literals.

Supported terms (all repeatable, all negatable with a leading `-`, plus free text ANDed across the haystack):

| Key                          | Values                                                               |
| ---------------------------- | -------------------------------------------------------------------- |
| `type`                       | `plan`, `phase`, `task`                                              |
| `tier`                       | `plan`, `epic`                                                       |
| `status`                     | the five real statuses, plus derived `blocked`, `launched`, `triage` |
| `size`                       | `xsmall`…`xlarge`                                                    |
| `project`                    | project key or display name                                          |
| `assignee`, `owner`, `model` | free string                                                          |
| `has`                        | `plan`, `bug`, `deps`, `notes`, `triage`                             |
| `since`, `until`             | `Nh`/`Nd`/`Nw`, `today`, `yesterday`, `YYYY-MM-DD`                   |

`beads_filtering.py` builds a prefolded `BeadFilterIndex` from a snapshot (one record per bead, folded haystack over id,
title, description, notes, design, refs, assignee, owner, model, ChangeSpec fields, plus the label sets) and compiles
values into a cheap predicate — the same shape as `plans_filtering.py`. Rebuild the index only when
`snapshot.source_key` changes.

`beads_filter_bar.py` subclasses the shared `FilterBar` with `#D787FF` accent, key and value completion, and observed
value sources (projects, assignees, owners, models) refreshed from the loaded snapshot. `beads_filter_session.py` ports
the Plans session mixin: `f` opens the bar, the bar is visible only during an edit session, live filtering as you type,
submit commits, escape dismisses without discarding the committed query.

The pane opens with a committed default of `-status:closed`. Because the default is a normal committed query, the info
line already renders it and `f` already edits or clears it — there is no hidden state. Section headers switch to
`matched/total` whenever a filter is active.

Tests: `test_artifacts_beads_filtering.py` for parsing (including negation and bad-token errors), each term's matching
semantics, derived statuses, the hide-closed default being visible and clearable, and index reuse across reloads.

---

## Bead mutations, close-with-reason, and triage-gate settlement

**Goal:** the Beads pane is a full bead workbench, and closing a triaged task bead clears its notification immediately.

### Backend helpers

In `src/sase/bead/task_gate.py`, add two public helpers so the TUI never re-implements gate lookup:

- `find_pending_task_triage(project, bead_id) -> str | None` — scan pending `task_triage` bundles under
  `interaction_requests_dir()`, skip any with a response or cancellation, and return the request id whose
  `payload.bead_id` and `payload.project` match. Match on payload, not on the request-id format.
- `cancel_task_triage(project, bead_id, *, reason, source="ace") -> bool` — resolve through the helper above and call
  `cancel_gate`, which settles the notification. Treat `already_answered` as a benign `False`.

In `src/sase/bead/task_launch.py`, add `submit_task_launch_for_project(project, bead_id, *, feedback=None, origin=None)`
that resolves the project's launch cwd and delegates to `submit_task_launch_task`. Refactor
`task_gate._resolve_task_triage_project_cwd` to reuse it so the gate path and the TUI path cannot drift.

### TUI mutation path

Every write in `actions/artifacts_beads.py` follows the same shape: optimistic UI where safe → `_submit_tracked_task`
with a `beads:<op>:<project>:<bead_id>` dedup key → a worker body that opens
`bead_store_mutation(auto_commit_bead_store, cwd=<project workspace>)`, performs the mutation, and commits with a
message from `sase.bead.mutation_commit` → a UI-thread `on_complete` that refreshes the pane and toasts the outcome.
Delete `_update_scoped_bead`, `_commit_scoped_bead_store`, and `_launch_scoped_epic` from `actions/artifacts_plans.py`
in favor of that shared path.

### Actions

- **`s` — cycle status.** Port `_next_plans_bead_status` (type-aware; only task beads reach `ready`).
- **`e` — edit.** Replace `modals/bead_edit_modal.py` with `modals/bead_editor_modal.py`: a field-driven editor over
  title, description, notes, status, assignee, owner, model, size, `design`, ChangeSpec name, and bug id. Fields render
  only when valid for the bead's type — `Issue.validate` rejects tier on phases and tasks, size on plans, and `ready` on
  non-tasks, so the modal must not offer them. Submit only changed fields, through a single `BeadProject.update`.
  Editing dependencies is out of scope for this epic; the detail panel keeps showing them read-only.
- **`N` — add note.** `modals/bead_note_modal.py`, a small TextArea that appends through the note path (never
  `update --notes`, which replaces the whole field).
- **`n` — create task bead.** `modals/bead_create_modal.py` with title, description, size, and a ready/draft toggle,
  scoped to the current project (blocked with a toast when browsing all projects). _This is a designer's addition, not
  an explicit request — it is the natural counterpart of the Bugs pane's `c`, and it is safe to cut at approval time._
- **`w` — launch work.** Epic beads take the existing `launch_epic_bead_work` path behind a confirm; task beads that are
  `ready` or `open` take `submit_task_launch_for_project`. Phase beads toast "phases launch with their epic". A
  successful task launch also calls `cancel_task_triage(..., reason="launched_from_ace")` so the pending notification
  does not outlive the decision.
- **`o` — open linked bug.** Port `action_plans_open_bug` unchanged apart from resolving the epic through the beads
  snapshot.
- **`c` — close or reopen.**
  - Closed bead → confirm, then `BeadProject.open`, which also reopens closed ancestors. Say so in the confirm text.
  - Open bead → `modals/bead_close_modal.py`: resolution (`done` default, `canceled`, `superseded`), a **required**
    reason, an optional note, and a force checkbox that is disabled unless the bead has unclosed descendants. Enabling
    force must also force a non-`done` resolution, matching `sase bead close --force`. List the blocking descendants
    inline so the user sees why force is offered.
  - Closing an epic plan bead or a phase bead an agent may be working shows an extra warning line in the modal. Warn, do
    not block: the human owner is allowed to override.
  - On success, if the bead is a task, call `cancel_task_triage(..., reason="closed_from_ace")`. Refresh the pane and
    the notification indicator.
  - Surface a rejected close verbatim (for example the unclosed-descendant error naming each bead) as an error toast;
    never swallow it.

### Tests

`test_artifacts_beads_mutations.py` covers each action's tracked-task submission, dedup, field diffing, and the
type-aware status cycle. `test_bead_task_triage_lookup.py` covers payload matching, skipping answered and cancelled
bundles, and the benign `already_answered` path. `test_bead_close_modal.py` covers required reason, the force/resolution
coupling, and the descendant preview. Add an end-to-end test that closing a ready task bead with a pending gate writes
the close event **and** cancels the gate.

---

## Plans sub-sub-tab dedicated to plan documents

**Goal:** every row on Plans is a plan document.

### Sections

1. `── Proposals (N) ──` — pending plan approvals, unchanged loading path (`load_proposals`), with `A` approve and `X`
   reject.
2. `── Active plans (N) ──` — committed plan documents currently linked from a non-closed bead, deduplicated by resolved
   path. Each row shows the plan title, tier chip, owning bead id and status chip, age, and project badge.
3. `── Archive (N) ──` — every other committed plan document, including plans of closed beads and unlinked plans. Keeps
   today's bounded first pass, the lazy deep-archive browse, and the truncation notice.

A plan document appears in exactly one section. Rows that have an owning bead expose `L`; rows that do not, do not.

### Loading

`plans_data.py` stops building epic and phase row models. It still reads beads, but only to project
`(project, bead_id) -> resolved plan path` plus the owning bead's id and status, so section 2 can be built and `L` can
resolve. Factor that projection into `widgets/artifacts/bead_plan_links.py`, shared by both panes, so the two snapshots
cannot disagree about what "linked" means. Drop `phases_by_epic`, `ready_ids`, `blocked_ids`, and the epic/phase
document loading from `PlansSnapshot`; keep `linked_plan_documents` only for the plan bodies the detail panel renders.

### Filters, detail, copy

- `kind:` becomes `proposal | active | archive`, plus the document-sidecar roles (`plans`, `research`) already folded in
  as kind labels. Remove `task`, `epic`, and `phase` from the vocabulary and completions.
- `status:` comes from plan frontmatter plus `proposed`; `tier:` stays.
- `plans_detail.py` keeps only the proposal and document helpers; the bead helpers now live in `beads_detail.py`. Delete
  whatever is left unused rather than leaving it for Symvision to flag.
- Copy targets: `artifacts_plans` keeps reference, link, design, path, title, body, JSON, handoff, and snapshot, and
  gains a bead-id target for the owning bead. Add the `artifacts_beads` group: id, reference, link, title, body
  (description plus notes), design reference, JSON, handoff, snapshot.

### Tests

Update `test_artifacts_plans_data.py`, `_filtering.py`, `_rendering.py`, `_interactions.py`, `_loading.py`, and
`_linked_documents.py` to the new model, and add `test_artifacts_plans_sections.py` for section assignment, dedup
between Active and Archive, and the removal of bead rows.

---

## Bidirectional bead and plan jumps with conditional footer hints

**Goal:** `L` moves between a bead and its plan document, reliably, from either side.

Add to `ArtifactsMixin`:

- `_request_artifacts_entry(pane_key, target)` — switch `current_artifacts_subtab` (and `current_files_subtab` when the
  target is a Files child), then either select the target immediately or hand it to the target pane as a pending
  selection.
- Each entry-navigator pane gains `request_entry_target(target)`: select now if the target is currently visible,
  otherwise store it, apply it after the next successful `_refresh_options`, and clear it on project-scope change. After
  the pane finishes a load without finding the target, warn once and clear — a pending selection must never linger and
  hijack a later, unrelated load.
- `action_beads_open_plan` resolves the selected bead's plan path from `snapshot.plan_links` and jumps to
  `("plan", project, "active"|"archive", path)`. `action_plans_open_bead` resolves the owning bead from
  `bead_plan_links` and jumps to `("bead", project, kind, bead_id)`. Both toast a specific reason when there is no
  counterpart ("This bead links no plan file", "No bead links this plan file") rather than doing nothing.
- If a filter on the destination pane hides the target row, clear that pane's committed filter for the jump and toast
  that it was cleared. Landing on a pane that appears not to contain the row is the failure mode to design out.

Footer: extend `KeybindingFooter.show_artifacts_pane` to accept conditional entries supplied by the active pane, and
have the Beads and Plans panes report the ones that depend on the selected row — `L` jump, `w` launch, `c` close/reopen,
`o` open bug. This follows the repo's footer rule: these keys are sometimes available and sometimes not, so they belong
in the footer; everything unconditional stays in the help modal.

Tests: `test_artifacts_bead_plan_jump.py` for both directions, the unloaded-pane handoff, the filtered-destination case,
missing-counterpart toasts, and the conditional footer entries.

---

## Help, docs, onboarding, and visual snapshots

- **Help modal** (`modals/help_modal/changespecs_bindings.py`): rename the sub-tab section for the new order, add `(` /
  `)`, add a Beads section covering all fifteen bead keys, trim the Plans section to plan actions, retitle the Files
  section as Other, and add the `artifacts_beads` copy-mode rows. Keep the 57-character box width and the 32-character
  description cap from `src/sase/ace/CLAUDE.md`, and re-check `COLUMN_SPLITS["changespecs"]` still balances the two
  columns.
- **`docs/ace.md`**: rewrite the sub-tab list (six sub-tabs become five plus three nested views), the `1`–`5` and `(` /
  `)` navigation text, the copy-mode table, the marks and filtering sections, the "Standalone Tasks in the Plans Pane"
  section (now the Beads pane), and add a Beads Pane section documenting triage, close-with-reason, launch, and the bead
  filter vocabulary.
- **Onboarding and empty states**: `widgets/changespec_onboarding.py` and `widgets/agent_onboarding.py` copy; the Beads
  pane's first-run empty state should point at `sase bead create` and the triage flow.
- **Visual snapshots**: add `test_ace_png_snapshots_artifacts_beads.py` and an empty-state variant, add a Files-view
  snapshot showing the nested strip, and re-record the Plans goldens. Use `--sase-update-visual-snapshots` only to
  accept these intentional changes, and inspect `.pytest_cache/sase-visual/` artifacts before accepting.
- Confirm the command palette and `jump_all_modal` list the new sub-tabs, and that `sase ace --profile` startup and the
  j/k bench (`bench_artifacts_jk.py`) show no regression on the Beads pane.

## Out of scope

- Editing bead dependencies from the TUI (`sase bead dep add|rm`). The detail panel shows them read-only.
- Bulk operations over marked beads. Marks work, but multi-bead close/update is a follow-up.
- Any change to the Bugs, Commits, PRs, or Chats panes beyond re-keying and gating.
- Promoting the bead filter query into the Rust core. It is added in Python next to the bead domain, matching the
  existing `plan_search` and `vcs_log` filter modules, and is a candidate for later promotion.

## Verification for every phase

Run `just install` first (workspaces may have drifted dependencies), then `just check` before finishing. Phase workers
must not create beads: record any discovered follow-up as a `PROPOSED FOLLOW-UP: <summary — detail>` note on their own
phase bead instead. Do not edit any file under `sase/memory/`, `AGENTS.md`, or the generated provider instruction shims.
