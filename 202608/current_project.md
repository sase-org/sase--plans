---
tier: epic
status: done
title: Current project, derived from the VCS xprompt MRU store
goal: 'SASE has one "current project" derived from the VCS xprompt MRU head, shown as a
  uniquely colored `+<project>` chip in the ACE top bar, and used as the default project
  filter on every TUI surface that can filter by project.

  '
phases:
  - id: resolve
    title: Current-project resolver over the VCS xprompt MRU
    depends_on: []
    size: medium
    description: "resolve: add `sase.current_project` with a `CurrentProject` record, an
      MRU-head-first resolver that maps project refs and Patch names to one enabled
      project, and a cheap stat-based change token for pollers.

      "
  - id: palette
    title: Per-project accent colors
    depends_on: []
    size: small
    description: "palette: add `sase.ace.tui.project_styles` with a curated accent
      palette and a hash-plus-probe assignment that gives every enabled project a
      distinct, deterministic color.

      "
  - id: config
    title: ace.current_project configuration
    depends_on: []
    size: small
    description: "config: add the `ace.current_project` config block (indicator,
      seed_filters, seed_agents_query), its JSON-schema entry, a typed reader on the
      app, and its configuration docs.

      "
  - id: indicator
    title: Top-bar +project indicator
    depends_on:
      - resolve
      - palette
      - config
    size: medium
    description: "indicator: add the `CurrentProjectIndicator` widget, mount it in the
      top-bar cluster right of the model indicator, and give it the poll/peek/off-thread
      resolve lifecycle, tooltip, and click action.

      "
  - id: artifacts
    title: Artifacts scope and Stitches startup filter
    depends_on:
      - resolve
      - config
    size: medium
    description: "artifacts: seed the shared Artifacts project scope from the current
      project and make it the single owner of the Stitches startup project filter,
      replacing the synchronous cwd-derived seed.

      "
  - id: panes
    title: Statistics, inventory, Glossary, and the + picker
    depends_on:
      - resolve
      - config
    size: medium
    description: "panes: seed the Statistics project filter, the Repos/Workspaces
      inventory filters, the Glossary project ring, and the `+` project-select cursor
      from the current project.

      "
  - id: agents
    title: Agents-tab project scoping
    depends_on:
      - resolve
      - config
    size: medium
    description: "agents: seed the Agents-tab search query with the current project
      behind the default-off `seed_agents_query` setting, and attribute a seeded scope
      visibly in the info panel.

      "
  - id: cli
    title: sase project current
    depends_on:
      - resolve
      - palette
    size: small
    description: "cli: add the `sase project current` subcommand with colored and
      `--json` output so the resolved current project is inspectable outside the TUI.

      "
  - id: polish
    title: Visual snapshot, help text, and full verification
    depends_on:
      - indicator
      - artifacts
      - panes
      - agents
      - cli
    size: small
    description:
      "polish: add the top-bar PNG snapshot, refresh help/command-palette wording for
      seeded project scopes, finish the ACE docs, and run the exhaustive verification
      lane."
proposed_by: bbugyi200.athena.062.f1
bead_id: sase-pw
create_time: 2026-09-09 19:50:12
---

- **PROMPT:**
  [prompts/202608/current_project.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/current_project.md)
- **BEAD:**
  [sase-pw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-pw/README.md)

# Plan: Current project, derived from the VCS xprompt MRU store

## Goal

SASE gains one **current project**: the project the user is actually working in right
now, derived from the head of the VCS xprompt MRU store. It is

- **shown** as a `+<project>` chip in the ACE top bar, right of the default-model
  indicator, in a color unique to that project;
- **used** as the default project filter on every TUI surface that can filter by
  project, whenever nothing more specific already selects one.

The MRU head can be a project ref (`#gh:sase`) or a Patch name (`#gh:my_patch`). When it
is a Patch, the current project is that Patch's owning project.

## Design

### One store, one writer, one reader

