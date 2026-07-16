---
tier: epic
title: Artifacts tab with PRs, Commits, Bugs, and Plans sub-tabs
goal: 'The ACE TUI''s "PRs" tab becomes an "Artifacts" tab hosting four sub-tabs:
  PRs (functionally unchanged), Commits (a rich cross-repo commit timeline with first-class
  message and diff viewing), Bugs (external-tracker bug triage backed by GitHub issues,
  with create/view/modify), and Plans (browse and manage the plan pipeline and bead
  DAG per project). Every sub-tab is intuitive, responsive under the TUI performance
  budget, and visually consistent with the existing ACE aesthetic.

  '
phases:
- id: artifacts-scaffold
  title: Artifacts tab rename and sub-tab scaffold
  depends_on: []
  description: 'Rename the PRs tab to Artifacts, introduce the sub-tab strip and content
    switcher with the PRs pane hosting the existing surface unchanged, add sub-tab
    navigation/keymaps/palette/help wiring, the shared project-scope control, lazy
    pane-activation lifecycle, and placeholder panes with empty states for Commits/Bugs/Plans.

    '
- id: issue-provider-seam
  title: Issue-tracker provider hooks and bug-link model
  depends_on: []
  description: 'Add optional issue CRUD hooks to the VCS provider plugin seam in this
    repo, an IssueWire model, capability detection, a fake in-memory issue provider
    for tests, and a bug-link helper that cross-references external bug ids with epic
    beads and ChangeSpec BUG tags.

    '
- id: commits-subtab
  title: Commits sub-tab
  depends_on:
  - artifacts-scaffold
  description: 'Build the Commits pane on top of the existing sase vcs log collection
    backend: constellation legend, day-grouped timeline with presence glyphs and tag
    chips, debounced detail panel with message + diff, CommitViewModal integration,
    and interactive filters matching the CLI''s options.

    '
- id: github-issues-plugin
  title: GitHub issues implementation in sase-github
  depends_on:
  - issue-provider-seam
  description: 'Implement the new issue hooks in the sase-github plugin repo using
    the gh CLI with JSON output: list/get/create/edit/close/reopen, error and auth
    mapping, and plugin-side tests. Hooks are optional so older plugin versions simply
    report the capability as absent.

    '
- id: bugs-subtab
  title: Bugs sub-tab
  depends_on:
  - artifacts-scaffold
  - issue-provider-seam
  description: 'Build the Bugs pane: capability-gated issue list and detail with markdown
    body, create/edit/state-toggle actions as tracked background tasks, open-in-browser
    and copy actions, linked epic beads and ChangeSpecs via the bug-link helper, and
    a start-agent-from-bug action that seeds a prompt with the issue context and bug
    id.

    '
- id: plans-subtab
  title: Plans sub-tab
  depends_on:
  - artifacts-scaffold
  - issue-provider-seam
  description: 'Build the Plans pane: pending plan proposals with approve/reject,
    epic beads with fold-out phase trees and dependency/readiness state, the plan
    archive with rendered markdown viewing, bead status and description edits as tracked
    tasks, launch-ready-epic work, and external-bug chips on epic beads.

    '
- id: integration-polish
  title: Integration, visual polish, and end-to-end verification
  depends_on:
  - commits-subtab
  - github-issues-plugin
  - bugs-subtab
  - plans-subtab
  description: 'Cohesion pass across all four sub-tabs: consistent accents and empty
    states, help/quickstart/palette copy audit, PNG visual snapshots for every pane,
    PRs-parity regression checklist, and a TUI performance audit of the Artifacts
    tab against the p95 key-to-paint budget.

    '
create_time: 2026-07-15 20:26:51
status: done
bead_id: sase-69
---

# Plan: Artifacts tab with PRs, Commits, Bugs, and Plans sub-tabs

## Product context

The ACE TUI currently has three top-level tabs: Agents, PRs, and AXE. The PRs tab is the only surface for reviewing work
products, yet SASE produces several other artifact streams the user must drop to the CLI for:

- **Commits** — `sase vcs log` renders a beautiful cross-repo timeline (constellation legend, day grouping, presence
  glyphs, SASE tag chips), but it is read-only text: no interactive browsing of commit messages or diffs.
- **Bugs** — there is no bug/issue surface at all. `bug_id` already threads from epic plans into beads and `BUG:` PR
  tags, but the bugs themselves live on GitHub, invisible to the TUI.
