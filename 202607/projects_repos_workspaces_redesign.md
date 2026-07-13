---
create_time: 2026-07-13 09:57:15
status: wip
prompt: 202607/prompts/projects_repos_workspaces_redesign.md
bead_id: sase-5w
tier: epic
---
# Plan: Redesign SASE Projects / Repos / Workspaces + Admin Center Projects Tab

## Product context

SASE's project-adjacent concepts have grown organically and are now muddled:

- The Admin Center "Projects" tab (`ACTIVE / sibling / inactive / all` sub-tabs) shows **non-projects**: any
  subdirectory of `~/.sase/projects/` is treated as a project by the Rust enumeration (`list_project_records` defaults
  missing `PROJECT_STATE:` to `active`). Directories created only by `skill_uses.jsonl` telemetry (e.g. `basher`,
  `symvision`, `toobig` in the 2026-07-13 screenshot) appear as "active projects" with warnings like "ProjectSpec file
  not found" and "WORKSPACE_DIR is not set".
- Linked-repo bookkeeping records (`PROJECT_STATE: sibling`) masquerade as a third project state even though linked
  repos are not projects.
- "Companion repos" is a confusing name for the SDD side-repositories (`<project>--plans`, `<project>--research>`),
  especially since they are _also_ auto-injected as linked repos.
- There is no TUI surface that answers "what repos does sase know about?" or "what workspace directories exist, who has
  claimed them, and which are stale?" — even though rich data already exists (per-project workspace `registry.json`,
  `RUNNING:` claims in ProjectSpecs, `workspace.cleanup_ttl_days`).

This plan reinvents the taxonomy around three crisp terms — **projects**, **repos**, **workspaces** — and rebuilds the
Admin Center Projects tab around them with three sub-tabs: **Projects · Repos · Workspaces**.

## The new conceptual model (canonical definitions)

These definitions are the product spec; the glossary entry added in the final phase must match them.

**Project** — A named unit of work registered with sase. Projects are created _only_ when the user passes a new argument
to a VCS xprompt workflow that resolves to a valid project name:

- `#git:<name>` — any project name made of valid project-name characters is accepted (auto-created).
- `#gh:<org>/<repo>` — the argument must be a valid, _existing_ GitHub repository.

A project is backed by a ProjectSpec file at `~/.sase/projects/<name>/<name>.sase`. Projects have exactly **two states:
enabled and disabled**. A project is always enabled unless the user explicitly disables it from the Projects sub-tab.
Both enabled and disabled projects are listed there. (The `home` project remains system-managed and hidden, as today.)

**Repo** — Any repository known by sase:

1. **Primary repos** — the main repo of a project (so _some_ repos are associated directly with projects).
2. **Sidecar repos** — the SDD side-repositories (`<project>--plans`, `<project>--research`). This is a rename of
   today's "companion repos"; all references must be replaced.
3. **Linked repos** — repos declared under `linked_repos:` in a project's `sase.yml`.

Workspace directories are explicitly **not** repos.

**Workspace** — A numbered workspace directory (clone of a project's primary repo) managed by the workspace store and
tracked in the project's workspace `registry.json`. Each sase agent claims exactly one workspace before launch (a
`RUNNING:` claim in the ProjectSpec) and does not release it until completion. Workspaces are always associated with a
main project repo. Linked-repo clones materialized inside/alongside a workspace are repo checkouts, not workspaces of
their own.

### State-model mapping (old → new)

| Today (persisted `PROJECT_STATE:`)       | New model                                                                                                                                                             |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `active` (or missing — the default)      | `enabled` (missing still defaults to enabled)                                                                                                                         |
| `inactive` (+legacy `archived`/`closed`) | `disabled`                                                                                                                                                            |
| `sibling`                                | Not a project state. Kept as an _internal_ backing-record marker for linked-repo claim bookkeeping; records with it are classified as repos, never shown as projects. |

