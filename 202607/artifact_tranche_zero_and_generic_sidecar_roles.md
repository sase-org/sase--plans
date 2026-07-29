---
tier: epic
title: Artifact tranche-zero defects and generic document-sidecar roles
goal: 'Copy mode and marks work on every Artifacts sub-tab, artifact-file path copies
  are unambiguous, the text-artifact fallback viewer is safe without `bat`, and no
  SASE code names the `research` sidecar: every user-defined document sidecar gets
  the behavior `research` has today.

  '
phases:
- id: copy-mode
  title: Copy mode on every Artifacts sub-tab
  depends_on: []
  size: medium
  description: 'copy-mode: admit `copy_tab_content` on non-PR Artifacts sub-tabs,
    dispatch the second copy key on the active sub-tab before falling through to the
    tab id, add per-sub-tab copy key blocks and real copy menus for Commits, Plans,
    Chats, and Bugs, and make the COPY footer and its restore path sub-tab aware.

    '
- id: subtab-marks
  title: Marks on non-PR Artifacts sub-tabs
  depends_on:
  - copy-mode
  size: medium
  description: 'subtab-marks: route `toggle_mark`/`clear_marks` on the Artifacts tab
    through the active sub-tab instead of the PR list, store marks per sub-tab keyed
    on the pane''s existing stable entry target, render the mark glyph, and let the
    sub-tab copy menus copy a marked set.

    '
- id: path-copy
  title: Anchored artifact-file path copy
  depends_on: []
  size: small
  description: 'path-copy: stop emitting bare workspace-relative paths from the artifact-file
    modal''s path copy, prefer the always-present stored path, anchor any source-path
    answer, and say in the toast which of the two was copied.

    '
- id: text-fallback
  title: Safe text-artifact fallback viewer
  depends_on: []
  size: small
  description: 'text-fallback: replace the bare `cat` fallback in the artifact text
    viewer with a bounded, binary-detecting, control-sequence-neutralizing reader
    and give the `bat` branch''s argument-boundary hygiene to the fallback too.

    '
- id: store-roles
  title: Generic sidecar roles in the SDD store
  depends_on: []
  size: medium
  description: 'store-roles: replace the store record''s fixed plans/research/beads
    fields with a role-keyed sidecar map, make `SddStore` resolve roots for any configured
    role, and stop requiring a sidecar literally named `research` before a sidecar-storage
    record can be written or considered materialized.

    '