- **Plans/beads** — plans and the bead DAG live in the `<project>--plans` sidecar repo and are only visible today
  through agent-detail panels or the `sase plan` / `sase bead` CLIs. There is no way to browse pending proposals, epics,
  or phase readiness independently of a running agent.

This epic renames the PRs tab to **Artifacts** and gives it four sub-tabs — **PRs · Commits · Bugs · Plans** — turning
it into the single place where all work products are reviewed and managed. Each sub-tab must earn its place with real
workflow value:

- PRs: exactly what exists today, moved without regression.
- Commits: answer "what actually landed, everywhere, and what does the diff look like?" without leaving the TUI.
- Bugs: triage GitHub issues, create/edit/close them, and jump from a bug to the epics and PRs addressing it (or start a
  new agent from a bug).
- Plans: approve/reject plan proposals, watch epic phase progress and readiness, mutate bead state, and launch ready
  epic work.

## Current architecture (verified anchors)

Top-level tabs are hand-rolled: `src/sase/ace/tui/tab_order.py` defines
`TabName = Literal["changespecs", "agents", "axe"]` and `TAB_ORDER`; the "PRs" string is only a display label in
`src/sase/ace/tui/widgets/tab_bar.py` (`_TAB_DISPLAY_NAMES`), mirrored in `modals/help_modal/binding_common.py`,
`modals/jump_all_modal.py`, and `modals/command_palette_modal.py`. `AceApp.compose()` (`src/sase/ace/tui/app.py`) mounts
three always-alive sibling containers (`#changespecs-view`, `#agents-view`, `#axe-view`) toggled by the `current_tab`
reactive in `watch_current_tab`; tab-scoped actions are gated in `AceApp.check_action`. `sase ace --tab`
(`src/sase/main/parser_ace.py`) accepts the internal tab names.

Other load-bearing facts:

- **Sub-tab precedent**: `widgets/panel_tab_strip.py` (`PanelTabStrip`, `PanelTab`) plus Textual `ContentSwitcher` is
  the established sub-tab pattern; `modals/projects_pane.py` is a complete working example (sub-tab enum, `[`/`]`
  cycling, click routing, per-pane project filter via `InventoryProjectPicker`). `modals/help_modal/modal.py` and
  `modals/config_center_modal.py` use the same pattern.
- **PRs tab internals**: `ChangeSpecList`/`ChangeSpecDetail` widgets, data from `find_all_changespecs_cached()`
  (`actions/changespec/_loading.py`), query-driven filtering through the Rust query corpus, project filtering via the
  `project:<name>` query token (`sase/ace/query/introspection.py:get_sole_project_filter`).
- **Commits machinery**: `sase vcs log` = argparse in `src/sase/main/parser_vcs.py`, collection in
  `src/sase/vcs_log/collect.py` (`run_vcs_log`, `CommitFilters`, `VcsLogResult`), repo resolution in
  `vcs_log/resolve.py`, rendering in `vcs_log/render.py` with palette/glyphs in `vcs_log/_style.py` and tag chips in
  `vcs_log/_tag_style.py`. Parsing/aggregation/presence is Rust-backed (`src/sase/core/vcs_log_facade.py`). Diff text is
  a git subprocess (`VCSProvider.show_revision`) or persisted `.diff` file, and the agents tab already has a polished
  commit viewer: `modals/commit_view_modal.py` (worker-thread diff loading, Rich `Syntax` "diff" rendering via
  `util/lazy_syntax.py:lazy_renderable`, change summary via `widgets/prompt_panel/_agent_deltas.py`), driven by
  `CommitViewSpec` records.
- **Provider seam**: two pluggy groups (`sase_vcs`, `sase_workspace`) declared in `pyproject.toml`, hookspecs in
  `src/sase/vcs_provider/_hookspec.py` and `src/sase/workspace_provider/_hookspec.py`. Optional capabilities follow the
  "hook returns None / method raises NotImplementedError = unsupported" convention (e.g.
  `ws_generate_submitted_check_script`). **There are no issue/tracker hooks today**; the GitHub implementation lives in
  the external `sase-github` plugin repo.