`73b55f0fb` ("back Ctrl+Space with the VCS xprompt MRU store") just collapsed two
competing "last selection" stores into one: `~/.sase/vcs_xprompt_mru.json`, written only
by a _successful_ launch, in `launch_query`, which every launch surface reaches. The
current project is a **pure derivation of that store**. This plan adds no second store
and no separate "set current project" write path — reintroducing one would rebuild
exactly the divergence that commit removed.

The direct consequence, which must be stated in the docs and the indicator tooltip:
**you change the current project by launching an agent**, not by picking one in a menu.
Clicking the indicator therefore opens the `+` launch picker (`start_custom_agent`),
which is the real way to move it.

### Resolution

Walk `load_launchable_vcs_xprompt_mru_pairs()` head-first. For each canonical prefix:

1. `extract_project_from_vcs_tag(prefix)` → `ref`; skip the entry when it is `None`.
2. Skip structural refs — `"/" in ref`, `ref.startswith("~")`, `ref == "home"`. These
   are external checkouts and the home workspace, not SASE projects.
3. Resolve aliases/display forms with `load_project_alias_map`, then
   `resolve_known_project_ref(...)` against `get_known_project_workspaces()`. A hit
   whose project record is `enabled` resolves with `origin="project"`.
4. Otherwise, match `ref` against `find_all_patches_cached()` by name. A hit resolves to
   `patch.project_name` with `origin="patch"`, again only when that project is enabled.
5. Otherwise continue to the next entry.

Walking rather than reading only the head is what makes this reliable: MRU pruning
already drops entries whose ref is gone, but it deliberately _keeps_ external
`#gh:owner/repo` and `~/path` refs, and a project can be disabled between a launch and a
read. A head that maps to no enabled project must fall through to the next real one
instead of blanking the current project.

Empty MRU, or no entry that resolves, yields `None`. Every consumer must behave exactly
as it does today when the result is `None`.

### Freshness without blocking

The MRU is written by a _different process_ (`sase run`), so the TUI has to notice
changes it did not make. `sase/memory/tui_perf.md` rules 1, 8 and 10 forbid doing the
real resolve on a timer tick: it reads JSON, project records, and the Patch cache.

Mirror `sase/llm_provider/launch_default_peek.py` exactly:

- `peek_current_project_change_token()` — `(mtime_ns, size)` of the MRU file plus
  `current_config_token()`, behind a `0.5 s` monotonic floor, `os.stat` only, degrading
  to a sentinel token on error so a broken read cannot cause a refresh storm.
- `resolve_current_project()` — the real read, **worker-thread only**, never called from
  a render path, message handler, or timer callback.

To stat the MRU file without reaching into `vcs_xprompt_mru`'s private `_mru_file`, add
a public `vcs_xprompt_mru_path()` accessor there and have `_mru_file()` delegate to it,
so the existing `_MRU_FILE` test hook keeps working for both.

### Seed, never enforce

This is the invariant that keeps the feature intuitive rather than annoying:

> The current project **seeds** a filter that has no value yet. It never overrides an
> explicit choice, and it never retroactively re-scopes a surface that is already open.

So the indicator moves live as the user launches agents, but a Statistics pane that is
already showing "All projects" stays that way. Each surface seeds once, on its first
open in the session. Concretely, the precedence for the Artifacts scope becomes:

1. an explicit `project:`/`+name` term in the startup query (`get_sole_project_filter`)
2. a scope the user picked this session (`_artifacts_scope_was_picked`)
3. **the current project**, when it is in the enabled set _(new)_
4. the existing sole-enabled-project auto-select
5. `None` — all projects

Rule 4 stays as the fallback for an empty MRU; rule 3 simply reaches the right answer
first, and correctly, when the user has more than one project.

### Color

`_provider_accent_for_kind` (`_artifact_tab_descriptors.py:272`) already establishes the
repo's pattern for "stable color for an identifier we cannot enumerate at authoring
time": sha256 the identifier, index a frozen palette. Reuse the shape, with one addition
the user's request requires — the colors must be _unique per project_, and a plain hash
collides at ~74% for eight projects over a nine-color palette.