Canonical persisted values become `enabled`/`disabled`; all legacy spellings are normalized on read forever (the Rust
core already has this normalization pattern for `archived`/`closed`).

### What makes a record a "true project"

Derived in the Rust core per record:
`is_project = active ProjectSpec file exists ∧ not system-managed ∧ state is not the sibling backing marker`. This
single predicate removes the `basher`/`symvision`/`toobig` class of junk rows (telemetry-only directories with no
`.sase` file) and the linked-repo backing records. Each project record also gains a derived `vcs_kind` (`git` when the
spec carries `BARE_REPO_DIR:`; `gh` for GitHub-provider projects) so the TUI can badge projects by origin.

## Architecture overview

### Rust core boundary

Project enumeration, ProjectSpec parsing, lifecycle state, and claim parsing are owned by the Rust core (`sase-core`
linked repo, crate `sase_core`, exposed via `sase_core_rs` bindings and the Python facades in
`src/sase/core/project_lifecycle_facade.py` / `project_lifecycle_wire.py`). All lifecycle/state and record-shape changes
land there first, then the Python mirrors update. Implementing agents access the Rust repo via
`sase workspace open -p sase-core -r "<reason>" <workspace_num>` (never by guessing paths).

Repo and workspace _inventories_ (new) compose Rust-owned project records with Python-owned subsystems (linked-repo
config resolution in `src/sase/linked_repos.py` / `_linked_repo_config.py`, SDD store records in `src/sase/sdd/`,
workspace store/registry in `src/sase/workspace_provider/`). Since those inputs are Python today, the inventories are
built as **thin, frontend-neutral Python domain modules** (not TUI code) with CLI surfaces, explicitly documented as
adapters that can migrate into the Rust core later. TUI panes stay presentation-only consumers.

### Data sources already available (reuse, don't reinvent)

- Project records: Rust `list_project_records` → `ProjectRecordWire`.
- Claims: `RUNNING:` field via `sase.running_field` (Rust-backed parse); includes agent/workflow name, PID, CL name,
  timestamp, pinned.
- Workspace registry: `<managed-root>/registry.json` per project (`WorkspaceStore`, `workspace_provider/registry.py`):
  per-workspace `checkout_dir`, `role` (primary/claim), `materialization`, `pinned`, `generation`, `created_at`,
  `last_used_at`.
- Staleness: `workspace.cleanup_ttl_days` (default 14) — same predicate `sase workspace cleanup --stale` uses.
- Linked/sidecar repos: `resolve_linked_repos_for_project()` + `inject_default_linked_repos()` + SDD store records.

## TUI design — the new Projects tab