- id: role-consumers
  title: Route hardcoded role tuples through the role registry
  depends_on:
  - store-roles
  size: medium
  description: 'role-consumers: replace every remaining literal `("plans", "research",
    "beads")` tuple and `research`-named branch across repo inventory, linked-repo
    paths, `sase repo path`, doctor, commit finalization, SDD path helpers, agent
    env, and the ACE file panel with the generic role registry, and document the role
    model.

    '
- id: core-corpora
  title: Rust core document corpora for plan discovery
  depends_on: []
  size: medium
  description: 'core-corpora: teach the Rust plan reader to scan caller-supplied `(root,
    kind)` document corpora instead of only its hardcoded `plans`/`research` sub-directory
    pair, and expose that through `search_plans` and the PyO3 `plan_search` binding
    without changing default behavior.

    '
- id: plan-search-roles
  title: Plan search and CLI over document-sidecar roles
  depends_on:
  - store-roles
  - core-corpora
  size: medium
  description: 'plan-search-roles: make the Python plan-search facade discover every
    configured document-sidecar root and pass them to the core as labeled corpora,
    and make `sase plan search --kind` and its result styling accept any discovered
    role rather than a fixed four-value choice list.

    '
- id: ace-documents
  title: ACE Plans pane over every document sidecar
  depends_on:
  - plan-search-roles
  size: medium
  description: 'ace-documents: browse and search documents from every configured document
    sidecar in the Artifacts Plans pane instead of only the plans sidecar, add a kind
    facet to its filter bar, show each row''s kind, and update the ACE docs and help
    popup.

    '
create_time: 2026-07-29 10:30:48
status: done
bead_id: sase-as
---

- **PROMPT:**
  [202607/prompts/artifact_tranche_zero_and_generic_sidecar_roles.md](prompts/artifact_tranche_zero_and_generic_sidecar_roles.md)
- **BEAD:** [sase-as](https://github.com/sase-org/sase--beads/blob/main/pages/sase-as/README.md)

# Plan: Artifact tranche-zero defects and generic document-sidecar roles

## Context

A consolidated research report — `202607/artifact_refs_and_inspector/artifact_refs_and_inspector.md` in the `research`
sidecar (open it with the `/sase_repo` skill) — ranked its recommendations and put four small, independently shippable
defect fixes first, under _"Tranche zero"_:

1. Restore copy mode and marks on every Artifacts sub-tab.
2. Fix the artifact-file modal's path copy (`Y`).
3. Stop the raw `cat` fallback in the artifact text viewer.
4. "Surface `research` in Plans" — described there as a two-line change adding `"research"` to a hardcoded kinds tuple.

Items 1–3 are taken as written. Item 4 is **not**. The `<project>--research` sidecar is a _user-defined_ sidecar repo,
declared like any other entry under `repos.sidecar` in a project's `sase/sase.yml`. Nothing about the name `research` is
intrinsic to SASE, so hardcoding it a second time in the ACE Plans pane would deepen an existing mistake rather than fix
one. This epic therefore replaces item 4 with the real defect it exposes: **SASE code names the `research` sidecar in
~30 modules, and every behavior it grants that sidecar is unavailable to any other user-defined sidecar.**

### The generic model this epic introduces

A project's sidecars come from `repos.sidecar` in its `sase/sase.yml`. Three role names are reserved and have
SASE-defined meaning:

| Role     | Meaning                                                                                              |
| :------- | :--------------------------------------------------------------------------------------------------- |
| `plans`  | Canonical plan store: month-sharded plan documents plus nested prompt snapshots, `tier: tale\|epic`. |
| `beads`  | Bead event store and projections; not a document corpus.                                             |
| `agents` | Hidden agent-data sidecar; never exposed to launched agents or cloned into workspaces.               |

**Every other configured sidecar role is a _document sidecar_**: a repo of month-sharded markdown documents whose
plan-search `kind` label is simply the role name. `research` becomes one instance of that class — the one
`sase repo init` happens to seed by default — and stops being a name the code knows.

Two lines are drawn deliberately:

- **Behavior must be role-generic.** Store roots, materialization, `sase repo path`, doctor checks, commit routing,
  agent env vars, plan discovery, and the ACE Plans pane must work for a sidecar named `designs` exactly as they work
  for one named `research`.
- **Shipped presentation presets may stay name-keyed.** `src/sase/sdd/templates/sidecar-research-README.md` and
  `research-directory-map.png` are hand-authored guide content for one role. Keep them as an optional preset looked up
  by role name, with the existing generic README fallback (`_generic_sidecar_readme`) for roles that have no preset. Do
  not delete a real user's guide content in the name of purity.

### Verified current state

Everything below was checked against the working tree; cite it rather than re-deriving it.

**Copy mode and marks are denied on 4 of 5 sub-tabs.** `%` is bound to `copy_tab_content` and `m` to `toggle_mark`
(`src/sase/ace/tui/bindings.py:189,169`), but neither is in `NON_PRS_ARTIFACT_ACTIONS`
(`src/sase/ace/tui/actions/artifacts.py:39`), and `check_action` denies by default for any action outside that
allow-list while a non-PR sub-tab is active (`src/sase/ace/tui/app.py:371-377`). Commits, Plans, Chats, and Bugs have no
copy mode at all — not even `%s`, the tab-independent tmux pane snapshot.

**Lifting the gate alone is wrong, twice over.** The Artifacts tab's internal id _is_ `"changespecs"`
(`src/sase/ace/tui/tab_order.py:18`). `_handle_copy_key` branches only on `self.current_tab`
(`src/sase/ace/tui/actions/clipboard/_core.py:47-53`), so `%n` on the Chats sub-tab would route into
`_handle_changespecs_copy_key` and copy the selected **PR's** name. Symmetrically, `action_toggle_mark`
(`src/sase/ace/tui/actions/marking.py:26-33`) dispatches on `current_tab` only, so an un-gated `m` on Chats would mark a
**ChangeSpec**. Both need sub-tab dispatch, not just an allow-list entry.

**The COPY footer has no sub-tab path.** `update_copy_bindings` accepts only `"changespecs"`/`"agents"`/`"axe"`
(`src/sase/ace/tui/widgets/_keybinding_modes.py:390`). The restore path is worse: `_handle_copy_key` finishes by calling
`_refresh_current_tab`, which routes the Artifacts tab into `_refresh_display`, and both of that method's footer
branches early-return when `current_artifacts_subtab != "prs"`
(`src/sase/ace/tui/actions/changespec/_display.py:255,285`). The footer would stay stuck in COPY. The correct restore
for a non-PR sub-tab is `_sync_active_artifacts_entry_state` (`src/sase/ace/tui/actions/artifacts.py:204`).

**`Y` commonly copies a path that resolves to the wrong file.** `_artifact_file_clipboard_path`
(`src/sase/ace/tui/modals/artifact_files_modal.py:107-131`) tries workspace-relative on the **stored** path first, but
stored paths live under `~/.sase/artifacts/` and are never inside a workspace, so that branch effectively never fires.
It falls through to the **`source_path`** branch, where `_workspace_relative_path` returns `relative.as_posix()` — a
bare relative path with no workspace identity, e.g. `tests/ace/tui/visual/snapshots/png/foo.png`. Scanning the 3,985
index records, the stored `path` exists 100% of the time while 31% of `source_path` values are already gone; and
`sase_<N>` workspaces are recycled and reused, so the same bare relative path can later name a _different_ file.

**The text viewer's fallback is a bare `cat`.** `artifact_text_viewer_command`
(`src/sase/ace/tui/graphics/_viewer_loop_media.py:30-43`) uses
`bat --paging=always --color=always --decorations=always -- <path>` when available — already argument-boundary safe, and
`bat` refuses to print binary to a terminal. The fallback is `["cat", str(expanded)]`: no `--`, no binary detection, no
control-sequence neutralization. The exposure is exactly "`bat` not installed".

**`research` is load-bearing in the SDD store, and that is a live bug.** `SddStoreRecord` has fixed
`plans`/`research`/`beads` fields and a `sidecar_for_kind` that raises for anything else
(`src/sase/sdd/_store_types.py:49,65`); `SddStore` has `research_dir`/`research_remote_url` and a `kind_root` that
raises for unknown roles (`:81,99`). Worse, `_write_compatibility_store` returns `(None, None)` when `research is None`
(`src/sase/sdd/_sidecar_init.py:350`) and `_is_materialized_record` requires `record.research is not None`
(`src/sase/sdd/_store_records.py:207`). **A project that disables or renames the `research` sidecar cannot write a
sidecar-storage record at all.**

**But the generic machinery already exists underneath.** `initialize_sidecars` already accepts arbitrary
`SidecarInitSpec` roles from config, resolves each clone root, creates/adopts/seeds/pushes each one, and
`expected_sdd_sidecar_files` already falls back to `_generic_sidecar_readme` for roles outside its guide set
(`src/sase/sdd/_init_files.py:87-93`). The store record's JSON is _already_ a role-keyed `sidecars` map
(`src/sase/sdd/_store_records.py:288`); only the Python model narrows it to three fields. The choke point is small.

**Plan discovery is hardcoded on both sides.** The Rust core scans
`REPO_PLAN_KINDS = [("plans","tale"), ("research","research")]` as sub-directories of one SDD root
(`crates/sase_core/src/plan/read.rs:36`). The Python facade resolves a single repo root and, in split-sidecar mode,
takes the `_is_flat_plans_root` path that classifies everything as `tale`/`epic`
(`src/sase/plan_search/facade.py:150-225`). `sase plan search --kind` is a fixed
`choices=["tale","epic","prompt","research"]` (`src/sase/main/parser_plan.py:341`) and `_KIND_STYLES` hardcodes a
`research` color (`src/sase/main/plan_search_render.py:53`).

**So the research bullet's "two lines and a chip" would not have worked.** `load_project_archive` and
`load_deep_plan_archive` (`src/sase/ace/tui/widgets/artifacts/plans_data_sources.py:170,185`) search with
`repo_root=plans_root`, where `plans_root` is `resolve_sdd_kind_dir(workspace_dir, 1, "plans")` — the _plans_ sidecar.
Research documents live in a different repo entirely. Adding `"research"` to the kinds tuple would have matched nothing.

### Ground rules for every phase

- Run `just install` first (workspace virtualenvs are ephemeral and may be stale), then `just check` before finishing.
  See the repo's build-and-run memory.
- Per the Rust-core-boundary memory: reference parsing, document discovery, and classification are core backend logic
  and belong in `sase-core`. Keymaps, footers, modal bindings, panes, and rendering stay in this repo. Open the
  `sase-core` linked repo with the `/sase_repo` skill; never clone or web-fetch it.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims. This plan does not authorize it.
- Keep user-facing project labels rendering `PROJECT_NAME`, never ProjectSpec keys.
- New or changed config keys require a matching update to `src/sase/config/sase.schema.json`; new keymap defaults
  require a matching update to `src/sase/default_config.yml`.

---

## Copy mode on every Artifacts sub-tab

Give Commits, Plans, Chats, and Bugs a real copy menu instead of the single hardcoded target each has today, without
letting any of it leak into the hidden PR list.

**Admit the action.** Add `copy_tab_content` to `NON_PRS_ARTIFACT_ACTIONS` (`src/sase/ace/tui/actions/artifacts.py:39`).
Leave `toggle_mark` to the next phase.

**Dispatch on the sub-tab first.** In `_handle_copy_key` (`src/sase/ace/tui/actions/clipboard/_core.py:31`), when
`current_tab == ARTIFACTS_TAB and current_artifacts_subtab != "prs"`, route to a new per-sub-tab handler _before_ the
existing `current_tab` chain. Put the new handlers in a new module beside the existing ones (`_axe.py`, `_agents.py`,
`_changespec.py`) rather than growing `_core.py`. `action_start_copy_mode`'s `"No ChangeSpec to copy"` guard must not
fire on a non-PR sub-tab.

**Keymap blocks.** Add nested key groups `artifacts_commits`, `artifacts_plans`, `artifacts_chats`, `artifacts_bugs`
under `copy_mode.keys`, in both `CopyModeKeymaps` (`src/sase/ace/tui/keymaps/mode_keymaps.py:51`) and
`src/sase/default_config.yml` (the `keymaps.modes.copy_mode` block near line 361) — the two must stay in sync, per the
keymap-sync gotcha. The config schema already permits arbitrary nested key groups
(`src/sase/config/sase.schema.json:807-835`), so no schema change is needed; confirm that before assuming it.

Proposed targets (every sub-tab keeps `s` = tmux pane snapshot, which is tab-independent and already works everywhere
else):

| Sub-tab | Keys                                                                                   |
| :------ | :------------------------------------------------------------------------------------- |
| Commits | `%` full SHA · `m` message · `r` `repo@sha` · `p` linked plan reference · `s` snapshot |
| Plans   | `p` path · `t` title · `b` body · `s` snapshot                                         |
| Chats   | `p` path · `a` agent name · `t` transcript contents · `s` snapshot                     |
| Bugs    | `b` `#N` · `u` url · `t` title · `p` agent-ready prompt · `s` snapshot                 |