`project_accent_map(project_keys)` therefore:

- iterates the keys in sorted canonical order (deterministic, independent of record
  order),
- takes each key's preferred slot from `sha256(key)` modulo the palette length,
- **forward linear-probes** to the next free slot on collision.

This yields distinct colors whenever `n <= len(PROJECT_ACCENTS)` while keeping a
project's color stable: it can only move if a project that sorts before it is added or
removed _and_ that project's arrival changes the probe outcome. With a palette of 18 and
three enabled projects on this machine, the ceiling is not a practical concern; above
it, assignment degrades to hash-only and repeats colors rather than failing.

The palette itself is a frozen literal tuple of 18 hex colors, derived at authoring time
by stepping hue evenly at a fixed saturation/lightness band chosen to match the
legibility of the existing `_PROVIDER_ACCENTS` on a dark terminal, and recorded as such
in a module comment. Freezing it keeps rendering deterministic and testable; the
_assignment_ is what is programmatic, which is what the user's "we have no way of
knowing what projects a user has enabled" constraint is actually about.

### The `+` sigil is not arbitrary

`sase/ace/query/project_scope.py` already treats a leading `+` as the shorthand spelling
of a project scope: `+sase` and `project:sase` are the same query term, and
`rewrite_project_scope` preserves whichever the user typed. Rendering the current
project as `+sase` therefore reads as "this is the project scope currently in effect" in
the exact vocabulary the query language already uses.

### Placement

`tests/ace/tui/test_top_bar_order.py` pins the top-bar order and documents that the
non-default override pill "sits just right of [the model indicator] so the two override
indicators read as a pair". Splitting that pair would contradict a deliberate, tested
decision.

Mount `#current-project-indicator` **immediately after `#provider-disables-indicator`**.
Both intervening pills render `Text("")` — zero width — whenever no override or provider
disable is active, which is the normal case, so the chip sits visually flush against the
model indicator exactly as asked while the tested override pairing survives.

### No feature flag; one config block

`sase/memory/sase_flags.md`: a flag is for behavior that reaches users before it is
ready to become unconditional, and "if users are meant to choose the value forever, it
was never a feature flag." Every phase here lands complete and ready — the indicator is
useful without the seeding, and each seeded surface is useful on its own — so there is
no partially-landed path to shield and no old branch to keep reachable.

What _is_ a forever choice is whether a user with many projects wants their views
pre-scoped at all. That is a config field:

```yaml
ace:
  current_project:
    indicator: true # show the +<project> chip in the top bar
    seed_filters: true # seed project filters that have no value yet
    seed_agents_query: false # also seed the Agents-tab search query
```

`seed_agents_query` defaults **off** on purpose, and this is the one place where I am
deliberately not taking the most aggressive reading of "use it everywhere". The Agents
tab is the primary at-a-glance view, and `_agent_search_query` is read by more than the
visible list — `_prospective_clan.py:140`, `_loading_finalize.py:111`, and
`_unread_jump_candidates.py:74` all consume it, so seeding it silently changes which
agents are considered for unread jumps and prospective clans, not just which rows are
drawn. Hiding rows there by default is the most surprising thing this feature could do.
The capability is fully built and tested in phase `agents`; it is one line of config to
turn on, and the plan says so in the docs. If you want it on by default, flip the
default in that phase — nothing else changes.

## Phases

### resolve: Current-project resolver over the VCS xprompt MRU

New module `src/sase/current_project.py`:

```python
@dataclass(frozen=True, slots=True)
class CurrentProject:
    project_key: str      # canonical directory key, e.g. gh_sase-org__sase
    display_name: str     # configured PROJECT_NAME, e.g. sase
    origin: str           # "project" | "patch"
    origin_ref: str       # the MRU ref that produced it
    workflow_type: str    # e.g. gh, git
```

- `resolve_current_project(*, projects_dir: Path | None = None) -> CurrentProject | None`
  implementing the walk in **Resolution** above. Build the alias map, known-project map,
  Patch names, and project records **once per call**, not per entry.