- **Beads/plans**: bead store = `beads/` inside the plans sidecar clone (`issues.jsonl` event stream, Rust-backed
  reads/mutations through `src/sase/core/bead_read_facade.py` / `bead_mutation_facade.py`, Python API
  `src/sase/bead/project.py:BeadProject`). Epic = `issue_type=plan` + `tier=epic`; phases reference `parent_id` with
  `Dependency` edges. `Issue.changespec_name` / `Issue.changespec_bug_id` already link an epic to a ChangeSpec and
  free-form bug id, flowing from epic-plan frontmatter `bug_id` (`src/sase/sdd/plan_validate.py`,
  `src/sase/bead/epic_from_plan.py`) into `BUG:` PR tags (`src/sase/workflows/commit/pr_operations.py`). Plan pipeline
  inventory: `src/sase/main/plan_inventory.py`; approval flow:
  `src/sase/llm_provider/_plan_utils.py:handle_plan_approval` + `modals/plan_approval_modal.py`. Per-project bead store
  resolution: `sase.bead.workspace.get_project_beads_dirs_for_project`. Epic launch: `src/sase/bead/cli_work_handler.py`
  (`sase bead work`).
- **Performance doctrine** (memory/tui_perf.md, binding on all TUI phases): no synchronous I/O on the event loop;
  multi-second user operations go through `_submit_tracked_task()` (`actions/task_actions.py`); detail panels debounce
  through `DetailPanelDebouncer`; refreshes reuse the cached-then-async pattern with last-request-wins coalescing; p95
  key-to-paint < 16 ms on every tab.

## Design overview

### Rename strategy

User-visible rename only, plus ergonomic aliases; the internal tab id stays `changespecs`:

- Change the display label to "Artifacts" in `tab_bar.py` and every mirror (help modal, jump-all modal, command palette,
  quickstart/onboarding copy).
- Add `artifacts` as an accepted value for `sase ace --tab` (keep `changespecs` working).
- Introduce a named constant (e.g. `ARTIFACTS_TAB: TabName = "changespecs"`) in `tab_order.py` so new code reads
  correctly without a mass literal rename. A full internal rename of the `TabName` value is an explicit non-goal (broad
  mechanical diff across app, actions, commands, testing harness, and repro/replay for zero user value); note it as a
  possible follow-up in the scaffold phase's summary.

### Sub-tab architecture

Inside the Artifacts view container, mount a `PanelTabStrip` (labels `PRs · Commits · Bugs · Plans`, each with its own
accent color) above a `ContentSwitcher` with four pane widgets. Follow `projects_pane.py` conventions:

- A `current_artifacts_subtab` reactive on the app (or the Artifacts view) drives `ContentSwitcher.current` +
  `PanelTabStrip.set_active_tab`.
- Navigation: `[` / `]` cycle sub-tabs (gated to the Artifacts tab via `check_action`; the agents tab already uses these
  keys for other actions, so gating must be airtight), clickable strip, one command-palette entry per sub-tab for direct
  jumps, help-modal documentation. All new keys are declared in `src/sase/default_config.yml` and the keymap registry.
- The PRs pane hosts the existing widgets (`ChangeSpecInfoPanel`, `ChangeSpecList`, `AncestorsChildrenPanel`,
  `SearchQueryPanel`, `ChangeSpecDetail`, `TabQuickStart`) with **zero behavioral change**: all existing changespec
  actions gain an additional "PRs sub-tab is active" gate but keep their keys, semantics, saved queries, grouping modes,
  and refresh behavior.
- **Pane lifecycle seam**: panes expose `on_first_activate` / `on_activate` / `on_deactivate` hooks. Nothing loads at
  app startup except the PRs pane (startup time must not regress); Commits/Bugs/Plans load lazily on first activation
  and refresh only while active. The auto-refresh loop (`actions/event_refresh/_auto_refresh.py`) consults the active
  sub-tab in addition to `current_tab`.
- Per-pane selection state survives sub-tab switches (panes stay mounted, mirroring how top-level tabs already work).

### Shared project scope

Commits, Bugs, and Plans are project-scoped surfaces; PRs remains query-scoped as today. Add one shared "Artifacts
project scope" control:

- A scope state (project name or "all" where a pane supports it) shown as a chip in each pane's info panel, changed with
  a `p` project-picker modal (reuse the `InventoryProjectPicker` pattern).
- Default resolution order: sole `project:` filter from the PRs query if set → the single enabled project if only one
  exists → last picked value this session → pane-specific fallback (Commits: current project constellation; Bugs/Plans:
  an explicit "pick a project" empty state).
- Semantics per pane: Commits maps scope to the `sase vcs log` repo constellation (with an all-projects toggle mirroring
  `--all`); Bugs maps it to the project's primary repo's tracker; Plans maps it to the project's plans sidecar store.