Tab description becomes "Manage projects, repos, and workspace directories". The four state-filter pseudo-tabs (a static
text line cycled with `[`/`]`) are replaced by a real, clickable `PanelTabStrip`
(`src/sase/ace/tui/widgets/panel_tab_strip.py` — the same widget the Admin Center's main tabs use) with three sub-tabs.
`[`/`]` keep cycling sub-tabs (consistent with the 2026-07-13 keymap-swap commit `8a1a4f467`); clicking works too. Each
sub-tab keeps the established pane anatomy: summary line → filter input → column header + `OptionList` rows → detail
panel → hint bar. All data loads happen off-thread with cached rows shown instantly (per `memory/tui_perf.md` rules);
detail-panel updates stay debounced.

### 1. Projects sub-tab (default)

Successor of today's ACTIVE view — true projects only, enabled and disabled together (enabled sorted first). `ctrl+x`
("mix inactive into active") disappears; there is nothing hidden to reveal.

```
  PROJECTS  ·  Repos  ·  Workspaces
  enabled:3 · disabled:1 · marked:0

  MARK NAME                             VCS  STATE       CLAIMS  WS  REPOS  WARN
       sase (gh_sase-org__sase)         gh   ● enabled        3  38      5     -
       actstat (gh_bbugyi200__actstat)  gh   ● enabled        0   2      3     -
       bob-cli (gh_bobs-org__bob-cli)   gh   ● enabled        0   4      4     -
       old-experiment                   git  ○ disabled       0   0      1     -
 ┌ Details ────────────────────────────────────────────────────────────────────┐
 │ sase   ● enabled    VCS: gh (sase-org/sase)                                 │
 │ Project file:  ~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase.sase    │
 │ Primary repo:  ~/projects/github/sase-org/sase                              │
 │ Repos: 5 (1 primary · 2 sidecar · 2 linked)   Workspaces: 38 (3 claimed)    │
 │ Aliases: -    Display name: sase    Launchable: yes                         │
 └─────────────────────────────────────────────────────────────────────────────┘
```

- Columns: MARK, NAME (`effective_project_name` + dir key), VCS (`git`/`gh` badge), STATE (`● enabled` / `○ disabled`),
  CLAIMS (active claim count), WS (workspace count), REPOS (associated repo count), WARN. The old WORKSPACE column
  (primary repo path) moves to the detail panel — the counts are more useful at a glance and the path was mostly
  truncated anyway.
- Keymaps preserved from today: `j/k` navigate, `/` filter, `m`/`u` marks, `e` edit spec, `A` aliases, `ctrl+d` delete,
  `F` force, `R` reload, `enter` default action. `a`/`d` become **enable/disable** (same keys, new verbs; disable keeps
  the live-work guard + `F` force path).
- **New cross-nav keymaps**: `r` → switch to the Repos sub-tab pre-filtered to the selected project; `w` → switch to the
  Workspaces sub-tab pre-filtered to the selected project (both keys are currently unbound in this pane). Filtering to a
  disabled project via `w` is what surfaces a disabled project's workspaces.

### 2. Repos sub-tab

All repos sase knows about, across enabled projects by default.

```
  Projects  ·  REPOS  ·  Workspaces
  repos:14 · primary:4 · sidecar:6 · linked:4 · project: all        p pick project

  NAME               KIND     PROJECT   CLONED  PATH
  sase               primary  sase      yes     ~/projects/github/sase-org/sase
  sase--plans        sidecar  sase      yes     ~/projects/github/sase-org/sase--plans
  sase--research     sidecar  sase      yes     ~/projects/github/sase-org/sase--research
  sase-core          linked   sase      yes     ~/projects/github/sase-org/sase-core
  sase-github        linked   sase      yes     ~/projects/github/sase-org/sase-github
  chezmoi            linked   home      yes     ~/.local/share/chezmoi
```

- Columns: NAME, KIND (`primary`/`sidecar`/`linked`), PROJECT (owning/host project), CLONED (does the checkout exist on
  disk; `missing` styled as a warning), PATH (shortened).
- Detail panel: full path, kind, source of the record (ProjectSpec / `sase.yml linked_repos` / auto-injected sidecar),
  description from config, `auto_clone`, env var name (`SASE_LINKED_REPO_*`), and for sidecars the SDD storage mode.
- `p` opens the **project picker** (below) to filter by project; the summary line always shows the active project
  filter; `escape` (or picking "All projects") clears it. `/` text-filters within the current view.

### 3. Workspaces sub-tab

All valid workspaces — i.e. present in a project's workspace `registry.json` (created via `sase workspace open` or agent
launches) — for enabled projects by default; filtering to a disabled project shows its workspaces.

```
  Projects  ·  Repos  ·  WORKSPACES
  workspaces:41 · claimed:3 · pinned:2 · stale:12 · project: sase   p pick project

    #  PROJECT  CLAIMED BY               ROLE     PIN  LAST USED   STALE  PATH
    0  sase     -                        primary  ●    2m ago      -      ~/projects/github/sase-org/sase
   11  sase     ace(run)-260713_094059   claim    -    now         -      …/sase-org/sase/sase_11
   42  sase     -                        claim    -    31d ago     ●      …/sase-org/sase/sase_42
 ┌ Details ────────────────────────────────────────────────────────────────────┐
 │ sase #11   claim · git-clone · generation 0                                 │
 │ Claim: ace(run)-260713_094059  pid 2362714 (alive)  since 260713_094059     │
 │ Created: 2026-06-01 10:15   Last used: just now   Stale: no (TTL 14d)       │
 │ Checkout: …/workspaces/sase-org/sase/sase_11                                │
 └─────────────────────────────────────────────────────────────────────────────┘
```