- `peek_current_project_change_token() -> tuple[object, ...]` as described in
  **Freshness without blocking**, copying the structure, floor constant, and error
  sentinel of `sase/llm_provider/launch_default_peek.py`.
- `display_name` comes from `effective_project_name` on the project record, never from
  the directory key — `CLAUDE.md` "Show Project Names, Never ProjectSpec Keys".
- Every consumer of the MRU already lives in Python and `vcs_xprompt_mru.py` is a Python
  store, so this derivation stays in Python next to it rather than crossing the
  `rust_core_backend_boundary`; porting it would mean porting the store.

Also add `vcs_xprompt_mru_path()` to `src/sase/history/vcs_xprompt_mru.py` and have the
existing `_mru_file()` delegate to it, preserving the `_MRU_FILE` hook.

**Tests** — `tests/test_current_project.py`, all under the existing SASE-home isolation
fixtures, never touching the real `~/.sase/vcs_xprompt_mru.json` (see `cce40d885`):

- head is a project ref → that project, `origin="project"`
- head is a Patch name → the Patch's project, `origin="patch"`
- head is `#gh:owner/repo`, `#git:~/path`, or `#git:home` → skipped, next entry wins
- head names a disabled project → skipped, next entry wins
- head is an alias / `PROJECT_NAME` spelling → resolves to the canonical key
- empty MRU, and an MRU where nothing resolves → `None`
- the token changes after `record_vcs_xprompt_usage` rewrites the file, and is stable
  across repeated calls with no write
- a resolve performs at most one project-records read and one Patch-cache read

### palette: Per-project accent colors

New module `src/sase/ace/tui/project_styles.py`:

- `PROJECT_ACCENTS: tuple[str, ...]` — 18 frozen hex colors, with a module comment
  recording the hue/saturation/lightness derivation and the dark-terminal legibility
  target.
- `project_accent_map(project_keys: Iterable[str]) -> Mapping[str, str]` — sorted-order
  hash-plus-forward-probe assignment, memoized on the sorted key tuple so repeated
  renders do not recompute.
- `project_accent(project_key: str, *, among: Iterable[str] | None = None) -> str` — the
  single-key convenience; without `among` it degrades to hash-only.

**Tests** — `tests/ace/tui/test_project_styles.py`:

- deterministic across calls and across process-level input ordering
- all-distinct for every `n <= len(PROJECT_ACCENTS)`, checked over a generated key
  corpus that is verified to contain at least one natural hash collision
- adding a project that sorts _after_ an existing one never changes the existing one's
  color when no probe was involved
- `n > len(PROJECT_ACCENTS)` degrades to repeats instead of raising
- `project_accent` without `among` matches `project_accent_map` for a single key

### config: ace.current_project configuration

- `src/sase/default_config.yml` — the `ace.current_project` block above, commented in
  the style of the surrounding `ace.artifacts` entries.
- `src/sase/config/sase.schema.json` — `ace` has `additionalProperties: false` (line
  937), so the new key **must** be added to its `properties` block or every config load
  fails validation.
- New `src/sase/ace/tui/current_project_settings.py` with a frozen settings dataclass
  and `parse_current_project_settings(ace_cfg)` that tolerates non-mapping and
  non-boolean values by falling back to defaults, matching `parse_agents_sync_config`.
- Wire it in `src/sase/ace/tui/actions/_state_init_late.py` as
  `self._current_project_settings`, and declare the attribute on `AceApp`
  (`src/sase/ace/tui/app.py`).
- `docs/configuration.md` — document all three fields, including the explicit note that
  `seed_agents_query` is off by default and why.

**Tests** — defaults, each field overridden, malformed values, and a schema-validation
test that the documented block is accepted.

### indicator: Top-bar +project indicator

New `src/sase/ace/tui/widgets/current_project_indicator.py`, modeled directly on
`llm_override_indicator.py` — same cached-state/peek/worker shape, same 5 s
`set_interval`, same `on_worker_state_changed` group filter:

- `_build_content()` returns `Text("")` when unresolved or when `settings.indicator` is
  false, so the chip occupies no width — matching `AliasOverridesIndicator` and
  `ProviderDisablesIndicator`.
- Otherwise
  `Text(" +", style=f"dim {accent}") + Text(display_name, style=f"bold {accent}") + Text(" ")`.
  The accent comes from `project_accent(key, among=enabled_keys)`; the enabled key set
  is resolved in the same worker call, never on the UI thread.
- Tooltip: the project name; `via Patch <name>` when `origin == "patch"`; the MRU ref it
  came from; and "Launch an agent on a project to make it current."
- `async def on_click` → `await self.app.run_action("start_custom_agent")`, i.e. the `+`
  picker — the surface that actually moves the MRU.
- Register in `widgets/__init__.py` and `widgets/__init__.pyi` next to the other
  indicators.

Layout and styling:

- `src/sase/ace/tui/_app_layout.py` — yield
  `CurrentProjectIndicator(id="current-project-indicator")` immediately after
  `ProviderDisablesIndicator`.
- `src/sase/ace/tui/styles.tcss` —
  `#current-project-indicator { width: auto; content-align: right middle; }`.
- `tests/ace/tui/test_top_bar_order.py` — insert the id into `EXPECTED_TOP_BAR_ORDER`
  and extend the comment with the reasoning from **Placement**. The two narrow-terminal
  bounds tests in that file must still pass with the chip rendered.

**Tests** — `tests/ace/tui/test_current_project_indicator.py`:

- resolved project renders `" +sase "` with the accent applied to both runs
- unresolved, and `indicator: false`, both render `""` and take zero width
- a Patch-origin current project still renders the _project_ name, with the Patch named
  only in the tooltip
- the tick calls `peek_current_project_change_token` but does **not** call
  `resolve_current_project` when the token is unchanged
- a token change schedules exactly one worker, and a second tick before it lands does
  not schedule another
- click dispatches `start_custom_agent`

### artifacts: Artifacts scope and Stitches startup filter

`src/sase/ace/tui/actions/artifacts.py`

- `_collect_artifacts_project_choices()` also resolves the current project (it already
  runs on a worker thread via `asyncio.to_thread`), returning it on
  `_ArtifactsProjectChoices`.
- In `_ensure_artifacts_project_choices._runner`, replace the
  `len(result.enabled_projects) == 1` special case with the full precedence ladder from
  **Seed, never enforce**, keeping the sole-enabled-project rule as the last resort.
  Seed with `picked=False` so a seeded scope is still distinguishable from a chosen one.

`src/sase/ace/tui/actions/_state_init_late.py`

- Stop deriving the Stitches startup project from
  `ensure_project_file_and_get_workspace_num`, and pass `current_project=None` into
  `merge_commits_startup_project`. That synchronous, cwd-based read is what forces
  project discovery into the startup path today (`tui_perf.md` rule 9), and it competes
  with the async seed for ownership of the same field.
- The async Artifacts seed becomes the single owner: `_set_artifacts_project_scope`
  already forwards to the Stitches pane when `commits.filters.project is None`
  (`artifacts.py:238`), which is now always true at seed time. An explicit `project:`
  term in `ace.artifacts.stitches.default_query` still wins, because
  `merge_commits_startup_project` keeps `values.project` ahead of the fallback.
- Keep `resolve_current_project`'s own fallback ordering honest: when the MRU yields
  nothing, fall back to the cwd-derived project so today's behavior is preserved for a
  first-run user with an empty MRU. Do that fallback in the _worker_, not at startup.
- Note the one accepted regression: the Stitches pane briefly shows all projects before
  the async inventory lands, where it previously scoped synchronously. The pane loads
  asynchronously anyway, so this is a sub-frame difference in practice; assert the
  settled state in tests rather than the transient one.

`src/sase/ace/tui/widgets/artifacts/commit_config.py` — update
`merge_commits_startup_project`'s docstring to describe the new precedence; the
signature does not need to change.