Reuse the existing single-target implementations where they exist (`commits_copy_sha`, `chats_copy_path`, `copy_bug`)
rather than duplicating their resolution logic; the `CommitViewModal` already loads a commit's linked plan, so the
Commits `p` target should resolve the same way that modal does. Chats' `p` target must agree with the artifact-file
modal's path answer after the `path-copy` phase — one question, one answer.

**Footer.** Extend `update_copy_bindings` (`src/sase/ace/tui/widgets/_keybinding_modes.py:390`) to take the active
Artifacts sub-tab and render the matching menu; update all four call sites (`actions/clipboard/_core.py:70`,
`actions/changespec/_display.py:270`, `actions/agents/_display_detail_footer.py:72`,
`actions/axe_display/_render.py:204`). Fix the restore path: when `_handle_copy_key` finishes on a non-PR sub-tab it
must call `_sync_active_artifacts_entry_state` (`src/sase/ace/tui/actions/artifacts.py:204`), not
`_refresh_current_tab`, or the footer stays in COPY.

**Docs.** Update the `?` help popup and `docs/ace.md` per the ACE guidelines in `src/sase/ace/CLAUDE.md` (help-popup
sync is mandatory; the footer convention there governs what belongs in the footer versus the help modal).

**Tests.** Cover per sub-tab: `%` opens copy mode; each menu key copies the right value; an unknown key warns and names
the sub-tab's key set; `%escape` cancels; the footer shows the sub-tab's menu and is restored afterwards; and — the
regression that motivates the whole phase — `%n` on Chats does **not** copy a ChangeSpec name.