- Columns: `#` (workspace number), PROJECT, CLAIMED BY (agent/workflow name from the `RUNNING:` claim, `-` when free; a
  dead-PID claim is styled as a warning like `⚠ dead`), ROLE (primary/claim), PIN, LAST USED (relative), STALE (`●` when
  past `workspace.cleanup_ttl_days` and unclaimed/unpinned), PATH.
- Detail panel: full checkout path, claim details (agent, PID + liveness, CL name, claim timestamp), created/last-used
  absolute timestamps, materialization, generation, registry path, staleness verdict. Registry entries whose checkout is
  missing on disk are shown with a `missing` warning (pointing at `sase workspace repair`) rather than silently dropped.
- `p` opens the project picker (includes disabled projects, marked `○`, which is how a disabled project's workspaces
  become visible). `/` text-filters.

### Project picker (shared modal)

A small reusable modal (in the spirit of existing pickers like the agent-workspace tmux modal): a filterable list of
projects — "All projects" first, then enabled (●), then disabled (○), each with repo/workspace counts. `enter` applies
the filter, `escape` cancels. Used by both Repos and Workspaces sub-tabs and by the `r`/`w` cross-nav (which pre-applies
the selection without opening the picker).

### Beauty & consistency notes

- Sub-tab strip styling matches the Admin Center main tab strip (active tab colored with the Projects tab accent
  `#FFAF5F`, inactive dim) so the hierarchy reads instantly.
- Reuse the existing state-dot language: `●` enabled (green `#00D7AF`), `○` disabled (dim `#FFD700`); repo kinds get
  fixed accent colors (primary/teal, sidecar/purple, linked/blue) used consistently in rows, summary counts, and detail
  panels.
- Column layouts follow the existing fixed-width renderer pattern (`project_management_rendering.py`) so rows stay
  aligned in PNG snapshots; keep hint bars to one line per sub-tab.

## Phases

Each phase is one agent instance, lands independently (passing `just check` + tests), and builds on the previous ones.
Rust-touching phases must follow the Rust core boundary workflow (change `sase-core` wire/API

- bindings + tests via `sase workspace open -p sase-core`, then the Python callers). Phases adding CLI surface must
  first read `memory/cli_rules.md`; TUI phases must first read `memory/tui_perf.md`.

### Phase 1 — Core domain: true-project predicate + enabled/disabled lifecycle

**Goal:** the Rust core and Python facades speak the new model.

- `sase-core` (`crates/sase_core/src/project_spec.rs` + `sase_core_py` bindings):
  - Canonical lifecycle states become `enabled`/`disabled`; normalize legacy `active` → `enabled` and
    `inactive`/`archived`/`closed` → `disabled` on read; missing `PROJECT_STATE:` still defaults to `enabled`. `sibling`
    remains parseable as the internal backing marker (not a project state).
  - `apply_project_lifecycle_update` accepts and writes the new canonical values (legacy inputs accepted).
  - `ProjectRecordWire` gains derived `is_project: bool` and `vcs_kind: "git" | "gh" | none` (from `BARE_REPO_DIR:` /
    GitHub-provider naming); `list_project_records` supports a projects-only view.
  - Rust unit tests for normalization, the predicate, and junk-directory exclusion.