**Tests** — extend the existing Artifacts scope tests:

- the full precedence table: explicit query term > session pick > current project > sole
  enabled project > `None`
- a seeded scope reaches the Stitches, Beads, Files, and Plans panes
- `seed_filters: false` reproduces today's behavior exactly
- a mid-session MRU change does **not** re-scope an already-open Artifacts pane

### panes: Statistics, inventory, Glossary, and the + picker

**Statistics pane** (`src/sase/ace/tui/modals/statistics_pane.py`) — `_project_filter`
starts `None` (line 123) and `_project_filter_options` is only populated once a result
lands (line 665). Seed on the first successful result: if the user has not cycled yet
and the current project appears in `result.views.projects.projects`, adopt it and
reload. Add a `_project_filter_seeded` guard so cycling back to "All projects" is not
re-seeded on the next load — without it, `p` would be unable to escape the seed.

**Repos / Workspaces inventory** (`src/sase/ace/tui/modals/config_center_session.py`,
`projects_pane.py`) — `ProjectsSessionState.repos_project_filter` /
`workspaces_project_filter` default to `None` and persist for the ACE session. Seed both
once when the state is first constructed for a session, with a `project_filter_seeded`
field so an explicit `<escape>` clear-to-all sticks for the rest of the session.
`ProjectInventoryPaneBase.set_project_records` already drops a filter naming an unknown
project, so a stale seed self-heals.

**Glossary panel** (`src/sase/ace/tui/modals/glossary_panel.py`,
`glossary_panel_load.py`) — `_project_index` starts at `0`, i.e. alphabetically first.
Precedence becomes: `launch_workspace`'s project (the prompt the panel was opened from,
unchanged) > the current project's index in the ring > `0`. The ring is built from
enabled projects with a glossary, so a current project with no glossary simply is not
found and falls through.

**`+` project-select modal** (`src/sase/ace/tui/modals/project_select_modal.py`) — the
option list starts highlighted at row 0. When the current project appears in
`all_items`, highlight that row on mount instead. This is cursor placement only: no
filtering, no selection, no dismissal behavior changes. Set the highlight behind
`ProgrammaticSelectionGuard`-style handling per `tui_perf.md` rule 12 — a programmatic
`highlighted = X` emits an `OptionHighlighted` echo.

**Tests** — one focused test per surface for the seeded and unseeded cases, plus:
`p`/`P` can always cycle away from a seeded Statistics filter; `<escape>` clears a
seeded inventory filter and it does not come back; typing in the `+` picker's filter box
still resets the highlight to the first match.

### agents: Agents-tab project scoping

`src/sase/ace/tui/actions/_state_init_agents.py:109` initializes
`self._agent_search_query = ""`. When `seed_agents_query` is true and the current
project resolves, seed it with that project's `project:` term instead — built through
the query grammar rather than string concatenation, and using the **display name**,
matching the `project_query_name` semantics in `sase/ace/patch/models/patch.py:177`.

Because the resolve must not run synchronously at startup, seed from the same worker
that already loads agent data, then apply through the existing `_refilter_agents()` →
`_schedule_agents_async_refresh()` fast path (`tui_perf.md` rule 5) rather than a new
refresh route.

Make a seeded scope visible rather than mysterious: `AgentInfoPanel.update_search_query`
(`_display_detail_info.py:201`) already renders the active query, so mark a
not-yet-edited seeded query distinctly (e.g. a dim `seeded` tag) and drop the marker as
soon as the user edits the query through `_edit_agent_search_query`.

Audit the three non-list consumers before landing — `_prospective_clan.py:140`,
`_loading_finalize.py:111`, `_unread_jump_candidates.py:74` — and state in the phase's
commit message what a seeded scope does to each.

**Tests** — both states, as the config field is permanent:

- `seed_agents_query: false` (the default) leaves the query empty — this is the
  regression guard for every existing Agents-tab test
- `seed_agents_query: true` scopes the list, and the info panel shows the seeded marker
- editing the query clears the marker and the edited value survives a refresh
- unread-jump candidates and prospective clans behave as documented under a seeded scope