### Data-flow and performance policy (applies to every new pane)

- All disk/subprocess/network collection happens off the event loop (`asyncio.to_thread` / `run_worker(thread=True)`),
  coalesced last-request-wins, rendering from cached results immediately on re-activation.
- Mutations (issue CRUD, bead updates, plan approval, epic launch) run as tracked background tasks via
  `_submit_tracked_task()` with an optimistic UI stage where safe and typed outcomes applied on the UI thread.
- Detail panels debounce through `DetailPanelDebouncer`; list highlight movement never blocks.
- Caches are keyed by source mtime (bead store, plan archive) or an explicit TTL (remote issue lists), invalidated after
  our own mutations.

## Sub-tab designs

### PRs (unchanged)

Everything the changespecs tab does today, hosted in the first pane. The acceptance bar is parity: existing changespec
unit/visual tests pass with at most mechanical updates (container ids, gating), and a manual parity checklist (status
changes, bulk ops, mail, diff, reword, hooks, rebase, rewind, marks, saved queries, grouping, onboarding) is exercised
in the polish phase.

### Commits

The `sase vcs log` experience, interactive:

- **Info panel (top)**: the constellation legend — per-repo accent-colored names with commit counts, `↑ahead ↓behind`,
  presence key, fetch freshness, and active-filter chips (author/since/until/repo/limit/SDD/all) so state is always
  visible.
- **Timeline list (left)**: an `OptionList` of commits, newest-first, with non-selectable day-header banner rows
  (`── Today ──` style, reusing the `_day_label` logic), presence glyph, `HH:MM`, gold short SHA, repo-colored label,
  subject, inline SASE tag chips, dim author. Reuse `vcs_log/_style.py` and `_tag_style.py` renderers directly — they
  build plain Rich `Text`.
- **Detail panel (right, debounced)**: full commit message, tag chips, change summary (`+A ~M -D · N files`, via
  `parse_unified_diff_deltas`), and the diff rendered with `lazy_renderable(text, "diff", line_numbers=True)`; diff text
  loaded off-thread through `VCSProvider.show_revision` and cached per SHA.
- **Actions**: `enter` opens `CommitViewModal` seeded with the visible timeline (synthesize
  `CommitViewSpec(sha=full_id, cwd=repo.path, ...)`) so `ctrl+n/p` pages between commits; `y` copies the SHA; `f` opens
  a Log Filters modal (author substrings, since/until using the CLI DATE grammar, repo multi-select, limit); quick
  toggles for SDD inclusion and all-projects; `F` fetches remote refs now as a tracked task (default collection is
  no-fetch, honoring the freshness cache); `R` refreshes.
- **Backend**: call `run_vcs_log` with a `CommitFilters` built from pane state — no new collection logic, so CLI and TUI
  can never drift. The Rust-backed parse/aggregate/presence path is reused as-is (respecting the Rust core boundary: no
  re-implementation on either side).

### Bugs

External-tracker bug management, GitHub issues first.

**Provider seam (this repo)** — add optional issue hooks to the `sase_vcs` pluggy group, mirroring existing
optional-capability conventions:

- Hooks: `vcs_list_issues(state, limit)`, `vcs_get_issue(number)`, `vcs_create_issue(title, body, labels)`,
  `vcs_update_issue(number, title/body/state/labels)`, plus an issue-URL accessor. Declared in
  `vcs_provider/_hookspec.py`, surfaced as `NotImplementedError`-defaulted `VCSProvider` methods, dispatched through the
  existing registry keyed off the scoped project's primary repo.
- Wire model: a frozen `IssueWire` dataclass (number, title, state, body, labels, assignees, author, created/updated
  timestamps, url, comment count) defined beside the other provider wire types. This stays a host-side provider-plugin
  surface (pure gh-CLI dispatch and JSON normalization, no cross-frontend domain logic), so it does not cross the Rust
  core boundary; if a second frontend later needs it, promote the model to sase-core then.
- Capability detection: "provider implements the hook" — `bare_git` deliberately does not, and the Bugs pane renders an
  explanatory empty state ("Bugs needs a tracker-capable provider (GitHub)") for such projects. Older sase-github
  versions without the hooks are automatically treated as unsupported (no version coupling).
- A fake in-memory issue provider registered directly on the pluggy manager powers unit and TUI tests without network or
  gh.