---

## Marks on non-PR Artifacts sub-tabs

**Route the action.** Add `toggle_mark` and `clear_marks` to `NON_PRS_ARTIFACT_ACTIONS`, and give
`action_toggle_mark`/`action_clear_marks` (`src/sase/ace/tui/actions/marking.py:26,69`) an Artifacts-sub-tab branch
ahead of the existing `current_tab` chain. Without that branch the un-gated `m` marks a ChangeSpec from the Chats pane.

**Identity.** Key marks on `ArtifactEntryTarget` (`src/sase/ace/tui/widgets/artifacts/entry_navigation.py:11`) — the
typed identity tuple every non-PR pane already computes for every row (`("commit", repo, sha)`, `("chat", path)`,
`("bug", project, n)`, `(kind, project, identity)` for plans). It is discriminated, project-scoped, and stable across
refresh, which is exactly what it was built for; that is why it is the right key and why marks must not be stored by row
index. Hold one mark set per sub-tab on the app, cleared when the pane's project scope changes.

**Rendering.** Show the mark glyph on marked rows using each pane's existing row-painting path (the same one
`apply_entry_jump_hints`/`clear_entry_jump_hints` use), so marks survive a refresh and repaint without a full rebuild.
Follow the ACE footer convention: a mark-count indicator belongs in the footer only because it is sometimes present and
sometimes absent.