### cli: sase project current

`src/sase/main/parser_project.py` and `src/sase/main/project_handler.py`:

- Add a `current` subparser, placed alphabetically between `close` and `deactivate` in
  the choices list, per `sase/memory/cli_rules.md`.
- Default output: the project name in its own accent color, its canonical key, the
  origin (`project` or `patch`, naming the Patch when applicable), and the MRU ref it
  came from. Colored output over black-and-white, per the same memory.
- `--json` / `-j` for machine-readable output — every public long option needs a short
  alias.
- No project resolves → a clear, non-error message explaining that launching an agent on
  a project makes it current, and exit non-zero only if that is the established
  convention for the sibling `show` subcommand.
- Excellent `-h` output; `docs/cli.md`.

**Tests** — resolved project, patch-origin project, empty MRU, and `--json` shape.

### polish: Visual snapshot, help text, and full verification

- Add a top-bar PNG snapshot covering the chip alongside
  `tests/ace/tui/visual/test_ace_png_snapshots_alias_overrides_indicator.py`, following
  the pinned-color/fontconfig fixtures described in `CLAUDE.md`. Accept the golden with
  `--sase-update-visual-snapshots` only after eyeballing the rendered PNG.
- Sweep help and command-palette text for places that describe project filtering and now
  need to mention a seeded default: `help_modal/patches_artifact_bindings.py`,
  `statistics_help_modal.py`, `statistics_pane_legends.py`, `commands/_app_metadata.py`.
- `docs/ace.md` — a short "Current project" section: what it is, that launching moves
  it, what it seeds, and how to turn each part off.
- Run the exhaustive lane. `just check-full` routinely outruns a single agent turn, so
  run it **only** through `/sase_monitor` with a `--next` action, per `CLAUDE.md`.

## Verification

- `just install` first — these are ephemeral workspace directories.
- `just check` while iterating in every phase.
- `just check-full` via `/sase_monitor` before landing the combined tree. This epic
  touches the ACE TUI action surface, the top-bar layout, the config schema, and the CLI
  parser, so scoped test selection is not sufficient.
- `just test-visual` for the `polish` phase's snapshot.
- Manual smoke in `sase ace`:
  1. Launch `#gh:<projA>` and confirm the chip reads `+<projA>` in projA's color.
  2. Launch `#gh:<projB>` from a shell with `sase run` and confirm the chip flips to
     `+<projB>` within one poll interval — this is the cross-process path.
  3. Confirm an already-open Artifacts pane did **not** re-scope, and that opening
     Statistics for the first time afterwards seeds to projB.
  4. Launch on a Patch belonging to projA and confirm the chip reads `+<projA>` with the
     Patch named in the tooltip.
  5. Set `ace.current_project.seed_filters: false` and confirm every surface reverts to
     today's behavior while the chip remains.

## Out of scope

- **Any way to set the current project other than launching.** That would mean a second
  store and would rebuild the divergence `73b55f0fb` removed. The `+` picker reached
  from the chip is the affordance.
- **Recoloring project names elsewhere in the TUI** (Agents/Patches project group
  headers, patch rows, statistics tables) with the new accents. `project_styles` is
  public and built for it, but doing it here would balloon the epic and mix a large
  visual change into a behavioral one. Worth a follow-up.
- **Ordering the Agents tab's `STANDARD` (by-project) grouping so the current project's
  group sorts first.** Attractive, but grouping order is not filtering and this epic is
  already wide.
- **Passing the current project into the Admin Center xprompt browser's `project=`
  expansion context** (`xprompt_browser_pane.py:75`). That parameter selects an
  expansion context, not a filter; changing it changes what xprompts _render as_, which
  needs its own reasoning.
- **Any change to `<ctrl+p>`/`<ctrl+n>` cycling, MRU pruning rules, or what
  `record_vcs_xprompt_usage` records.** This epic only reads the store.
- **Persisting the current project across ACE restarts.** It is already durable, because
  the MRU file is.