- `sase` (Python): mirror the wire/facade (`project_lifecycle_wire.py`, `project_lifecycle_facade.py`), sweep all
  `"active"`/`"inactive"` state-literal usages (`project_handler`, launch gating/`launchable` consumers, doctor checks,
  ace actions), and update `sase project` CLI verbs to enable/disable (legacy aliases retained). Keep the _existing_ TUI
  functional with minimally relabeled states (full restructure is Phase 4); update affected unit + visual snapshot
  fixtures.

**Acceptance:** telemetry-only project dirs and sibling backing records report `is_project=false`; states round-trip as
enabled/disabled; legacy spec files parse unchanged; `#gh` still refuses nonexistent GitHub repos and `#git` still
auto-creates valid names (add regression tests around `resolve_git_ref` / `gh_setup` creation paths).

### Phase 2 — Rename companion → sidecar (sase + sase-github)

**Goal:** zero remaining "companion" terminology.

- Rename identifiers, config values, hooks, templates, and docs across this repo (~118 files) and the `sase-github`
  linked repo (~14 files): `SddCompanion*`, `_companion_init.py`, `companion_for_kind`, `COMPANION_REPO_CLONES_SUBDIR`,
  hookspec `ws_preflight_sdd_companion` / `SddCompanionPreflight*`, `sdd_companion_suffix` option,
  `companion-research-README.md` template, `sase.schema.json`, `default_config.yml`, and all `docs/` prose.
- Persisted-value compatibility: `.sase/sdd-store.json` values `"companion_repos"` normalize to `"sidecar_repos"` on
  read and are written back in the new spelling; the pluggy hook rename ships as a coordinated change with `sase-github`
  (same release train — all runtimes/plugins are uniform, no runtime-specific carve-outs).
- Editorial call (flagging for approval at plan review): dated blog posts under `docs/blog/` get a short "(now called
  sidecar repos)" note rather than rewriting historical posts; everything else is renamed outright.

**Acceptance:** repo-wide grep for `companion` (case-insensitive) hits only the read-compat shim and the blog-post
notes; existing SDD stores load without migration steps; `just check` green in both repos.

### Phase 3 — Repo & workspace inventories (backend + CLI)

**Goal:** one frontend-neutral source of truth each for "all repos" and "all workspaces".

- New domain module (e.g. `src/sase/repo_inventory.py`):
  `RepoRecord(name, kind: primary|sidecar|linked, project, path, exists, auto_clone, description, source, env_name)`
  aggregated from project records (Rust)
  - SDD store records + resolved `linked_repos` config, deduping the sidecar/linked overlap (sidecar wins).
- New workspace inventory (e.g. in `src/sase/workspace_provider/`): `WorkspaceInventoryRecord` joining each project's
  `registry.json` entries with `RUNNING:` claims (agent name, PID + liveness, CL name, pinned), checkout existence, and
  TTL staleness — across many projects, with per-project errors isolated (one corrupt registry must not blank the
  inventory).
- CLI surfaces (read `memory/cli_rules.md` first): `sase repo list [-p PROJECT] [--json]`, and extend
  `sase workspace list` with an all-projects mode (e.g. `--all`), both sharing the inventory modules so the TUI and CLI
  can never disagree.