**Bulk copy.** Extend the previous phase's menus so that when a sub-tab has marks, its copy targets act on the marked
set instead of the selection, and the toast names the count (_"Copied 3 chat paths"_). Multi-value output should reuse
`format_multi_copy_content` (`src/sase/ace/tui/actions/clipboard/_helpers.py`).

**Tests.** Per sub-tab: mark/unmark toggling, mark survival across a pane refresh, `u` clearing, the footer count, and
bulk copy of a marked set. Include a regression that `m` on a non-PR sub-tab leaves `marked_indices` (the ChangeSpec
mark set) untouched.

---

## Anchored artifact-file path copy

Rewrite `_artifact_file_clipboard_path` (`src/sase/ace/tui/modals/artifact_files_modal.py:107`) so `Y` always copies
something that resolves to the intended file from anywhere.

Rules:

1. Prefer the **stored** path. It exists for 100% of indexed artifacts. Emit it home-relative (`~/...`) through the
   existing `_clipboard_path`.
2. Keep the PDF special case (`_artifact_file_display_path` returns `source_path` for `kind == "pdf"`): for a
   markdown-derived PDF the source document is what a human wants.
3. Never emit a bare workspace-relative path. If a source-path answer is chosen, anchor it — emit the absolute
   (home-relative) form. Deleting `_workspace_relative_path`'s unanchored `relative.as_posix()` return is the point of
   this phase; do not preserve it for "compatibility".
4. Say which one was copied in the toast — _"Copied stored path: ~/.sase/artifacts/..."_ versus _"Copied source path:
   ..."_ — and warn when the chosen source path no longer exists.

Four existing tests in `tests/ace/tui/modals/test_artifact_files_modal_copy.py` assert the old behavior
(`..._Y_copies_workspace_relative_path_and_stays_open`, `..._Y_without_workspace_falls_back_to_home_relative_path`,
`..._copy_uses_pdf_markdown_source_path`, `..._Y_uses_workspace_relative_source_path_for_global_artifact`). Rewriting
them to the new contract is expected and intended — but state the new contract in each test's name so the change is
legible rather than silently inverted. Add a regression for the recycled-workspace case: a stored artifact whose
`source_path` points into a workspace that has since been reused must not yield a bare relative path.

---

## Safe text-artifact fallback viewer

Keep the `bat` branch of `artifact_text_viewer_command` (`src/sase/ace/tui/graphics/_viewer_loop_media.py:30`) exactly
as it is. Replace the `["cat", str(expanded)]` fallback.

`display_text_artifact` (`:137`) runs the command through an injected `run_command`, and the injection is what makes the
viewer testable, so keep returning a command list. Add a small module invoked as
`[sys.executable, "-m", "sase.ace.tui.graphics.artifact_text_dump", "--", <path>]` that:

- reads a bounded prefix of the file rather than the whole thing, and says so when it truncates;
- refuses to dump binary content (NUL bytes, or a decode failure ratio above a threshold), printing a one-line notice
  instead — this is the behavior `bat` already gives on its branch;
- decodes as UTF-8 with replacement, so legitimate non-ASCII text stays readable. `cat -v` is **not** an acceptable
  shortcut: GNU `cat -v` escapes non-ASCII bytes and would mangle every UTF-8 document;