**GitHub implementation (sase-github repo)** — implement the hooks with
`gh issue list/view/create/edit/close/reopen --json ...`, mapping auth and rate-limit failures to typed errors the TUI
can render as a non-blocking banner. This phase's changes land in the sase-github linked repo; the implementing agent
must open it with the `/sase_repo` skill and commit there through the standard commit workflow.

**TUI pane**:

- **List**: state glyph (open/closed), `#number`, title, label chips, assignee, updated age. Filter toggle
  open/closed/all; TTL-cached with explicit `R` refresh (tracked task).
- **Detail (debounced)**: title, metadata row, markdown-rendered body, labels, and two link sections computed by the
  bug-link helper: **Epics** (epic beads whose `changespec_bug_id` matches the issue number) and **PRs** (ChangeSpecs
  whose `BUG` tag matches). Links jump to the Plans sub-tab / PRs sub-tab with the target selected.
- **Actions**: `c` create (modal: title, body text area, labels), `e` edit, state toggle close/reopen (with confirm),
  `o` open in browser, `y` copy URL/number — every remote mutation a tracked task with the list updated optimistically
  and reconciled on completion. Plus the loop-closer: **start agent from bug** — reuse the start-agent-from-changespec
  machinery to pre-seed the prompt bar with the issue title/body/number so the resulting epic plan carries `bug_id` and
  its PRs carry the `BUG:` tag. This, plus the link sections, delivers the epic-bead ↔ external-bug association
  end-to-end using the existing `changespec_bug_id` thread — no bead schema change required for the MVP (a dedicated
  `external_ref` bead field in the sase-core schema is the designed follow-up once bidirectional write-back is wanted).

### Plans

The plan pipeline and bead DAG for the scoped project:

- **List (left)**, three groups with banner headers:
  1. **Proposals** — pending plan proposals (from the plan-inventory pending markers), each showing name, tier, age. `A`
     approves / `X` rejects through the existing `handle_plan_approval` flow (tracked task; approval side effects like
     coder/epic launches go through the existing notification and launch paths untouched).
  2. **Epics** — epic beads (open/in-progress first), each expandable (reuse the agents-tab `h`/`l` fold convention)
     into phase children showing status glyph (open ○ / in-progress ◐ / closed ●), blocked-vs- ready derived from
     dependency edges, and per-phase model. Epic rows show progress (`3/7 phases`), ChangeSpec name, and a `#bug` chip
     when `changespec_bug_id` is set (`o` opens the issue URL when the scoped project's provider supports issues).
  3. **Plan archive** — recent committed plans from the sidecar store (month-sharded `.md` files), with tier and
     `wip/done` status chips, searchable via the existing plan-search facade.
- **Detail (right, debounced)**: for a plan file, frontmatter summary (tier, goal, status, linked `bead_id`) above the
  rendered markdown body; for a bead, full details (description, design ref, deps list with ready/blocked state,
  ChangeSpec + bug id, timestamps, close reason).
- **Actions**: `enter` opens a full-screen plan/bead viewer; `s` cycles bead status and `e` edits title/description
  (bead mutation facade, tracked tasks — mutations commit to the sidecar and must tolerate the existing lock-retry
  behavior); `w` launches work for a ready epic (`sase bead work` path, confirm modal, tracked task, honoring the
  existing launch-approval machinery); `R` refreshes (bead reads are local Rust-backed JSONL reads — cheap, but still
  off-thread and mtime-cached).
- **Backend**: reads through `BeadProject` and the plan-inventory/search facades; no new storage. All bead/plan domain
  logic stays behind the existing Rust-backed facades.

## Phase goals and deliverables

**artifacts-scaffold** — Rename + scaffold, shippable on its own: Artifacts label everywhere, `--tab artifacts` alias,
sub-tab strip + switcher with the PRs pane at full parity, `[`/`]`/click/palette/help navigation, keymap config,
`check_action` gating extended to sub-tab awareness, project-scope state + picker, pane lifecycle seam wired into the
auto-refresh loop, and three placeholder panes with quickstart-style empty states that state what is coming. Establish
the module layout so later phases touch disjoint files (suggested: one package per pane under the TUI widgets tree plus
a matching actions module). Tests: tab-bar/label snapshot updates, sub-tab cycling and gating unit tests, PRs regression
suite green.

**issue-provider-seam** — Hookspec additions, `VCSProvider` optional methods, `IssueWire`, registry dispatch +
capability probe, the fake issue provider, and the bug-link helper (bug id → epic beads via `changespec_bug_id`; bug id
→ ChangeSpecs via `BUG` tag). Pure backend/CLI-free phase with thorough unit tests; runs in parallel with the scaffold.

**commits-subtab** — The Commits pane as designed above. Tests: renderer unit tests against fixture `VcsLogResult`s, a
pilot test driving filter/refresh/modal flows with a stubbed collector, and PNG snapshots for the pane (timeline +
detail).

**github-issues-plugin** — The gh-backed hook implementations plus tests in the sase-github repo (opened via
`/sase_repo`). Deliverable includes a manual smoke recipe against a real repo (documented in the phase summary, not run
in CI).

**bugs-subtab** — The Bugs pane as designed above, developed against the fake provider so it does not block on the
plugin phase; capability empty state; cross-links to Plans/PRs sub-tabs; start-agent-from-bug. Tests: pilot tests for
CRUD flows via the fake provider, capability-gating tests, PNG snapshots.

**plans-subtab** — The Plans pane as designed above. Tests: pilot tests over a fixture bead store + plan archive (create
in tmp sidecar layout), approval/reject flow tests with stubbed side effects, PNG snapshots.

**integration-polish** — Everything together: visual cohesion (accent colors per sub-tab consistent with the existing
palette — PRs keeps `#00D7AF`; Commits gold, Bugs warm red, Plans violet, exact values chosen to match
`provider_styles`/existing palette conventions), copy audit (quickstart, onboarding, help, palette), full PNG snapshot
suite for all four panes in representative states (populated, empty, unsupported), the PRs parity checklist, and a
performance audit: `SASE_TUI_PERF=1` j/k p95 < 16 ms on the Artifacts tab in every sub-tab, startup profile unchanged
(lazy panes), and no event-loop blocking in the new code paths. Fix what the audit finds.

## Testing strategy

- Unit tests colocated per module as the repo convention dictates; TUI behavior via the existing ace testing harness /
  Textual pilot patterns.
- Visual regression via the PNG snapshot suite (`just test-visual`); new goldens for each pane state, added deliberately
  (exact pixel equality locally).
- No test may hit the network or require gh auth: the fake issue provider and fixture bead stores/log results cover
  everything; sase-github plugin tests stub the gh subprocess boundary.
- Every phase runs `just install` then `just check` before finishing, and keeps `src/sase/default_config.yml` in sync
  with any keymap changes.

## Risks and mitigations

- **PRs regression** — the highest-value existing surface moves containers. Mitigation: zero-behavior-change rule in the
  scaffold phase, existing test suite as the gate, explicit parity checklist in polish.
- **Event-loop stalls** — commits collection spawns many git subprocesses; bead mutations and issue CRUD are
  multi-second. Mitigation: the data-flow policy above is binding; TUI phases must read memory/tui_perf.md first (per
  repo instructions) and the polish phase audits with SASE_TUI_PERF.
- **Startup-time regression** — new panes must not add startup work; only the PRs pane initializes eagerly, verified in
  the polish phase profile.
- **Keymap collisions** — `[`/`]` and pane keys collide with other tabs' bindings if gating is sloppy. Mitigation:
  `check_action` gating on tab + sub-tab, default_config.yml as the single source, help-modal audit.
- **gh variability** (auth missing, rate limits, offline) — typed errors rendered as a pane banner; the TUI never blocks
  or crashes on provider failure; capability detection keeps bare-git projects clean.
- **Cross-repo phases** — sase-github changes must go through `/sase_repo` and land independently; because hooks are
  optional, core and plugin can land in either order without breaking each other.
- **Bead store contention** — TUI mutations race agent-driven bead commits; rely on the existing lock-retry path and
  refresh-after-mutation, never hold the store open across user think-time.

## Non-goals (explicit)

- Renaming the internal `changespecs` tab identifier or any ChangeSpec data types (follow-up candidate).
- A new `external_ref` field on the bead schema in sase-core; the MVP rides the existing `changespec_bug_id`/`BUG:`
  thread. Adding the dedicated field (Rust schema + wire + JSONL + `sase bead update` flag) is the designed next step
  once write-back association is needed.
- Non-GitHub trackers (the hook seam is provider-shaped; nothing else is GitHub-specific).
- Issue comments (read or write) beyond the comment count; a follow-up can add a comments drawer to the Bugs detail
  panel.
- New CLI subcommands (the TUI reuses existing backends; `sase vcs log`, `sase bead`, and `sase plan` remain the CLI
  surfaces).