- Document the Rust-core migration intent on both modules (they are adapters; linked-repo config parsing and the
  workspace store are Python-owned today, which is why these don't start in Rust).

**Acceptance:** CLI output matches reality for fixture projects (unit tests over tmp dirs covering all three repo kinds,
claimed/free/stale/missing workspaces, disabled projects).

### Phase 4 — TUI: sub-tab shell + Projects sub-tab

**Goal:** the Projects tab gets its new skeleton and a finished Projects sub-tab.

- Replace the state-filter line in `ProjectsPane` with a clickable `PanelTabStrip` (Projects · Repos · Workspaces);
  `[`/`]` cycle sub-tabs; Repos/Workspaces show a clean placeholder empty-state until Phase 5.
- Rebuild the Projects sub-tab per the design above: `is_project`-filtered records, enabled+disabled rows, new columns
  (VCS, WS, REPOS via the Phase-3 inventories, loaded off-thread), enable/disable verbs on `a`/`d`, removal of `ctrl+x`
  and the sibling/inactive/all filters, updated detail panel and hint bar.
- Tests: rewrite `test_projects_pane.py` / `test_project_management_modal_*` interaction tests; refresh the four
  `config_center_projects_*` PNG snapshots and fixtures (`_patch_project_records`, deterministic record set) to the new
  model, adding a disabled-project row to the fixture spread.

**Acceptance:** visual + interaction suites green; junk/sibling rows are impossible to surface in the UI; j/k p95 stays
<16 ms on the tab (`SASE_TUI_PERF` spot-check).

### Phase 5 — TUI: Repos & Workspaces sub-tabs + project picker + cross-nav

**Goal:** the remaining two sub-tabs, fully wired.

- Implement the Repos and Workspaces sub-tab views over the Phase-3 inventories (async load, cached rows, loading state,
  coalesced refreshes), including detail panels, warnings (`missing` checkouts, dead-PID claims), and summary lines with
  kind/claim/stale counts.
- Implement the shared project-picker modal; wire `p` on both sub-tabs and the `r`/`w` cross-nav keymaps on the Projects
  sub-tab (pre-applied project filter + sub-tab switch); `escape` clears an active project filter before falling through
  to modal-close.
- Tests: interaction tests for filtering/picker/cross-nav; new PNG snapshot modules for both sub-tabs and the picker,
  following the existing `_ace_config_center_*` helper pattern.

**Acceptance:** the three sub-tabs form one coherent surface (identical anatomy, consistent colors/keys);
disabled-project workspaces reachable exactly via project filtering; visual suite green.

### Phase 6 — Glossary, docs, doctor, and polish

**Goal:** the new model is documented, self-diagnosing, and pleasant.

- Add the single glossary entry **"Projects, Repos, and Workspaces"** to `memory/glossary.md` matching the definitions
  above (provider shims regenerate via `sase init`). ⚠ Memory edits require explicit user permission in the implementing
  agent's launch prompt — plan approval alone does not grant it.
- Update `docs/` (ace.md, workspace.md, configuration.md, architecture.md, cli.md, rust_backend.md) for the new
  taxonomy, sub-tabs, keymaps, and CLI commands.
- `sase doctor`: add a check that lists junk directories under `~/.sase/projects/` (no ProjectSpec) with a cleanup hint,
  and one for registry entries whose checkout is missing.
- Polish (as capacity allows, in priority order): `t` on a Workspaces row opens tmux at the checkout (reusing the
  existing workspace tmux machinery); a stale-cleanup affordance that shells out to `sase workspace cleanup --stale`;
  project-picker fuzzy filtering.

**Acceptance:** docs/glossary match shipped behavior; full `just check` + visual suite green; final QA pass over all
three sub-tabs.

## Risks & open questions

- **State-literal sweep breadth (Phase 1):** many Python call sites compare against `"active"`; the wire constants make
  the sweep tractable, but this is the highest-regression-risk step — lean on the existing test suite plus new
  normalization tests.
- **Pluggy hook rename (Phase 2)** breaks third-party workspace plugins that implement `ws_preflight_sdd_companion`.
  Known implementers are in-house (`sase-github`); if outside plugins exist, a one-release deprecated-alias hookspec can
  be added instead of a hard cutover.
- **Blog-post rename policy (Phase 2):** recommendation is annotate-not-rewrite for dated posts — needs a yes/no at plan
  review.
- **Inventory cost:** repo/workspace inventories read `sase.yml`, `registry.json`, and ProjectSpecs across all projects
  (dozens of small files at current scale). Loads are off-thread with mtime-keyed caching, so the TUI never blocks; if
  project counts grow, the adapters are the natural seam to push into the Rust core.
- **`sase-core` release coupling (Phase 1):** the Python repo pins `sase_core_rs`; the phase must bump and
  publish/install the updated binding as part of its workflow.