- neutralizes ANSI/OSC escape sequences so a hostile artifact cannot retitle the terminal, emit OSC 8 links, or drive
  the cursor;
- treats everything after `--` as a path, giving the fallback the same argument-boundary hygiene the `bat` branch has.

Unit-test the dump module directly (it is pure): a UTF-8 document round-trips, a file with ANSI/OSC sequences comes out
inert, a NUL-containing file is refused, a file larger than the bound is truncated with a notice, and a filename
beginning with `-` is handled. Update the two existing tests at
`tests/ace/tui/artifact_file_viewer/test_rendering.py:438,456`.

---

## Generic sidecar roles in the SDD store

This phase removes the name `research` from the SDD store layer and, in doing so, fixes a real defect: a project that
disables or renames that sidecar currently cannot materialize sidecar storage at all.

**Model.** In `src/sase/sdd/_store_types.py`:

- Replace `SddStoreRecord.plans/research/beads` with `sidecars: Mapping[str, SddSidecar]`. Keep `plans` and `beads` as
  read-only convenience properties (they are genuinely reserved roles); delete `research` as a field. Make
  `sidecar_for_kind(role)` a plain lookup returning `None` for absent roles instead of raising for unknown names.
- Replace `SddStore.research_dir`/`research_remote_url` with role-keyed maps. `kind_root(role)` and
  `repo_root_for_kind(role)` must resolve any configured role: the reserved `plans`/`beads` mappings stay as they are
  today, and every other role resolves to its own sidecar clone root (in-tree and local storage keep resolving
  `sdd/<role>`).
- Add the reserved-role constants and a `document_sidecar_roles(...)` helper in one place — reserved roles are `plans`,
  `beads`, `agents`; document roles are every other configured role, plus `plans` itself where a caller wants the full
  document corpus. Every later phase imports from here rather than re-deriving the rule.

**Serialization.** `src/sase/sdd/_store_records.py` already writes and reads a role-keyed `sidecars` JSON map
(`:288,300`); make `_sidecars_from_raw` return a dict over all present roles instead of a fixed triple. Keep the
existing `schema_version` rule (`3` when a `beads` sidecar is present, else `2`): extra document roles are additive keys
inside `sidecars`, so an older sase build reads such a record without error and simply ignores roles it does not know.
Record that compatibility decision in a comment — it is the reason no version bump is needed. Reading existing v1/v2/v3
records, including the legacy sidecars key, must be byte-for-byte unchanged.

**Remove the hard dependency.** `_write_compatibility_store` (`src/sase/sdd/_sidecar_init.py:335`) must require only
`plans`, and `_is_materialized_record` (`src/sase/sdd/_store_records.py:200`) likewise.
`_existing_compatibility_sidecar` (`:319`) must stop filtering on `SPLIT_SIDECAR_KINDS`. `initialize_sidecars` already
handles arbitrary roles; the narrowing at the end of the transaction is the only thing to undo.

**Commit routing.** `sdd_commit_partitions` (`src/sase/sdd/_commit_store.py:134`) must partition split-store paths
across every configured sidecar root, not the fixed plans/research/beads triple.

**Tests.** Extend `tests/sdd_store/` with: a project configuring a document sidecar under a name other than `research`
gets a store record, roots, and commit partitioning for it; a project with **no** `research` sidecar materializes
successfully (the current defect); existing v2 and v3 records round-trip unchanged; and a record carrying an unknown
extra role is read without error.

---

## Route hardcoded role tuples through the role registry

Mechanical follow-through, one call site at a time. Every literal `("plans", "research", "beads")` tuple and every
`research`-named branch outside the shipped-preset exception becomes a lookup through the previous phase's role
registry. Known sites:

- `src/sase/repo_inventory.py:204,224` — the store-record sidecar loop and the default-description lookup. Keep
  `DEFAULT_RESEARCH_DESCRIPTION` (`src/sase/_linked_repo_config.py:34`) as the seed description `sase repo init` writes
  into config for the `research` role; it must no longer be consulted as a code-level fallback for a role whose
  description comes from config.
- `src/sase/_linked_repo_paths.py:128,147` and `src/sase/_linked_repo_identity.py:92` — clone-dirname mapping and the
  store-identity lookup.
- `src/sase/main/repo_handler_path.py:89` — the `sase repo path <role>` legacy fallback must accept any configured role.
- `src/sase/doctor/checks_config_sdd.py:264,316` — the sidecar clone/remote checks and `_regressed_split_sidecar_paths`
  must iterate configured roles; issue codes stay per-role (`missing-<role>-sidecar-clone`).
- `src/sase/llm_provider/commit_finalizer.py:407,468` — repo roots and the split-clone probe.
- `src/sase/sdd/_paths.py:8,86` — `SDD_CANONICAL_DIRS` and the month-dir branch in `sdd_kind_roots`. Note its consumers
  (`workflows/commit/commit_tracking.py:404`, `ace/tui/widgets/prompt_panel/_agent_commits.py:301,368`) and keep
  `is_sdd_internal_path`'s behavior for `plans`/`beads`/prompts intact.
- `src/sase/sdd/_link_files.py:122` and `src/sase/sdd/_init_files.py:17-23` — `SDD_SIDECAR_KINDS` and the guide-kind set
  become "reserved roles plus roles that have a shipped preset", with `_generic_sidecar_readme` covering the rest.
- `src/sase/sdd/env.py` — export one `SASE_SDD_<ROLE>_DIR` per configured document sidecar (uppercased, non-identifier
  characters mapped to `_`). Keep `SASE_SDD_RESEARCH_DIR` emitted whenever a `research` role exists so existing user
  xprompts keep working, but stop resolving it unconditionally — today it is derived from a `kind_root("research")` call
  that will now raise for projects without that role.
- `src/sase/ace/tui/widgets/file_panel/_linked_deltas.py:154` — the linked-workspace candidate scan.

**Docs.** Add the role model to `docs/sdd_storage.md` and `docs/configuration.md`: which roles are reserved, that any
other `repos.sidecar` entry is a document sidecar, what a document sidecar gets (clone, store root, `sase repo path`,
doctor coverage, commit routing, an env var, plan-search visibility, an ACE Plans kind), and that `research` is just the
default-seeded instance. Check `docs/sdd.md`, `docs/init.md`, and `docs/workspace.md` for statements this invalidates.

**Tests.** A project configuring a sidecar named something other than `research` gets: an inventory record,
`sase repo path <role>`, doctor coverage, a `SASE_SDD_<ROLE>_DIR` env var, and commit routing. A project with no
`research` role produces no `SASE_SDD_RESEARCH_DIR` and no doctor error about it.

---

## Rust core document corpora for plan discovery

Work in the `sase-core` linked repo; open it with the `/sase_repo` skill and use the printed path. Document discovery
and kind classification are core backend logic — the mobile gateway and any future web or editor client need identical
semantics — so this does not belong in the Python facade.

`read_plans` (`crates/sase_core/src/plan/read.rs:50`) currently walks the fixed
`REPO_PLAN_KINDS = [("plans","tale"), ("research","research")]` as sub-directories of a single `sdd_root`, and passes
`*dir_name != "research"` as the tier-override flag to `build_plan`.

Add an optional parameter carrying explicit document corpora — a list of `(root, kind_label)` pairs. When supplied, it
replaces the hardcoded sub-directory walk; when absent, behavior is exactly as it is today (this keeps every existing
caller and test green). For each supplied corpus: walk both flat `<root>/*.md` and sharded `<root>/<shard>/*.md`,
compute `relpath` relative to _that_ corpus root, set `source = "repo"` and `kind = <label>`, and enable the
`tier: tale|epic` frontmatter override only for the reserved `plans` label — a document sidecar is directory-classified,
exactly as `research` is today.

Thread the parameter through `search_plans` (`crates/sase_core/src/plan/search.rs:85`) and the PyO3 `plan_search`
binding (`crates/sase_core_py/src/lib.rs:2707`). Keep the wire additive and optional so a Python build that has not yet
been updated keeps working.

Cover in Rust tests: two separate corpus roots with distinct labels; flat and sharded layouts under one root; a
`plans`-labeled corpus honoring `tier`, a non-`plans` one ignoring it; `kinds` filtering across labels; and unchanged
results when the parameter is omitted.

---

## Plan search and CLI over document-sidecar roles

**Facade.** `src/sase/plan_search/facade.py` resolves one repo root today and, for a flat split-sidecar plans root,
falls into `_search_flat_plans_root`, which relabels everything as `tale`/`epic`. Replace root resolution with
document-corpus resolution: ask the store-roles registry for every configured document sidecar root for the project and
pass them to the core as labeled corpora. The `plans` corpus keeps its `tier` override and prompt-snapshot handling
(`_search_repo_prompts` must keep scanning the plans corpus only); every other corpus is labeled with its role name.
In-tree and local storage keep working: there, the corpora are the `sdd/<role>` sub-directories, which the core's
default path already handles. Preserve the `repo_root`/`local_dir` overrides tests rely on.

**CLI.** `sase plan search --kind` (`src/sase/main/parser_plan.py:340`) must stop being
`choices=["tale","epic","prompt","research"]`. argparse cannot know a project's roles at parse time, so drop `choices`,
validate after parsing against the reserved kinds plus the resolved document roles, and emit a clear error naming the
valid values for the current project. Update the help text and the examples near `:314`. `_KIND_STYLES`
(`src/sase/main/plan_search_render.py:50`) must derive a stable color for an arbitrary role rather than hardcoding
`research`; keep the existing colors for `tale`, `epic`, and `local` so current output does not shift.

Also review `src/sase/plan_search/model.py:21` and the facade module docstring, whose prose enumerates the four kinds.

**Tests.** `sase plan search --kind <custom-role>` finds documents in a sidecar named something other than `research`;
`--kind research` still works when that role exists; an unknown kind errors with a message naming the valid values;
`--kind tale` results are unchanged; prompt-snapshot search is unchanged.

**Note on the corpus.** This repo's own `research` sidecar holds real documents, so exercise the new path against a
fixture sidecar rather than depending on live content.

---

## ACE Plans pane over every document sidecar

Make the Artifacts Plans pane show documents from every configured document sidecar — which is what the research
report's item 4 actually asked for, now that the substrate supports it.

`project_plans_root` (`src/sase/ace/tui/widgets/artifacts/plans_data_sources.py:129`) resolves a single
`resolve_sdd_kind_dir(workspace_dir, 1, "plans")`, and both archive loaders (`:170` and `:185`) search
`kinds=("tale","epic")` against it. Replace the single root with the project's full set of document-sidecar roots, and
let the kinds filter come from the pane's facet state rather than a literal tuple. `PlansSnapshot.plans_roots`
(`src/sase/ace/tui/widgets/artifacts/plans_data_models.py`) becomes a per-project mapping of role to root; update
`load_plans_snapshot`, `load_deep_plan_archive`, the dedup keyed on `match.plan.path`, and the per-project error map,
which should attribute a failure to the role that failed.

Add a kind facet to the Plans filter bar (`plan_filter_bar.py`, `plans_filtering.py`, `plans_filter_session.py`),
populated from the roles actually present, and show each archive row's kind so a document sidecar's entries are
distinguishable from plans. Follow the pane's existing pattern: query membership stays with the Python matcher so
preview and deep reconciliation cannot drift.

Watch two things. The archive limits (`_ARCHIVE_PER_PROJECT_LIMIT = 50`, `_ARCHIVE_MERGED_LIMIT = 100`,
`DEEP_ARCHIVE_PER_PROJECT_LIMIT = 500`) are now shared across more corpora — keep the caps meaningful and keep
`archive_truncated` honest rather than silently dropping a whole sidecar. And all of this runs on a worker thread;
consult the TUI-performance memory with the `/sase_memory_read` skill before changing anything in the refresh or
snapshot path.

**Docs and tests.** Update `docs/ace.md` and the `?` help popup per `src/sase/ace/CLAUDE.md`. Test that a project with a
document sidecar other than `plans` shows its documents in the archive, that the kind facet filters them, that a project
with only a plans sidecar is unchanged, and that a failure in one role's root does not lose the others. The PNG snapshot
suite covers this pane — run `just test-visual`, and if the filter bar or row layout genuinely changed, accept the new
goldens with `--sase-update-visual-snapshots` and say so in the commit.

---

## Out of scope

Named here so no phase drifts into them. All are later items in the same research report:

- A kind-tagged artifact reference grammar (`commit:`, `chat:`, `bug:`, `file:`) generalizing `plans:` refs.
- `sase artifact` as a read CLI, and the `sha256`/`size_bytes`/`mime_type` index fields.
- Making `PreviewPanelModal` a real reader (search, rendered Markdown, open-in-editor).
- A sixth **Files** sub-tab reaching the artifact-file viewer from the Artifacts panel.
- A unified "Copy as…" palette and OSC 52 clipboard transport.
- Artifact refs in the prompt bar, and Jump All entries for artifacts.

Re-minting artifact ids is explicitly not wanted: ids are location-derived, and the managed store is 100% intact.
